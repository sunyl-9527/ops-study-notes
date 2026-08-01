# Nginx 入门学习指南

## 目录

- [1. Nginx 是什么](#1-nginx-是什么)
- [2. 核心概念](#2-核心概念)
  - [2.1 Web 服务器模式](#21-web-服务器模式)
  - [2.2 反向代理模式](#22-反向代理模式)
  - [2.3 负载均衡](#23-负载均衡)
- [3. 安装](#3-安装)
  - [3.1 Linux（Ubuntu/Debian）](#31-linuxubuntudebian)
  - [3.2 Linux（CentOS/RHEL）](#32-linuxcentosrhel)
  - [3.3 Windows](#33-windows)
- [4. 配置文件结构](#4-配置文件结构)
  - [4.1 整体结构](#41-整体结构)
  - [4.2 四层嵌套关系](#42-四层嵌套关系)
  - [4.3 location 匹配规则](#43-location-匹配规则)
- [5. 常用场景配置示例](#5-常用场景配置示例)
  - [5.1 托管静态网站](#51-托管静态网站)
  - [5.2 反向代理后端应用](#52-反向代理后端应用)
  - [5.3 负载均衡](#53-负载均衡)
  - [5.4 同一端口区分前后端](#54-同一端口区分前后端)
  - [5.5 HTTPS 配置](#55-https-配置)
- [6. 关键指令速查](#6-关键指令速查)
- [7. 常用内置变量](#7-常用内置变量)
- [8. 调试技巧](#8-调试技巧)
- [9. 与 Apache 的简要对比](#9-与-apache-的简要对比)
- [10. 学习路径建议](#10-学习路径建议)
- [11. 动手实践：从零到反向代理](#11-动手实践从零到反向代理)
  - [实践 1：安装并验证 Nginx 运行](#实践-1安装并验证-nginx-运行)
  - [实践 2：托管你自己的静态页面](#实践-2托管你自己的静态页面)
  - [实践 3：配置反向代理](#实践-3配置反向代理代理一个-python-小服务)
  - [实践 4：同一端口区分静态文件和后端](#实践-4同一端口区分静态文件和后端)
  - [实践 5：负载均衡](#实践-5负载均衡用两个-python-进程模拟)
  - [实践 6：查看日志](#实践-6查看日志理解请求流程)
  - [实践 7：清理和整理](#实践-7清理和整理)
  - [实践检查清单](#实践检查清单)

---

## 1. Nginx 是什么

Nginx（读作 "engine-x"）是一款高性能的开源软件，最初由 Igor Sysoev 于 2004 年发布。它能扮演多种角色：

- **Web 服务器**：直接把静态文件（HTML、CSS、图片等）响应给浏览器
- **反向代理**：把客户端请求转发给后端应用（Node.js、Python、Java 等），再把响应返回给客户端
- **负载均衡器**：把流量分发到多台后端服务器，避免单点过载
- **HTTP 缓存**：缓存后端响应，减少后端压力

与 Apache 的"一请求一线程/进程"不同，Nginx 采用**事件驱动、异步非阻塞**架构，用少量 worker 进程处理海量并发连接，在高并发场景下内存占用极低。

---

## 2. 核心概念

### 2.1 Web 服务器模式

```
浏览器  ──请求──▶  Nginx  ──读取──▶  磁盘上的静态文件
                     │
                  直接响应
```

适合托管纯静态网站、前端 SPA 构建产物（`dist/` 目录）。

### 2.2 反向代理模式

```
浏览器  ──请求──▶  Nginx  ──转发──▶  后端应用（Node.js / Python / Java）
                     ▲                        │
                     └────────────响应─────────┘
```

客户端只看到 Nginx，不直接接触后端服务。好处：
- 统一入口，可集中处理 HTTPS、鉴权、限流
- 隐藏后端真实地址和端口
- 可同时代理多个后端服务

### 2.3 负载均衡

```
                       ┌─▶  后端服务器 A
浏览器 ──▶  Nginx ─────┼─▶  后端服务器 B
                       └─▶  后端服务器 C
```

支持多种策略：
- `round-robin`（默认）：轮流分发
- `least_conn`：优先发给连接数最少的服务器
- `ip_hash`：同一客户端 IP 固定路由到同一服务器（用于 Session 保持）
- `weight`：按权重分发

---

## 3. 安装

### 3.1 Linux（Ubuntu/Debian）

```bash
# 更新包索引
sudo apt update

# 安装
sudo apt install nginx -y

# 启动并设置开机自启
sudo systemctl start nginx
sudo systemctl enable nginx

# 验证是否运行
sudo systemctl status nginx
# 或直接在浏览器访问 http://localhost，看到 "Welcome to nginx!" 即成功
```

常用管理命令：

```bash
sudo nginx -t              # 检查配置文件语法
sudo nginx -s reload       # 热重载配置（不中断连接）
sudo nginx -s stop         # 立即停止
sudo systemctl restart nginx  # 完全重启
```

### 3.2 Linux（CentOS/RHEL）

```bash
sudo yum install epel-release -y
sudo yum install nginx -y
sudo systemctl start nginx
sudo systemctl enable nginx
```

### 3.3 Windows

1. 前往 [nginx.org/en/docs/windows.html](https://nginx.org/en/docs/windows.html) 下载 zip 包
2. 解压到任意目录，例如 `C:\nginx`
3. 在该目录下运行：

```powershell
# 启动
.\nginx.exe

# 热重载配置
.\nginx.exe -s reload

# 停止
.\nginx.exe -s stop

# 检查配置语法
.\nginx.exe -t
```

> Windows 版 Nginx 主要用于开发调试，生产环境推荐 Linux。

---

## 4. 配置文件结构

主配置文件路径：
- Linux：`/etc/nginx/nginx.conf`
- Windows：`C:\nginx\conf\nginx.conf`

### 4.1 整体结构

```nginx
# 全局块（主进程配置）
worker_processes auto;

# events 块（连接处理）
events {
    worker_connections 1024;
}

# http 块（HTTP 服务配置）
http {
    include       mime.types;
    default_type  application/octet-stream;

    # server 块（虚拟主机，可以有多个）
    server {
        listen       80;
        server_name  example.com;

        # location 块（路由规则，可以有多个）
        location / {
            root   /var/www/html;
            index  index.html;
        }
    }
}
```

### 4.2 四层嵌套关系

```
nginx.conf
└── http {}          ← HTTP 全局设置
    ├── server {}    ← 一个虚拟主机（类似 Apache 的 VirtualHost）
    │   ├── location /api/ {}    ← 匹配特定路径的规则
    │   └── location / {}
    └── server {}    ← 另一个虚拟主机（不同域名或端口）
```

### 4.3 location 匹配规则

| 写法 | 类型 | 优先级 |
|------|------|--------|
| `location = /exact` | 精确匹配 | 最高 |
| `location ^~ /prefix` | 前缀匹配（禁止后续正则） | 次高 |
| `location ~ \.php$` | 正则匹配（区分大小写） | 中 |
| `location ~* \.jpg$` | 正则匹配（不区分大小写） | 中 |
| `location /prefix` | 普通前缀匹配 | 低 |

---

## 5. 常用场景配置示例

### 5.1 托管静态网站

```nginx
server {
    listen 80;
    server_name mysite.com www.mysite.com;

    root /var/www/mysite;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }

    # 静态资源缓存 30 天
    location ~* \.(jpg|jpeg|png|gif|css|js|ico|svg)$ {
        expires 30d;
        add_header Cache-Control "public";
    }
}
```

### 5.2 反向代理后端应用

```nginx
server {
    listen 80;
    server_name api.mysite.com;

    location / {
        proxy_pass http://127.0.0.1:3000;  # 后端监听 3000 端口

        # 把真实客户端信息传给后端
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # 超时设置
        proxy_connect_timeout 10s;
        proxy_read_timeout    60s;
    }
}
```

### 5.3 负载均衡

```nginx
# 定义后端服务器组
upstream backend_pool {
    least_conn;                      # 策略：最少连接
    server 192.168.1.10:8080 weight=3;
    server 192.168.1.11:8080 weight=1;
    server 192.168.1.12:8080 backup; # 备用服务器
}

server {
    listen 80;
    server_name mysite.com;

    location / {
        proxy_pass http://backend_pool;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

### 5.4 同一端口区分前后端

```nginx
server {
    listen 80;
    server_name mysite.com;

    # /api 开头的请求转发给后端
    location /api/ {
        proxy_pass http://127.0.0.1:3000/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    # 其余请求服务前端静态文件
    location / {
        root /var/www/frontend/dist;
        index index.html;
        try_files $uri $uri/ /index.html;  # SPA 路由支持
    }
}
```

### 5.5 HTTPS 配置

```nginx
server {
    listen 80;
    server_name mysite.com;
    return 301 https://$host$request_uri;  # HTTP 强制跳转 HTTPS
}

server {
    listen 443 ssl;
    server_name mysite.com;

    ssl_certificate     /etc/ssl/certs/mysite.crt;
    ssl_certificate_key /etc/ssl/private/mysite.key;

    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_ciphers         HIGH:!aNULL:!MD5;

    location / {
        root /var/www/mysite;
        index index.html;
    }
}
```

---

## 6. 关键指令速查

| 指令 | 作用 | 示例 |
|------|------|------|
| `listen` | 监听端口/地址 | `listen 80;` / `listen 443 ssl;` |
| `server_name` | 虚拟主机域名 | `server_name example.com *.example.com;` |
| `root` | 静态文件根目录 | `root /var/www/html;` |
| `index` | 默认首页文件 | `index index.html index.htm;` |
| `try_files` | 按顺序尝试文件，都找不到则返回最后一项 | `try_files $uri $uri/ /index.html;` |
| `proxy_pass` | 反向代理目标地址 | `proxy_pass http://127.0.0.1:3000;` |
| `proxy_set_header` | 修改或新增请求头 | `proxy_set_header X-Real-IP $remote_addr;` |
| `upstream` | 定义后端服务器组 | `upstream pool { server 10.0.0.1:8080; }` |
| `return` | 直接返回状态码或跳转 | `return 301 https://$host$request_uri;` |
| `rewrite` | URL 重写 | `rewrite ^/old(.*)$ /new$1 permanent;` |
| `expires` | 设置响应缓存时间 | `expires 7d;` |
| `worker_processes` | worker 进程数，`auto` 等于 CPU 核心数 | `worker_processes auto;` |
| `worker_connections` | 每个 worker 最大连接数 | `worker_connections 1024;` |
| `gzip` | 开启 gzip 压缩 | `gzip on;` |
| `include` | 引入其他配置文件 | `include /etc/nginx/conf.d/*.conf;` |

---

## 7. 常用内置变量

| 变量 | 含义 |
|------|------|
| `$host` | 请求的 Host 头 |
| `$remote_addr` | 客户端 IP |
| `$request_uri` | 完整请求 URI（含参数） |
| `$uri` | 当前 URI（不含参数） |
| `$args` | 查询字符串 |
| `$scheme` | 协议（http 或 https） |
| `$server_port` | 服务器端口 |
| `$http_user_agent` | 客户端 User-Agent |

---

## 8. 调试技巧

```bash
# 1. 永远先检查配置语法再 reload
sudo nginx -t

# 2. 查看错误日志（最有用的调试入口）
sudo tail -f /var/log/nginx/error.log

# 3. 查看访问日志
sudo tail -f /var/log/nginx/access.log

# 4. 测试某个域名的路由
curl -H "Host: mysite.com" http://127.0.0.1/
```

---

## 9. 与 Apache 的简要对比

| 维度 | Nginx | Apache |
|------|-------|--------|
| 并发模型 | 异步事件驱动 | 线程/进程 |
| 高并发内存 | 极低 | 较高 |
| 静态文件 | 非常快 | 快 |
| 动态内容 | 需反向代理 | 可通过模块直接处理（mod_php） |
| 配置热重载 | 支持（`-s reload`） | 支持 |
| `.htaccess` | 不支持 | 支持 |
| 学习曲线 | 相对平缓 | 略陡 |

---

## 10. 学习路径建议

1. **第一步**：在本机装好 Nginx，能看到默认欢迎页
2. **第二步**：托管一个本地静态 HTML 文件，理解 `root` + `location`
3. **第三步**：跑一个本地 Node.js/Python 应用，配置反向代理访问它
4. **第四步**：学习 `upstream` 负载均衡，模拟多后端
5. **第五步**：用 Let's Encrypt（`certbot`）配置真实 HTTPS

---

## 11. 动手实践：从零到反向代理

> 以下步骤在 **Ubuntu/Debian** 上操作，每步都能立即验证结果。Windows 用户看括号里的替代命令。

---

### 实践 1：安装并验证 Nginx 运行

```bash
# 安装
sudo apt update && sudo apt install nginx -y

# 启动
sudo systemctl start nginx

# 验证：用 curl 请求本机，看到 HTML 输出说明成功
curl http://localhost
```

**预期结果**：终端输出包含 `Welcome to nginx!` 的 HTML。

也可以在浏览器访问 `http://localhost` 或 `http://127.0.0.1`，看到欢迎页面。

---

### 实践 2：托管你自己的静态页面

**第一步**：创建网站目录和文件

```bash
sudo mkdir -p /var/www/mysite
sudo nano /var/www/mysite/index.html
```

写入以下内容后保存（`Ctrl+O` 保存，`Ctrl+X` 退出）：

```html
<!DOCTYPE html>
<html>
<head><meta charset="UTF-8"><title>我的第一个 Nginx 网站</title></head>
<body>
  <h1>Hello, Nginx!</h1>
  <p>这是我自己配置的静态页面。</p>
</body>
</html>
```

**第二步**：创建 Nginx 配置文件

```bash
sudo nano /etc/nginx/conf.d/mysite.conf
```

写入：

```nginx
server {
    listen 8080;              # 用 8080 端口，避免和默认的 80 冲突
    server_name localhost;

    root /var/www/mysite;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

**第三步**：检查语法并重载

```bash
sudo nginx -t          # 出现 "syntax is ok" 才继续
sudo nginx -s reload
```

**第四步**：验证

```bash
curl http://localhost:8080
```

**预期结果**：看到你写的 HTML 内容，页面显示"Hello, Nginx!"。

---

### 实践 3：配置反向代理（代理一个 Python 小服务）

**第一步**：启动一个简单的 HTTP 后端（用 Python 内置模块）

新开一个终端窗口，运行：

```bash
# Python 3，在 9000 端口启动一个简单 HTTP 服务
python3 -m http.server 9000
```

这会把当前目录作为静态文件服务，监听 9000 端口。让这个窗口保持运行。

**第二步**：修改 Nginx 配置，加入反向代理

```bash
sudo nano /etc/nginx/conf.d/mysite.conf
```

在原有内容下方新增一个 server 块：

```nginx
server {
    listen 8081;
    server_name localhost;

    location / {
        proxy_pass http://127.0.0.1:9000;
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
    }
}
```

**第三步**：重载并验证

```bash
sudo nginx -t && sudo nginx -s reload

# 通过 Nginx 的 8081 端口访问后端服务
curl http://localhost:8081
```

**预期结果**：看到 Python HTTP 服务返回的文件列表 HTML。说明请求经过了 Nginx 转发给 Python 后端。

**理解发生了什么**：

```
你的 curl ──▶ Nginx:8081 ──▶ Python:9000
                               │
                             响应返回
```

---

### 实践 4：同一端口区分静态文件和后端

这个场景模拟前后端分离项目（前端静态 + 后端 API）。

**第一步**：新建 API 目录模拟后端数据

```bash
mkdir -p /tmp/fakeapi
echo '{"status": "ok", "message": "来自后端的数据"}' > /tmp/fakeapi/data.json

# 在 9001 端口启动 API 服务
cd /tmp/fakeapi && python3 -m http.server 9001
```

新开终端，保持上面这个服务运行。

**第二步**：新增配置

```bash
sudo nano /etc/nginx/conf.d/spa.conf
```

写入：

```nginx
server {
    listen 8082;
    server_name localhost;

    # /api/ 路径转发给后端
    location /api/ {
        proxy_pass http://127.0.0.1:9001/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    # 其他路径服务静态文件
    location / {
        root /var/www/mysite;
        index index.html;
        try_files $uri $uri/ /index.html;
    }
}
```

**第三步**：重载并分别测试两个路径

```bash
sudo nginx -t && sudo nginx -s reload

# 访问静态页面（走 /var/www/mysite）
curl http://localhost:8082/

# 访问 API（走 Python 后端）
curl http://localhost:8082/api/data.json
```

**预期结果**：
- 第一条：返回你的 HTML 页面
- 第二条：返回 `{"status": "ok", "message": "来自后端的数据"}`

---

### 实践 5：负载均衡（用两个 Python 进程模拟）

**第一步**：准备两个不同内容的后端

```bash
mkdir -p /tmp/server1 /tmp/server2
echo "我是服务器 A (端口 9100)" > /tmp/server1/index.html
echo "我是服务器 B (端口 9101)" > /tmp/server2/index.html

# 分别在两个终端窗口里启动
cd /tmp/server1 && python3 -m http.server 9100
cd /tmp/server2 && python3 -m http.server 9101
```

**第二步**：配置负载均衡

```bash
sudo nano /etc/nginx/conf.d/lb.conf
```

写入：

```nginx
upstream my_servers {
    server 127.0.0.1:9100;
    server 127.0.0.1:9101;
}

server {
    listen 8083;
    server_name localhost;

    location / {
        proxy_pass http://my_servers;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

**第三步**：重载并连续请求，观察轮询

```bash
sudo nginx -t && sudo nginx -s reload

# 连续请求 4 次，观察响应在两台服务器之间交替
for i in 1 2 3 4; do curl -s http://localhost:8083/index.html; done
```

**预期结果**：

```
我是服务器 A (端口 9100)
我是服务器 B (端口 9101)
我是服务器 A (端口 9100)
我是服务器 B (端口 9101)
```

请求在 A 和 B 之间轮流分发，这就是默认的 round-robin 负载均衡。

---

### 实践 6：查看日志，理解请求流程

```bash
# 实时查看访问日志（另开一个终端）
sudo tail -f /var/log/nginx/access.log

# 在另一个终端发送请求
curl http://localhost:8082/
curl http://localhost:8083/index.html
```

日志格式解读：

```
127.0.0.1 - - [28/Jun/2026:10:00:01 +0800] "GET / HTTP/1.1" 200 312 "-" "curl/7.81.0"
│           │   │                            │   │    │
客户端IP    时间  请求行                     状态码 响应大小 User-Agent
```

**故意制造一个 404**：

```bash
curl http://localhost:8082/notexist.html
# 日志里会出现 404
```

然后查看错误日志：

```bash
sudo tail -f /var/log/nginx/error.log
```

---

### 实践 7：清理和整理

实践完成后，停掉所有 Python 服务（`Ctrl+C`），清理临时配置：

```bash
# 删除实践用的配置文件
sudo rm /etc/nginx/conf.d/mysite.conf
sudo rm /etc/nginx/conf.d/spa.conf
sudo rm /etc/nginx/conf.d/lb.conf

# 重载，恢复默认状态
sudo nginx -t && sudo nginx -s reload

# 验证 80 端口默认页还在
curl http://localhost
```

---

### 实践检查清单

完成以下每项说明你已掌握对应概念：

- [ ] 安装 Nginx，浏览器能看到欢迎页
- [ ] 修改配置托管自己的 HTML，用 `curl` 能访问
- [ ] 配置反向代理，请求经过 Nginx 转发到后端
- [ ] 同一端口区分 `/api/` 和静态文件路径
- [ ] 配置 `upstream`，多次请求能看到轮询效果
- [ ] 会看 `access.log` 和 `error.log`
- [ ] `nginx -t` 检查语法成为你每次改配置后的条件反射

---

*参考资料：nginx.org 官方文档 / DigitalOcean Community Tutorials*
