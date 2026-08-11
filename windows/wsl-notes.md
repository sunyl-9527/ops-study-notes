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
| 网络模型 | 与 Windows 更接近 | 有独立虚拟网络（可选镜像模式） |
| 推荐程度 | 特殊场景使用 | 日常推荐 |

建议：

- 新学习者优先使用 `WSL 2`。
- Docker、Kubernetes、云原生实验优先使用 `WSL 2`。
- 如果大量操作 Windows 文件系统，可以关注 WSL 2 与 Windows 目录之间的性能差异（见「文件互通」一节的性能提示）。

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

### 3. 保持 WSL 本体更新

WSL 引擎本身（内核、`wsl.exe`）会独立于发行版更新，建议定期检查：

```powershell
wsl --update
wsl --version
```

如果遇到内核相关的异常行为或新特性（如镜像网络模式）不生效，优先确认 WSL 版本是否是最新。

### 4. 查看已安装发行版

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

### 5. 设置默认 WSL 版本

```powershell
wsl --set-default-version 2
```

### 6. 修改某个发行版的 WSL 版本

```powershell
wsl --set-version Ubuntu 2
```

### 7. 设置默认发行版

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

在 WSL 中直接执行 Linux 命令（这一步在 Windows 侧的 PowerShell / CMD 中执行，不是在 WSL 内部）：

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

### 4. 导出、导入与迁移

导出、导入、跨磁盘迁移和 VHD 相关命令统一放在「十六、备份、迁移与恢复」一节说明，避免重复。日常只需记住：**注销或重装前，先导出一份备份**。

---

## 五、Windows 与 Linux 文件互通

### 1. 在 WSL 中访问 Windows 文件

Windows 磁盘会挂载到 `/mnt` 下。

```bash
cd /mnt/c/Projects
ls
```

常见路径：

```text
C:\Projects
```

在 WSL 中对应：

```text
/mnt/c/Projects
```

### 2. 在 Windows 中访问 WSL 文件

现在推荐使用 `\\wsl.localhost\`：

```text
\\wsl.localhost\Ubuntu
```

旧路径 `\\wsl$` 仍然可用（作为兼容别名保留），但新文档和新版本资源管理器地址栏建议统一使用 `\\wsl.localhost\`：

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
C:\Projects\ops-study-notes
```

实践建议：

- Linux 工具频繁读写的项目，优先放在 WSL 内部文件系统。
- Windows 软件频繁使用的文档和笔记，可以放在 Windows 文件系统。
- 不要在 Windows 和 WSL 中同时用两个工具高频修改同一个文件，避免状态混乱。

### 4. 性能与注意事项

- 跨文件系统访问（WSL 读写 `/mnt/c/...`，或 Windows 读写 `\\wsl.localhost\...`）比在各自原生文件系统内操作慢很多，大量小文件的项目（如 `node_modules`）尤其明显。
- 文件监听（inotify）在 `/mnt` 挂载的 Windows 目录下经常不可靠，webpack、Vite 等工具的热更新在这类路径下可能不生效；需要热更新的前端项目建议放进 WSL 原生文件系统。
- Git 换行符：跨系统协作建议在 WSL 内统一设置 `git config --global core.autocrlf input`，避免因为换行符差异产生大量无意义的 diff。
- `/etc/wsl.conf` 中 `[automount] options=metadata` 可以让 `/mnt` 下的文件支持 Linux 权限位（chmod 生效），否则挂载的 Windows 文件权限固定，某些工具（如 SSH 私钥权限检查）会报错。

---

## 六、常用 WSL 命令速查

| 场景 | 命令 |
|---|---|
| 查看状态 | `wsl --status` |
| 查看版本 | `wsl --version` |
| 更新 WSL 本体 | `wsl --update` |
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
| 查看/管理发行版属性 | `wsl --manage Ubuntu --set-sparse true` |
| 导出备份（tar） | `wsl --export Ubuntu D:\backup\ubuntu.tar` |
| 导出备份（VHD，体积更小） | `wsl --export Ubuntu D:\backup\ubuntu.vhdx --vhd` |
| 导入恢复（tar） | `wsl --import Ubuntu2 D:\WSL\Ubuntu2 D:\backup\ubuntu.tar --version 2` |
| 原地导入已有 VHD | `wsl --import-in-place Ubuntu2 D:\WSL\Ubuntu2\ext4.vhdx` |
| 移动发行版磁盘位置 | `wsl --manage Ubuntu --move D:\WSL\Ubuntu` |
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

推荐优先编辑 WSL 内部的 `/etc/wsl.conf`，这是跨发行版都适用的通用方式：

```ini
[user]
default=username
```

修改后在 Windows 中执行：

```powershell
wsl --terminate Ubuntu
wsl -d Ubuntu
```

较新版本的 WSL（先用 `wsl --version` 确认版本）也提供了不进入发行版即可设置默认用户的命令：

```powershell
wsl --manage Ubuntu --set-default-user username
```

早期文档中常见的 `ubuntu config --default-user username` 依赖官方商店发行版自带的启动器命令，不是所有发行版都提供，不建议作为首选方式。

---

## 八、wsl.conf 配置（发行版级）

WSL 的部分行为可以通过发行版内部的 `/etc/wsl.conf` 配置，作用范围仅限该发行版。

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

- `[boot] systemd=true`：启用 systemd，详见「systemd 与服务管理」一节。
- `[user] default=username`：设置默认登录用户。
- `[automount] enabled=true`：自动挂载 Windows 磁盘。
- `[automount] root=/mnt/`：设置挂载根路径。
- `[automount] options=metadata`：让 `/mnt` 下文件支持 Linux 权限位，见「文件互通」中的性能提示。
- `[interop] enabled=true`：允许 WSL 调用 Windows 程序。
- `appendWindowsPath=true`：把 Windows PATH 加入 WSL。

修改 `/etc/wsl.conf` 后，需要关闭并重新启动发行版：

```powershell
wsl --shutdown
wsl -d Ubuntu
```

---

## 九、.wslconfig 配置（宿主机级）

`/etc/wsl.conf` 只影响单个发行版，而 `.wslconfig` 影响的是 WSL 2 虚拟机本身（内存、CPU、网络模式等），对所有发行版生效。

文件位置（Windows 侧）：

```text
%UserProfile%\.wslconfig
```

常见配置示例：

```ini
[wsl2]
memory=6GB
processors=4
swap=2GB
localhostForwarding=true
networkingMode=mirrored
dnsTunneling=true
autoMemoryReclaim=gradual
sparseVhd=true
```

常见配置说明：

- `memory` / `processors` / `swap`：限制 WSL 2 虚拟机可使用的内存、CPU 核心数和交换空间上限。不设置时 WSL 默认会占用较多宿主机内存，长期开发机建议显式限制，避免和其他程序抢内存。
- `networkingMode=mirrored`：启用镜像网络模式（需要较新版本 Windows 11 和 WSL），让 WSL 直接共享 Windows 主机的网络接口，`localhost` 在两个方向都能互通，也能更好地兼容 VPN。默认模式是 NAT，网络访问方式见「网络访问」一节的说明。
- `dnsTunneling=true`：让 WSL 的 DNS 请求通过 Windows 主机转发，通常能明显改善 VPN 环境下 WSL 内部 DNS 解析失败的问题。
- `autoMemoryReclaim=gradual`：WSL 2 虚拟机在空闲时逐步归还已分配但未使用的内存给 Windows，缓解“内存只增不减”的问题。
- `sparseVhd=true`：让虚拟磁盘文件保持稀疏，删除 WSL 内部文件后磁盘占用能自动收缩，而不是一直增长。

修改 `.wslconfig` 后，需要完全关闭 WSL 才能生效：

```powershell
wsl --shutdown
```

---

## 十、systemd 与服务管理

### 1. 启用 systemd

在「wsl.conf 配置」一节中把 `[boot] systemd=true` 写入对应发行版的 `/etc/wsl.conf`，然后执行 `wsl --shutdown` 并重新进入该发行版即可生效。

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

## 十一、网络访问

WSL 2 有两种网络模式，行为差异较大，排查网络问题前先确认当前用的是哪种模式。

### 1. NAT 模式（默认）

这是没有配置 `.wslconfig` 中 `networkingMode` 时的默认模式，WSL 处于一个独立的虚拟子网中。

- **Windows 访问 WSL**：默认开启 `localhostForwarding`，Windows 上直接访问 `http://localhost:端口` 就能打到 WSL 里监听的服务，无需额外配置。
- **WSL 访问 Windows**：`localhost` 通常不指向 Windows 主机，需要用 WSL 侧看到的网关地址，例如：

```bash
ip route show default | awk '{print $3}'
# 或
cat /etc/resolv.conf   # nameserver 字段在多数默认配置下等于宿主机地址
```

`/etc/resolv.conf` 里的 `nameserver` 通常等于宿主机在 NAT 子网中的地址，可以用来访问 Windows 上监听在该地址的服务；但如果发行版设置了自定义 DNS（`generateResolvConf=false` 或手动编辑过该文件），这个值就不再可靠，优先用 `ip route` 拿网关地址。

### 2. 镜像网络模式（mirrored，较新版本 WSL 支持）

在 `.wslconfig` 中设置 `networkingMode=mirrored` 后，WSL 直接共享 Windows 主机的网络接口：

- `localhost` 在 Windows 和 WSL 之间双向互通，不用再区分方向。
- 对 VPN、防火墙的兼容性通常更好，尤其是企业 VPN 环境下 WSL 网络异常的问题，很多可以通过切换到镜像模式解决。
- 需要确认 Windows 版本和 WSL 版本支持该特性（`wsl --update` 到最新版本）。

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

- 先确认当前网络模式（NAT 还是 mirrored），按对应模式的访问方式验证。
- 检查 Windows 网络是否正常。
- 执行 `wsl --shutdown` 后重新启动。
- VPN 环境下优先尝试 `.wslconfig` 里的 `networkingMode=mirrored` 和 `dnsTunneling=true`。
- 检查代理环境变量：`http_proxy`、`https_proxy`。
- 检查服务是否监听在 `0.0.0.0` 或正确地址上。

---

## 十二、在 WSL 中使用 Windows 程序

WSL 可以直接调用 Windows 程序。

### 1. 打开资源管理器

```bash
explorer.exe .
```

### 2. 使用 Windows VS Code

```bash
code .
```

详细的插件配置和使用建议见「WSL 与 VS Code」一节。

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
wslpath 'C:\Projects'
```

---

## 十三、开发环境实践

### 1. Git

```bash
git --version
git config --global user.name "your-name"
git config --global user.email "your-email@example.com"
git config --global core.autocrlf input
```

建议：

- Windows Git 和 WSL Git 可以同时存在。
- 同一个仓库尽量固定使用一种环境操作。
- `core.autocrlf input` 可以避免 Windows/Linux 换行符差异造成大量无意义的 diff，详见「文件互通」中的性能提示。

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

## 十四、WSL 与 Docker

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

不建议在同一个发行版里再额外安装、启动独立的 Docker Engine（`dockerd`），这会和 Docker Desktop 提供的 daemon 争抢端口和资源，出现难以定位的连接异常。二选一即可：要么用 Docker Desktop 的 WSL 集成，要么在发行版内自行安装 Docker Engine（不装 Docker Desktop）。

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

## 十五、WSL 与 VS Code

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

## 十六、备份、迁移与恢复

### 1. 导出备份

传统 tar 格式，通用但体积较大：

```powershell
wsl --export Ubuntu D:\backup\ubuntu-2026-08-12.tar
```

较新版本支持直接导出为 VHD，体积更小、恢复更快：

```powershell
wsl --export Ubuntu D:\backup\ubuntu-2026-08-12.vhdx --vhd
```

### 2. 导入恢复

从 tar 导入：

```powershell
wsl --import Ubuntu-Restore D:\WSL\Ubuntu-Restore D:\backup\ubuntu-2026-08-12.tar --version 2
```

如果已经有现成的 VHD 磁盘文件（例如从旧机器直接复制过来的），可以原地导入，不需要重新解压：

```powershell
wsl --import-in-place Ubuntu-Restore D:\WSL\Ubuntu-Restore\ext4.vhdx
```

### 3. 迁移到其他磁盘

较新版本可以直接移动发行版的磁盘位置，不需要导出再导入：

```powershell
wsl --manage Ubuntu --move D:\WSL\Ubuntu
```

如果 WSL 版本不支持 `--move`，回退到传统方式：

1. `wsl --export` 导出原发行版。
2. `wsl --unregister` 删除原发行版。
3. `wsl --import` 导入到新路径。
4. 重新设置默认用户和默认发行版。

### 4. 磁盘瘦身

启用稀疏 VHD 后，磁盘占用能随着内部文件删除自动收缩：

```powershell
wsl --manage Ubuntu --set-sparse true
```

也可以在 `.wslconfig` 中全局设置 `sparseVhd=true`，对新导入的发行版默认生效。

### 5. 备份建议

- 重要环境改动前先导出备份。
- 做大型实验前先保留一个干净快照。
- 不要只备份 Windows 目录，WSL 内部的 `/home` 也需要备份。
- 涉及密钥、凭据、token 的备份文件要妥善保存。

---

## 十七、常见故障排查

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
- VPN 环境优先尝试 `.wslconfig` 中的 `dnsTunneling=true` 或 `networkingMode=mirrored`。
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

如果使用 Docker Desktop，也可以在 Docker Desktop 中清理镜像、容器和卷；长期占用持续增长，也可以配合「备份、迁移与恢复」一节的稀疏 VHD 设置。

---

## 十八、学习路线

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
- 导出与导入（含 VHD 方式）。
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
- Kubernetes 本地实验入口（规划中，暂无对应笔记）。

### 第六阶段：会排查和维护

目标：能处理 WSL 常见问题，保证环境稳定。

需要掌握：

- 网络故障排查（含 NAT / 镜像模式区分）。
- `.wslconfig` 与 `wsl.conf` 调优。
- 文件权限问题处理。
- 备份恢复与磁盘瘦身。

---

## 十九、实践任务

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
- 在 Windows 资源管理器中通过 `\\wsl.localhost\` 访问该目录。

### 任务三：搭建基础开发环境

完成：

- 安装 `git`、`curl`、`wget`、`vim`。
- 配置 Git 用户名和邮箱。
- 创建 SSH key。
- 使用 VS Code Remote - WSL 打开项目。

### 任务四：运行一个 Web 服务并验证网络模式

完成：

- 在 WSL 中运行 `python3 -m http.server 8000`。
- 在 Windows 浏览器访问 `http://localhost:8000`。
- 使用 `ss -lntp` 查看监听端口。
- 确认当前是 NAT 模式还是镜像模式，并验证对应的 Windows↔WSL 访问方式。

### 任务五：Docker 实验

完成：

- 开启 Docker Desktop 的 WSL Integration。
- 在 WSL 中运行 `docker run hello-world`。
- 使用 Docker 运行 Nginx。
- 在 Windows 浏览器访问容器映射端口。

### 任务六：备份与恢复

完成：

- 使用 `wsl --export` 导出发行版（tar 或 VHD 格式）。
- 使用 `wsl --import` 或 `wsl --import-in-place` 导入为一个新发行版。
- 验证新发行版可以正常启动。

---

## 二十、学习建议

- WSL 适合作为学习和实验环境，但不要把它完全等同于生产 Linux 服务器。
- Linux 项目尽量放在 WSL 内部文件系统，性能和权限体验更好。
- 重要环境变更前先导出备份。
- 做 Docker 和云原生实验时优先使用 WSL 2。
- 学习 Windows 运维时结合 PowerShell，学习 Linux 运维时结合 WSL。
- 记录每次环境配置步骤，方便重装或迁移时快速恢复。
- 长期使用建议同时维护好 `.wslconfig`（宿主机资源与网络）和 `/etc/wsl.conf`（发行版行为），两者作用范围不同，容易混淆。

---

## 二十一、总结

WSL 的学习主线可以概括为：

```text
安装启动 -> 发行版管理 -> 文件互通 -> 网络访问 -> 开发环境 -> Docker 集成 -> 备份迁移 -> 故障排查
```

对于云计算运维学习来说，WSL 的价值在于：你可以在 Windows 上快速获得一个 Linux 实验环境，并把 PowerShell、Docker、VS Code、Git、Linux 命令和运维工具串联起来。

掌握 WSL 后，Windows 不再只是桌面系统，也可以成为一台很顺手的开发和运维学习工作站。
