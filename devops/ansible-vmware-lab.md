# VMware + Arch Linux 搭建 Ansible 实践环境

本文搭建一套隔离的 Ansible 学习环境：1 台控制节点管理 4 台被控节点。重点是理解 inventory、SSH 认证、ad-hoc 命令、playbook 和故障排查。

> 本文管"搭环境"；环境就绪后，Ansible 本身的系统学习见 [ansible-notes.md](./ansible-notes.md)。

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

完成基础实验后，可继续练习 package、service、template、copy、user、firewalld/ufw 等模块，以及 role、handler、tag 和幂等性验证。Ansible 本身的系统学习（模块速查、playbook 语法、变量与模板、roles、错误处理、调试排错）见 [ansible-notes.md](./ansible-notes.md)。

---

## 附录 A：批量创建虚拟机（PowerShell 脚本）

如果有 4 台被控节点需要从零创建，可以用 PowerShell 脚本批量生成 VMX 虚拟机。脚本按需编写，以下为等效的交互式创建命令：

```powershell
$vmBase = "D:\Virtual Machines\ansible"
$iso = "C:\Users\<your-user>\Downloads\archlinux-x86_64.iso"
$vmrun = "C:\Program Files (x86)\VMware\VMware Workstation\vmrun.exe"

1..4 | ForEach-Object {
    $name = "ansible-node$_"
    $path = "$vmBase\$name"
    $vmx = "$path\$name.vmx"
    New-Item -ItemType Directory -Force -Path $path | Out-Null

    # 生成 VMX 配置文件（闭合分隔符 "@ 必须顶格写，否则 PowerShell 会报语法错误）
    @"
.encoding = "UTF-8"
config.version = "8"
virtualHW.version = "18"
memsize = "4096"
numvcpus = "2"
ide1:0.fileName = "$iso"
ide1:0.deviceType = "cdrom-image"
scsi0:0.fileName = "$name.vmdk"
scsi0:0.present = "TRUE"
ide0:0.present = "FALSE"
ethernet0.present = "TRUE"
ethernet0.connectionType = "nat"
ethernet0.virtualDev = "e1000"
ethernet0.addressType = "generated"
displayName = "$name"
guestOS = "archlinux-64"
"@ | Out-File -FilePath $vmx -Encoding utf8

    # 创建对应的空磁盘，再启动虚拟机（首次启动会从挂载的 ISO 引导安装）
    & $vmrun -T ws createvdisk $vmx -adapter lsilogic -size 20GB
    & $vmrun -T ws start $vmx
}
```

- 之前版本在创建 VMX 之前调用 `vmrun stop`、以及用 `VMware.exe -X` 尝试"打开"虚拟机，这两步在 `.vmx` 文件还不存在时没有意义，也不会真正创建磁盘或启动安装，属于脚本错误，已删除并替换为 `vmrun createvdisk` + `vmrun start`。
- PowerShell 的 here-string（`@"..."@`）要求闭合的 `"@`必须顶格写在行首，前面不能有空格或制表符，否则会直接报语法错误；原示例中闭合分隔符前有缩进，无法正常执行。

控制节点 `ansible-ctl` 可用 VMware 图形界面新建，配置保持一致（2C / 4GB / 20GB），网络选择 NAT，ISO 选择同一个 Arch Linux 镜像。

## 附录 B：通过 HTTP 脚本服务器引导安装

在无图形化 TTY 中手动输入大量命令容易出错。可以在宿主机开一个临时 HTTP 服务，让虚拟机从宿主机下载安装脚本。

### 宿主机启动 HTTP 服务

```powershell
cd "<path-to-scripts-directory>"
python -m http.server 8000 --bind 192.168.88.1
```

保持窗口不要关闭，直到所有节点安装完成。确认防火墙允许 `8000/tcp` 入站。

### 被控节点安装流程

每台被控节点从 Arch ISO 启动后，先确认网络可达：

```bash
ping -c 3 archlinux.org
```

然后下载并执行安装脚本（脚本内容根据实际需求编写）：

```bash
curl -fLO http://192.168.88.1:8000/install-managed.sh
chmod +x install-managed.sh
./install-managed.sh <hostname>
```

安装完成后关机，断开 ISO，再从硬盘启动。

### 被控节点初始化（从宿主机远程执行）

被控节点首次启动后，可通过 SSH 远程执行初始化脚本，完成基础包安装、sshd 启用、sudo 配置和静态 IP 设置：

```powershell
# 首次连接前，用 ssh-keyscan 采集并核对主机指纹，而不是直接关闭校验
ssh-keyscan -H <temp-ip> | Out-File -Append -Encoding utf8 "$env:USERPROFILE\.ssh\known_hosts"

# 提前设置好环境变量，避免密码出现在命令行参数和 shell 历史中
$env:LAB_SUDO_PASSWORD = Read-Host -AsSecureString "输入临时 sudo 密码" |
    ForEach-Object { [Runtime.InteropServices.Marshal]::PtrToStringAuto([Runtime.InteropServices.Marshal]::SecureStringToGlobalAllocUnicode($_)) }

ssh -tt <user>@<temp-ip> `
    "curl -fL http://192.168.88.1:8000/bootstrap-managed.sh -o /tmp/bootstrap-managed.sh && " `
  + "sudo -S bash /tmp/bootstrap-managed.sh <hostname> <static-ip> <<< `"`$env:LAB_SUDO_PASSWORD`" && sudo reboot"

Remove-Item Env:\LAB_SUDO_PASSWORD
```

- 不要用 `StrictHostKeyChecking=no` 跳过主机指纹校验，即便是临时实验环境——这会让中间人替换目标主机而不被发现。改用 `ssh-keyscan` 显式采集指纹并写入 `known_hosts`，出现指纹变化时能及时察觉。
- 不要把 `sudo` 密码明文拼进命令行（如 `echo <password> | sudo -S ...`），命令行参数会被记录进 shell 历史文件，同机其他用户也能通过 `ps` 在极短窗口内看到进程参数。改用环境变量或标准输入传递，用完立即清除。生产环境应直接用 SSH 密钥 + 禁用密码登录，彻底不需要传递密码。

初始化后验证 SSH 可达：

```powershell
Test-NetConnection 192.168.88.21 -Port 22
Test-NetConnection 192.168.88.22 -Port 22
Test-NetConnection 192.168.88.23 -Port 22
Test-NetConnection 192.168.88.24 -Port 22
```

### 控制节点安装

控制节点从 Arch ISO 启动后同样使用 HTTP 方式：

```bash
curl -fLO http://192.168.88.1:8000/install-control.sh
chmod +x install-control.sh
./install-control.sh
```

控制节点脚本应自动安装 `ansible`、`sshpass`、`python`、`openssh`、`sudo`、`git`、`vim`、`NetworkManager` 等基础工具。

### 脚本文件清单

实验结束后，保留以下脚本以便重建：

```text
scripts/
├── create-ansible-lab.ps1        # 批量创建 VMX
├── install-managed.sh            # 被控节点 Arch 安装
├── bootstrap-managed-node.sh     # 被控节点初始化
├── install-control.sh            # 控制节点 Arch 安装
├── ansible/
│   ├── ansible.cfg
│   ├── inventory.ini
│   └── ping.yml
```

> 不要将安装脚本中的密码、token 或本地路径提交到 Git。脚本中的敏感值应从环境变量读取或使用占位符。
