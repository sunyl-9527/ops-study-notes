# VMware + Arch Linux 搭建 Ansible 实践环境

本文搭建一套隔离的 Ansible 学习环境：1 台控制节点管理 4 台被控节点。重点是理解 inventory、SSH 认证、ad-hoc 命令、playbook 和故障排查。

## 1. 实验拓扑

```text
Windows 宿主机
└─ VMware VMnet8 NAT: 192.168.88.0/24
   ├─ ansible-ctl    192.168.88.10
   ├─ ansible-node1  192.168.88.21
   ├─ ansible-node2  192.168.88.22
   ├─ ansible-node3  192.168.88.23
   └─ ansible-node4  192.168.88.24
```

建议每台虚拟机配置：

| 项目 | 建议值 |
|---|---|
| CPU | 2 vCPU |
| 内存 | 2-4 GB |
| 磁盘 | 20 GB |
| 系统 | Arch Linux |
| 网络 | VMware VMnet8 NAT |

这是实验网段。若本机已有相同网段，应统一替换本文地址，避免路由冲突。

## 2. 前置条件

- VMware Workstation 和 Arch Linux ISO。
- `VMnet8` 已配置为 NAT，宿主机可访问虚拟机。
- 每台虚拟机完成基础系统安装并创建普通用户 `ansible`。
- 节点时间同步正常，主机名和 IP 唯一。

不要在公开笔记中记录实验机真实密码。安装阶段可使用临时密码，完成 SSH 密钥配置后立即更换或禁用密码登录。

## 3. 检查 VMware NAT

在管理员 PowerShell 中检查 VMware 服务：

```powershell
Get-Service -Name "VMnetDHCP", "VMware NAT Service"
```

需要时启动服务：

```powershell
Start-Service -Name "VMnetDHCP"
Start-Service -Name "VMware NAT Service"
```

确认 `VMnet8` 的子网、掩码、网关和 DHCP 范围没有与静态地址冲突。建议把 `192.168.88.10` 和 `192.168.88.21-24` 排除在 DHCP 地址池之外。

## 4. 初始化所有节点

设置主机名，每台机器只执行与自身对应的一条：

```bash
sudo hostnamectl set-hostname ansible-ctl
sudo hostnamectl set-hostname ansible-node1
sudo hostnamectl set-hostname ansible-node2
sudo hostnamectl set-hostname ansible-node3
sudo hostnamectl set-hostname ansible-node4
```

安装基础软件并启用 SSH：

```bash
sudo pacman -Syu --needed python sudo openssh
sudo systemctl enable --now sshd
```

被控节点必须有 Python；控制节点还需要安装 Ansible：

```bash
sudo pacman -S --needed ansible
ansible --version
```

将主机名写入所有节点的 `/etc/hosts`：

```text
192.168.88.10 ansible-ctl
192.168.88.21 ansible-node1
192.168.88.22 ansible-node2
192.168.88.23 ansible-node3
192.168.88.24 ansible-node4
```

静态 IP 的配置方式取决于使用 NetworkManager、systemd-networkd 还是其他网络管理工具。修改前先执行以下命令确认当前方案：

```bash
systemctl is-active NetworkManager
systemctl is-active systemd-networkd
ip -brief address
ip route
```

## 5. 配置 SSH 密钥认证

在控制节点以 `ansible` 用户生成密钥：

```bash
ssh-keygen -t ed25519 -a 100
```

复制公钥到各被控节点：

```bash
ssh-copy-id ansible@192.168.88.21
ssh-copy-id ansible@192.168.88.22
ssh-copy-id ansible@192.168.88.23
ssh-copy-id ansible@192.168.88.24
```

逐台验证：

```bash
ssh ansible@192.168.88.21 hostname
ssh ansible@192.168.88.22 hostname
ssh ansible@192.168.88.23 hostname
ssh ansible@192.168.88.24 hostname
```

不要在 inventory 中保存明文 `ansible_password`。必须使用密码时，用 Ansible Vault 加密变量。

## 6. 配置 sudo

在每个被控节点使用 `visudo` 创建 `/etc/sudoers.d/ansible`：

```sudoers
ansible ALL=(ALL:ALL) ALL
```

验证语法和权限：

```bash
sudo chmod 440 /etc/sudoers.d/ansible
sudo visudo -cf /etc/sudoers.d/ansible
```

实验环境也不建议默认配置无限制免密 sudo。需要自动提权时使用 `--ask-become-pass`，或将提权密码放进 Vault。

## 7. 创建 Ansible 项目

在控制节点执行：

```bash
mkdir -p ~/ansible-lab
cd ~/ansible-lab
```

创建 `ansible.cfg`：

```ini
[defaults]
inventory = ./inventory.ini
interpreter_python = auto_silent
host_key_checking = True
retry_files_enabled = False
```

创建 `inventory.ini`：

```ini
[managed]
ansible-node1 ansible_host=192.168.88.21
ansible-node2 ansible_host=192.168.88.22
ansible-node3 ansible_host=192.168.88.23
ansible-node4 ansible_host=192.168.88.24

[managed:vars]
ansible_user=ansible
```

首次连接前写入 SSH 主机指纹：

```bash
ssh-keyscan -H 192.168.88.21 192.168.88.22 192.168.88.23 192.168.88.24 >> ~/.ssh/known_hosts
```

实验中应核对显示的指纹。不要为了省事长期关闭 `host_key_checking`。

## 8. 验证 Ansible

检查 inventory：

```bash
ansible-inventory --graph
ansible managed --list-hosts
```

测试 SSH 和 Python：

```bash
ansible managed -m ping
```

查看基础信息：

```bash
ansible managed -m command -a 'hostnamectl --static'
ansible managed -m setup -a 'filter=ansible_distribution*'
```

需要 sudo 时：

```bash
ansible managed -b --ask-become-pass -m command -a 'id'
```

## 9. 第一个 playbook

创建 `ping.yml`：

```yaml
---
- name: Verify managed nodes
  hosts: managed
  gather_facts: true

  tasks:
    - name: Test Ansible connection
      ansible.builtin.ping:

    - name: Show hostname
      ansible.builtin.debug:
        var: ansible_hostname
```

先做语法检查，再执行：

```bash
ansible-playbook --syntax-check ping.yml
ansible-playbook ping.yml
```

## 10. 使用 Ansible Vault

创建加密变量文件：

```bash
mkdir -p group_vars/managed
ansible-vault create group_vars/managed/vault.yml
```

文件中可保存：

```yaml
ansible_become_password: "<temporary-lab-password>"
```

执行时输入 Vault 密码：

```bash
ansible managed -b --ask-vault-pass -m command -a 'id'
```

Vault 文件可以提交，Vault 密码不能提交。生产环境优先使用专用密钥管理系统。

## 11. 常见问题

### SSH 不通

在宿主机和控制节点分别检查：

```powershell
Test-NetConnection 192.168.88.21 -Port 22
```

```bash
ip route
ping -c 3 192.168.88.21
ssh -vvv ansible@192.168.88.21
```

### Ansible 报 Python 不存在

```bash
ssh ansible@192.168.88.21 'command -v python && python --version'
```

Arch Linux 安装 `python` 包后再重试。

### 主机指纹变化

确认虚拟机确实被重装或替换后，再删除旧记录：

```bash
ssh-keygen -R 192.168.88.21
ssh-keyscan -H 192.168.88.21 >> ~/.ssh/known_hosts
```

### sudo 失败

```bash
ssh -t ansible@192.168.88.21 'sudo -v && sudo id'
```

检查用户组、`/etc/sudoers.d/ansible` 权限和 `visudo` 校验结果。

## 12. 实验完成检查表

- [ ] 5 台虚拟机的主机名和 IP 与规划一致。
- [ ] 控制节点能通过 SSH 密钥登录 4 台被控节点。
- [ ] inventory 能正确列出 `managed` 组。
- [ ] `ansible managed -m ping` 全部返回 `pong`。
- [ ] `ping.yml` 通过语法检查并执行成功。
- [ ] inventory 和 Git 中没有明文密码或私钥。
- [ ] 已记录快照、重建步骤和故障排查结果。

完成基础实验后，可继续练习 package、service、template、copy、user、firewalld/ufw 等模块，以及 role、handler、tag 和幂等性验证。
