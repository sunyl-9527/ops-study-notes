# 云服务器运维常用命令

> 适用场景：Linux 云主机日常巡检、服务维护、故障排查、快速应急处理  
> 使用原则：先查看，再修改；先备份，再删除；涉及网络、防火墙、磁盘操作时优先保留回滚手段

## 目录

1. [系统信息](#1-系统信息)
2. [用户与权限](#2-用户与权限)
3. [文件与目录](#3-文件与目录)
4. [进程管理](#4-进程管理)
5. [系统资源监控](#5-系统资源监控)
6. [网络](#6-网络)
7. [磁盘与存储](#7-磁盘与存储)
8. [服务管理（systemd）](#8-服务管理systemd)
9. [日志查看](#9-日志查看)
10. [防火墙](#10-防火墙)
11. [软件包管理](#11-软件包管理)
12. [文件传输](#12-文件传输)
13. [定时任务](#13-定时任务)
14. [性能排查速查](#14-性能排查速查)

---

## 1. 系统信息

```bash
# 查看系统版本
cat /etc/os-release
lsb_release -a

# 查看内核版本
uname -r
uname -a

# 查看主机名
hostname
hostnamectl

# 修改主机名
# 提醒：修改后建议同时检查 /etc/hosts，避免本地解析异常
hostnamectl set-hostname my-server

# 查看系统运行时间 / 负载
uptime

# 查看当前时间与时区
date
timedatectl

# 修改时区
timedatectl set-timezone Asia/Shanghai

# 查看 CPU 信息
lscpu
cat /proc/cpuinfo | grep "model name" | head -1

# 查看内存信息
cat /proc/meminfo | head -5

# 查看系统位数
getconf LONG_BIT
```

---

## 2. 用户与权限

### 用户管理

```bash
# 查看当前用户
whoami
id

# 查看所有用户
cat /etc/passwd | cut -d: -f1

# 创建用户
useradd -m -s /bin/bash username
# -m 创建家目录  -s 指定 shell

# 设置 / 修改密码
passwd username

# 删除用户（-r 同时删除家目录）
userdel -r username

# 切换用户
su - username

# 以 root 执行命令
sudo command

# 查看 sudo 权限配置
visudo
```

### 权限管理

```bash
# 查看文件权限
ls -la

# 修改文件权限（数字方式）
chmod 755 file       # rwxr-xr-x
chmod 644 file       # rw-r--r--
chmod -R 755 dir/    # 递归修改目录

# 修改文件所有者
chown user:group file
chown -R user:group dir/

# 常用权限数字速记
# 7 = rwx（读写执行）
# 6 = rw-（读写）
# 5 = r-x（读执行）
# 4 = r--（只读）
```

### SSH 密钥配置

```bash
# 生成 SSH 密钥对
ssh-keygen -t ed25519 -C "your_email"

# 将公钥复制到远程服务器
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@server_ip

# 手动追加公钥
cat ~/.ssh/id_ed25519.pub >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys

# SSH 连接
ssh user@server_ip
ssh -p 22 user@server_ip
ssh -i ~/.ssh/id_ed25519 user@server_ip
```

---

## 3. 文件与目录

```bash
# 查看目录内容
ls -la              # 详细列表（含隐藏文件）
ls -lh              # 人性化显示文件大小
tree -L 2           # 树形显示（2层深度）

# 创建目录
mkdir -p /opt/app/config    # -p 递归创建

# 复制
cp file1 file2
cp -r dir1/ dir2/           # 递归复制目录

# 移动 / 重命名
mv file1 file2
mv dir1/ /opt/

# 删除
rm file
rm -rf dir/                 # 递归强制删除（谨慎）

# 查找文件
find /opt -name "*.log"                  # 按文件名
find /opt -name "*.log" -mtime -7        # 7天内修改
find /var/log -size +100M                # 大于100M
find / -user username                    # 按所有者

# 搜索文件内容
grep "error" /var/log/nginx/error.log
grep -r "keyword" /opt/app/             # 递归搜索
grep -n "error" app.log                 # 显示行号
grep -i "error" app.log                 # 忽略大小写
grep -v "DEBUG" app.log                 # 反向匹配（排除）

# 查看文件内容
cat file
less file                               # 翻页查看（q退出）
head -20 file                           # 前20行
tail -50 file                           # 后50行
tail -f /var/log/nginx/access.log       # 实时跟踪

# 文件内容统计
wc -l file             # 行数
wc -c file             # 字节数

# 软链接
ln -s /opt/app/current /opt/app/latest
```

---

## 4. 进程管理

```bash
# 查看进程
ps aux                          # 所有进程
ps aux | grep nginx             # 过滤指定进程
ps -ef --forest                 # 树形显示父子关系

# 实时进程监控
top
htop                            # 更友好（需安装）

# 查看进程占用的端口
lsof -i :8080
lsof -p <PID>

# 查找进程 PID
pgrep nginx
pidof nginx

# 终止进程
kill <PID>                      # 优雅终止（SIGTERM）
kill -9 <PID>                   # 强制终止（SIGKILL）
killall nginx                   # 按名称终止所有同名进程
pkill -f "python app.py"        # 按命令行匹配终止

# 后台运行
nohup python app.py &           # 脱离终端运行
nohup python app.py > app.log 2>&1 &

# 查看后台任务
jobs
fg %1                           # 将后台任务调到前台
```

---

## 5. 系统资源监控

### CPU / 内存

```bash
# 实时负载与资源
top
# top 内常用按键：P 按CPU排序，M 按内存排序，k 终止进程，q 退出

# 查看内存使用
free -h

# 查看 CPU 使用率（每秒刷新，显示3次）
vmstat 1 3

# 查看详细 CPU 统计（需 sysstat）
mpstat -P ALL 1

# 查看系统负载（1/5/15 分钟平均）
cat /proc/loadavg
uptime
```

### 磁盘 I/O

```bash
# 查看磁盘读写（需 sysstat）
iostat -x 1

# 实时磁盘 I/O（需 iotop）
iotop -o               # 只显示有 I/O 的进程
```

### 内存详情

```bash
# 查看内存使用详情
free -h

# 查看 OOM 记录（内存不足杀进程）
dmesg | grep -i "oom\|killed"
grep -i "oom\|killed" /var/log/syslog
```

---

## 6. 网络

### 连接与状态

```bash
# 查看网络接口信息
ip addr
ip addr show eth0

# 查看路由表
ip route
route -n

# 查看所有端口监听情况
ss -tlnp               # TCP 监听端口（推荐）
netstat -tlnp          # 同上（老版本系统）
ss -tunlp              # TCP + UDP

# 查看已建立的连接
ss -s                  # 连接统计汇总
ss -antp               # 所有 TCP 连接

# 测试连通性
ping -c 4 baidu.com
traceroute baidu.com
mtr baidu.com          # 实时路由追踪（需安装）

# 测试端口连通性
telnet server_ip 3306
nc -zv server_ip 3306
curl -v telnet://server_ip:3306

# DNS 查询
nslookup domain.com
dig domain.com
dig domain.com +short  # 只显示 IP
```

### 网络流量

```bash
# 实时网络流量（需安装）
iftop -n               # 按连接显示流量
nethogs               # 按进程显示流量
nload                  # 按网卡显示流量

# 抓包
tcpdump -i eth0 port 80 -n
tcpdump -i eth0 host 192.168.1.1 -w capture.pcap
```

### 常用网络请求

```bash
# curl 常用操作
curl https://example.com                          # GET 请求
curl -o file.zip https://example.com/file.zip     # 下载文件
curl -X POST -H "Content-Type: application/json" \
     -d '{"key":"value"}' https://api.example.com

# 测试接口响应时间
curl -o /dev/null -s -w "total: %{time_total}s\n" https://example.com

# wget 下载文件
wget https://example.com/file.tar.gz
wget -c https://example.com/file.tar.gz           # 断点续传
wget -P /opt/ https://example.com/file.tar.gz     # 指定目录
```

---

## 7. 磁盘与存储

```bash
# 查看磁盘使用情况
df -h                  # 各分区使用率
df -h /opt             # 指定目录所在分区

# 查看目录 / 文件占用大小
du -sh /var/log        # 目录总大小
du -sh /var/log/*      # 各子目录大小
du -sh /* 2>/dev/null  # 根目录各文件夹大小

# 找出占用最大的前10个目录
du -h /var | sort -rh | head -10

# 查看磁盘分区
lsblk
fdisk -l

# 查看 inode 使用情况（inode 满同样导致无法写文件）
df -i

# 找出 inode 占用最多的目录
find /var -xdev -printf '%h\n' | sort | uniq -c | sort -k 1 -n | tail -10

# 压缩与解压
tar -czf archive.tar.gz dir/          # 压缩
tar -xzf archive.tar.gz               # 解压
tar -xzf archive.tar.gz -C /opt/      # 解压到指定目录
tar -tzf archive.tar.gz               # 查看内容不解压

zip -r archive.zip dir/
unzip archive.zip
unzip archive.zip -d /opt/

# 清理大文件示例
# 清理7天前的日志
find /var/log -name "*.log" -mtime +7 -delete
# 清理已被删除但仍占用空间的文件（需重启或重载服务）
lsof | grep deleted
```

---

## 8. 服务管理（systemd）

```bash
# 服务基本操作
systemctl start nginx
systemctl stop nginx
systemctl restart nginx
systemctl reload nginx       # 优雅重载配置（不断开连接）
systemctl status nginx

# 开机自启
systemctl enable nginx
systemctl disable nginx

# 查看所有服务状态
systemctl list-units --type=service
systemctl list-units --type=service --state=failed   # 失败的服务

# 查看服务日志
journalctl -u nginx
journalctl -u nginx -f                # 实时跟踪
journalctl -u nginx --since "1 hour ago"
journalctl -u nginx -n 50            # 最近50条
journalctl -u nginx --since "2026-01-01" --until "2026-01-02"

# 编写自定义 service 文件
# 文件路径：/etc/systemd/system/myapp.service
```

```ini
# /etc/systemd/system/myapp.service 示例
[Unit]
Description=My Application
After=network.target

[Service]
Type=simple
User=appuser
WorkingDirectory=/opt/myapp
ExecStart=/usr/bin/python3 /opt/myapp/app.py
Restart=always
RestartSec=5
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

```bash
# 加载新 service 文件后需执行
systemctl daemon-reload
systemctl enable myapp
systemctl start myapp
```

---

## 9. 日志查看

```bash
# 系统日志
tail -f /var/log/syslog          # Ubuntu/Debian
tail -f /var/log/messages        # CentOS/RHEL

# 内核日志
dmesg
dmesg | tail -20
dmesg | grep -i error

# 认证 / 登录日志
tail -f /var/log/auth.log        # Ubuntu
tail -f /var/log/secure          # CentOS

# 查看最近登录记录
last
last -n 20
lastb                            # 登录失败记录

# Nginx 日志
tail -f /var/log/nginx/access.log
tail -f /var/log/nginx/error.log

# 统计 Nginx 访问日志
# 访问量最高的 IP TOP10
awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head -10

# 状态码分布
awk '{print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -rn

# 访问量最高的 URL TOP10
awk '{print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head -10

# 日志切割（logrotate 配置示例）
# /etc/logrotate.d/myapp
```

```
/opt/myapp/logs/*.log {
    daily
    rotate 14
    compress
    missingok
    notifempty
    sharedscripts
    postrotate
        systemctl reload myapp
    endscript
}
```

---

## 10. 防火墙

### ufw（Ubuntu 推荐）

```bash
# 查看状态
ufw status
ufw status verbose

# 启用 / 禁用
ufw enable
ufw disable

# 允许端口
ufw allow 80
ufw allow 443
ufw allow 22/tcp
ufw allow 8000:8100/tcp       # 端口范围

# 允许指定 IP 访问端口
ufw allow from 192.168.1.0/24 to any port 3306

# 拒绝端口
ufw deny 3306

# 删除规则
ufw delete allow 80
ufw status numbered            # 查看规则编号
ufw delete 3                   # 按编号删除
```

### firewalld（CentOS/RHEL 推荐）

```bash
# 查看状态
firewall-cmd --state
firewall-cmd --list-all

# 永久开放端口（--permanent 持久化，需 reload 生效）
firewall-cmd --permanent --add-port=80/tcp
firewall-cmd --permanent --add-port=443/tcp
firewall-cmd --reload

# 移除端口
firewall-cmd --permanent --remove-port=8080/tcp
firewall-cmd --reload

# 允许服务
firewall-cmd --permanent --add-service=http
firewall-cmd --permanent --add-service=https
```

### iptables（通用）

```bash
# 查看规则
iptables -L -n -v

# 允许端口
iptables -A INPUT -p tcp --dport 80 -j ACCEPT

# 保存规则
# 提醒：不同发行版持久化方式可能不同，执行前先确认目标系统的 iptables 持久化方案
iptables-save > /etc/iptables/rules.v4
```

---

## 11. 软件包管理

### apt（Ubuntu/Debian）

```bash
# 更新软件包列表
apt update

# 升级所有软件包
apt upgrade -y

# 安装软件
apt install -y nginx git curl wget vim htop

# 删除软件
apt remove nginx
apt purge nginx              # 同时删除配置文件

# 搜索软件包
apt search nginx

# 查看软件包信息
apt show nginx

# 清理缓存
apt autoremove -y
apt clean
```

### yum / dnf（CentOS/RHEL）

```bash
# 更新
yum update -y
dnf update -y               # CentOS 8+

# 安装
yum install -y nginx
dnf install -y nginx

# 删除
yum remove nginx

# 搜索
yum search nginx
```

---

## 12. 文件传输

### scp（基于 SSH）

```bash
# 上传本地文件到服务器
scp /local/file.txt user@server:/remote/path/

# 下载服务器文件到本地
scp user@server:/remote/file.txt /local/path/

# 传输目录（-r 递归）
scp -r /local/dir/ user@server:/remote/path/

# 指定端口
scp -P 22 file.txt user@server:/path/
```

### rsync（增量同步，推荐）

```bash
# 同步目录到服务器（-a 归档，-v 显示，-z 压缩，--progress 进度）
rsync -avz --progress /local/dir/ user@server:/remote/dir/

# 同步并删除目标端多余文件
rsync -avz --delete /local/dir/ user@server:/remote/dir/

# 只同步新文件（跳过已存在）
rsync -avz --ignore-existing /local/dir/ user@server:/remote/dir/

# 排除文件
rsync -avz --exclude="*.log" --exclude=".git" /local/dir/ user@server:/remote/dir/

# 从服务器同步到本地
rsync -avz user@server:/remote/dir/ /local/dir/
```

---

## 13. 定时任务

```bash
# 编辑当前用户的 crontab
crontab -e

# 查看当前用户的 crontab
crontab -l

# 删除 crontab
crontab -r

# 查看系统级 crontab
cat /etc/crontab
ls /etc/cron.d/
```

### cron 表达式格式

```
*    *    *    *    *   command
分   时   日   月   周
```

```bash
# 示例
0 2 * * *     /opt/backup.sh          # 每天凌晨2点
*/5 * * * *   /opt/check.sh           # 每5分钟
0 9 * * 1     /opt/weekly.sh          # 每周一9点
0 0 1 * *     /opt/monthly.sh         # 每月1日0点
@reboot       /opt/startup.sh         # 开机时执行

# 输出重定向（避免发邮件）
0 2 * * * /opt/backup.sh >> /var/log/backup.log 2>&1
```

---

## 14. 性能排查速查

### CPU 高

```bash
# 找出 CPU 占用高的进程
top -c              # 按 P 排序
ps aux --sort=-%cpu | head -10

# 查看进程的线程
top -H -p <PID>
ps -T -p <PID>
```

### 内存高

```bash
# 找出内存占用高的进程
ps aux --sort=-%mem | head -10
top                 # 按 M 排序

# 查看内存详情
free -h
cat /proc/meminfo

# 查看 OOM 日志
dmesg | grep -i oom
```

### 磁盘满

```bash
# 快速定位大文件
df -h                               # 确定哪个分区满了
du -sh /var/log/* | sort -rh | head -10
find /var -size +500M 2>/dev/null   # 找500M以上的文件
lsof | grep deleted | awk '{print $7,$9}' | sort -rn | head  # 已删除但未释放
```

### 连接数高

```bash
# 查看当前连接总数
ss -s

# 查看各状态连接数
ss -ant | awk '{print $1}' | sort | uniq -c | sort -rn

# 查看连接数最多的 IP
ss -ant | awk '{print $5}' | cut -d: -f1 | sort | uniq -c | sort -rn | head -10

# Nginx 当前活跃连接（需开启 stub_status）
curl http://localhost/nginx_status
```

### 系统卡顿

```bash
# 综合看板
uptime              # 负载
free -h             # 内存
df -h               # 磁盘
ss -s               # 连接数
iostat -x 1 3       # I/O
```

---

## 附录：常用快捷操作

```bash
# 历史命令
history
!!                  # 执行上一条命令
!nginx              # 执行最近一条以 nginx 开头的命令
Ctrl+R              # 搜索历史命令

# 输出重定向
command > file      # 覆盖写入
command >> file     # 追加写入
command 2>&1        # 标准错误重定向到标准输出
command &>/dev/null # 丢弃所有输出

# 管道与常用组合
ps aux | grep nginx | grep -v grep
netstat -tlnp | grep :80
awk '{print $1}' access.log | sort | uniq -c | sort -rn | head

# 多命令执行
cmd1 && cmd2        # cmd1 成功才执行 cmd2
cmd1 || cmd2        # cmd1 失败才执行 cmd2
cmd1 ; cmd2         # 顺序执行，不管成败
```
