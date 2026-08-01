# VMware + Arch Linux 搭建 Ansible 实践环境

本文档用于把 5 台 VMware 虚拟机搭建为一套 Ansible 学习环境：

- 1 台控制节点：`ansible-ctl`
- 4 台被控节点：`ansible-node1`、`ansible-node2`、`ansible-node3`、`ansible-node4`

## 规划

虚拟机统一配置：

- CPU：`2C`
- 内存：`4GB`
- 磁盘：`20GB`
- 系统：`Arch Linux`
- 网络：VMware `VMnet8 NAT`
- ISO：`C:\Users\DELL\Downloads\archlinux-x86_64.iso`

IP 规划：

```text
ansible-ctl    192.168.88.10
ansible-node1  192.168.88.21
ansible-node2  192.168.88.22
ansible-node3  192.168.88.23
ansible-node4  192.168.88.24
gateway        192.168.88.2
host VMnet8    192.168.88.1
```

账号密码：

```text
root/root
ansible/ansible
```

## 1. 准备 VMware NAT 网络

在 VMware Virtual Network Editor 中确认 `VMnet8` 存在，并配置为：

- 类型：`NAT`
- 子网：`192.168.88.0`
- 掩码：`255.255.255.0`
- 网关：`192.168.88.2`
- DHCP：开启

管理员 PowerShell 检查服务：

```powershell
Get-Service -Name "VMnetDHCP","VMware NAT Service"
```

正常状态：

```text
Running  VMnetDHCP
Running  VMware NAT Service
```

如果服务没有启动，管理员 PowerShell 执行：

```powershell
Set-Service -Name "VMnetDHCP" -StartupType Automatic
Set-Service -Name "VMware NAT Service" -StartupType Automatic
Start-Service -Name "VMnetDHCP"
Start-Service -Name "VMware NAT Service"
```

## 2. 创建 4 台被控节点虚拟机

在 Windows PowerShell 中执行：

```powershell
powershell -ExecutionPolicy Bypass -File "C:\Users\DELL\Desktop\work_note\scripts\create-ansible-lab.ps1"
```

虚拟机会创建在：

```text
D:\Virtual Machines\ansible
```

生成内容：

```text
D:\Virtual Machines\ansible\ansible-node1
D:\Virtual Machines\ansible\ansible-node2
D:\Virtual Machines\ansible\ansible-node3
D:\Virtual Machines\ansible\ansible-node4
```

控制节点 `ansible-ctl` 可以用 VMware 图形界面新建，配置保持 `2C / 4GB / 20GB`，网络选择 `NAT`，ISO 选择同一个 Arch Linux 镜像。

## 3. 开启宿主机临时 HTTP 服务

为了避免在无图形化 TTY 里大量复制粘贴，在宿主机开启临时 HTTP 服务，让虚拟机从 `192.168.88.1:8000` 下载安装脚本。

在 Windows PowerShell 中执行：

```powershell
cd "C:\Users\DELL\Desktop\work_note"
python -m http.server 8000 --bind 192.168.88.1
```

保持这个窗口不要关闭，直到 5 台虚拟机都安装完成。

## 4. 安装 4 台被控节点

每台被控节点从 Arch ISO 启动后，先确认网络：

```bash
ping -c 3 archlinux.org
```

安装 `ansible-node1`：

```bash
curl -fLO http://192.168.88.1:8000/i.sh
chmod +x i.sh
./i.sh ansible-node1
```

安装 `ansible-node2`：

```bash
curl -fLO http://192.168.88.1:8000/i.sh
chmod +x i.sh
./i.sh ansible-node2
```

安装 `ansible-node3`：

```bash
curl -fLO http://192.168.88.1:8000/i.sh
chmod +x i.sh
./i.sh ansible-node3
```

安装 `ansible-node4`：

```bash
curl -fLO http://192.168.88.1:8000/i.sh
chmod +x i.sh
./i.sh ansible-node4
```

脚本提示时输入大写：

```text
YES
```

安装完成后关机，断开 ISO，再从硬盘启动。

## 5. 初始化 4 台被控节点

被控节点系统安装完成并启动后，可以从宿主机远程初始化。该步骤会：

- 安装 `python`、`sudo`、`openssh` 等基础包
- 启用 `sshd`
- 配置 `ansible` 用户免密 sudo
- 写入 `/etc/hosts`
- 固定静态 IP 为 `192.168.88.21-24`

当前已使用过的初始化命令如下，可作为重做参考：

```powershell
ssh -tt -o StrictHostKeyChecking=no ansible@192.168.88.128 "curl -fL http://192.168.88.1:8000/scripts/bootstrap-managed-node.sh -o /tmp/bootstrap-managed-node.sh && echo ansible | sudo -S bash /tmp/bootstrap-managed-node.sh ansible-node1 192.168.88.21 && sudo reboot"
ssh -tt -o StrictHostKeyChecking=no ansible@192.168.88.129 "curl -fL http://192.168.88.1:8000/scripts/bootstrap-managed-node.sh -o /tmp/bootstrap-managed-node.sh && echo ansible | sudo -S bash /tmp/bootstrap-managed-node.sh ansible-node2 192.168.88.22 && sudo reboot"
ssh -tt -o StrictHostKeyChecking=no ansible@192.168.88.130 "curl -fL http://192.168.88.1:8000/scripts/bootstrap-managed-node.sh -o /tmp/bootstrap-managed-node.sh && echo ansible | sudo -S bash /tmp/bootstrap-managed-node.sh ansible-node3 192.168.88.23 && sudo reboot"
ssh -tt -o StrictHostKeyChecking=no ansible@192.168.88.131 "curl -fL http://192.168.88.1:8000/scripts/bootstrap-managed-node.sh -o /tmp/bootstrap-managed-node.sh && echo ansible | sudo -S bash /tmp/bootstrap-managed-node.sh ansible-node4 192.168.88.24 && sudo reboot"
```

初始化后验证 SSH：

```powershell
Test-NetConnection 192.168.88.21 -Port 22
Test-NetConnection 192.168.88.22 -Port 22
Test-NetConnection 192.168.88.23 -Port 22
Test-NetConnection 192.168.88.24 -Port 22
```

## 6. 安装控制节点

控制节点 `ansible-ctl` 从 Arch ISO 启动后，确认网络：

```bash
ping -c 3 archlinux.org
```

执行：

```bash
curl -fLO http://192.168.88.1:8000/ctl.sh
chmod +x ctl.sh
./ctl.sh
```

脚本提示时输入大写：

```text
YES
```

控制节点脚本会自动安装：

- `ansible`
- `sshpass`
- `python`
- `openssh`
- `sudo`
- `git`
- `vim`
- `NetworkManager`

安装完成后关机，断开 ISO，再从硬盘启动。

## 7. 在控制节点验证 Ansible

登录控制节点：

```text
user: ansible
password: ansible
```

进入实验目录：

```bash
cd ~/ansible-lab
```

查看清单：

```bash
cat inventory.ini
```

内容应为：

```ini
[managed]
ansible-node1 ansible_host=192.168.88.21
ansible-node2 ansible_host=192.168.88.22
ansible-node3 ansible_host=192.168.88.23
ansible-node4 ansible_host=192.168.88.24

[managed:vars]
ansible_user=ansible
ansible_password=ansible
```

执行连通性测试：

```bash
ansible managed -m ping
```

执行测试 playbook：

```bash
ansible-playbook ping.yml
```
```
echo 'root:V^BAF*sAtTajZjDz' | sudo chpasswd && echo 'ansible:V^BAF*sAtTajZjDz' | sudo chpasswd
```

正常时四台节点都会返回成功，并显示各自主机名和 IP。

## 8. 保留文件说明

必要文件：

```text
scripts/create-ansible-lab.ps1
scripts/arch-install-managed-node.sh
scripts/bootstrap-managed-node.sh
scripts/arch-install-control-node.sh
i.sh
ctl.sh
ansible/ansible.cfg
ansible/inventory.ini
ansible/ping.yml
docs/ansible-vmware-lab.md
```

当前文档已经合并了原来的控制节点和被控节点安装说明。
