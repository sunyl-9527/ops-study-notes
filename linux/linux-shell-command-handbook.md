# Linux Shell 命令手册

> 适用发行版：Ubuntu / Arch Linux  
> 无特殊标注的命令两者通用。  
> 阅读建议：先看目录定位主题，再按“查看信息 -> 修改配置 -> 验证结果”的顺序执行命令。

---

## 目录

1. [系统信息](#1-系统信息)
2. [系统基础设置](#2-系统基础设置)
3. [包管理与软件安装](#3-包管理与软件安装)
4. [网络配置与管理](#4-网络配置与管理)
5. [文件与目录操作](#5-文件与目录操作)
6. [文本处理](#6-文本处理)
7. [用户与权限管理](#7-用户与权限管理)
8. [进程管理](#8-进程管理)
9. [服务管理 systemd](#9-服务管理-systemd)
10. [磁盘与存储](#10-磁盘与存储)
11. [压缩与解压](#11-压缩与解压)
12. [SSH 远程连接](#12-ssh-远程连接)
13. [环境变量](#13-环境变量)
14. [Shell 脚本基础](#14-shell-脚本基础)
15. [实用技巧](#15-实用技巧)
16. [两个发行版主要差异对比](#16-两个发行版主要差异对比)

---

## 1. 系统信息

### 1.1 查看系统与发行版

```bash
# 查看发行版信息（Ubuntu）
lsb_release -a

# 查看发行版信息（通用）
cat /etc/os-release

# 查看内核版本
uname -r

# 查看完整系统信息
uname -a

# 查看主机名
hostname

# 查看系统运行时长
uptime
uptime -p

# 查看当前登录用户
who
w
```

### 1.2 硬件信息

```bash
# CPU 信息
lscpu
cat /proc/cpuinfo

# 内存信息
free -h
cat /proc/meminfo

# 所有硬件概览
lshw -short          # 需安装 lshw
lspci                # PCI 设备
lsusb                # USB 设备

# 块设备（磁盘/分区）
lsblk
lsblk -o NAME,SIZE,TYPE,MOUNTPOINT

# 磁盘详细信息
sudo fdisk -l
sudo parted -l
```

### 1.3 系统资源监控

```bash
# 实时进程与资源监控
top
htop                           # 更友好，需安装

# 磁盘使用
df -h                          # 所有挂载点
df -h /home                    # 指定目录
df -i                          # inode 使用情况

# 目录/文件大小
du -sh /path/to/dir
du -sh *                       # 当前目录每项大小
du -h --max-depth=1 /var       # 一级子目录大小

# 内核日志
dmesg
dmesg -T                       # 带时间戳
dmesg | tail -50

# 系统日志
journalctl -xe
journalctl --since "1 hour ago"
journalctl -b                  # 本次启动的日志
```

---

## 2. 系统基础设置

### 2.1 时区与时间

```bash
# 查看当前时间和时区
timedatectl
timedatectl status

# 列出可用时区
timedatectl list-timezones
timedatectl list-timezones | grep Asia

# 设置时区
sudo timedatectl set-timezone Asia/Shanghai

# 启用 NTP 自动同步
sudo timedatectl set-ntp true

# 手动设置时间
sudo timedatectl set-time "2024-01-01 12:00:00"

# 查看 / 同步硬件时钟
sudo hwclock --show
sudo hwclock --systohc
```

### 2.2 语言与字符集（Locale）

```bash
# 查看当前 locale
locale
locale -a                        # 列出所有可用 locale

# Ubuntu — 生成和配置 locale
sudo locale-gen zh_CN.UTF-8
sudo update-locale LANG=zh_CN.UTF-8

# Arch — 生成 locale
# 1. 编辑 /etc/locale.gen，取消注释 zh_CN.UTF-8 和 en_US.UTF-8
sudo vim /etc/locale.gen
# 2. 生成
sudo locale-gen
# 3. 设置默认
sudo localectl set-locale LANG=zh_CN.UTF-8

# 临时修改（当前 session）
export LANG=en_US.UTF-8
export LC_ALL=en_US.UTF-8
```

### 2.3 主机名与 hosts

```bash
# 查看和设置主机名
hostnamectl
sudo hostnamectl set-hostname myserver

# 编辑本地 DNS 覆写
sudo vim /etc/hosts
# 格式示例：
# 127.0.0.1   localhost
# 192.168.1.10  myserver myserver.local

# 验证解析
ping myserver
getent hosts myserver
```

### 2.4 内核参数（sysctl）

```bash
# 查看所有内核参数
sysctl -a
sysctl -a | grep net.ipv4

# 读取指定参数
sysctl net.ipv4.ip_forward
sysctl vm.swappiness

# 临时修改（重启失效）
sudo sysctl -w net.ipv4.ip_forward=1
sudo sysctl -w vm.swappiness=10

# 永久修改
echo 'net.ipv4.ip_forward=1' | sudo tee -a /etc/sysctl.d/99-custom.conf
echo 'vm.swappiness=10'       | sudo tee -a /etc/sysctl.d/99-custom.conf
sudo sysctl --system           # 重新加载所有配置
```

### 2.5 开机自启配置

```bash
# 设置服务开机自启
sudo systemctl enable sshd
sudo systemctl enable NetworkManager

# 查看所有开机自启服务
systemctl list-unit-files --type=service --state=enabled

# 设置默认启动目标（运行级别）
sudo systemctl set-default multi-user.target   # 无图形（服务器）
sudo systemctl set-default graphical.target    # 图形界面

# 查看当前默认目标
systemctl get-default
```

### 2.6 系统关机与重启

```bash
sudo poweroff                  # 立即关机
sudo reboot                    # 立即重启
sudo shutdown -h now           # 立即关机
sudo shutdown -r now           # 立即重启
sudo shutdown -h +10           # 10 分钟后关机
sudo shutdown -r 23:00         # 23:00 重启
sudo shutdown -c               # 取消定时操作

logout                         # 注销当前用户
```

---

## 3. 包管理与软件安装

### 3.1 Ubuntu — apt

```bash
# ── 更新 ──────────────────────────────────────────
sudo apt update                          # 同步软件源索引
sudo apt upgrade                         # 升级所有包
sudo apt full-upgrade                    # 升级（含依赖变更）
sudo do-release-upgrade                  # 跨大版本升级

# ── 安装 ──────────────────────────────────────────
sudo apt install <package>
sudo apt install <pkg1> <pkg2>
sudo apt install <package>=<version>     # 指定版本
sudo apt install ./local.deb             # 安装本地 deb 包
sudo apt install -y <package>            # 自动确认

# ── 删除 ──────────────────────────────────────────
sudo apt remove <package>                # 删除程序（保留配置）
sudo apt purge <package>                 # 删除程序 + 配置
sudo apt autoremove                      # 删除无用依赖
sudo apt autoremove --purge              # 同上 + 清配置

# ── 搜索 / 查询 ────────────────────────────────────
apt search <keyword>
apt show <package>
apt list --installed
apt list --upgradable
dpkg -l                                  # 所有已安装包
dpkg -L <package>                        # 包安装了哪些文件
dpkg -S /usr/bin/ls                      # 文件属于哪个包

# ── 清理 ─────────────────────────────────────────
sudo apt clean                           # 清除所有缓存
sudo apt autoclean                       # 清除旧版本缓存

# ── 软件源管理 ───────────────────────────────────
cat /etc/apt/sources.list
sudo add-apt-repository ppa:<ppa-name>
sudo add-apt-repository --remove ppa:<ppa-name>
```

### 3.2 Arch Linux — pacman

```bash
# ── 更新 ──────────────────────────────────────────
sudo pacman -Sy                          # 仅同步索引，通常不建议单独使用
sudo pacman -Syu                         # 同步 + 升级全系统
sudo pacman -Syuu                        # 允许降级（镜像不一致时）

# ── 安装 ──────────────────────────────────────────
sudo pacman -S <package>
sudo pacman -S <pkg1> <pkg2>
sudo pacman -U /path/to/pkg.tar.zst      # 安装本地包
sudo pacman -S --needed <package>        # 已安装则跳过

# ── 删除 ──────────────────────────────────────────
sudo pacman -R <package>                 # 删除包
sudo pacman -Rs <package>                # 删除包 + 无用依赖
sudo pacman -Rns <package>               # 删除包 + 依赖 + 配置

# ── 搜索 / 查询 ────────────────────────────────────
pacman -Ss <keyword>                     # 搜索仓库
pacman -Si <package>                     # 仓库包信息
pacman -Q                                # 已安装所有包
pacman -Qs <keyword>                     # 搜索已安装包
pacman -Qi <package>                     # 已安装包详情
pacman -Ql <package>                     # 包安装了哪些文件
pacman -Qo /usr/bin/ls                   # 文件属于哪个包
pacman -Qdt                              # 列出孤立依赖包

# ── 清理 ─────────────────────────────────────────
sudo pacman -Sc                          # 清除旧版本缓存
sudo pacman -Scc                         # 清除所有缓存（慎用）
sudo pacman -Rns $(pacman -Qtdq)         # 删除所有孤立包

# ── 软件源与镜像 ─────────────────────────────────
cat /etc/pacman.conf
sudo vim /etc/pacman.d/mirrorlist

# 用 reflector 自动选最快镜像
sudo reflector --country China --sort rate --save /etc/pacman.d/mirrorlist
```

> 提醒：在 Arch 中单独执行 `pacman -Sy` 后长期不升级，可能造成部分升级问题。日常更推荐直接使用 `pacman -Syu`。

### 3.3 Arch AUR 助手（yay / paru）

```bash
# 安装 yay
sudo pacman -S --needed base-devel git
git clone https://aur.archlinux.org/yay.git
cd yay && makepkg -si

# yay 常用命令
yay -S <package>          # 安装 AUR 包
yay -Syu                  # 更新所有包（含 AUR）
yay -Ss <keyword>         # 搜索 AUR + 官方仓库
yay -Rns <package>        # 删除包
yay -Yc                   # 清理不需要的依赖

# paru 与 yay 命令基本相同
paru -S <package>
paru -Syu
```

### 3.4 从源码编译安装

```bash
# 通用流程
./configure --prefix=/usr/local
make -j$(nproc)
sudo make install

# Ubuntu — 安装编译依赖
sudo apt install build-essential

# Arch — 安装编译依赖
sudo pacman -S base-devel

# 查看已安装到哪里
which <command>
whereis <command>
```

### 3.5 Snap（Ubuntu）

```bash
# Ubuntu 自带 snapd
snap find <keyword>              # 搜索
sudo snap install <package>      # 安装
sudo snap remove <package>       # 删除
snap list                        # 已安装列表
sudo snap refresh                # 更新所有 snap 包
sudo snap refresh <package>      # 更新指定包
```

### 3.6 Flatpak（通用）

```bash
# 安装 flatpak
sudo apt install flatpak         # Ubuntu
sudo pacman -S flatpak           # Arch

# 添加 Flathub 仓库
flatpak remote-add --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo

# 常用命令
flatpak search <keyword>
flatpak install flathub <app.id>
flatpak uninstall <app.id>
flatpak list
flatpak update
```

---

## 4. 网络配置与管理

### 4.1 NetworkManager（nmcli）

```bash
# 查看状态
nmcli
nmcli device status
nmcli connection show
nmcli connection show --active

# 连接 WiFi
nmcli device wifi list
nmcli device wifi connect "SSID" password "密码"
nmcli connection up "连接名"
nmcli connection down "连接名"
nmcli connection delete "连接名"

# 配置静态 IP（有线）
nmcli connection modify "Wired connection 1" \
    ipv4.method manual \
    ipv4.addresses 192.168.1.100/24 \
    ipv4.gateway 192.168.1.1 \
    ipv4.dns "8.8.8.8 114.114.114.114"
nmcli connection up "Wired connection 1"

# 改回 DHCP
nmcli connection modify "Wired connection 1" ipv4.method auto
nmcli connection up "Wired connection 1"

# 重启网络
sudo systemctl restart NetworkManager
```

### 4.2 Netplan（Ubuntu 18.04+）

```bash
# 查看当前配置
cat /etc/netplan/*.yaml

# 静态 IP 配置示例（/etc/netplan/01-netcfg.yaml）
# network:
#   version: 2
#   ethernets:
#     eth0:
#       addresses:
#         - 192.168.1.100/24
#       routes:
#         - to: default
#           via: 192.168.1.1
#       nameservers:
#         addresses: [8.8.8.8, 114.114.114.114]
#       dhcp4: no

# DHCP 配置示例
# network:
#   version: 2
#   ethernets:
#     eth0:
#       dhcp4: true

# 应用配置
sudo netplan apply
sudo netplan try               # 测试，60s 后自动回滚
```

### 4.3 DNS 配置

```bash
# 查看当前 DNS
cat /etc/resolv.conf
nmcli device show | grep DNS
resolvectl status

# Ubuntu — 通过 NetworkManager 设置永久 DNS
sudo nmcli connection modify "连接名" ipv4.dns "8.8.8.8 114.114.114.114"
sudo nmcli connection up "连接名"

# Arch — systemd-resolved
sudo vim /etc/systemd/resolved.conf
# [Resolve]
# DNS=8.8.8.8 114.114.114.114
# FallbackDNS=1.1.1.1
sudo systemctl restart systemd-resolved

# 常用公共 DNS
# 8.8.8.8 / 8.8.4.4        Google DNS
# 1.1.1.1 / 1.0.0.1        Cloudflare
# 114.114.114.114           国内 114 DNS
# 223.5.5.5 / 223.6.6.6    阿里云 DNS
```

### 4.4 网络接口与 IP

```bash
# 查看接口和 IP
ip addr
ip a
ip addr show eth0

# 查看路由
ip route
ip route show

# 查看 ARP 缓存
ip neigh

# 临时设置 IP（重启失效）
sudo ip addr add 192.168.1.100/24 dev eth0
sudo ip addr del 192.168.1.100/24 dev eth0
sudo ip link set eth0 up
sudo ip link set eth0 down

# 添加 / 删除路由
sudo ip route add 10.0.0.0/8 via 192.168.1.1
sudo ip route del 10.0.0.0/8
```

### 4.5 连通性测试

```bash
# ping
ping google.com
ping -c 4 google.com
ping -i 2 google.com
ping -s 1024 google.com

# 追踪路由
traceroute google.com
traceroute -n google.com
mtr google.com                 # 动态版（需安装）

# DNS 查询
nslookup google.com
dig google.com
dig google.com A
dig google.com MX
dig @8.8.8.8 google.com        # 指定 DNS 服务器
host google.com
```

### 4.6 端口与连接监控

```bash
# ss（推荐）
ss -tlnp                       # 监听的 TCP 端口 + 进程
ss -ulnp                       # 监听的 UDP 端口
ss -anp                        # 所有连接
ss -s                          # 统计摘要
ss -tnp | grep :80

# 查找端口被哪个进程占用
sudo ss -tlnp | grep :8080
sudo lsof -i :8080
sudo fuser 8080/tcp

# 测试端口是否开放
nc -zv host 80
nc -zv host 20-25              # 测试端口范围
telnet host 80
```

### 4.7 下载与 HTTP 请求

```bash
# wget
wget https://example.com/file.tar.gz
wget -O output.tar.gz https://example.com/file.tar.gz
wget -c https://example.com/file.tar.gz        # 断点续传
wget --limit-rate=1m https://example.com/file  # 限速

# curl
curl https://example.com                       # GET 请求
curl -O https://example.com/file.tar.gz        # 下载文件
curl -L -o out.tar.gz https://example.com/...  # 跟随跳转
curl -I https://example.com                    # 只看响应头
curl -v https://example.com                    # 调试输出
curl -X POST -d 'key=value' https://api.com    # POST 表单
curl -X POST -H "Content-Type: application/json" \
     -d '{"key":"value"}' https://api.com      # JSON POST
curl -u user:pass https://api.com              # 基础认证
curl -x http://proxy:8080 https://example.com  # 使用代理
```

### 4.8 防火墙

```bash
# ── Ubuntu — ufw ─────────────────────────────────
sudo ufw status verbose
sudo ufw enable
sudo ufw disable
sudo ufw allow 22
sudo ufw allow 22/tcp
sudo ufw allow from 192.168.1.0/24
sudo ufw allow from 192.168.1.0/24 to any port 22
sudo ufw deny 80
sudo ufw delete allow 80
sudo ufw reset

# ── Arch — firewalld ─────────────────────────────
sudo systemctl enable --now firewalld
sudo firewall-cmd --state
sudo firewall-cmd --list-all
sudo firewall-cmd --add-port=80/tcp --permanent
sudo firewall-cmd --remove-port=80/tcp --permanent
sudo firewall-cmd --add-service=http --permanent
sudo firewall-cmd --reload

# ── iptables（低层通用）──────────────────────────
sudo iptables -L -n -v
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT
# 谨慎：下面这条会丢弃其他未放行流量，远程服务器上操作前务必先确认 SSH 已放行
sudo iptables -A INPUT -j DROP
sudo iptables-save > rules.v4
sudo iptables-restore < rules.v4
```

### 4.9 代理设置

```bash
# 临时设置（当前 session）
export http_proxy="http://127.0.0.1:7890"
export https_proxy="http://127.0.0.1:7890"
export all_proxy="socks5://127.0.0.1:7890"
export no_proxy="localhost,127.0.0.1"

# 取消
unset http_proxy https_proxy all_proxy

# 永久（写入 ~/.bashrc 或 ~/.zshrc）
echo 'export http_proxy="http://127.0.0.1:7890"'   >> ~/.bashrc
echo 'export https_proxy="http://127.0.0.1:7890"'  >> ~/.bashrc

# apt 代理（Ubuntu，/etc/apt/apt.conf.d/proxy.conf）
# Acquire::http::Proxy  "http://127.0.0.1:7890/";
# Acquire::https::Proxy "http://127.0.0.1:7890/";

# git 代理
git config --global http.proxy http://127.0.0.1:7890
git config --global --unset http.proxy
```

---

## 5. 文件与目录操作

### 5.1 基本操作

```bash
# 查看目录内容
ls
ls -la                         # 含隐藏文件的长格式
ls -lh                         # 人性化大小
ls -lt                         # 按时间排序
ls -lS                         # 按大小排序

# 目录导航
pwd
cd /path/to/dir
cd ..                          # 上级目录
cd -                           # 上次所在目录
cd ~                           # 家目录

# 创建目录
mkdir dirname
mkdir -p a/b/c                 # 递归创建

# 复制
cp file.txt /dest/
cp -r src/ /dest/              # 递归复制目录
cp -a src/ /dest/              # 保留权限/时间戳

# 移动 / 重命名
mv old.txt new.txt
mv file.txt /dest/
mv -i file.txt /dest/          # 覆盖前确认

# 删除
rm file.txt
rm -f file.txt                 # 强制（不提示）
rm -rf dir/                    # 强制递归删除（危险）
rmdir emptydir                 # 删除空目录

# 创建文件
touch file.txt

# 链接
ln source link_name            # 硬链接
ln -s /path/to/target link     # 软链接
readlink -f link               # 查看软链接真实路径
```

### 5.2 查找文件

```bash
# find
find /path -name "*.log"
find /path -iname "*.log"      # 忽略大小写
find /path -type f             # 只找文件
find /path -type d             # 只找目录
find /path -size +100M
find /path -mtime -7           # 7 天内修改
find /path -mtime +30          # 30 天前修改
find /path -newer ref.txt
find /path -perm 644
find /path -user username
find /path -name "*.tmp" -delete
find /path -name "*.log" -exec ls -lh {} \;
find . -name "*.py" | xargs grep "keyword"

# locate（更快，基于索引库）
sudo updatedb
locate filename
```

### 5.3 查看文件内容

```bash
cat file.txt
cat -n file.txt                # 带行号
less file.txt                  # 分页（推荐，q 退出）
head -n 20 file.txt            # 前 20 行
tail -n 20 file.txt            # 后 20 行
tail -f /var/log/syslog        # 实时追踪
tail -F /var/log/syslog        # 文件轮转后自动重开

# 统计
wc -l file.txt                 # 行数
wc -w file.txt                 # 单词数
wc -c file.txt                 # 字节数

# 查看文件类型
file filename
```

---

## 6. 文本处理

### 6.1 grep — 搜索内容

```bash
grep "keyword" file.txt
grep -i "keyword" file.txt     # 忽略大小写
grep -n "keyword" file.txt     # 显示行号
grep -v "keyword" file.txt     # 反向匹配
grep -c "keyword" file.txt     # 统计匹配行数
grep -l "keyword" *.txt        # 只列文件名
grep -r "keyword" /path/
grep -rn "keyword" /path/
grep -E "regex" file.txt       # 扩展正则
grep -A 3 "keyword" file.txt   # 匹配行 + 后 3 行
grep -B 3 "keyword" file.txt   # 匹配行 + 前 3 行
grep -C 3 "keyword" file.txt   # 前后各 3 行
grep --include="*.py" -r "keyword" .
```

### 6.2 sed — 流编辑器

```bash
# 替换
sed 's/old/new/' file.txt              # 替换每行第一个
sed 's/old/new/g' file.txt             # 替换所有
sed -i 's/old/new/g' file.txt          # 直接修改文件
sed -i.bak 's/old/new/g' file.txt      # 修改前备份

# 删除行
sed '5d' file.txt
sed '/pattern/d' file.txt
sed '1,3d' file.txt

# 打印指定行
sed -n '5p' file.txt
sed -n '5,10p' file.txt
sed -n '/pattern/p' file.txt
```

### 6.3 awk — 文本分析

```bash
awk '{print $1}' file.txt              # 第 1 列
awk '{print $1, $3}' file.txt
awk '{print NR, $0}' file.txt          # 带行号
awk -F: '{print $1}' /etc/passwd       # 指定分隔符
awk 'NR==5' file.txt                   # 第 5 行
awk '/pattern/' file.txt
awk '$3 > 100' file.txt
awk '{sum += $1} END {print sum}' file # 求和

# 实用示例
awk -F: '{print $1, $7}' /etc/passwd
df -h | awk 'NR>1 {print $1, $5}'
```

### 6.4 sort / uniq / cut / tr

```bash
# sort
sort file.txt
sort -n file.txt               # 数字排序
sort -r file.txt               # 反向
sort -k2 file.txt              # 按第 2 列
sort -u file.txt               # 排序并去重

# uniq（需先排序）
sort file.txt | uniq
sort file.txt | uniq -c        # 统计重复次数
sort file.txt | uniq -d        # 只显示重复行

# cut
cut -d: -f1 /etc/passwd        # 按 : 分割取第 1 列
cut -d, -f1,3 file.csv
cut -c1-10 file.txt            # 取前 10 个字符

# tr
tr 'a-z' 'A-Z' < file.txt     # 转大写
tr -d '\n' < file.txt          # 删除换行
tr -s ' ' < file.txt           # 压缩连续空格
```

---

## 7. 用户与权限管理

### 7.1 用户管理

```bash
# 查看用户
whoami
id
id username
cat /etc/passwd

# 切换用户
su - username                  # 切换（加载环境）
sudo su -                      # 切换到 root

# 创建用户
sudo useradd -m username                           # 创建 + 家目录
sudo useradd -m -s /bin/bash username              # 指定 shell
sudo useradd -m -G sudo,docker username            # Ubuntu，附加组
sudo useradd -m -G wheel,docker username           # Arch，附加组

# 修改用户
sudo usermod -s /bin/zsh username
sudo usermod -aG sudo username                     # Ubuntu 加入 sudo 组
sudo usermod -aG wheel username                    # Arch 加入 wheel 组
sudo usermod -l newname oldname                    # 重命名
sudo usermod -d /new/home -m username              # 修改家目录

# 删除用户
sudo userdel username
sudo userdel -r username                           # 含家目录

# 密码管理
sudo passwd username
passwd                         # 修改当前用户密码
sudo passwd -l username        # 锁定账户
sudo passwd -u username        # 解锁账户
sudo chage -l username         # 查看密码有效期
```

### 7.2 组管理

```bash
cat /etc/group
groups
groups username

sudo groupadd groupname
sudo groupdel groupname
sudo gpasswd -a username groupname     # 加入组
sudo gpasswd -d username groupname     # 移出组
sudo groupmod -n newname oldname       # 重命名组
```

### 7.3 权限管理

```bash
# 查看权限（格式：类型 所有者rwx 组rwx 其他rwx）
ls -l file.txt

# chmod — 数字方式
chmod 644 file.txt             # rw-r--r--
chmod 755 file.txt             # rwxr-xr-x
chmod 700 dir/
chmod -R 755 dir/              # 递归

# chmod — 符号方式
chmod +x script.sh
chmod u+x script.sh
chmod go-w file.txt
chmod u=rwx,g=rx,o=r file.txt

# chown — 修改所有者
sudo chown user file.txt
sudo chown user:group file.txt
sudo chown -R user:group dir/

# 特殊权限
chmod u+s /usr/bin/program     # SetUID
chmod g+s dir/                 # SetGID
chmod +t /tmp                  # Sticky Bit

# sudo 配置
sudo visudo                    # 安全编辑 sudoers
sudo -l                        # 查看当前用户 sudo 权限
```

---

## 8. 进程管理

### 8.1 查看进程

```bash
ps aux                         # 所有进程
ps -ef
ps aux | grep nginx
ps --forest                    # 树形显示

top
htop                           # 需安装

pstree
pstree -p                      # 显示 PID

pidof nginx
pgrep nginx
pgrep -l nginx
pgrep -a nginx

lsof -p <PID>                  # 进程打开的文件
```

### 8.2 控制进程

```bash
# 发送信号
kill <PID>                     # SIGTERM（优雅终止）
kill -9 <PID>                  # SIGKILL（强制终止）
kill -HUP <PID>                # 重载配置
kill -l                        # 列出所有信号

killall nginx
killall -9 nginx
pkill nginx
pkill -u username              # 终止某用户所有进程

# 前后台管理
command &                      # 后台运行
Ctrl + Z                       # 挂起当前进程
bg %1                          # 放到后台继续
fg %1                          # 调到前台
jobs -l                        # 查看后台任务

# 持久运行（退出终端后继续）
nohup command &
nohup command > output.log 2>&1 &

# tmux（更推荐的持久方案）
tmux new -s session_name
tmux ls
tmux attach -t session_name
# Ctrl+B D 分离会话
```

### 8.3 进程优先级

```bash
# nice 值：-20（最高）到 19（最低）
nice -n 10 command
sudo nice -n -5 command

# 修改运行中进程的优先级
sudo renice -n 5 -p <PID>
sudo renice -n 5 -u username
```

---

## 9. 服务管理 systemd

### 9.1 服务控制

```bash
sudo systemctl start nginx
sudo systemctl stop nginx
sudo systemctl restart nginx
sudo systemctl reload nginx         # 重载配置（不中断）
sudo systemctl enable nginx         # 开机自启
sudo systemctl disable nginx        # 取消自启
sudo systemctl enable --now nginx   # 启用 + 立即启动
sudo systemctl mask nginx           # 屏蔽服务
sudo systemctl unmask nginx
```

### 9.2 查看状态

```bash
systemctl status nginx
systemctl is-active nginx
systemctl is-enabled nginx

systemctl list-units --type=service --state=running
systemctl list-units --type=service --state=failed
systemctl list-unit-files --type=service

systemctl list-dependencies nginx
```

### 9.3 日志管理 journalctl

```bash
journalctl -u nginx                        # 服务日志
journalctl -u nginx -f                     # 实时跟踪
journalctl -u nginx -n 100                 # 最后 100 行
journalctl -u nginx --since "1 hour ago"
journalctl -u nginx --since "2024-01-01"
journalctl -b                              # 本次启动的日志
journalctl -b -1                           # 上次启动的日志
journalctl -p err                          # 只看错误
journalctl --disk-usage
sudo journalctl --vacuum-time=7d           # 清理 7 天前的日志
sudo journalctl --vacuum-size=500M
```

### 9.4 定时任务

```bash
# crontab
crontab -e                     # 编辑
crontab -l                     # 查看
crontab -r                     # 删除

# 时间格式：分 时 日 月 周
# 每天凌晨 2 点备份
0 2 * * * /home/user/backup.sh

# 每 5 分钟执行
*/5 * * * * /path/to/script.sh

# 每周一早上 8 点
0 8 * * 1 /path/to/script.sh

# 查看系统定时器
systemctl list-timers --all
```

---

## 10. 磁盘与存储

### 10.1 查看磁盘

```bash
lsblk
lsblk -f                       # 含文件系统和 UUID
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINT

df -h
df -h /home
df -i                          # inode 使用情况

sudo fdisk -l
sudo parted -l
sudo blkid                     # 查看设备 UUID
```

### 10.2 分区操作

```bash
# fdisk（MBR）
sudo fdisk /dev/sdb
# n 新建，d 删除，p 查看，w 写入，q 退出

# parted（MBR + GPT）
sudo parted /dev/sdb
# mklabel gpt, mkpart, print, rm, quit

# gdisk（GPT 专用）
sudo gdisk /dev/sdb
```

### 10.3 格式化

```bash
sudo mkfs.ext4 /dev/sdb1
sudo mkfs.xfs /dev/sdb1
sudo mkfs.btrfs /dev/sdb1
sudo mkswap /dev/sdb2          # swap 分区
```

### 10.4 挂载与卸载

```bash
# 挂载
sudo mount /dev/sdb1 /mnt/data
sudo mount -t ext4 /dev/sdb1 /mnt/data
sudo mount -o ro /dev/sdb1 /mnt/data   # 只读

# 卸载
sudo umount /mnt/data
sudo umount -l /mnt/data               # 懒卸载

# 查看挂载
mount | column -t
findmnt

# 永久挂载（/etc/fstab）
# UUID=xxx /mnt/data ext4 defaults 0 2

# swap
sudo swapon /dev/sdb2
sudo swapoff /dev/sdb2
swapon --show
```

### 10.5 磁盘健康检测

```bash
sudo smartctl -a /dev/sda              # SMART 信息
sudo smartctl -t short /dev/sda        # 短自检
sudo fsck /dev/sdb1                    # 文件系统检查（先卸载）
sudo e2fsck -f /dev/sdb1               # ext 文件系统检查
```

---

## 11. 压缩与解压

```bash
# tar.gz
tar -czvf archive.tar.gz dir/          # 压缩
tar -xzvf archive.tar.gz               # 解压
tar -xzvf archive.tar.gz -C /target/   # 解压到指定目录
tar -tzvf archive.tar.gz               # 查看内容

# tar.bz2（压缩率更高）
tar -cjvf archive.tar.bz2 dir/
tar -xjvf archive.tar.bz2

# tar.xz（压缩率最高）
tar -cJvf archive.tar.xz dir/
tar -xJvf archive.tar.xz

# tar 通用解压（自动识别格式）
tar -xvf archive.tar.gz
tar -xvf archive.tar.bz2
tar -xvf archive.tar.xz

# zip
zip -r archive.zip dir/
zip -e archive.zip file.txt            # 加密
unzip archive.zip
unzip archive.zip -d /target/
unzip -l archive.zip                   # 查看内容

# gzip / bzip2 / xz（单文件）
gzip file.txt                          # 生成 file.txt.gz
gzip -k file.txt                       # 保留原文件
gzip -d file.txt.gz                    # 解压
bzip2 file.txt
bunzip2 file.txt.bz2
xz file.txt
xz -d file.txt.xz

# 7zip
7z a archive.7z dir/
7z x archive.7z
7z l archive.7z
```

---

## 12. SSH 远程连接

### 12.1 基本连接

```bash
ssh user@host
ssh user@192.168.1.100
ssh -p 2222 user@host
ssh -i ~/.ssh/id_ed25519 user@host

# SSH 配置文件（~/.ssh/config）
# Host myserver
#   HostName 192.168.1.100
#   User myuser
#   Port 2222
#   IdentityFile ~/.ssh/id_ed25519

ssh myserver                   # 使用配置连接

# SSH 隧道（本地端口转发）
ssh -N -L 8080:localhost:80 user@host
```

### 12.2 密钥管理

```bash
# 生成密钥对
ssh-keygen -t ed25519 -C "email"        # 推荐
ssh-keygen -t rsa -b 4096 -C "email"    # 兼容性好

# 复制公钥到远程（免密登录）
ssh-copy-id user@host
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@host

# 查看已有密钥
ls -la ~/.ssh/
cat ~/.ssh/id_ed25519.pub

# ssh-agent
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
ssh-add -l                     # 查看已加载密钥
```

### 12.3 文件传输

```bash
# scp
scp file.txt user@host:/remote/path/
scp -r dir/ user@host:/remote/path/
scp user@host:/remote/file.txt ./
scp -P 2222 file.txt user@host:/path/

# rsync（增量同步，推荐）
rsync -avz src/ user@host:/dest/
rsync -avz user@host:/src/ ./dest/
rsync -avz --delete src/ user@host:/dest/      # 删除目标多余文件
rsync -avz --exclude '*.log' src/ user@host:/
rsync -avz -e "ssh -p 2222" src/ user@host:/   # 指定端口
rsync --progress -avz src/ user@host:/dest/    # 显示进度
```

---

## 13. 环境变量

```bash
# 查看
env
printenv
echo $PATH
echo $HOME
echo $USER
echo $SHELL

# 临时设置（当前 session）
export MY_VAR="hello"
MY_VAR="hello" command         # 只为这条命令设置

# 取消变量
unset MY_VAR

# 永久设置（当前用户）
echo 'export MY_VAR="hello"' >> ~/.bashrc   # bash
echo 'export MY_VAR="hello"' >> ~/.zshrc    # zsh
source ~/.bashrc               # 立即生效
-
# 系统级（所有用户）
sudo vim /etc/environment

# PATH 管理
export PATH="$PATH:/new/path"                   # 追加
export PATH="/new/path:$PATH"                   # 前置（优先级更高）
echo 'export PATH="$PATH:/new/path"' >> ~/.bashrc
```

---

## 14. Shell 脚本基础

### 14.1 变量与字符串

```bash
#!/bin/bash

NAME="World"
echo "Hello, $NAME"
echo "Hello, ${NAME}!"

# 获取命令输出
today=$(date +%Y-%m-%d)
file_count=$(ls -1 | wc -l)

# 字符串操作
str="Hello World"
echo ${#str}                   # 字符串长度
echo ${str:0:5}                # 截取：Hello
echo ${str/World/Linux}        # 替换：Hello Linux
echo ${str,,}                  # 转小写
echo ${str^^}                  # 转大写

# 数组
fruits=("apple" "banana" "cherry")
echo ${fruits[0]}
echo ${fruits[@]}              # 所有元素
echo ${#fruits[@]}             # 长度
```

### 14.2 条件判断

```bash
# 常用判断条件
# 字符串：== != -z（空）-n（非空）
# 数字：  -eq -ne -lt -le -gt -ge
# 文件：  -f（文件）-d（目录）-e（存在）-r -w -x（权限）

if [ -f "/etc/passwd" ]; then
    echo "文件存在"
elif [ -d "/etc" ]; then
    echo "是目录"
else
    echo "不存在"
fi

# [[ 双括号（支持正则，推荐）
if [[ "$str" =~ ^[0-9]+$ ]]; then echo "纯数字"; fi
if [[ "$str" == *.txt ]]; then echo "txt 文件"; fi
```

### 14.3 循环

```bash
# for 循环
for i in {1..10}; do echo $i; done

for file in *.txt; do
    echo "Processing: $file"
done

for ((i=0; i<10; i++)); do echo $i; done

# while 循环
count=0
while [ $count -lt 5 ]; do
    echo "Count: $count"
    ((count++))
done

# 逐行读取文件
while IFS= read -r line; do
    echo "$line"
done < file.txt
```

### 14.4 函数

```bash
greet() {
    local name="$1"            # 局部变量
    echo "Hello, $name!"
}

greet "World"

# 通过 echo + $() 返回值
get_date() { echo $(date +%Y-%m-%d); }
today=$(get_date)
```

### 14.5 常用脚本模式

```bash
# 检查参数
if [ $# -ne 2 ]; then
    echo "用法: $0 <param1> <param2>"
    exit 1
fi

# 严格模式
set -euo pipefail

# 检查命令是否存在
if ! command -v git &> /dev/null; then
    echo "git 未安装"
    exit 1
fi

# 日志函数
log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"; }

# 清理陷阱
trap "rm -f /tmp/tmpfile; exit" EXIT INT TERM
```

---

## 15. 实用技巧

### 15.1 命令历史

```bash
history
history 20
history | grep apt
!123                           # 执行第 123 条
!!                             # 重复上一条
sudo !!                        # 以 sudo 重执行
!apt                           # 执行最近以 apt 开头的命令
Ctrl + R                       # 反向搜索历史

history -c                     # 清空当前 shell 历史，执行前确认是否真的需要
```

### 15.2 输入输出重定向

```bash
command > file.txt             # 覆盖写入 stdout
command >> file.txt            # 追加
command 2> error.log           # 只重定向 stderr
command > out.txt 2>&1         # stdout + stderr
command &> out.txt             # 同上（bash 简写）
command > /dev/null 2>&1       # 丢弃所有输出

command | tee output.txt       # 输出到终端 + 文件
command | tee -a output.txt    # 追加模式
```

### 15.3 别名

```bash
alias ll='ls -la'
alias ..='cd ..'
alias grep='grep --color=auto'

alias                          # 查看所有别名
unalias ll                     # 取消别名

# 永久别名
echo "alias ll='ls -la'" >> ~/.bashrc
```

### 15.4 实用组合命令

```bash
# 查看最大的 10 个文件
find / -type f -printf '%s %p\n' 2>/dev/null | sort -rn | head -10

# 实时监控目录文件变化
watch -n 2 ls -la /path/

# 批量创建文件
touch file_{1..10}.txt

# 查找并删除 30 天前的 .log 文件
find /var/log -name "*.log" -mtime +30 -delete

# 去掉空行和注释行
grep -v '^\s*#' config.txt | grep -v '^\s*$'

# 递归替换所有文件中的字符串
find . -type f -name "*.txt" -exec sed -i 's/old/new/g' {} +

# 最常用的 10 条命令
history | awk '{print $2}' | sort | uniq -c | sort -rn | head -10

# 监控命令每 2 秒刷新
watch -n 2 "df -h && free -h"

# 提取 IP 地址
grep -oE '([0-9]{1,3}\.){3}[0-9]{1,3}' file.txt
```

### 15.5 快捷键

| 快捷键 | 功能 |
|--------|------|
| `Ctrl + C` | 终止当前命令 |
| `Ctrl + Z` | 挂起当前命令 |
| `Ctrl + D` | 退出当前 shell |
| `Ctrl + L` | 清屏 |
| `Ctrl + A` | 光标移到行首 |
| `Ctrl + E` | 光标移到行尾 |
| `Ctrl + U` | 删除光标到行首 |
| `Ctrl + K` | 删除光标到行尾 |
| `Ctrl + W` | 删除光标前一个单词 |
| `Ctrl + R` | 反向搜索命令历史 |
| `Alt + F` | 光标前移一个单词 |
| `Alt + B` | 光标后移一个单词 |
| `Tab` | 自动补全 |
| `Tab Tab` | 显示所有可能补全 |

---

## 16. 两个发行版主要差异对比

| 功能 | Ubuntu | Arch Linux |
|------|--------|------------|
| 包管理器 | `apt` | `pacman` |
| AUR 支持 | 无（有 PPA） | `yay` / `paru` |
| sudo 组 | `sudo` | `wheel` |
| 发行版版本查看 | `lsb_release -a` | `cat /etc/os-release` |
| 默认 Shell | bash | bash（通常换 zsh/fish） |
| 更新策略 | LTS 版本，稳定 | 滚动更新，始终最新 |
| 初始化系统 | systemd | systemd |
| 网络配置工具 | netplan + NetworkManager | NetworkManager / systemd-networkd |
| 防火墙工具 | ufw | firewalld / iptables |
| 安装后开箱 | 包含桌面/基础软件 | 极简，完全自定义 |
| 软件包格式 | .deb | .pkg.tar.zst |
| snap 支持 | 默认支持 | 需手动安装 |
| flatpak 支持 | 需安装 | 需安装 |
