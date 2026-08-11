# 运维网络基础笔记

> 面向运维方向学习者，聚焦排障所需的网络知识，不是考研级网络原理。

## 一、运维网络的基本思路

学习网络不是为了背协议，而是为了回答这些问题：
1. 本机网卡、IP、网关和 DNS 是否正确？
2. 数据包能否到达目标主机？
3. 目标端口是否可达？
4. 服务是否在正确地址和端口监听？
5. 防火墙、安全组或 NAT 是否阻断流量？
6. DNS、TLS 和应用层协议是否正常？
先判断故障范围：
- 只有一个客户端失败：检查客户端配置、DNS、代理和本机防火墙。
- 同一网段都失败：检查网关、二层网络和上游路由。
- 所有客户端都失败：检查服务进程、监听端口和服务端策略。
- IP 能访问但域名不能访问：优先检查 DNS。
- 端口能连接但请求失败：继续检查 TLS、HTTP、认证和应用日志。
- 偶发超时或丢包：关注链路质量、负载、连接数和上游依赖。
排障前记录发生时间、源和目标、协议和端口、原始报错、近期变更。
不要一开始就重启服务或关闭防火墙，应先保留现场并进行最小化验证。
---
## 二、网络分层模型：每层排查什么

运维场景使用 TCP/IP 四层模型即可，重点是每一层能观察什么。
| 层次 | 常见协议或对象 | 排障重点 | 常用工具 |
|---|---|---|---|
| 应用层 | HTTP、HTTPS、DNS、SSH | 域名、状态码、证书、应用日志 | `curl`、`dig`、`openssl` |
| 传输层 | TCP、UDP | 端口、握手、重传、连接状态 | `ss`、`nc`、`tcpdump` |
| 网际层 | IPv4、IPv6、ICMP | IP、子网、路由、丢包 | `ip`、`ping`、`traceroute` |
| 网络接口层 | Ethernet、Wi-Fi、ARP | 网卡、MAC、链路、邻居解析 | `ip link`、`ip neigh`、`ethtool` |
### 2.1 网络接口层
主要检查网卡是否存在并处于 `UP`、链路是否正常、是否有错误包，以及邻居解析是否成功。
```bash
ip -br link
ip -s link show dev <网卡名>
ip neigh show
sudo ethtool <网卡名>
```
`ip neigh` 常见状态：
- `REACHABLE`：邻居近期可达。
- `STALE`：缓存存在，需要时会重新确认。
- `PROBE`：正在探测邻居。
- `FAILED`：邻居解析失败，可能是二层不通或目标不存在。
### 2.2 网际层
主要检查 IP、前缀长度、默认路由、实际选路、源地址和中间路径。
```bash
ip -br addr
ip route show
ip route get <目标IP>
ping -c 4 <目标IP>
traceroute <目标IP>
```
### 2.3 传输层
主要检查服务是否监听、监听地址是否正确、TCP 握手在哪一步失败，以及 UDP 请求和响应是否都出现。
```bash
ss -lntup
ss -ant
nc -vz <目标主机> <端口>
sudo tcpdump -ni any host <目标IP>
```
### 2.4 应用层
主要检查 DNS 结果、HTTP 状态码、TLS 证书、协议交互和应用日志。
```bash
dig <域名>
curl -v https://<域名>/
openssl s_client -connect <域名>:443 -servername <域名>
journalctl -u <服务名> --since "10 minutes ago"
```
分层只是缩小范围：`ping` 不通不等于服务不可用，端口可连接也不等于应用正常。
---
## 三、IP 地址与子网

### 3.1 IPv4 与 CIDR
IPv4 地址有 32 位，CIDR 用 `/前缀长度` 表示网络位数量。
以 `192.168.10.20/24` 为例：
- 子网掩码：`255.255.255.0`。
- 网络地址：`192.168.10.0`。
- 广播地址：`192.168.10.255`。
- 通常可用地址：`192.168.10.1` 到 `192.168.10.254`。
传统子网的可用地址数通常为 `2^(32-前缀长度)-2`，云网络和点对点链路可能有特殊规则。
### 3.2 私有地址与特殊地址
| 地址段 | CIDR | 常见用途 |
|---|---|---|
| `10.0.0.0` 到 `10.255.255.255` | `10.0.0.0/8` | 大型内网、云 VPC |
| `172.16.0.0` 到 `172.31.255.255` | `172.16.0.0/12` | 企业和容器网络 |
| `192.168.0.0` 到 `192.168.255.255` | `192.168.0.0/16` | 家庭和实验网络 |
常见特殊地址：
- `127.0.0.0/8`：IPv4 回环地址。
- `169.254.0.0/16`：链路本地地址，意外出现时可能表示 DHCP 失败。
- `0.0.0.0`：未指定地址；监听场景中常表示所有 IPv4 地址。
- `::1`：IPv6 回环地址。
- `::`：未指定 IPv6 地址。
### 3.3 子网划分实用速算
| CIDR | 子网掩码 | 地址总数 | 通常可用数 |
|---|---|---:|---:|
| `/24` | `255.255.255.0` | 256 | 254 |
| `/25` | `255.255.255.128` | 128 | 126 |
| `/26` | `255.255.255.192` | 64 | 62 |
| `/27` | `255.255.255.224` | 32 | 30 |
| `/28` | `255.255.255.240` | 16 | 14 |
| `/29` | `255.255.255.248` | 8 | 6 |
| `/30` | `255.255.255.252` | 4 | 2 |
速算方法：
1. 找到前缀未占满的八位组。
2. 用 `256 - 掩码值` 得到块大小。
3. 从 0 开始按块大小列出子网边界。
4. 所在区间起点是网络地址，下一边界减 1 是广播地址。
示例：`192.168.10.70/26`。
- `/26` 掩码为 `255.255.255.192`，块大小为 64。
- 边界是 `0、64、128、192`，70 位于 64 到 127。
- 网络地址为 `192.168.10.64`。
- 广播地址为 `192.168.10.127`。
- 通常可用范围为 `192.168.10.65` 到 `192.168.10.126`。
```bash
ipcalc 192.168.10.70/26
```
两个地址的网络部分相同，通常直接通信；不同则交给路由表选择下一跳。
前缀长度配置错误，即使 IP 看起来正确也可能无法通信。
```bash
ip -br addr
ip addr show dev <网卡名>
sudo ip addr add 192.168.10.20/24 dev <网卡名>
sudo ip addr del 192.168.10.20/24 dev <网卡名>
```
`ip` 修改通常是临时的，持久化配置应使用系统网络管理工具。
---
## 四、TCP、UDP 与端口

### 4.1 TCP 三次握手
TCP 面向连接并提供可靠、有序的字节流。建立连接的简化过程：
1. 客户端发送 `SYN`。
2. 服务端返回 `SYN, ACK`。
3. 客户端发送 `ACK`，连接建立。
抓包时的运维判断：
- 只有重复 `SYN`：可能是路由、防火墙、安全组或回程路径问题。
- 收到 `RST`：主机通常可达，但端口未监听或被主动拒绝。
- 收到 `SYN, ACK` 后客户端不回 `ACK`：检查客户端策略和回程路径。
- 握手后立即断开：继续检查 TLS、协议、认证和服务日志。
```bash
nc -vz -w 3 <目标主机> <端口>
```
`Connection refused` 通常表示目标可达但端口被拒绝；超时更像丢弃、路径不通或无回包。
### 4.2 TCP 四次挥手
TCP 是全双工协议，双方分别关闭发送方向：
1. 一方发送 `FIN`。
2. 另一方返回 `ACK`。
3. 另一方处理完成后发送 `FIN`。
4. 最初一方返回 `ACK`。
应用不正确关闭套接字，可能造成状态堆积和文件描述符耗尽。
```bash
ss -s
ss -ant
```
### 4.3 TIME_WAIT 与 CLOSE_WAIT
`TIME-WAIT`：
- 通常位于主动关闭连接的一方。
- 用于处理最后确认的重传，并避免旧报文干扰新连接。
- 一定数量是正常的；大量出现时检查短连接、连接复用和流量模式。
`CLOSE-WAIT`：
- 对端已经关闭，本机应用尚未关闭套接字。
- 长时间大量堆积通常指向应用未正确释放连接。
```bash
ss -ant state time-wait
ss -ant state close-wait
sudo ss -antp state close-wait
```
其他常见状态：`LISTEN`、`SYN-SENT`、`SYN-RECV`、`ESTAB`、`FIN-WAIT-1`、`FIN-WAIT-2`、`LAST-ACK`。
### 4.4 UDP
UDP 无连接，不保证到达、顺序或去重，常用于 DNS、DHCP、NTP 和实时通信。
UDP 排障应同时观察请求是否发出、响应是否返回，不能只看“连接成功”。
```bash
ss -lnup
nc -vzu -w 3 <目标主机> <端口>
sudo tcpdump -ni any udp and host <目标IP>
```
### 4.5 常见端口速查
| 端口 | 协议 | 常见服务 |
|---:|---|---|
| 22 | TCP | SSH |
| 25 | TCP | SMTP |
| 53 | TCP、UDP | DNS |
| 67、68 | UDP | DHCP 服务端、客户端 |
| 80 | TCP | HTTP |
| 123 | UDP | NTP |
| 443 | TCP | HTTPS |
| 465、587 | TCP | 邮件加密或提交 |
| 993 | TCP | IMAPS |
| 3306 | TCP | MySQL |
| 5432 | TCP | PostgreSQL |
| 6379 | TCP | Redis |
| 8080、8443 | TCP | 常见备用 Web 端口 |
端口只是默认约定，实际服务可以修改。
```bash
sudo ss -lntup
sudo ss -lntp '( sport = :<端口> )'
```
`ifconfig` 和 `netstat` 属于旧版 `net-tools`，现代 Linux 优先使用 `ip` 和 `ss`。
---
## 五、DNS 域名解析

### 5.1 解析流程
简化流程如下：
1. 应用检查自身缓存。
2. 系统按名称服务配置检查缓存和 `/etc/hosts`。
3. 查询发送给递归 DNS 服务器。
4. 递归服务器使用缓存，或查询根、顶级域和权威 DNS。
5. 客户端获得记录与 TTL，再连接目标地址。
Linux 名称解析顺序受 `/etc/nsswitch.conf` 中 `hosts:` 配置影响。
```bash
grep '^hosts:' /etc/nsswitch.conf
getent hosts <域名>
getent ahosts <域名>
```
`getent` 更接近普通应用通过系统解析器获得的结果。
### 5.2 常见记录
| 记录 | 作用 | 排障重点 |
|---|---|---|
| `A` | 名称映射到 IPv4 | 地址和 TTL 是否正确 |
| `AAAA` | 名称映射到 IPv6 | IPv6 路径是否可用 |
| `CNAME` | 名称指向另一名称 | 目标是否存在、链是否过长 |
| `MX` | 邮件接收服务器 | 优先级和目标解析 |
| `TXT` | 文本验证信息 | 域名验证、SPF 等内容 |
| `NS` | 权威 DNS | 委派是否一致 |
| `PTR` | IP 反向解析 | 邮件、审计和访问控制 |
```bash
dig A <域名>
dig CNAME <域名>
dig MX <域名>
dig TXT <域名>
dig +short <域名>
dig -x <目标IP>
```
指定 DNS 服务器和追踪委派：
```bash
dig @<DNS服务器IP> <域名>
dig +trace <域名>
```
### 5.3 nslookup 与 resolvectl
```bash
nslookup <域名>
nslookup <域名> <DNS服务器IP>
resolvectl status
resolvectl query <域名>
resolvectl statistics
sudo resolvectl flush-caches
```
查看解析配置：
```bash
ls -l /etc/resolv.conf
cat /etc/resolv.conf
```
`/etc/resolv.conf` 可能由 NetworkManager、`systemd-resolved` 或 DHCP 自动生成，不要在不清楚来源时直接覆盖。
### 5.4 常见 DNS 故障
`NXDOMAIN`：名称不存在，检查拼写、记录、委派、搜索域和内外网视图。
查询超时：检查 DNS 服务器可达性、UDP 53、TCP 53、防火墙和上游状态。
不同客户端结果不同：检查 TTL、递归服务器、分离解析、负载均衡和 `/etc/hosts`。
能解析但不能访问：核对返回地址、IPv4/IPv6 路径、目标端口和服务状态。
```bash
getent ahosts <域名>
dig +short A <域名>
dig +short AAAA <域名>
curl -v https://<域名>/
```
---
## 六、HTTP 与 HTTPS

### 6.1 请求与响应结构
简化请求：
```text
GET /health HTTP/1.1
Host: <域名>
Accept: */*
```
请求包含方法、路径、协议版本、请求头和可选请求体。
`Host` 对虚拟主机和反向代理路由非常重要。
简化响应：
```text
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 15
{"status":"ok"}
```
响应包含状态行、响应头、空行和可选响应体。
```bash
curl -I https://<域名>/
curl -v https://<域名>/
curl -sS -o /dev/null -w '%{http_code}\n' https://<域名>/
```
查看各阶段耗时：
```bash
curl -sS -o /dev/null \
  -w '解析=%{time_namelookup} 连接=%{time_connect} TLS=%{time_appconnect} 首包=%{time_starttransfer} 总计=%{time_total}\n' \
  https://<域名>/
```
绕过 DNS 指向实验地址，同时保留正确的 `Host` 和 TLS SNI：
```bash
curl --resolve <域名>:443:192.168.10.20 https://<域名>/
```
### 6.2 常见状态码
| 状态码 | 含义 | 运维排查方向 |
|---:|---|---|
| `200` | 成功 | 继续确认响应内容和业务数据 |
| `204` | 成功但无响应体 | 不要误判为空响应故障 |
| `301`、`302` | 重定向 | 检查 `Location`、协议和循环 |
| `304` | 使用缓存 | 检查缓存头和客户端行为 |
| `400` | 请求错误 | 检查参数、请求头和代理改写 |
| `401` | 未认证 | 检查令牌、凭据和时间偏差 |
| `403` | 禁止访问 | 检查权限、访问控制和 WAF |
| `404` | 资源不存在 | 检查路径、路由和发布版本 |
| `429` | 请求过多 | 检查限流、重试和流量突增 |
| `500` | 应用内部错误 | 查看应用日志和依赖 |
| `502` | 网关收到无效上游响应 | 检查代理到上游的连接和协议 |
| `503` | 服务暂不可用 | 检查实例健康、过载和维护状态 |
| `504` | 网关等待上游超时 | 检查慢查询、依赖和超时设置 |
应判断响应来自 CDN、负载均衡、反向代理还是应用，而不是只看状态码。
### 6.3 TLS 握手与证书链
HTTPS 通常先建立 TCP，再进行 TLS 握手，最后传输 HTTP。
简化过程：
1. 客户端发送支持的 TLS 参数和 SNI 域名。
2. 服务端选择参数并返回证书链。
3. 客户端验证域名、有效期、签发链和信任根。
4. 双方协商会话密钥，开始加密通信。
常见故障：证书过期、域名不匹配、中间证书缺失、根证书不受信、系统时间错误、TLS 参数不兼容、SNI 配置错误。
```bash
openssl s_client -connect <域名>:443 -servername <域名> -showcerts </dev/null
```
重点查看 `subject`、`issuer`、有效期、SAN 和 `Verify return code`。
```bash
openssl s_client -connect <域名>:443 -servername <域名> </dev/null 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates -ext subjectAltName
```
验证本地证书链：
```bash
openssl verify -CAfile <根证书文件> -untrusted <中间证书文件> <服务端证书文件>
```
`curl -k` 会跳过证书验证，只适合临时对照，不应作为修复方案。
---
## 七、Linux 网络配置与查看

### 7.1 ip 命令族
```bash
ip -br link
ip -br addr
ip addr show dev <网卡名>
ip -s link show dev <网卡名>
ip neigh show
```
临时启停接口：
```bash
sudo ip link set dev <网卡名> up
sudo ip link set dev <网卡名> down
```
删除异常邻居缓存以触发重新解析：
```bash
sudo ip neigh del 192.168.10.1 dev <网卡名>
```
修改命令仅应在实验或明确影响范围时使用。
### 7.2 ss 查看套接字
```bash
sudo ss -lntup
ss -ant
ss -ant '( dport = :<端口> or sport = :<端口> )'
ss -ti dst <目标IP>
```
常用参数：
- `-l`：只看监听套接字。
- `-n`：数字显示，不做名称解析。
- `-t`：TCP。
- `-u`：UDP。
- `-p`：显示进程，需要相应权限。
### 7.3 路由表解读
```bash
ip route show
```
典型输出：
```text
default via 192.168.10.1 dev eth0 proto dhcp src 192.168.10.20 metric 100
192.168.10.0/24 dev eth0 proto kernel scope link src 192.168.10.20 metric 100
```
字段含义：
- `default`：没有更具体路由时使用。
- `via`：下一跳网关。
- `dev`：出口接口。
- `src`：优先源地址。
- `metric`：同等条件下通常越小越优先。
- `scope link`：可通过本地链路直接到达。
Linux 先进行最长前缀匹配，不是单纯选择最小 `metric`。
```bash
ip route get <目标IP>
ip rule show
ip route show table all
```
临时实验路由：
```bash
sudo ip route add 192.168.20.0/24 via 192.168.10.1 dev <网卡名>
sudo ip route del 192.168.20.0/24 via 192.168.10.1 dev <网卡名>
```
### 7.4 NetworkManager 与 systemd-networkd
NetworkManager 常用于桌面和服务器，通过连接配置管理网卡、DNS 与路由，命令是 `nmcli`。
```bash
nmcli device status
nmcli connection show
nmcli connection show <连接名>
```
`systemd-networkd` 常用于精简服务器，配置通常位于 `/etc/systemd/network/`。
```bash
networkctl list
networkctl status <网卡名>
```
判断当前管理器：
```bash
systemctl is-active NetworkManager
systemctl is-active systemd-networkd
```
同一接口不应由多个管理器同时配置。远程修改 IP、路由前必须准备控制台或回滚方案。
---
## 八、防火墙基础

### 8.1 包过滤概念
防火墙可按地址、协议、端口、接口和连接状态匹配数据包。
- `ACCEPT`：允许。
- `DROP`：静默丢弃，客户端常表现为超时。
- `REJECT`：明确拒绝，客户端较快收到错误。
- `LOG`：记录日志，通常还需配合后续动作。
云环境还可能有安全组、网络 ACL、负载均衡规则和应用访问控制。
### 8.2 iptables 的表和链
常见表：
- `filter`：普通包过滤。
- `nat`：地址转换。
- `mangle`：修改标记或部分字段。
- `raw`：连接跟踪前的特殊处理。
常见链：
- `INPUT`：目标为本机。
- `OUTPUT`：本机产生。
- `FORWARD`：由本机转发。
- `PREROUTING`：路由决策前，常用于 DNAT。
- `POSTROUTING`：路由决策后，常用于 SNAT。
```bash
sudo iptables -L -n -v --line-numbers
sudo iptables -t nat -L -n -v --line-numbers
sudo iptables-save
```
不要在不了解持久化方式和 SSH 影响时清空规则。
### 8.3 nftables
nftables 是现代 Linux 推荐的包过滤框架，可统一处理多种协议族。
```bash
sudo nft list ruleset
sudo nft list tables
```
系统可能通过 firewalld、ufw 或 `iptables-nft` 管理 nftables，不要同时手工修改多个管理层。
### 8.4 firewalld
```bash
sudo firewall-cmd --state
sudo firewall-cmd --get-active-zones
sudo firewall-cmd --list-all
sudo firewall-cmd --add-port=<端口>/tcp
sudo firewall-cmd --permanent --add-port=<端口>/tcp
sudo firewall-cmd --reload
sudo firewall-cmd --query-port=<端口>/tcp
```
删除永久规则：
```bash
sudo firewall-cmd --permanent --remove-port=<端口>/tcp
sudo firewall-cmd --reload
```
### 8.5 ufw
```bash
sudo ufw status verbose
sudo ufw status numbered
sudo ufw allow <端口>/tcp
sudo ufw allow from 192.168.10.0/24 to any port <端口> proto tcp
sudo ufw delete allow <端口>/tcp
```
启用 ufw 前，应先确认 SSH 放行规则和控制台回滚能力。
---
## 九、NAT、端口转发与云安全组

### 9.1 NAT 概念
NAT 修改源地址或目标地址，并利用连接跟踪处理返回流量。
- SNAT：修改源地址，常用于内网访问外部网络。
- MASQUERADE：适合出口地址动态变化的 SNAT。
- DNAT：修改目标地址，常用于发布内部服务。
- PAT：多个连接共享地址并通过端口区分。
外部主机通常不能直接访问私有地址，需要 DNAT、端口映射、负载均衡或 VPN 等入口。
### 9.2 Linux 转发
```bash
sysctl net.ipv4.ip_forward
sudo sysctl -w net.ipv4.ip_forward=1
```
第二条只适合实验中的临时启用。仅开启转发还不够，还需正确路由、FORWARD 规则和必要的 NAT。
端口转发失败时检查：
1. 外部流量是否到达转发主机。
2. DNAT 规则计数器是否增加。
3. 内核转发是否开启。
4. FORWARD 链是否允许。
5. 转发主机是否能到后端。
6. 后端是否监听正确地址和端口。
7. 后端回程是否经过正确路径。
8. 是否需要 SNAT 避免非对称回程。
```bash
sudo tcpdump -ni <外部网卡> tcp port <外部端口>
sudo tcpdump -ni <内部网卡> host 192.168.20.20 and tcp port <后端端口>
```
### 9.3 云安全组的关系
云安全组位于云虚拟网络层，主机防火墙位于操作系统内部。
请求成功通常要求安全组、网络 ACL、路由、主机防火墙和服务监听全部正确。
常见误区：
- 安全组开放不代表服务已经监听。
- 主机防火墙开放不代表安全组已经开放。
- 服务只监听 `127.0.0.1` 时，外部仍无法访问。
- NAT 网关通常解决出站访问，不自动发布入站服务。
- 安全组常为有状态策略，网络 ACL 的行为应查阅云平台文档。
---
## 十、网络排障方法论

推荐顺序：`ping → 路由 → 端口 → 服务 → 应用`，同时先检查本机配置和 DNS。
### 10.1 明确现象
记录客户端、目标域名和 IP、协议、端口、路径、发生时间、失败范围、原始错误和近期变更。
不要只写“网络不通”，应写成“DNS 返回某地址，TCP 连接三秒后超时”。
### 10.2 检查本机配置
```bash
ip -br link
ip -br addr
ip route show
resolvectl status
```
判断：网卡为 `UP`；地址和前缀正确；存在默认路由或更具体路由；DNS 符合当前环境。
```bash
nmcli device status
nmcli connection show --active
```
### 10.3 检查 DNS
```bash
getent ahosts <域名>
dig <域名>
```
判断是否解析成功、地址是否预期、是否同时返回 IPv4 和 IPv6、不同 DNS 的结果是否一致。
```bash
curl --resolve <域名>:443:192.168.10.20 https://<域名>/
```
绕过 DNS 后成功，重点检查记录、缓存和解析路径。
### 10.4 检查 ping 与邻居
```bash
ping -c 4 127.0.0.1
ping -c 4 192.168.10.1
ping -c 4 <目标IP>
ip neigh show
```
判断：
- 回环失败说明本机环境异常。
- 网关失败可能是本机配置、邻居解析、网关或 ICMP 策略问题。
- 目标不回 ICMP 不等于实际 TCP 服务失败。
- 延迟和丢包要多次观察并与正常基线比较。
### 10.5 检查路由与路径
```bash
ip route get <目标IP>
traceroute -n <目标IP>
mtr -rwzc 20 <目标IP>
traceroute -T -p <端口> <目标主机>
```
判断出口接口、下一跳和源地址是否正确，并观察异常从哪一跳开始。
中间跳不响应可能只是限制 ICMP；最终目标持续丢包更能说明端到端问题。
### 10.6 检查端口
```bash
nc -vz -w 3 <目标主机> <端口>
curl -v --connect-timeout 3 https://<域名>:<端口>/<路径>
```
- 成功：TCP 握手完成，继续检查 TLS 或应用。
- `Connection refused`：端口未监听、监听地址错误或策略主动拒绝。
- 超时：检查路由、防火墙、安全组和回程路径。
### 10.7 检查服务端
```bash
sudo ss -lntup
sudo ss -lntp '( sport = :<端口> )'
systemctl status <服务名> --no-pager
journalctl -u <服务名> --since "10 minutes ago" --no-pager
```
监听地址含义：
- `127.0.0.1:<端口>`：仅本机 IPv4。
- `192.168.10.20:<端口>`：仅指定地址。
- `0.0.0.0:<端口>`：所有 IPv4 地址。
- `[::1]:<端口>`：仅本机 IPv6。
- `[::]:<端口>`：所有 IPv6 地址，是否兼容 IPv4 取决于系统配置。
```bash
curl -v http://127.0.0.1:<端口>/<路径>
curl -v http://192.168.10.20:<端口>/<路径>
```
回环成功但接口地址失败时，重点检查监听地址和主机防火墙。
### 10.8 检查防火墙与云策略
```bash
sudo nft list ruleset
sudo iptables -L -n -v --line-numbers
sudo firewall-cmd --list-all
sudo ufw status verbose
```
只使用系统实际采用的管理工具。检查链方向、协议、端口、源地址、规则顺序和计数器。
同时确认安全组、网络 ACL 和负载均衡监听规则。
### 10.9 检查 TLS 与应用
```bash
curl -v https://<域名>:<端口>/<路径>
openssl s_client -connect <域名>:<端口> -servername <域名> </dev/null
```
判断失败发生在 DNS、TCP、TLS 还是 HTTP，核对证书、状态码来源、响应头和应用日志时间。
### 10.10 双端抓包
客户端：
```bash
sudo tcpdump -ni any host <目标IP> and tcp port <端口>
```
服务端：
```bash
sudo tcpdump -ni any host <客户端IP> and tcp port <端口>
```
对比判断：
- 客户端发出 `SYN`，服务端看不到：中间路径或入口策略拦截。
- 服务端收到 `SYN` 但不回应：检查监听、防火墙和内核状态。
- 服务端发出 `SYN, ACK`，客户端看不到：回程或出口策略异常。
- 握手正常但应用无响应：检查 TLS、线程、依赖和日志。
结束后恢复临时规则，保存输出和抓包，记录根因、恢复动作与预防措施。
---
## 十一、抓包入门

### 11.1 tcpdump 基础
```bash
tcpdump -D
sudo tcpdump -ni <网卡名>
sudo tcpdump -ni any
```
常用选项：
- `-i`：指定接口。
- `-n`、`-nn`：禁止名称和端口解析。
- `-c`：达到指定报文数后退出。
- `-s 0`：抓取完整报文。
- `-v`、`-vv`：显示更多细节。
- `-A`：按 ASCII 显示载荷。
- `-X`：显示十六进制和 ASCII。
- `-w`：保存 pcap。
- `-r`：读取 pcap。
### 11.2 过滤表达式
```bash
sudo tcpdump -ni any host 192.168.10.20
sudo tcpdump -ni any src host 192.168.10.20
sudo tcpdump -ni any dst host 192.168.10.20
sudo tcpdump -ni any net 192.168.10.0/24
sudo tcpdump -ni any tcp port <端口>
sudo tcpdump -ni any udp port <端口>
sudo tcpdump -ni any icmp
```
组合条件：
```bash
sudo tcpdump -ni any 'host 192.168.10.20 and tcp port <端口>'
sudo tcpdump -ni any 'src net 192.168.10.0/24 and (tcp port 80 or tcp port 443)'
sudo tcpdump -ni any 'not port 22'
```
远程抓包时可排除当前 SSH 端口，避免产生大量自身流量。
只看 SYN 或 RST：
```bash
sudo tcpdump -ni any 'tcp[tcpflags] & tcp-syn != 0'
sudo tcpdump -ni any 'tcp[tcpflags] & tcp-rst != 0'
```
### 11.3 保存 pcap
```bash
sudo tcpdump -ni <网卡名> -s 0 -w /tmp/network-check.pcap \
  'host 192.168.10.20 and tcp port <端口>'
```
限制数量并读取：
```bash
sudo tcpdump -ni <网卡名> -c 200 -s 0 -w /tmp/network-check.pcap host 192.168.10.20
tcpdump -nnr /tmp/network-check.pcap
```
把 pcap 安全复制到分析机后可用 Wireshark 打开。
抓包可能包含内部地址、域名、令牌和业务数据，应限制范围、时长和访问权限。
### 11.4 常见抓包线索
正常 TCP 握手：`SYN → SYN, ACK → ACK`。
- 重复 `SYN`：连接请求未获得正常响应。
- `RST`：连接被主动重置。
- `Dup ACK`：可能存在丢包或乱序。
- `Retransmission`：发送方因未及时收到确认而重传。
- `Zero Window`：接收方暂时无法继续接收数据。
Wireshark 标记只是线索，应结合双端抓包、日志和链路背景判断。
---
## 十二、常用工具速查

| 工具 | 用途 | 示例 | 关键判断 |
|---|---|---|---|
| `ping` | ICMP 可达性与延迟 | `ping -c 4 <目标IP>` | 回复、丢包、延迟 |
| `traceroute` | 路径跳点 | `traceroute -n <目标IP>` | 异常从哪一跳开始 |
| `mtr` | 连续路径质量 | `mtr -rwzc 20 <目标IP>` | 端到端丢包趋势 |
| `ip` | 接口、地址、路由、邻居 | `ip route get <目标IP>` | 出口、网关、源地址 |
| `ss` | 套接字和 TCP 状态 | `ss -lntup` | 监听地址、端口、进程 |
| `dig` | DNS 查询 | `dig +short <域名>` | 记录、TTL、状态 |
| `curl` | HTTP、HTTPS 调试 | `curl -v https://<域名>/` | DNS、连接、TLS、状态码 |
| `tcpdump` | 抓取报文 | `tcpdump -ni any host <目标IP>` | 包是否经过、握手阶段 |
| `nc` | 端口测试 | `nc -vz <目标主机> <端口>` | 成功、拒绝、超时 |
常用组合：
```bash
ping -c 4 -W 2 <目标IP>
traceroute -T -p <端口> <目标IP>
mtr -rwzc 20 <目标IP>
ss -s
sudo ss -lntup
dig +short <域名>
curl -L --connect-timeout 3 --max-time 10 https://<域名>/
nc -vz -w 3 <目标主机> <端口>
```
`nc` 不同实现的参数可能略有差异，应查看本机 `nc -h` 或手册页。
---
## 十三、学习路线与实践任务

所有修改类任务都应在本机、虚拟机或 `192.168.x.x` 实验网络中完成。
### 13.1 接口、地址和子网
- [ ] 用 `ip -br link` 记录所有接口和状态。
- [ ] 用 `ip -br addr` 区分回环、物理和虚拟接口。
- [ ] 用 `ip -s link` 查看收发包、丢包和错误计数。
- [ ] 手工计算 `192.168.10.70/26` 的网络地址和可用范围。
- [ ] 手工计算 `192.168.20.130/27` 的网络地址和可用范围。
- [ ] 使用 `ipcalc` 验证结果。
- [ ] 判断 `192.168.10.20/25` 与 `192.168.10.200/25` 是否直连。
- [ ] 在快照保护下临时添加并删除一个实验地址。
### 13.2 路由与连通性
- [ ] 找出默认网关、直连路由和默认源地址。
- [ ] 用 `ip route get 192.168.10.20` 解释实际选路。
- [ ] 分别 ping 回环、本机地址、网关和另一台实验机。
- [ ] 对同一目标执行 `ping`、`traceroute -n` 和 `mtr`。
- [ ] 记录三种工具提供的信息差异。
- [ ] 在双网卡实验机观察最长前缀匹配与 `metric`。
- [ ] 临时添加并删除一条到 `192.168.20.0/24` 的路由。
### 13.3 TCP、UDP 与服务
- [ ] 用 `nc -lv <端口>` 在实验机监听。
- [ ] 从另一台实验机连接 `192.168.10.20:<端口>`。
- [ ] 用 `ss -lntp` 找到监听端口和进程。
- [ ] 用 `ss -ant` 观察连接状态变化。
- [ ] 对未监听端口执行 `nc -vz`，比较拒绝和超时。
- [ ] 抓取一次完整的三次握手和四次挥手。
- [ ] 统计 `ESTAB`、`TIME-WAIT` 和 `CLOSE-WAIT`。
- [ ] 用 UDP 实验理解“无连接”测试的局限。
### 13.4 DNS、HTTP 与 TLS
- [ ] 用 `getent hosts <域名>` 查看系统解析结果。
- [ ] 用 `dig` 查询 `A`、`AAAA`、`MX` 和 `TXT`。
- [ ] 对比默认 DNS 和指定 DNS 的结果。
- [ ] 查看 `/etc/resolv.conf` 的来源。
- [ ] 用 `curl -I` 查看响应头。
- [ ] 用 `curl -v` 区分 DNS、TCP、TLS 和 HTTP 阶段。
- [ ] 用 `curl -w` 输出各阶段耗时。
- [ ] 用 `curl --resolve` 把测试域名指向 `192.168.10.20`。
- [ ] 用 `openssl s_client` 检查证书链、有效期和 SAN。
- [ ] 写出 `404`、`502`、`504` 各自的排查起点。
### 13.5 防火墙、NAT 与抓包
- [ ] 判断实验机使用 nftables、firewalld 还是 ufw。
- [ ] 只读查看规则集和计数器。
- [ ] 在有回滚能力的实验机开放测试端口并验证。
- [ ] 删除测试规则并确认恢复。
- [ ] 比较 `DROP` 和 `REJECT` 的客户端表现。
- [ ] 查看 `net.ipv4.ip_forward`。
- [ ] 画出 SNAT 和 DNAT 前后的地址变化。
- [ ] 分别按主机、网段、端口和协议过滤抓包。
- [ ] 保存一个不超过 200 个报文的 pcap。
- [ ] 用 Wireshark 找出 TCP 握手、TLS 握手和 HTTP 响应。
### 13.6 综合故障演练
- [ ] 模拟错误 DNS 记录，按分层方法定位并恢复。
- [ ] 模拟服务只监听 `127.0.0.1`，从远端验证并修复。
- [ ] 模拟端口未开放，对比 `ss`、`nc`、规则计数器和抓包。
- [ ] 为演练写出“现象、假设、验证、根因、恢复、预防”。
建议顺序：先掌握 `ip`、`ping`、`ss`，再学 `dig`、`curl`、`nc`，随后学习路由、防火墙、NAT，最后用 `tcpdump` 串联各层现象。
---
## 十四、总结

运维网络排障链路可以概括为：
```text
本机接口与地址
    ↓
DNS 与目标地址
    ↓
路由、网关与链路
    ↓
TCP 或 UDP 端口
    ↓
服务监听与防火墙
    ↓
TLS 与应用协议
    ↓
日志、抓包与根因
```
每个工具回答不同问题：
- `ip`：接口、地址、邻居和路由是否正确。
- `ping`：ICMP 层面的可达性、延迟和丢包。
- `traceroute`、`mtr`：路径和端到端质量。
- `ss`：端口、进程和连接状态。
- `dig`、`getent`：名称解析。
- `nc`：基础端口连接。
- `curl`：HTTP、HTTPS 各阶段。
- `openssl`：TLS 协商和证书链。
- `tcpdump`：数据包是否真正经过某处。
最后牢记：先确认现象再提出假设；同时关注去程和回程；区分端口开放、服务监听和应用正常；修改网络前准备回滚；使用同一时间范围关联命令输出、日志与抓包；恢复后记录根因和预防措施。
