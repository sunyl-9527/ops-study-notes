# FRP 内网穿透部署与排障

FRP 由服务端 `frps` 和客户端 `frpc` 组成。公网客户端先访问 FRP 服务端的映射端口，再由 `frps` 把流量转发到内网主机上的 `frpc`，最后到达本地服务。

```text
运维终端 -> 公网服务器:映射端口 -> frps -> frpc -> 内网服务
```

本文以把 Windows 远程桌面 `127.0.0.1:3389` 映射到公网服务器 TCP 端口 `10001` 为例。示例地址来自文档保留网段，不能直接用于生产。

## 1. 前置条件

- 一台具有公网 IP 的 Linux 服务器，用于运行 `frps`。
- 一台能够访问该服务器的内网主机，用于运行 `frpc`。
- 服务端和客户端使用相同的 FRP 版本。
- 云防火墙和系统防火墙只放行必需端口。
- 使用随机生成的高强度 token，不把真实值提交到 Git。

示例变量：

```text
FRP_SERVER_IP=203.0.113.10
FRP_BIND_PORT=7000
RDP_REMOTE_PORT=10001
```

## 2. 安装 FRP 服务端

从 [FRP Releases](https://github.com/fatedier/frp/releases) 选择与你的系统架构匹配的版本。以下用 Linux AMD64 和 `v0.69.1` 演示：

```bash
FRP_VERSION=0.69.1
cd /tmp
curl -fLO "https://github.com/fatedier/frp/releases/download/v${FRP_VERSION}/frp_${FRP_VERSION}_linux_amd64.tar.gz"
tar -xzf "frp_${FRP_VERSION}_linux_amd64.tar.gz"
sudo install -m 0755 "frp_${FRP_VERSION}_linux_amd64/frps" /usr/local/bin/frps
sudo install -d -m 0750 /etc/frp
```

验证版本：

```bash
frps --version
```

## 3. 配置 frps

创建 `/etc/frp/frps.toml`：

```toml
bindPort = 7000

auth.method = "token"
auth.token = "<strong-random-token>"

allowPorts = [
  { start = 10000, end = 10100 }
]
```

生成随机 token 的一种方式：

```bash
openssl rand -base64 32
```

把 token 放入受权限控制的配置管理系统。配置文件至少限制为仅 root 可读：

```bash
sudo chown root:root /etc/frp/frps.toml
sudo chmod 600 /etc/frp/frps.toml
```

## 4. 使用 systemd 管理 frps

创建 `/etc/systemd/system/frps.service`：

```ini
[Unit]
Description=FRP server
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
ExecStart=/usr/local/bin/frps -c /etc/frp/frps.toml
Restart=on-failure
RestartSec=5s
NoNewPrivileges=true
LimitNOFILE=1048576

[Install]
WantedBy=multi-user.target
```

加载并启动：

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now frps
sudo systemctl status frps --no-pager
sudo journalctl -u frps -n 100 --no-pager
```

确认监听端口：

```bash
sudo ss -lntp | grep ':7000'
```

## 5. 配置 frpc

在 Windows 客户端下载同版本 FRP，创建 `frpc.toml`：

```toml
serverAddr = "203.0.113.10"
serverPort = 7000

auth.method = "token"
auth.token = "<same-token-as-server>"

[[proxies]]
name = "rdp-lab"
type = "tcp"
localIP = "127.0.0.1"
localPort = 3389
remotePort = 10001
transport.useEncryption = true
transport.useCompression = true
```

先在前台启动，便于观察日志：

```powershell
.\frpc.exe -c .\frpc.toml
```

看到代理启动成功后，再通过任务计划程序或服务管理工具设置开机启动。运行账户必须对 FRP 目录和配置文件具有读取权限。

## 6. 验证链路

在 Windows 客户端确认本地服务可用：

```powershell
Test-NetConnection 127.0.0.1 -Port 3389
Test-NetConnection 203.0.113.10 -Port 7000
```

在服务端确认端口：

```bash
sudo ss -lntp | grep -E ':(7000|10001)'
sudo journalctl -u frps -f
```

从授权的运维终端测试映射端口：

```powershell
Test-NetConnection 203.0.113.10 -Port 10001
mstsc /v:203.0.113.10:10001
```

## 7. 防火墙建议

- `7000/tcp` 只允许固定的 FRP 客户端出口 IP 访问。
- `10001/tcp` 只允许堡垒机或运维出口 IP 访问。
- 不要把 RDP、SSH 或 SOCKS5 映射端口开放给整个互联网。
- 优先通过 VPN、堡垒机或零信任接入系统访问 FRP 映射服务。
- 定期轮换 token，并检查异常登录和端口扫描日志。

不同发行版的防火墙命令不同。修改前先确认当前规则，避免把 SSH 管理端口一并阻断。

## 8. 常见故障排查

### frpc 无法连接 frps

按顺序检查：

```powershell
Resolve-DnsName <frp-server-domain>
Test-NetConnection <frp-server-domain> -Port 7000
```

```bash
sudo systemctl status frps --no-pager
sudo journalctl -u frps -n 100 --no-pager
sudo ss -lntp | grep ':7000'
```

重点确认服务器地址、端口、token、FRP 版本、云防火墙和系统防火墙。

### 映射端口可达，但业务服务不可用

在客户端检查本地服务：

```powershell
Get-NetTCPConnection -LocalPort 3389 -State Listen
Test-NetConnection 127.0.0.1 -Port 3389
```

如果本地端口没有监听，应先恢复业务服务，而不是反复重启 FRP。

### 连接间歇性中断

检查客户端休眠、网络切换、VPN 路由、进程退出和时间同步。服务端同时查看：

```bash
sudo journalctl -u frps --since "30 minutes ago"
uptime
free -h
df -h
```

### 配置修改后未生效

```bash
sudo systemctl restart frps
sudo systemctl status frps --no-pager
```

Windows 客户端需要停止旧 `frpc` 进程，再使用目标配置文件重新启动。

## 9. 变更与回滚记录

每次调整至少记录：

```text
变更时间：
变更人：
影响节点：
修改内容：
验证结果：
回滚方式：
关联工单：
```

资产地址、账号和凭据放在受访问控制的系统中，不写入公开笔记。
