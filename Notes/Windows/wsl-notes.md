# WSL 学习笔记

> 适用对象：希望在 Windows 上学习 Linux、开发环境搭建、云计算运维实验和容器化实践的学习者。
>
> 学习目标：掌握 WSL 的安装、发行版管理、Windows 与 Linux 文件互通、网络访问、systemd、Docker、备份迁移和常见故障排查。

---

## 一、WSL 是什么

WSL 全称是 Windows Subsystem for Linux，即 Windows Linux 子系统。

它允许你在 Windows 上直接运行 Linux 发行版，例如 Ubuntu、Debian、Arch、Kali 等，不需要像传统虚拟机那样完整安装一个独立操作系统。

WSL 适合这些场景：

- 学习 Linux 命令。
- 搭建开发环境。
- 运行 Shell、Python、Node.js、Go 等工具链。
- 做云计算运维和 DevOps 实验。
- 使用 Docker、Kubernetes、Ansible 等工具。
- 在 Windows 和 Linux 工具之间快速切换。

一句话理解：WSL 是 Windows 上的 Linux 实验环境，也是开发和运维学习里非常好用的轻量级 Linux 平台。

---

## 二、WSL 1 与 WSL 2

| 对比项 | WSL 1 | WSL 2 |
|---|---|---|
| 实现方式 | 系统调用转换层 | 轻量虚拟机 + Linux 内核 |
| Linux 兼容性 | 一般 | 更好 |
| 文件系统性能 | Windows 文件访问较快 | Linux 文件系统内性能更好 |
| Docker 支持 | 不适合 | 推荐 |
| 网络模型 | 与 Windows 更接近 | 有独立虚拟网络 |
| 推荐程度 | 特殊场景使用 | 日常推荐 |

建议：

- 新学习者优先使用 `WSL 2`。
- Docker、Kubernetes、云原生实验优先使用 `WSL 2`。
- 如果大量操作 Windows 文件系统，可以关注 WSL 2 与 Windows 目录之间的性能差异。

---

## 三、安装与基础配置

### 1. 查看 WSL 是否可用

在 PowerShell 中执行：

```powershell
wsl --status
```

查看 WSL 版本：

```powershell
wsl --version
```

查看帮助：

```powershell
wsl --help
```

### 2. 安装 WSL

在管理员 PowerShell 中执行：

```powershell
wsl --install
```

安装指定发行版：

```powershell
wsl --install -d Ubuntu
```

查看可安装发行版：

```powershell
wsl --list --online
```

### 3. 查看已安装发行版

```powershell
wsl --list --verbose
```

简写：

```powershell
wsl -l -v
```

输出中常见字段：

- `NAME`：发行版名称。
- `STATE`：运行状态。
- `VERSION`：WSL 版本，通常建议为 `2`。

### 4. 设置默认 WSL 版本

```powershell
wsl --set-default-version 2
```

### 5. 修改某个发行版的 WSL 版本

```powershell
wsl --set-version Ubuntu 2
```

### 6. 设置默认发行版

```powershell
wsl --set-default Ubuntu
```

---

## 四、发行版管理

### 1. 启动发行版

```powershell
wsl
```

启动指定发行版：

```powershell
wsl -d Ubuntu
```

以指定用户启动：

```powershell
wsl -d Ubuntu -u root
```

在 WSL 中直接执行 Linux 命令：

```powershell
wsl -d Ubuntu -- uname -a
wsl -d Ubuntu -- whoami
```

### 2. 关闭发行版

关闭指定发行版：

```powershell
wsl --terminate Ubuntu
```

关闭所有 WSL 实例：

```powershell
wsl --shutdown
```

### 3. 注销发行版

注销会删除该发行版及其所有数据，执行前必须确认已经备份。

```powershell
wsl --unregister Ubuntu
```

### 4. 导出与导入发行版

导出备份：

```powershell
wsl --export Ubuntu D:\backup\ubuntu-wsl.tar
```

导入恢复：

```powershell
wsl --import Ubuntu-Restore D:\WSL\Ubuntu-Restore D:\backup\ubuntu-wsl.tar --version 2
```

这组命令适合：

- 迁移 WSL 到其他磁盘。
- 重装系统前备份环境。
- 保存一个干净实验快照。
- 复制多套类似实验环境。

---

## 五、Windows 与 Linux 文件互通

### 1. 在 WSL 中访问 Windows 文件

Windows 磁盘会挂载到 `/mnt` 下。

```bash
cd /mnt/c/Users/DELL/Desktop
ls
```

常见路径：

```text
C:\Users\DELL\Desktop
```

在 WSL 中对应：

```text
/mnt/c/Users/DELL/Desktop
```

### 2. 在 Windows 中访问 WSL 文件

在资源管理器地址栏输入：

```text
\\wsl$
```

或者进入指定发行版：

```text
\\wsl$\Ubuntu
```

### 3. 推荐的文件存放方式

如果主要使用 Linux 工具，例如 `git`、`npm`、`python`、`go`、`docker`，建议把项目放在 WSL 的 Linux 文件系统中，例如：

```text
/home/username/projects
```

如果主要用 Windows 软件编辑和管理，可以放在 Windows 目录中，例如：

```text
C:\Users\DELL\Desktop\work_note
```

实践建议：

- Linux 工具频繁读写的项目，优先放在 WSL 内部文件系统。
- Windows 软件频繁使用的文档和笔记，可以放在 Windows 文件系统。
- 不要在 Windows 和 WSL 中同时用两个工具高频修改同一个文件，避免状态混乱。

---

## 六、常用 WSL 命令速查

| 场景 | 命令 |
|---|---|
| 查看状态 | `wsl --status` |
| 查看版本 | `wsl --version` |
| 查看已安装发行版 | `wsl -l -v` |
| 查看可安装发行版 | `wsl --list --online` |
| 安装默认发行版 | `wsl --install` |
| 安装指定发行版 | `wsl --install -d Ubuntu` |
| 启动默认发行版 | `wsl` |
| 启动指定发行版 | `wsl -d Ubuntu` |
| 以 root 启动 | `wsl -d Ubuntu -u root` |
| 关闭指定发行版 | `wsl --terminate Ubuntu` |
| 关闭所有 WSL | `wsl --shutdown` |
| 设置默认发行版 | `wsl --set-default Ubuntu` |
| 设置默认 WSL 版本 | `wsl --set-default-version 2` |
| 导出备份 | `wsl --export Ubuntu D:\backup\ubuntu.tar` |
| 导入恢复 | `wsl --import Ubuntu2 D:\WSL\Ubuntu2 D:\backup\ubuntu.tar --version 2` |
| 注销发行版 | `wsl --unregister Ubuntu` |

---

## 七、WSL 内部基础配置

### 1. 更新软件包

Ubuntu / Debian：

```bash
sudo apt update
sudo apt upgrade -y
```

Arch：

```bash
sudo pacman -Syu
```

### 2. 安装常用工具

Ubuntu / Debian：

```bash
sudo apt install -y git curl wget vim unzip jq net-tools iproute2 dnsutils
```

Arch：

```bash
sudo pacman -S git curl wget vim unzip jq net-tools iproute2 bind
```

### 3. 查看系统信息

```bash
uname -a
cat /etc/os-release
whoami
pwd
ip addr
df -h
free -h
```

### 4. 设置默认用户

不同发行版设置方式可能不同。Ubuntu 常见方式：

```powershell
ubuntu config --default-user username
```

通用方式是编辑 WSL 内部的 `/etc/wsl.conf`：

```ini
[user]
default=username
```

修改后在 Windows 中执行：

```powershell
wsl --terminate Ubuntu
wsl -d Ubuntu
```

---

## 八、wsl.conf 配置

WSL 的部分行为可以通过 `/etc/wsl.conf` 配置。

常见配置示例：

```ini
[boot]
systemd=true

[user]
default=username

[automount]
enabled=true
root=/mnt/
options=metadata,umask=22,fmask=11

[interop]
enabled=true
appendWindowsPath=true
```

常见配置说明：

- `[boot] systemd=true`：启用 systemd。
- `[user] default=username`：设置默认登录用户。
- `[automount] enabled=true`：自动挂载 Windows 磁盘。
- `[automount] root=/mnt/`：设置挂载根路径。
- `[interop] enabled=true`：允许 WSL 调用 Windows 程序。
- `appendWindowsPath=true`：把 Windows PATH 加入 WSL。

修改 `/etc/wsl.conf` 后，需要关闭并重新启动发行版：

```powershell
wsl --shutdown
wsl -d Ubuntu
```

---

## 九、systemd 与服务管理

### 1. 启用 systemd

编辑 `/etc/wsl.conf`：

```ini
[boot]
systemd=true
```

然后在 Windows 中执行：

```powershell
wsl --shutdown
wsl -d Ubuntu
```

### 2. 验证 systemd

```bash
ps -p 1 -o comm=
```

如果输出是：

```text
systemd
```

说明 systemd 已启用。

### 3. 管理服务

```bash
systemctl status ssh
sudo systemctl start ssh
sudo systemctl enable ssh
sudo systemctl restart ssh
```

注意：WSL 是学习和开发环境，不等同于完整生产服务器。部分依赖内核模块、硬件设备或底层网络能力的服务，行为可能和真实 Linux 主机不同。

---

## 十、网络访问

### 1. WSL 访问 Windows

WSL 通常可以访问 Windows 主机。常见方式是读取 `/etc/resolv.conf` 中的 nameserver。

```bash
cat /etc/resolv.conf
```

也可以测试访问 Windows 上运行的服务：

```bash
curl http://localhost:端口
```

### 2. Windows 访问 WSL

如果 WSL 中启动了 Web 服务：

```bash
python3 -m http.server 8000
```

Windows 浏览器通常可以访问：

```text
http://localhost:8000
```

### 3. 查看监听端口

WSL 中：

```bash
ss -lntp
```

Windows PowerShell 中：

```powershell
Get-NetTCPConnection -State Listen
```

### 4. 常见网络问题

常见现象：

- WSL 无法访问外网。
- DNS 解析失败。
- Windows 无法访问 WSL 服务。
- 代理配置不一致。
- VPN 开启后 WSL 网络异常。

排查思路：

- 检查 Windows 网络是否正常。
- 执行 `wsl --shutdown` 后重新启动。
- 检查 `/etc/resolv.conf`。
- 检查 Windows 防火墙。
- 检查代理环境变量：`http_proxy`、`https_proxy`。
- 检查服务是否监听在 `0.0.0.0` 或正确地址上。

---

## 十一、在 WSL 中使用 Windows 程序

WSL 可以直接调用 Windows 程序。

### 1. 打开资源管理器

```bash
explorer.exe .
```

### 2. 使用 Windows VS Code

```bash
code .
```

前提是已安装 VS Code，并安装 Remote - WSL 扩展。

### 3. 调用 Windows 命令

```bash
notepad.exe test.txt
cmd.exe /c dir
powershell.exe -Command "Get-Date"
```

### 4. 路径转换

把 Linux 路径转换为 Windows 路径：

```bash
wslpath -w /home/user/project
```

把 Windows 路径转换为 Linux 路径：

```bash
wslpath 'C:\Users\DELL\Desktop'
```

---

## 十二、开发环境实践

### 1. Git

```bash
git --version
git config --global user.name "your-name"
git config --global user.email "your-email@example.com"
```

建议：

- Windows Git 和 WSL Git 可以同时存在。
- 同一个仓库尽量固定使用一种环境操作。
- 注意换行符设置，避免 Windows 和 Linux 换行差异造成大量无意义变更。

### 2. Python

```bash
python3 --version
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
```

### 3. Node.js

推荐使用 `nvm` 管理 Node.js 版本。

```bash
node -v
npm -v
```

### 4. SSH

```bash
ssh-keygen -t ed25519 -C "your-email@example.com"
cat ~/.ssh/id_ed25519.pub
```

常见用途：

- 连接远程 Linux 服务器。
- 访问 GitHub / GitLab。
- 做 Ansible 实验。

---

## 十三、WSL 与 Docker

### 1. 推荐方式

在 Windows 上使用 Docker Desktop，并开启 WSL 2 后端。

常见路径：

```text
Docker Desktop -> Settings -> Resources -> WSL Integration
```

开启对应发行版集成后，可以在 WSL 中直接使用：

```bash
docker version
docker ps
docker run hello-world
```

### 2. Docker Compose

```bash
docker compose version
docker compose up -d
docker compose ps
docker compose logs
docker compose down
```

### 3. 常见问题

常见现象：

- WSL 中找不到 `docker` 命令。
- Docker Desktop 未启动。
- 当前发行版没有开启 WSL Integration。
- Docker 占用磁盘空间过大。
- 容器端口无法访问。

排查思路：

- 确认 Docker Desktop 正在运行。
- 确认 WSL Integration 已开启。
- 执行 `wsl --shutdown` 后重启 Docker Desktop。
- 使用 `docker system df` 查看空间占用。
- 使用 `docker ps` 确认端口映射。

---

## 十四、WSL 与 VS Code

### 1. 推荐插件

- Remote - WSL
- Dev Containers
- Docker

### 2. 从 WSL 打开项目

```bash
cd ~/projects/demo
code .
```

VS Code 左下角出现 `WSL: Ubuntu` 之类标识，说明当前窗口连接到了 WSL 环境。

### 3. 使用建议

- 在 WSL 项目中使用 WSL 终端。
- 依赖安装在 WSL 内部，不要混用 Windows 和 WSL 的包管理器。
- 需要容器开发时，可以结合 Dev Containers。

---

## 十五、备份、迁移与恢复

### 1. 导出备份

```powershell
wsl --export Ubuntu D:\backup\ubuntu-2026-07-11.tar
```

### 2. 导入恢复

```powershell
wsl --import Ubuntu-Restore D:\WSL\Ubuntu-Restore D:\backup\ubuntu-2026-07-11.tar --version 2
```

### 3. 迁移到其他磁盘

基本思路：

1. `wsl --export` 导出原发行版。
2. `wsl --unregister` 删除原发行版。
3. `wsl --import` 导入到新路径。
4. 重新设置默认用户和默认发行版。

### 4. 备份建议

- 重要环境改动前先导出备份。
- 做大型实验前先保留一个干净快照。
- 不要只备份 Windows 目录，WSL 内部的 `/home` 也需要备份。
- 涉及密钥、凭据、token 的备份文件要妥善保存。

---

## 十六、常见故障排查

### 1. WSL 启动失败

排查命令：

```powershell
wsl --status
wsl -l -v
wsl --shutdown
```

处理思路：

- 重启 WSL：`wsl --shutdown`。
- 重启 Windows。
- 检查虚拟化是否开启。
- 检查发行版是否损坏。
- 必要时先导出备份，再重建发行版。

### 2. 忘记 Linux 用户密码

以 root 进入发行版：

```powershell
wsl -d Ubuntu -u root
```

修改密码：

```bash
passwd username
```

退出后重新进入普通用户。

### 3. DNS 解析失败

检查：

```bash
cat /etc/resolv.conf
ping 8.8.8.8
ping github.com
```

处理思路：

- 先确认 Windows 网络正常。
- 执行 `wsl --shutdown` 重启 WSL。
- 检查 VPN、代理、防火墙影响。
- 必要时重新生成或手动配置 DNS。

### 4. 文件权限异常

常见原因：

- 在 `/mnt/c` 下操作 Linux 项目。
- Windows 和 Linux 工具混用修改文件。
- 挂载选项不合适。

处理建议：

- Linux 项目优先放在 `/home/username/projects`。
- 需要权限元数据时，关注 `/etc/wsl.conf` 中的 `metadata` 选项。
- 避免用 Windows 工具直接修改 WSL 系统文件。

### 5. WSL 占用空间过大

排查：

```bash
du -h -d 1 ~
docker system df
```

处理：

```bash
docker system prune
```

如果使用 Docker Desktop，也可以在 Docker Desktop 中清理镜像、容器和卷。

---

## 十七、学习路线

### 第一阶段：会安装和启动

目标：能安装 WSL，启动发行版，知道如何进入 Linux 环境。

需要掌握：

- `wsl --install`
- `wsl -l -v`
- `wsl -d`
- `wsl --shutdown`
- `wsl --terminate`

### 第二阶段：会管理发行版

目标：能管理多个 Linux 发行版，知道如何备份和恢复。

需要掌握：

- 设置默认发行版。
- 设置 WSL 版本。
- 导出与导入。
- 注销发行版。
- 设置默认用户。

### 第三阶段：会做 Linux 基础实验

目标：能把 WSL 当作 Linux 学习环境。

需要掌握：

- 包管理器。
- 文件目录。
- Shell 命令。
- SSH。
- Git。
- 网络排查。

### 第四阶段：会搭建开发环境

目标：能在 WSL 中完成日常开发和 DevOps 工具使用。

需要掌握：

- VS Code Remote - WSL。
- Python 虚拟环境。
- Node.js / nvm。
- Git 配置。
- 环境变量。

### 第五阶段：会做运维和容器实验

目标：能把 WSL 用作云计算运维学习平台。

需要掌握：

- Docker Desktop WSL 集成。
- Docker Compose。
- Nginx、Redis、MySQL 等中间件实验。
- Ansible 控制端实验。
- Kubernetes 本地实验入口。

### 第六阶段：会排查和维护

目标：能处理 WSL 常见问题，保证环境稳定。

需要掌握：

- 网络故障排查。
- DNS 问题处理。
- 文件权限问题处理。
- 备份恢复。
- 磁盘空间清理。

---

## 十八、实践任务

### 任务一：安装并验证 WSL

完成：

- 安装 WSL。
- 安装 Ubuntu 或其他发行版。
- 查看 `wsl -l -v`。
- 进入发行版执行 `whoami`、`uname -a`、`cat /etc/os-release`。

### 任务二：Windows 与 WSL 文件互通

完成：

- 在 Windows 桌面创建一个测试目录。
- 在 WSL 中通过 `/mnt/c/...` 访问该目录。
- 在 WSL 的 `/home` 下创建一个项目目录。
- 在 Windows 资源管理器中通过 `\\wsl$` 访问该目录。

### 任务三：搭建基础开发环境

完成：

- 安装 `git`、`curl`、`wget`、`vim`。
- 配置 Git 用户名和邮箱。
- 创建 SSH key。
- 使用 VS Code Remote - WSL 打开项目。

### 任务四：运行一个 Web 服务

完成：

- 在 WSL 中运行 `python3 -m http.server 8000`。
- 在 Windows 浏览器访问 `http://localhost:8000`。
- 使用 `ss -lntp` 查看监听端口。

### 任务五：Docker 实验

完成：

- 开启 Docker Desktop 的 WSL Integration。
- 在 WSL 中运行 `docker run hello-world`。
- 使用 Docker 运行 Nginx。
- 在 Windows 浏览器访问容器映射端口。

### 任务六：备份与恢复

完成：

- 使用 `wsl --export` 导出发行版。
- 使用 `wsl --import` 导入为一个新发行版。
- 验证新发行版可以正常启动。

---

## 十九、学习建议

- WSL 适合作为学习和实验环境，但不要把它完全等同于生产 Linux 服务器。
- Linux 项目尽量放在 WSL 内部文件系统，性能和权限体验更好。
- 重要环境变更前先导出备份。
- 做 Docker 和云原生实验时优先使用 WSL 2。
- 学习 Windows 运维时结合 PowerShell，学习 Linux 运维时结合 WSL。
- 记录每次环境配置步骤，方便重装或迁移时快速恢复。

---

## 二十、总结

WSL 的学习主线可以概括为：

```text
安装启动 -> 发行版管理 -> 文件互通 -> 网络访问 -> 开发环境 -> Docker 集成 -> 备份迁移 -> 故障排查
```

对于云计算运维学习来说，WSL 的价值在于：你可以在 Windows 上快速获得一个 Linux 实验环境，并把 PowerShell、Docker、VS Code、Git、Linux 命令和运维工具串联起来。

掌握 WSL 后，Windows 不再只是桌面系统，也可以成为一台很顺手的开发和运维学习工作站。
