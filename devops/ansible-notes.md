# Ansible 核心知识笔记

> 适用对象：已按 `ansible-vmware-lab.md` 搭好实验环境，并完成控制节点、4 台被控节点、SSH 密钥和第一个 `ping.yml` 验证。  
> 定位：实验环境笔记管“搭环境”，本笔记管“学 Ansible 本身”。

---

## 目录

1. [Ansible 的设计理念](#1-ansible-的设计理念)
2. [Inventory 进阶](#2-inventory-进阶)
3. [Ad-hoc 命令与常用模块速查](#3-ad-hoc-命令与常用模块速查)
4. [Playbook 语法详解](#4-playbook-语法详解)
5. [变量与 Facts](#5-变量与-facts)
6. [Jinja2 模板](#6-jinja2-模板)
7. [Handlers 与通知机制](#7-handlers-与通知机制)
8. [Roles](#8-roles)
9. [错误处理与流程控制](#9-错误处理与流程控制)
10. [Vault 进阶](#10-vault-进阶)
11. [调试与排错](#11-调试与排错)
12. [最佳实践](#12-最佳实践)
13. [完整实战：用 Role 批量部署 Nginx](#13-完整实战用-role-批量部署-nginx)
14. [学习路线与实践任务](#14-学习路线与实践任务)
15. [总结](#15-总结)

---

## 1. Ansible 的设计理念

### 1.1 无 Agent

Ansible 不要求在被控节点常驻专用代理进程。控制节点通常通过 SSH 登录 Linux 被控节点，把模块代码和参数临时传过去执行，再收集结果。

基本链路如下：

```text
控制节点读取 inventory 和 playbook
        ↓
通过 SSH 连接被控节点
        ↓
传输并执行 Ansible 模块
        ↓
返回 changed、failed、stdout 等结果
        ↓
清理临时文件并继续下一个任务
```

优点：

- 被控节点不用维护额外代理及其版本。
- 能直接利用现有 SSH、用户、密钥和 sudo 安全机制。
- 控制节点故障不会让被控节点上的业务随之停止。
- 学习门槛较低，适合从少量主机逐步扩展。

需要注意：

- “无 Agent”不等于“无依赖”，大多数模块仍依赖被控节点上的 Python。
- SSH 并发量、网络延迟和控制节点性能会影响大规模执行速度。
- Windows 被控节点通常使用 WinRM 等连接方式，不走本笔记的 Linux SSH 路径。

实验环境的 SSH 密钥、主机指纹、sudo 和初始 inventory 已在 [`ansible-vmware-lab.md`](./ansible-vmware-lab.md) 中完成，这里不重复搭建。

### 1.2 推送模型

Ansible 的常见工作方式是由控制节点主动推送配置：

```bash
ansible-playbook site.yml
```

控制节点决定何时执行、对哪些主机执行、执行哪些任务。它与被控节点定时拉取配置的模型不同。

推送模型适合：

- 人工发起的配置变更。
- CI/CD 流水线触发的部署。
- 按批次滚动更新服务器。
- 一次性巡检或故障处置。

如果需要定时执行，可由计划任务、流水线或自动化平台调度 `ansible-playbook`，而不是在每台被控节点自行维护脚本。

### 1.3 声明式与过程式

声明式任务描述“目标状态”：

```yaml
- name: 确保 Nginx 已安装
  become: true
  ansible.builtin.package:
    name: nginx
    state: present
```

过程式任务描述“执行某条命令”：

```yaml
- name: 直接运行安装命令
  become: true
  ansible.builtin.command:
    cmd: pacman -S --noconfirm nginx
```

前者让模块检查当前状态，只在需要时改变系统；后者每次都执行命令。Ansible 并非禁止过程式操作，但应优先使用能表达资源状态的专用模块。

### 1.4 什么是幂等性

幂等性指同一个自动化任务执行一次或多次，最终系统状态保持一致；系统已处于目标状态时，再次执行不应产生无意义变更。

例如：

- 软件已安装时，`package` 的 `state: present` 不会重复安装。
- 目录权限正确时，`file` 不会重复修改。
- 配置内容一致时，`template` 返回 `ok`，不会触发 handler。
- 用户已存在且属性相同时，`user` 不会重复创建。

验证幂等性的常用方法是连续运行两次 playbook。第二次应主要显示 `ok`，`changed` 应为零或只有确实具有时效性的任务。

### 1.5 为什么 shell 容易破坏幂等性

`shell` 的职责只是让 shell 执行字符串，它通常不知道命令执行前后的资源状态：

```yaml
- name: 追加配置行的错误示例
  become: true
  ansible.builtin.shell:
    cmd: echo 'vm.swappiness=10' >> /etc/sysctl.d/99-lab.conf
```

每运行一次都会追加一行，产生重复配置。即使命令本身没有真正修改系统，`shell` 默认也常被 Ansible 判断为 `changed`。

应优先改用专用模块：

```yaml
- name: 确保参数配置唯一存在
  become: true
  ansible.builtin.lineinfile:
    path: /etc/sysctl.d/99-lab.conf
    regexp: '^vm\.swappiness='
    line: 'vm.swappiness=10'
    create: true
    mode: '0644'
```

### 1.6 用 creates 和 changed_when 补救

确实需要命令时，可以给出幂等性判断条件。

`creates` 表示指定路径存在时跳过命令：

```yaml
- name: 仅在标记文件不存在时初始化数据
  become: true
  ansible.builtin.command:
    cmd: /usr/local/bin/init-lab-data
    creates: /var/lib/lab/.initialized
```

`shell` 也支持 `creates`：

```yaml
- name: 仅首次解压流水线产物
  become: true
  ansible.builtin.shell:
    cmd: tar -xzf /tmp/site.tar.gz -C /srv/site
    creates: /srv/site/index.html
```

查询型命令不应报告变更，可使用 `changed_when: false`：

```yaml
- name: 查询当前内核版本
  ansible.builtin.command:
    cmd: uname -r
  register: kernel_result
  changed_when: false
```

根据输出判断是否发生变更：

```yaml
- name: 运行只在更新时输出标记的脚本
  ansible.builtin.command:
    cmd: /usr/local/bin/sync-config
  register: sync_result
  changed_when: "'UPDATED' in sync_result.stdout"
```

不要为了让执行结果“更绿”而随意写 `changed_when: false`。它应反映真实状态，否则 handler、审计结果和幂等性判断都会失真。

---

## 2. Inventory 进阶

### 2.1 分组与嵌套组

实验环境已有 `managed` 组。进一步可以按业务角色划分子组，再用父组统一管理：

```ini
[web]
ansible-node1 ansible_host=192.168.88.21
ansible-node2 ansible_host=192.168.88.22

[app]
ansible-node3 ansible_host=192.168.88.23

[db]
ansible-node4 ansible_host=192.168.88.24

[managed:children]
web
app
db

[managed:vars]
ansible_user=ansible
```

这样可以选择不同范围：

```bash
ansible web --list-hosts
ansible 'web:app' --list-hosts
ansible 'managed:!db' --list-hosts
ansible 'web:&managed' --list-hosts
```

常见主机模式：

| 模式 | 含义 |
|---|---|
| `managed` | `managed` 组全部主机 |
| `web:app` | `web` 或 `app` 组 |
| `managed:!db` | 属于 `managed` 但不属于 `db` |
| `web:&managed` | 同时属于 `web` 和 `managed` |
| `ansible-node1` | 单台主机 |

### 2.2 group_vars 与 host_vars

变量不应全部挤在 inventory 行中。推荐目录结构：

```text
ansible-project/
├── ansible.cfg
├── inventory.ini
├── group_vars/
│   ├── all.yml
│   ├── managed.yml
│   └── web/
│       ├── vars.yml
│       └── vault.yml
├── host_vars/
│   ├── ansible-node1.yml
│   └── ansible-node2.yml
└── site.yml
```

示例 `group_vars/all.yml`：

```yaml
---
ntp_timezone: Asia/Shanghai
common_packages:
  - curl
  - vim
```

示例 `group_vars/web/vars.yml`：

```yaml
---
nginx_worker_connections: 1024
nginx_listen_port: 80
```

示例 `host_vars/ansible-node1.yml`：

```yaml
---
nginx_listen_port: 8080
node_role_description: 主 Web 节点
```

主机变量通常覆盖组变量，因此 `ansible-node1` 会使用端口 `8080`，其他 `web` 主机使用 `80`。

### 2.3 变量优先级要点

Ansible 变量优先级层次很多，学习阶段先记住以下趋势：

1. 角色默认变量 `roles/角色名/defaults/main.yml` 优先级较低，适合提供可覆盖默认值。
2. inventory 的组变量用于一组主机的共同配置。
3. inventory 的主机变量通常覆盖组变量。
4. play 的 `vars`、任务变量和 `set_fact` 优先级更高。
5. 命令行 `-e` 额外变量优先级很高，应谨慎使用。

例如：

```bash
ansible-playbook site.yml -e nginx_listen_port=8888
```

这会强制覆盖许多其他位置的同名变量。不要把 `-e` 当作日常修补手段，否则变量来源难以追踪。

变量优先级复杂时，可用 `debug` 输出最终值，并通过搜索变量名确认定义位置。变量名应带角色前缀，例如 `nginx_role_port`，减少碰撞。

### 2.4 动态 Inventory

静态 inventory 适合固定的 4 台实验机。云环境中的实例会频繁创建和销毁，此时可使用动态 inventory 插件从云平台、虚拟化平台或资产系统实时获取主机和分组。

动态 inventory 的核心思想：

- 主机清单由数据源生成，而不是人工维护固定文件。
- 可按标签、区域、项目或实例属性自动分组。
- 认证信息应来自环境变量或密钥管理系统。
- 必须缓存和限制查询范围，避免慢查询或误操作全部资源。

查看解析结果：

```bash
ansible-inventory -i inventory.ini --graph
ansible-inventory -i inventory.ini --list
ansible-inventory -i inventory.ini --host ansible-node1
```

学习动态 inventory 时先理解插件概念即可，不必为固定实验节点引入额外复杂度。

---

## 3. Ad-hoc 命令与常用模块速查

Ad-hoc 命令适合一次性检查或小范围操作；需要重复执行、评审和版本控制时应写成 playbook。

基本格式：

```bash
ansible <主机模式> -m <模块完整名称> -a '<模块参数>'
```

### 3.1 常用模块表

| 模块 | 主要用途 | 典型示例 |
|---|---|---|
| `ansible.builtin.ping` | 验证连接与 Python | `ansible managed -m ansible.builtin.ping` |
| `ansible.builtin.command` | 直接执行程序，不经过 shell | `ansible managed -m ansible.builtin.command -a 'uptime'` |
| `ansible.builtin.shell` | 使用 shell 管道、重定向等语法 | `ansible managed -m ansible.builtin.shell -a 'ss -lnt \| wc -l'` |
| `ansible.builtin.copy` | 复制文件或直接写入内容 | `ansible managed -b -m ansible.builtin.copy -a 'src=motd dest=/etc/motd mode=0644'` |
| `ansible.builtin.file` | 管理文件、目录、链接和权限 | `ansible managed -b -m ansible.builtin.file -a 'path=/srv/lab state=directory mode=0755'` |
| `ansible.builtin.package` | 跨发行版管理软件包 | `ansible managed -b -m ansible.builtin.package -a 'name=curl state=present'` |
| `community.general.pacman` | 管理 Arch Linux 软件包 | `ansible managed -b -m community.general.pacman -a 'name=nginx state=present'` |
| `ansible.builtin.service` | 通用服务管理 | `ansible managed -b -m ansible.builtin.service -a 'name=nginx state=started enabled=true'` |
| `ansible.builtin.systemd_service` | 管理 systemd 服务及 daemon reload | `ansible managed -b -m ansible.builtin.systemd_service -a 'name=sshd state=restarted'` |
| `ansible.builtin.user` | 管理用户账户 | `ansible managed -b -m ansible.builtin.user -a 'name=deploy state=present create_home=true'` |
| `ansible.builtin.lineinfile` | 确保文件中某一行符合要求 | `ansible managed -b -m ansible.builtin.lineinfile -a 'path=/etc/example.conf regexp=^Mode= line=Mode=safe'` |
| `ansible.builtin.template` | 用 Jinja2 渲染模板 | `ansible managed -b -m ansible.builtin.template -a 'src=app.conf.j2 dest=/etc/app.conf'` |
| `ansible.builtin.fetch` | 从被控节点取回文件 | `ansible managed -m ansible.builtin.fetch -a 'src=/etc/hostname dest=./collected/ flat=false'` |
| `ansible.builtin.setup` | 收集 facts | `ansible managed -m ansible.builtin.setup -a 'filter=ansible_default_ipv4'` |

`community.general.pacman` 属于 `community.general` collection。如果本机没有该 collection，可优先使用内置的 `ansible.builtin.package`，或安装后再使用：

```bash
ansible-galaxy collection install community.general
```

### 3.2 command 与 shell 的区别

`command` 不通过 shell 解释，因此不能直接使用：

- 管道 `|`
- 重定向 `>`、`>>`
- 通配符展开 `*`
- `&&`、`||`
- shell 内建变量和命令替换

能用 `command` 时优先用它，因为参数边界更清晰，也减少 shell 注入和转义问题。

```yaml
- name: 使用 command 查询服务状态
  ansible.builtin.command:
    cmd: systemctl is-active sshd
  register: sshd_state
  changed_when: false
```

确实需要管道时使用 `shell`，并固定 shell 或谨慎引用变量：

```yaml
- name: 统计正在监听的端口
  ansible.builtin.shell:
    cmd: set -o pipefail && ss -lnt | tail -n +2 | wc -l
    executable: /bin/bash
  register: listening_count
  changed_when: false
```

不要把不可信输入直接拼接到 `shell` 命令中。能用 `find`、`file`、`lineinfile`、`uri` 等专用模块时，不要用长 shell 管道替代。

### 3.3 常见 ad-hoc 操作

```bash
# 查看所有节点运行时间
ansible managed -m ansible.builtin.command -a 'uptime'

# 提权安装 curl
ansible managed -b --ask-become-pass -m ansible.builtin.package -a 'name=curl state=present'

# 启动并启用 Nginx
ansible managed -b --ask-become-pass -m ansible.builtin.service -a 'name=nginx state=started enabled=true'

# 只操作一台节点
ansible ansible-node1 -m ansible.builtin.setup -a 'filter=ansible_default_ipv4'

# 限制并发数为 2
ansible managed -f 2 -m ansible.builtin.ping
```

执行修改前先用 `--list-hosts` 核对目标范围，尤其是使用组合主机模式时。

---

## 4. Playbook 语法详解

### 4.1 Play 的基本结构

一个 play 把“哪些主机”“以什么权限”“执行哪些任务”连接起来：

```yaml
---
- name: 配置所有实验节点
  hosts: managed
  become: true
  gather_facts: true

  vars:
    lab_directory: /srv/ansible-lab

  pre_tasks:
    - name: 显示当前目标主机
      ansible.builtin.debug:
        msg: "正在处理 {{ inventory_hostname }}"

  tasks:
    - name: 创建实验目录
      ansible.builtin.file:
        path: "{{ lab_directory }}"
        state: directory
        owner: root
        group: root
        mode: '0755'

  post_tasks:
    - name: 确认目录状态
      ansible.builtin.stat:
        path: "{{ lab_directory }}"
      register: lab_directory_stat
```

常用 play 级关键字：

| 关键字 | 作用 |
|---|---|
| `name` | 描述 play 的目标 |
| `hosts` | 选择 inventory 主机或组 |
| `become` | 是否提权 |
| `gather_facts` | 是否在任务前收集 facts |
| `vars` | 定义 play 范围变量 |
| `serial` | 分批处理主机 |
| `roles` | 应用角色 |
| `handlers` | 定义通知触发的处理任务 |

### 4.2 任务常用关键字

`name`：必须写清动作与目标，便于日志、排错和 `--start-at-task` 使用。

`become`：只在需要权限的 play 或任务上启用：

```yaml
- name: 读取普通用户信息
  ansible.builtin.command:
    cmd: id
  changed_when: false

- name: 创建系统目录
  become: true
  ansible.builtin.file:
    path: /opt/lab
    state: directory
    mode: '0755'
```

`when`：按变量或 facts 条件执行，不需要 `{{ }}`：

```yaml
- name: 仅在 Arch Linux 上安装 Nginx
  become: true
  ansible.builtin.package:
    name: nginx
    state: present
  when: ansible_facts.distribution == 'Archlinux'
```

`loop`：重复执行任务，当前项使用 `item`：

```yaml
- name: 安装常用工具
  become: true
  ansible.builtin.package:
    name: "{{ item }}"
    state: present
  loop:
    - curl
    - vim
    - git
```

也可以一次把列表交给支持列表参数的模块，通常更高效：

```yaml
- name: 一次安装常用工具列表
  become: true
  ansible.builtin.package:
    name:
      - curl
      - vim
      - git
    state: present
```

`register`：保存任务结果：

```yaml
- name: 查询 Nginx 是否运行
  ansible.builtin.command:
    cmd: systemctl is-active nginx
  register: nginx_status
  changed_when: false
  failed_when: nginx_status.rc not in [0, 3]

- name: 输出查询结果
  ansible.builtin.debug:
    var: nginx_status.stdout
```

注册结果常见字段包括 `rc`、`stdout`、`stderr`、`changed`、`failed`、`results`。

`failed_when`：自定义失败条件：

```yaml
- name: 检查磁盘使用率
  ansible.builtin.shell:
    cmd: df -P / | tail -1 | awk '{print $5}' | tr -d '%'
  register: root_usage
  changed_when: false
  failed_when: root_usage.stdout | int > 90
```

`changed_when`：自定义变更条件，适用于查询命令或非标准脚本。

`ignore_errors`：忽略当前任务失败并继续，但会隐藏流程问题，应少用：

```yaml
- name: 尝试读取可选文件
  ansible.builtin.command:
    cmd: test -f /etc/optional.conf
  register: optional_file
  changed_when: false
  failed_when: false
```

对于预期失败，更推荐精确设置 `failed_when`，而不是宽泛使用 `ignore_errors: true`。

### 4.3 Tags

Tags 用于只执行或跳过一部分任务：

```yaml
- name: 安装 Nginx
  become: true
  ansible.builtin.package:
    name: nginx
    state: present
  tags:
    - nginx
    - packages

- name: 分发 Nginx 配置
  become: true
  ansible.builtin.template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
    mode: '0644'
  tags:
    - nginx
    - config
```

常用命令：

```bash
ansible-playbook site.yml --list-tags
ansible-playbook site.yml --tags nginx
ansible-playbook site.yml --tags config
ansible-playbook site.yml --skip-tags packages
```

Tags 是执行过滤器，不是依赖管理器。只运行 `config` 时，安装软件等前置任务可能被跳过，因此任务仍要设计成可理解、可组合的结构。

---

## 5. 变量与 Facts

### 5.1 变量定义位置

常见变量来源：

- `group_vars/all.yml`：所有主机共享。
- `group_vars/组名.yml`：某组共享。
- `host_vars/主机名.yml`：单台主机专属。
- play 的 `vars`、`vars_files`。
- role 的 `defaults/main.yml` 和 `vars/main.yml`。
- inventory 内联变量。
- `set_fact` 运行时变量。
- 命令行 `-e` 额外变量。

变量示例：

```yaml
---
nginx_listen_port: 80
nginx_server_name: _
nginx_document_root: /usr/share/nginx/html
nginx_features:
  access_log: true
  gzip: true
```

引用变量：

```yaml
- name: 显示服务参数
  ansible.builtin.debug:
    msg: >-
      主机 {{ inventory_hostname }} 将监听 {{ nginx_listen_port }}，
      根目录为 {{ nginx_document_root }}
```

### 5.2 Facts 与 setup

开启 `gather_facts: true` 时，Ansible 在 play 开始阶段运行 facts 采集。常见 facts：

```yaml
- name: 显示关键 Facts
  ansible.builtin.debug:
    msg:
      - "主机名：{{ ansible_facts.hostname }}"
      - "系统：{{ ansible_facts.distribution }}"
      - "架构：{{ ansible_facts.architecture }}"
      - "默认 IPv4：{{ ansible_facts.default_ipv4.address }}"
      - "处理器核数：{{ ansible_facts.processor_vcpus }}"
```

按范围查看：

```bash
ansible ansible-node1 -m ansible.builtin.setup
ansible ansible-node1 -m ansible.builtin.setup -a 'filter=ansible_distribution*'
ansible managed -m ansible.builtin.setup -a 'filter=ansible_default_ipv4'
```

如果 play 完全不需要 facts，可写 `gather_facts: false` 减少启动时间。不要在关闭 facts 后继续引用未定义的 `ansible_facts`。

### 5.3 Magic Variables

Magic variables 由 Ansible 提供，不需要自行定义。

`inventory_hostname` 是 inventory 中的主机名；`groups` 保存组到主机列表的映射；`hostvars` 可读取其他主机的变量。

```yaml
- name: 显示 managed 组成员
  ansible.builtin.debug:
    var: groups['managed']

- name: 显示第一台 managed 主机的地址
  ansible.builtin.debug:
    msg: >-
      {{ hostvars[groups['managed'][0]].ansible_host }}
  run_once: true
```

其他常用 magic variables：

| 变量 | 含义 |
|---|---|
| `inventory_hostname` | inventory 中当前主机名 |
| `inventory_hostname_short` | 主机名短形式 |
| `group_names` | 当前主机所属组列表 |
| `groups` | 全部组及其成员 |
| `hostvars` | 全部主机变量映射 |
| `play_hosts` | 当前 play 中仍活跃的主机 |
| `ansible_play_hosts_all` | 当前 play 最初选择的全部主机 |

访问其他主机变量前，要确保该变量确实存在；必要时用 `default` 过滤器提供默认值。

### 5.4 set_fact 与 debug

`set_fact` 在执行期间为当前主机生成变量：

```yaml
- name: 根据内存计算应用工作进程数
  ansible.builtin.set_fact:
    app_worker_count: "{{ [1, (ansible_facts.memtotal_mb / 1024) | int] | max }}"

- name: 显示计算结果
  ansible.builtin.debug:
    var: app_worker_count
```

`set_fact` 适合保存计算结果，不应用来替代清晰的静态变量文件。

常用调试方式：

```yaml
- name: 输出单个变量及其类型信息
  ansible.builtin.debug:
    msg:
      - "端口值：{{ nginx_listen_port }}"
      - "变量类型：{{ nginx_listen_port | type_debug }}"

- name: 仅在详细模式下输出全部主机变量
  ansible.builtin.debug:
    var: hostvars[inventory_hostname]
    verbosity: 2
```

第二个任务只有使用 `-vv` 或更高详细级别时才显示，可避免普通执行日志过长。

---

## 6. Jinja2 模板

### 6.1 template 模块工作方式

模板文件通常放在 `templates/`，使用 `{{ 变量 }}` 输出值，使用 `{% ... %}` 表示条件、循环等控制结构。`ansible.builtin.template` 在控制节点渲染后把结果发送到被控节点。

以下示例生成 Nginx 配置。

变量文件：

```yaml
---
nginx_user: http
nginx_worker_processes: auto
nginx_worker_connections: 1024
nginx_listen_port: 80
nginx_server_name: _
nginx_document_root: /usr/share/nginx/html
nginx_enable_access_log: true
nginx_extra_headers:
  - name: X-Content-Type-Options
    value: nosniff
  - name: X-Frame-Options
    value: SAMEORIGIN
```

模板 `templates/nginx.conf.j2`：

```jinja2
# 此文件由 Ansible 管理，请勿手工修改。
user {{ nginx_user }};
worker_processes {{ nginx_worker_processes }};
error_log /var/log/nginx/error.log warn;
pid /run/nginx.pid;

events {
    worker_connections {{ nginx_worker_connections }};
}

http {
    include mime.types;
    default_type application/octet-stream;
    sendfile on;

    server {
        listen {{ nginx_listen_port }};
        server_name {{ nginx_server_name }};
        root {{ nginx_document_root }};
        index index.html;

{% if nginx_enable_access_log %}
        access_log /var/log/nginx/access.log;
{% else %}
        access_log off;
{% endif %}

{% for header in nginx_extra_headers %}
        add_header {{ header.name }} "{{ header.value }}" always;
{% endfor %}

        location / {
            try_files $uri $uri/ =404;
        }
    }
}
```

Playbook 任务：

```yaml
- name: 渲染 Nginx 主配置
  become: true
  ansible.builtin.template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
    owner: root
    group: root
    mode: '0644'
    backup: true
    validate: /usr/bin/nginx -t -c %s
  notify: 重载 Nginx
```

`validate` 会先检查临时文件，只有检查成功才替换目标配置。`%s` 由模块替换成临时文件路径，不经过 shell。

### 6.2 常用 Filters

| Filter | 用途 | 示例 |
|---|---|---|
| `default` | 未定义时提供默认值 | `{{ app_port | default(8080) }}` |
| `int` | 转换为整数 | `{{ app_port | int }}` |
| `bool` | 转换为布尔值 | `{{ feature_enabled | bool }}` |
| `lower` / `upper` | 大小写转换 | `{{ environment_name | lower }}` |
| `join` | 连接列表 | `{{ groups['web'] | join(',') }}` |
| `length` | 取长度 | `{{ groups['managed'] | length }}` |
| `sort` | 排序 | `{{ package_list | sort }}` |
| `unique` | 去重 | `{{ package_list | unique }}` |
| `to_nice_yaml` | 便于查看 YAML 数据 | `{{ complex_value | to_nice_yaml }}` |
| `mandatory` | 未定义时立即报错 | `{{ required_token | mandatory }}` |

处理可能不存在的嵌套值时应谨慎：

```jinja2
监听地址：{{ service_bind_address | default('127.0.0.1') }}
节点数量：{{ groups['managed'] | default([]) | length }}
```

### 6.3 模板中的循环与条件

```jinja2
{% for host in groups['managed'] | sort %}
server {{ hostvars[host].ansible_host }}:8080;
{% endfor %}

{% if deployment_environment == 'production' %}
日志级别：warn
{% elif deployment_environment == 'staging' %}
日志级别：info
{% else %}
日志级别：debug
{% endif %}
```

模板应以生成配置为主。复杂计算尽量在变量、`set_fact` 或自定义过滤器中完成，避免模板变成难以测试的程序。

---

## 7. Handlers 与通知机制

### 7.1 notify 与 handler

普通任务通过 `notify` 发出通知；只有任务状态为 `changed` 时才会触发 handler。

```yaml
---
- name: 配置 Nginx
  hosts: managed
  become: true

  tasks:
    - name: 分发 Nginx 配置
      ansible.builtin.template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
        mode: '0644'
        validate: /usr/bin/nginx -t -c %s
      notify: 重载 Nginx

  handlers:
    - name: 重载 Nginx
      ansible.builtin.service:
        name: nginx
        state: reloaded
```

如果模板内容没有变化，任务返回 `ok`，handler 不会执行。这是配置管理中“变更后才重载”的标准模式。

### 7.2 Handler 去重

多个任务通知同一个 handler 时，handler 默认在 play 的任务阶段结束后只执行一次：

```yaml
- name: 更新主配置
  ansible.builtin.template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
    mode: '0644'
  notify: 重载 Nginx

- name: 更新站点配置
  ansible.builtin.template:
    src: lab.conf.j2
    dest: /etc/nginx/servers/lab.conf
    mode: '0644'
  notify: 重载 Nginx
```

即使两个模板都变化，也只重载一次，避免服务被连续重载。

可以用 `listen` 让多个 handler 监听同一主题：

```yaml
handlers:
  - name: 检查 Nginx 配置
    ansible.builtin.command:
      cmd: nginx -t
    changed_when: false
    listen: Nginx 配置已变化

  - name: 重载 Nginx 服务
    ansible.builtin.service:
      name: nginx
      state: reloaded
    listen: Nginx 配置已变化
```

任务只需：

```yaml
notify: Nginx 配置已变化
```

### 7.3 meta: flush_handlers

默认 handler 在当前 play 的任务结束后执行。如果后续任务必须立即依赖新配置，可提前刷新：

```yaml
- name: 分发 Nginx 配置
  ansible.builtin.template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
    mode: '0644'
  notify: 重载 Nginx

- name: 立即执行已收到通知的 handlers
  ansible.builtin.meta: flush_handlers

- name: 验证新监听端口
  ansible.builtin.uri:
    url: "http://127.0.0.1:{{ nginx_listen_port }}/"
    status_code: 200
```

不要到处刷新 handler。通常让它在阶段末尾统一执行更高效，只有存在明确依赖时才提前刷新。

### 7.4 配置变更的标准模式

```yaml
- name: 安装 Nginx
  become: true
  ansible.builtin.package:
    name: nginx
    state: present

- name: 校验并分发 Nginx 配置
  become: true
  ansible.builtin.template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
    owner: root
    group: root
    mode: '0644'
    backup: true
    validate: /usr/bin/nginx -t -c %s
  notify: 重载 Nginx

- name: 确保 Nginx 已启动并开机自启
  become: true
  ansible.builtin.service:
    name: nginx
    state: started
    enabled: true
```

该模式依次完成安装、配置校验与分发、服务状态保证；配置改变时再通知重载。

---

## 8. Roles

### 8.1 为什么使用 Role

平铺 playbook 变长后，任务、变量、模板、文件和 handlers 混在一起，难以复用。Role 用约定目录把同一功能封装起来。

创建角色：

```bash
ansible-galaxy role init roles/nginx
```

典型结构：

```text
roles/nginx/
├── README.md
├── defaults/
│   └── main.yml
├── files/
├── handlers/
│   └── main.yml
├── meta/
│   └── main.yml
├── tasks/
│   └── main.yml
├── templates/
├── tests/
│   ├── inventory
│   └── test.yml
└── vars/
    └── main.yml
```

各目录职责：

| 路径 | 作用 |
|---|---|
| `defaults/main.yml` | 可由调用者覆盖的默认变量，优先级较低 |
| `vars/main.yml` | 角色内部强约束变量，优先级较高，应少放可配置项 |
| `tasks/main.yml` | 角色任务入口，可继续 `include_tasks` 拆分 |
| `handlers/main.yml` | 角色 handlers |
| `templates/` | Jinja2 模板，调用时不用写完整目录 |
| `files/` | 原样复制的静态文件 |
| `meta/main.yml` | 角色元数据和依赖关系 |
| `tests/` | 简单测试 inventory 和 playbook |

### 8.2 平铺 Playbook 重构前后

重构前：

```yaml
---
- name: 部署 Nginx
  hosts: managed
  become: true
  vars:
    nginx_port: 80

  tasks:
    - name: 安装 Nginx
      ansible.builtin.package:
        name: nginx
        state: present

    - name: 分发配置
      ansible.builtin.template:
        src: templates/nginx.conf.j2
        dest: /etc/nginx/nginx.conf
        mode: '0644'
      notify: 重载 Nginx

    - name: 启动 Nginx
      ansible.builtin.service:
        name: nginx
        state: started
        enabled: true

  handlers:
    - name: 重载 Nginx
      ansible.builtin.service:
        name: nginx
        state: reloaded
```

重构后，`site.yml` 只负责选择主机和角色：

```yaml
---
- name: 部署 Web 服务
  hosts: managed
  become: true

  roles:
    - role: nginx
      vars:
        nginx_role_port: 80
```

`roles/nginx/tasks/main.yml`：

```yaml
---
- name: 安装 Nginx
  ansible.builtin.package:
    name: nginx
    state: present

- name: 分发 Nginx 配置
  ansible.builtin.template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
    mode: '0644'
  notify: 重载 Nginx

- name: 启动并启用 Nginx
  ansible.builtin.service:
    name: nginx
    state: started
    enabled: true
```

### 8.3 Roles 与 Collections

Role 是针对某个功能的目录封装；Collection 是更大的分发单位，可以包含多个 roles、modules、plugins 和文档。

完整限定名称如 `ansible.builtin.copy` 中：

- `ansible` 是命名空间。
- `builtin` 是 collection 名称。
- `copy` 是模块名。

查看已安装 collection：

```bash
ansible-galaxy collection list
ansible-galaxy role list
```

项目依赖可以写入 `requirements.yml`，再统一安装：

```yaml
---
collections:
  - name: community.general
    version: '>=10.0.0,<12.0.0'

roles:
  - name: geerlingguy.nginx
    version: 3.2.0
```

```bash
ansible-galaxy install -r requirements.yml
```

生产项目应固定经过验证的版本范围，升级前查看变更说明并在测试环境验证。

---

## 9. 错误处理与流程控制

### 9.1 block、rescue 与 always

`block` 把一组任务放在一起，并允许统一设置条件、提权和异常处理：

```yaml
- name: 更新应用配置并处理失败
  block:
    - name: 备份当前配置
      ansible.builtin.copy:
        src: /etc/lab/app.conf
        dest: /etc/lab/app.conf.backup
        remote_src: true
        mode: '0644'

    - name: 分发新配置
      ansible.builtin.template:
        src: app.conf.j2
        dest: /etc/lab/app.conf
        mode: '0644'
        validate: /usr/local/bin/lab-check-config %s

  rescue:
    - name: 恢复备份配置
      ansible.builtin.copy:
        src: /etc/lab/app.conf.backup
        dest: /etc/lab/app.conf
        remote_src: true
        mode: '0644'

    - name: 报告更新失败
      ansible.builtin.debug:
        msg: "{{ inventory_hostname }} 配置更新失败，已尝试恢复"

  always:
    - name: 删除临时备份
      ansible.builtin.file:
        path: /etc/lab/app.conf.backup
        state: absent

  become: true
```

`rescue` 只在 `block` 中任务失败时执行；`always` 无论成功或失败都会执行，适合清理临时资源和记录结果。

### 9.2 any_errors_fatal

默认情况下，一台主机失败不一定立即停止其他主机。对必须保持集群一致的操作，可设置：

```yaml
---
- name: 执行必须全局一致的变更
  hosts: managed
  any_errors_fatal: true

  tasks:
    - name: 检查统一前置条件
      ansible.builtin.assert:
        that:
          - ansible_facts.distribution == 'Archlinux'
        fail_msg: 检测到非 Arch Linux 节点，停止整个 play
```

一台主机出现未处理失败后，Ansible 会尽快终止整个 play。该选项适合无法接受部分成功的变更。

### 9.3 serial 滚动执行

逐批更新可以降低服务整体中断风险：

```yaml
---
- name: 滚动更新四台实验节点
  hosts: managed
  serial: 2
  become: true

  tasks:
    - name: 更新 Nginx 配置
      ansible.builtin.template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
        mode: '0644'
      notify: 重载 Nginx

  handlers:
    - name: 重载 Nginx
      ansible.builtin.service:
        name: nginx
        state: reloaded
```

实验中的 4 台节点会分两批，每批 2 台。也可以使用百分比，例如 `serial: '25%'`。

### 9.4 delegate_to 与 run_once

`delegate_to` 让任务在其他主机上执行，但任务上下文仍属于当前 inventory 主机：

```yaml
- name: 从控制节点检查被控节点 HTTP 端口
  ansible.builtin.uri:
    url: "http://{{ ansible_host }}:80/"
    status_code: 200
  delegate_to: localhost
  become: false
```

`run_once` 让任务只执行一次：

```yaml
- name: 只生成一次部署批次标识
  ansible.builtin.set_fact:
    deployment_batch_id: "{{ lookup('ansible.builtin.pipe', 'date +%Y%m%d%H%M%S') }}"
  run_once: true
  delegate_to: localhost
```

常见组合场景：

- 在控制节点调用负载均衡器接口摘除当前节点。
- 数据库迁移只执行一次。
- 生成一次报告或部署标识。
- 从控制节点验证每台被控节点的外部访问路径。

使用 `serial` 时，`run_once` 可能对每个批次执行一次。若要求整个 play 真正只执行一次，可增加条件，例如只允许 `ansible_play_hosts_all[0]` 执行。

---

## 10. Vault 进阶

基础的 `ansible-vault create`、Vault 变量文件和执行时输入密码已在 [`ansible-vmware-lab.md`](./ansible-vmware-lab.md) 中介绍。本节只补充多身份、局部加密和自动化使用。

### 10.1 Vault ID

Vault ID 可为不同环境或团队使用不同密码来源：

```bash
ansible-vault encrypt --vault-id dev@prompt group_vars/dev/vault.yml
ansible-vault encrypt --vault-id prod@prompt group_vars/prod/vault.yml
```

执行时提供多个身份：

```bash
ansible-playbook site.yml \
  --vault-id dev@prompt \
  --vault-id prod@/secure/path/prod-vault-pass
```

加密文件头会记录标签，便于 Ansible 选择对应密码。标签不是密码，也不提供额外加密强度。

### 10.2 encrypt_string 局部加密

不必加密整个变量文件，可以只加密某个敏感值：

```bash
ansible-vault encrypt_string --vault-id lab@prompt \
  --name 'lab_api_token' '占位敏感值'
```

把输出的 `!vault` 块复制进变量文件：

```yaml
---
lab_api_endpoint: https://example.invalid/api
lab_api_token: !vault |
  $ANSIBLE_VAULT;1.2;AES256;lab
  6638346236326662633866386137626139653264383630636164356232666631
  3938313361386439653330313863666566666561356538390a663462353431
```

上面只是格式示意，不是真实凭据，也不应直接当作可解密密文使用。

优点是普通变量仍可读；缺点是大量局部密文会降低评审可读性。敏感项较多时，单独的 `vault.yml` 更清晰。

### 10.3 CI 中使用密码文件

```bash
ansible-playbook site.yml --vault-id ci@/run/secrets/ansible_vault_password
```

注意事项：

- 密码文件不能提交到 Git。
- 使用 CI 平台的受保护变量或密钥挂载功能临时生成。
- 文件权限限制为当前执行用户可读，例如 `0600`。
- 任务结束后立即删除临时文件。
- 禁止在日志中输出密码文件内容、解密变量或带敏感值的 `debug`。
- 限制受保护分支和受信任执行器才能读取生产 Vault 密码。
- 能使用外部密钥管理系统时，不要长期共享静态 Vault 密码。

Vault 保护的是静态文件内容，不会自动隐藏执行日志，也不能阻止拿到 Vault 密码的人解密全部数据。

---

## 11. 调试与排错

### 11.1 执行前检查

```bash
ansible-playbook --syntax-check site.yml
ansible-playbook site.yml --list-hosts
ansible-playbook site.yml --list-tasks
ansible-playbook site.yml --list-tags
```

先确认语法、目标主机和任务范围，再执行变更。

### 11.2 --check 与 --diff

检查模式尝试预测变更而不真正执行：

```bash
ansible-playbook site.yml --check
ansible-playbook site.yml --check --diff
```

`--diff` 会显示模板或文件内容差异，可能泄露敏感配置，CI 日志中应谨慎启用。

并非所有模块都完整支持检查模式。命令类任务通常无法准确预测效果，因此 `--check` 不是实际测试的替代品。

### 11.3 详细日志与单步执行

```bash
ansible-playbook site.yml -v
ansible-playbook site.yml -vvv
ansible-playbook site.yml --step
ansible-playbook site.yml --start-at-task '分发 Nginx 配置'
ansible-playbook site.yml --limit ansible-node1
ansible-playbook site.yml --limit 'managed:!ansible-node4'
```

- `-v` 到 `-vvv` 逐步增加细节，SSH 问题常用 `-vvv`。
- `--step` 在每个任务前询问是否执行，适合学习和高风险变更。
- `--start-at-task` 从指定任务名开始，依赖前置任务时要谨慎。
- `--limit` 在 playbook 的 `hosts` 范围内再次缩小目标。

### 11.4 ansible-lint

`ansible-lint` 检查常见质量问题，例如缺少任务名、使用短模块名、命令不幂等、文件权限未明确等。

```bash
ansible-lint site.yml
ansible-lint roles/nginx
```

Lint 通过不等于逻辑正确，但适合作为提交前和 CI 中的基础检查。对规则例外应写清原因，不要为了通过检查而全局禁用规则。

### 11.5 常见报错

| 现象 | 常见原因 | 处理方向 |
|---|---|---|
| `UNREACHABLE` | IP、路由、22 端口、SSH 密钥、主机指纹或用户错误 | 先手工 `ssh -vvv`，再检查 inventory 和网络 |
| `Permission denied` | SSH 用户或密钥错误，密钥权限不正确 | 核对 `ansible_user`、密钥路径和 `authorized_keys` |
| `Missing sudo password` | 任务需要提权但未提供密码 | 使用 `--ask-become-pass` 或安全的 Vault 变量 |
| `MODULE FAILURE` | Python 不存在、模块依赖缺失或远端执行异常 | 用 `-vvv` 查看模块 stderr，验证远端 Python |
| `The task includes an option with an undefined variable` | 变量拼写、作用域、组名或 facts 错误 | 用 `debug`、`default` 和 `ansible-inventory --host` 检查 |
| `No hosts matched` | `hosts`、`--limit` 或组名与 inventory 不一致 | 运行 `ansible-inventory --graph` 和 `--list-hosts` |
| YAML 解析错误 | 缩进、冒号、引号或列表层级错误 | 运行 `--syntax-check`，统一使用空格缩进 |
| Handler 未执行 | 通知任务没有 `changed`，或名称不匹配 | 检查任务结果与 `notify`、handler 名称 |
| 服务启动失败 | 配置语法、端口冲突、权限或包内容错误 | 先用配置校验命令，再看 `systemctl status` 和日志 |
| Check 模式结果异常 | 模块不支持或任务依赖前一步真实变更 | 查模块文档，并在隔离测试节点实际执行 |

建议排错顺序：

1. 确认目标主机是否正确。
2. 手工验证 SSH 和 sudo。
3. 用 `--syntax-check` 排除语法问题。
4. 用 `--limit` 缩小到一台主机。
5. 用 `-vvv` 查看连接与模块错误。
6. 用 `debug` 查看变量最终值和类型。
7. 在被控节点检查服务状态、配置和日志。
8. 修复后连续执行两次，验证成功与幂等性。

---

## 12. 最佳实践

### 12.1 推荐项目结构

```text
ansible-project/
├── ansible.cfg
├── inventory/
│   ├── lab.ini
│   └── production.ini
├── group_vars/
│   ├── all.yml
│   └── managed.yml
├── host_vars/
├── playbooks/
│   ├── site.yml
│   └── verify.yml
├── roles/
│   └── nginx/
├── collections/
│   └── requirements.yml
├── .ansible-lint
└── .gitignore
```

实验项目可以更简单，但应保持 inventory、变量、playbook 和 roles 职责分离。

### 12.2 命名规范

- play 和任务都写清晰的中文 `name`，说明动作和对象。
- role 变量使用前缀，例如 `nginx_role_port`。
- 布尔变量使用 `enable`、`enabled` 等可读前缀。
- handler 名称使用明确动作，例如“重载 Nginx”。
- 文件名使用小写字母、数字和下划线，避免空格。
- inventory 主机名保持稳定，不把短期 IP 当作唯一业务含义。

### 12.3 模块与幂等性

- 优先 `package`、`service`、`user`、`template`、`file` 等专用模块。
- 必须使用命令时，先考虑 `creates`、`removes`、`changed_when` 和 `failed_when`。
- 配置文件使用 `validate`，避免错误配置直接覆盖线上文件。
- 修改配置后通过 handler 重载，不要每次都重启服务。
- 连续运行两次，第二次检查是否仍有意外 `changed`。

### 12.4 安全与评审

- inventory、Git 和执行日志中不保存明文密码、私钥或令牌。
- Vault 密码与 Vault 密文分开保存。
- 不长期关闭 SSH 主机指纹校验。
- `--check --diff` 输出可能包含敏感内容。
- 合并前运行语法检查、lint 和测试环境执行。
- 重要变更通过代码评审，说明目标范围、风险、验证和回滚方法。
- 固定 collection 和第三方 role 版本，升级时单独评审。

### 12.5 可维护性

- 一个任务只表达一个明确目标。
- 不复制粘贴大段相同任务，使用 role、变量和循环复用。
- 不在模板中堆积复杂业务逻辑。
- 不用 `ignore_errors: true` 掩盖未知问题。
- 对外部命令和接口设置明确失败条件。
- 用 tags 辅助选择任务，但不依赖 tags 构造隐式执行顺序。

---

## 13. 完整实战：用 Role 批量部署 Nginx

本实战直接使用实验环境的 `managed` 组，对应：

- `ansible-node1`：`192.168.88.21`
- `ansible-node2`：`192.168.88.22`
- `ansible-node3`：`192.168.88.23`
- `ansible-node4`：`192.168.88.24`

SSH、sudo 和初始主机指纹沿用实验环境笔记。以下项目不包含真实密码。

### 13.1 完整文件树

```text
nginx-lab/
├── ansible.cfg
├── inventory.ini
├── group_vars/
│   └── managed.yml
├── site.yml
└── roles/
    └── nginx_lab/
        ├── defaults/
        │   └── main.yml
        ├── handlers/
        │   └── main.yml
        ├── tasks/
        │   └── main.yml
        └── templates/
            ├── index.html.j2
            └── nginx.conf.j2
```

### 13.2 ansible.cfg

```ini
[defaults]
inventory = ./inventory.ini
interpreter_python = auto_silent
host_key_checking = True
retry_files_enabled = False
roles_path = ./roles

[ssh_connection]
pipelining = True
```

### 13.3 inventory.ini

```ini
[managed]
ansible-node1 ansible_host=192.168.88.21
ansible-node2 ansible_host=192.168.88.22
ansible-node3 ansible_host=192.168.88.23
ansible-node4 ansible_host=192.168.88.24

[managed:vars]
ansible_user=ansible
```

### 13.4 group_vars/managed.yml

```yaml
---
nginx_lab_listen_port: 80
nginx_lab_server_name: _
nginx_lab_worker_connections: 1024
nginx_lab_document_root: /usr/share/nginx/html
nginx_lab_page_title: Ansible Nginx 实验
```

### 13.5 site.yml

```yaml
---
- name: 在全部 managed 节点部署 Nginx
  hosts: managed
  become: true
  gather_facts: true
  serial: 2

  roles:
    - role: nginx_lab
      tags:
        - nginx
```

### 13.6 roles/nginx_lab/defaults/main.yml

```yaml
---
nginx_lab_package_name: nginx
nginx_lab_service_name: nginx
nginx_lab_listen_port: 80
nginx_lab_server_name: _
nginx_lab_worker_processes: auto
nginx_lab_worker_connections: 1024
nginx_lab_document_root: /usr/share/nginx/html
nginx_lab_page_title: Ansible 管理的 Nginx
```

### 13.7 roles/nginx_lab/tasks/main.yml

```yaml
---
- name: 断言目标系统为 Arch Linux
  ansible.builtin.assert:
    that:
      - ansible_facts.distribution == 'Archlinux'
    fail_msg: 此实验角色仅针对 Arch Linux 节点

- name: 安装 Nginx 软件包
  ansible.builtin.package:
    name: "{{ nginx_lab_package_name }}"
    state: present
  tags:
    - packages

- name: 确保站点根目录存在
  ansible.builtin.file:
    path: "{{ nginx_lab_document_root }}"
    state: directory
    owner: root
    group: root
    mode: '0755'

- name: 分发实验首页
  ansible.builtin.template:
    src: index.html.j2
    dest: "{{ nginx_lab_document_root }}/index.html"
    owner: root
    group: root
    mode: '0644'
  tags:
    - content

- name: 校验并分发 Nginx 主配置
  ansible.builtin.template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
    owner: root
    group: root
    mode: '0644'
    backup: true
    validate: /usr/bin/nginx -t -c %s
  notify: 重载 Nginx
  tags:
    - config

- name: 确保 Nginx 已启动并开机自启
  ansible.builtin.service:
    name: "{{ nginx_lab_service_name }}"
    state: started
    enabled: true

- name: 立即处理配置变更通知
  ansible.builtin.meta: flush_handlers

- name: 从被控节点本机验证 Nginx 响应
  ansible.builtin.uri:
    url: "http://127.0.0.1:{{ nginx_lab_listen_port }}/"
    return_content: true
    status_code: 200
  register: nginx_lab_http_result
  changed_when: false

- name: 断言首页包含当前主机名
  ansible.builtin.assert:
    that:
      - inventory_hostname in nginx_lab_http_result.content
    fail_msg: Nginx 首页没有包含预期主机名
```

### 13.8 roles/nginx_lab/handlers/main.yml

```yaml
---
- name: 重载 Nginx
  ansible.builtin.service:
    name: "{{ nginx_lab_service_name }}"
    state: reloaded
```

### 13.9 roles/nginx_lab/templates/nginx.conf.j2

```jinja2
# 此文件由 Ansible 管理，请勿手工修改。
user http;
worker_processes {{ nginx_lab_worker_processes }};
error_log /var/log/nginx/error.log warn;
pid /run/nginx.pid;

events {
    worker_connections {{ nginx_lab_worker_connections }};
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;
    access_log /var/log/nginx/access.log;
    sendfile on;
    keepalive_timeout 65;

    server {
        listen {{ nginx_lab_listen_port }};
        server_name {{ nginx_lab_server_name }};
        root {{ nginx_lab_document_root }};
        index index.html;

        location / {
            try_files $uri $uri/ =404;
        }
    }
}
```

### 13.10 roles/nginx_lab/templates/index.html.j2

```jinja2
<!doctype html>
<html lang="zh-CN">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>{{ nginx_lab_page_title }}</title>
</head>
<body>
  <h1>{{ nginx_lab_page_title }}</h1>
  <p>当前节点：{{ inventory_hostname }}</p>
  <p>节点地址：{{ ansible_host }}</p>
  <p>系统版本：{{ ansible_facts.distribution }} {{ ansible_facts.distribution_version }}</p>
  <p>此页面由 Ansible 模板统一生成。</p>
</body>
</html>
```

### 13.11 创建、检查与执行

在控制节点进入项目目录，先验证 inventory：

```bash
ansible-inventory --graph
ansible managed --list-hosts
ansible managed -m ansible.builtin.ping
```

再检查 playbook：

```bash
ansible-playbook --syntax-check site.yml
ansible-playbook site.yml --list-hosts
ansible-playbook site.yml --check --diff --ask-become-pass
```

确认范围和差异后执行：

```bash
ansible-playbook site.yml --ask-become-pass
```

如果提权密码已按实验笔记使用 Vault 安全配置，则按对应 Vault 方式执行，不要把密码写进命令行或仓库。

### 13.12 验证四台节点

从控制节点访问：

```bash
curl http://192.168.88.21/
curl http://192.168.88.22/
curl http://192.168.88.23/
curl http://192.168.88.24/
```

检查服务状态：

```bash
ansible managed -b --ask-become-pass \
  -m ansible.builtin.command \
  -a 'systemctl is-active nginx'
```

再次执行 playbook 验证幂等性：

```bash
ansible-playbook site.yml --ask-become-pass
```

第二次执行时，预期 `changed=0`。若仍有变更，使用 `--diff` 定位不断变化的文件或任务。

### 13.13 修改配置并观察 Handler

把 `group_vars/managed.yml` 中端口改为 `8080`：

```yaml
nginx_lab_listen_port: 8080
```

再次执行：

```bash
ansible-playbook site.yml --ask-become-pass --tags nginx
```

预期配置模板发生变化，通知 handler 重载服务。随后验证：

```bash
curl http://192.168.88.21:8080/
curl http://192.168.88.22:8080/
curl http://192.168.88.23:8080/
curl http://192.168.88.24:8080/
```

该实验串联了 inventory、组变量、facts、role、template、handler、serial、assert、uri、tags 和幂等性验证。

---

## 14. 学习路线与实践任务

建议按顺序完成，每项都保留命令、结果和故障记录。

### 第一阶段：主机选择与模块

- [ ] 在 `192.168.88.21-24` 上运行 `ansible.builtin.ping` 并解释 `pong` 代表什么。
- [ ] 用 `ansible.builtin.setup` 分别查看系统发行版、默认 IPv4 和内存 facts。
- [ ] 把 4 台主机划分为 `web`、`app`、`db` 子组，并建立 `managed:children`。
- [ ] 练习并解释 `managed:!db`、`web:app` 和 `web:&managed`。
- [ ] 分别用 `command` 和 `shell` 完成查询任务，说明为何优先使用 `command`。

### 第二阶段：幂等 Playbook

- [ ] 写 playbook，在 4 台节点创建 `/srv/ansible-practice` 目录。
- [ ] 用 `copy` 写入说明文件，并明确 owner、group 和 mode。
- [ ] 连续运行两次，确认第二次 `changed=0`。
- [ ] 故意用 `shell` 重复追加一行，观察幂等性被破坏，再改为 `lineinfile`。
- [ ] 给一个查询命令添加 `changed_when: false` 和合理的 `failed_when`。

### 第三阶段：变量与模板

- [ ] 在 `group_vars/managed.yml` 定义统一端口和页面标题。
- [ ] 在 `host_vars/ansible-node1.yml` 覆盖单台节点标题。
- [ ] 用 `debug` 验证变量优先级和变量类型。
- [ ] 编写包含循环、条件和 `default` filter 的模板。
- [ ] 修改一个变量并用 `--check --diff` 预览模板变化。

### 第四阶段：Handlers 与 Roles

- [ ] 给配置模板添加 handler，证明模板不变时不会重载服务。
- [ ] 让两个任务通知同一 handler，观察 handler 去重。
- [ ] 使用 `meta: flush_handlers` 后立即运行验证任务。
- [ ] 用 `ansible-galaxy role init` 创建一个练习 role。
- [ ] 把原有平铺 playbook 拆成 defaults、tasks、handlers 和 templates。

### 第五阶段：流程控制与排错

- [ ] 使用 `serial: 2` 分两批处理 `192.168.88.21-24`。
- [ ] 用 `delegate_to: localhost` 从控制节点逐台访问被控节点网页。
- [ ] 用 `run_once` 生成一次部署标识，并观察与 `serial` 的关系。
- [ ] 写一个 `block/rescue/always` 练习，故意让配置校验失败并恢复。
- [ ] 分别制造 SSH 用户错误、未定义变量和服务配置错误，再按排错顺序恢复。

### 第六阶段：安全与工程化

- [ ] 复习实验笔记中的 Vault 基础用法，不在仓库保存明文提权密码。
- [ ] 使用 `encrypt_string` 加密一个无真实价值的练习变量。
- [ ] 给开发和实验环境创建不同 Vault ID，理解密码来源分离。
- [ ] 安装并运行 `ansible-lint`，逐项理解而不是直接忽略规则。
- [ ] 为项目执行语法检查、检查模式、实际执行和第二次幂等验证。
- [ ] 在 Git 提交前检查 diff 中没有密码、私钥、令牌和本地敏感路径。

### 综合验收

- [ ] 4 台节点都能被 inventory 正确分组和访问。
- [ ] 完整 Nginx role 可从零部署并通过 HTTP 验证。
- [ ] 配置改变时 handler 恰好执行一次。
- [ ] `serial` 滚动执行不会同时修改全部节点。
- [ ] 第二次执行 `changed=0`。
- [ ] 能独立解释一次 `UNREACHABLE` 和一次变量未定义错误。
- [ ] 能说明变量来源、覆盖关系和最终值。
- [ ] 能从 Git 中还原项目，并在重建的实验环境再次运行。

---

## 15. 总结

Ansible 的核心不在于记住命令，而在于把系统状态表达成可读、可重复、可评审的自动化代码。

需要形成以下思维习惯：

1. 先确认目标主机范围，再执行任务。
2. 优先描述目标状态，而不是堆砌 shell 命令。
3. 用专用模块、条件和结果判断保持幂等。
4. 用 inventory、变量和 facts 分离环境差异。
5. 用 template 管理配置，用 handler 响应真实变更。
6. 用 role 组织可复用功能，用 collection 管理扩展内容。
7. 用 check、diff、limit、详细日志和 lint 降低变更风险。
8. 用 Vault 或外部密钥系统保护敏感信息，但同时防止日志泄露。
9. 用 serial、错误处理和验证任务控制发布影响范围。
10. 每次变更都能说明验证方法、失败处理和回滚路径。

完成本笔记的实战与检查表后，应能够从“会运行 Ansible 命令”进阶到“能设计、调试和维护一个小型 Ansible 项目”。
