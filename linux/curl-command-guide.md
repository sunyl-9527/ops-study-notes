# curl 与 wget 命令详解

> 适用场景：网络连通性测试、HTTP/HTTPS 接口排查、文件下载、断点续传、限速重试、镜像抓取、带认证访问
> 适用平台：Linux / macOS / Windows 10+（Windows 建议显式使用 `curl.exe`）
> 核心理解：`curl` 更擅长接口测试与请求构造，`wget` 更擅长稳定下载与递归抓取

---

## 1. 基本语法

```bash
curl [选项] URL
```

最简单的例子：

```bash
curl https://www.baidu.com
```

默认行为：

- 向目标 URL 发起请求
- 把响应内容输出到终端
- 默认使用 `GET`

---

## 2. 示例域名约定

为了让示例更接近真实使用，本文统一按场景选用这些真实域名：

- 通用网页测试：`https://www.baidu.com`
- API / JSON 回显测试：`https://httpbin.org`
- 下载测试（小文件）：`https://ftp.gnu.org/gnu/wget/README`
- 下载测试（续传 / 限速）：`https://ftp.gnu.org/gnu/wget/wget-latest.tar.gz`

这样做的好处是：

- 复制后更容易直接试
- 一眼能看出命令适合什么场景
- 不会把“网页测试”和“接口测试”混在一起

---

## 3. 最常用参数

```bash
-I                 只看响应头
-i                 输出响应头和响应体
-v                 显示详细调试信息
-L                 跟随重定向
-o FILE            保存为指定文件名
-O                 使用远端原文件名保存
-C -               断点续传
-k                 忽略 HTTPS 证书校验
-u user:pass       Basic Auth 认证
-H "K: V"          添加请求头
-d 'data'          发送请求体，常用于 POST
-X METHOD          指定请求方法
-m SECONDS         设置最大耗时
--connect-timeout  设置连接超时
--retry N          失败时重试
--limit-rate 1M    限制下载速度
-A "UA"            指定 User-Agent
```

---

## 4. 用于网络测试的常见操作

### 4.1 测试网站是否能访问

```bash
curl https://www.baidu.com
```

适合快速确认：

- DNS 是否解析正常
- 目标站点是否返回内容
- 本机到目标服务是否基本可达

### 4.2 只看响应头

```bash
curl -I https://www.baidu.com
```

适合确认：

- HTTP 状态码
- `Server`
- `Content-Type`
- `Content-Length`
- 是否有重定向

### 4.3 查看完整调试过程

```bash
curl -v https://www.baidu.com
```

会看到：

- DNS 解析
- TCP 连接
- TLS 握手
- 请求头
- 响应头

排查 HTTPS、重定向、代理问题时很有用。

### 4.4 只输出状态码

```bash
curl -o /dev/null -s -w "%{http_code}\n" https://www.baidu.com
```

Windows PowerShell 可用：

```powershell
curl.exe -o NUL -s -w "%{http_code}`n" https://www.baidu.com
```

常见状态码：

- `200`：成功
- `301/302`：重定向
- `403`：禁止访问
- `404`：资源不存在
- `500`：服务端错误
- `502/504`：网关或上游异常

### 4.5 查看总耗时和关键阶段耗时

```bash
curl -o /dev/null -s -w "dns=%{time_namelookup}s connect=%{time_connect}s tls=%{time_appconnect}s start=%{time_starttransfer}s total=%{time_total}s\n" https://www.baidu.com
```

适合区分问题发生在哪一层：

- DNS 慢
- 建连慢
- TLS 握手慢
- 服务端处理慢

### 4.6 跟随重定向

```bash
curl -L http://www.baidu.com
```

有些站点会把：

- `http` 跳到 `https`
- 裸域名跳到 `www`
- 旧地址跳到新地址

不加 `-L` 时，你看到的可能只是 `301/302`。

### 4.7 忽略证书错误

```bash
curl -k https://self-signed.badssl.com
```

适用场景：

- 自签名证书
- 测试环境证书不完整

注意：`-k` 只适合测试，不建议在生产脚本中长期使用。

### 4.8 测试接口是否支持某种 HTTP 方法

```bash
curl -X OPTIONS -i https://httpbin.org/anything
```

也可以测试：

```bash
curl -I https://www.baidu.com
curl -X DELETE -i https://httpbin.org/delete
```

### 4.9 指定请求头测试接口

```bash
curl -H "Accept: application/json" https://httpbin.org/get
```

带 Token：

```bash
curl -H "Authorization: Bearer YOUR_TOKEN" https://httpbin.org/bearer
```

### 4.10 发送 JSON 请求测试接口

```bash
curl -X POST https://httpbin.org/post \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"<example-password>"}'
```

说明：

- `-d` 一般会隐式使用 `POST`
- JSON 接口通常要显式指定 `Content-Type`

### 4.11 测试超时控制

```bash
curl --connect-timeout 5 -m 15 https://www.baidu.com
```

含义：

- `--connect-timeout 5`：连接阶段最多等 5 秒
- `-m 15`：整个请求最多等 15 秒

### 4.12 失败时自动重试

```bash
curl --retry 3 --retry-delay 2 https://httpbin.org/status/503
```

适合：

- 网络偶发抖动
- 临时性 5xx 错误
- 下载大文件时偶发中断

### 4.13 指定 Host 解析测试

当你想绕过本地 DNS，直接验证某个 IP 上的站点时：

```bash
curl --resolve www.baidu.com:443:39.156.66.10 https://www.baidu.com
```

说明：

- `www.baidu.com` 是真实域名
- `39.156.66.10` 只是示例 IP
- 生产排查时应替换成你实际要验证的源站 IP

适合排查：

- CDN 回源问题
- 新 IP 是否已正确部署
- DNS 切换前的验证

### 4.14 通过代理测试

```bash
curl -x http://127.0.0.1:7890 https://www.baidu.com
```

带认证代理：

```bash
curl -x http://user:pass@127.0.0.1:8080 https://httpbin.org/ip
```

---

## 5. 用于 API 调试的常见操作

### 5.1 查看服务端收到的请求内容

```bash
curl -s https://httpbin.org/get
```

这个接口会把请求信息原样回显出来，适合学习和排查：

- 请求头是否真的发出去了
- Query 参数是否拼对
- 源 IP 是否符合预期

### 5.2 携带 Query 参数

```bash
curl -G https://httpbin.org/get \
  --data-urlencode "wd=curl 命令" \
  --data-urlencode "page=1"
```

说明：

- `-G` 会把 `-d` 提供的数据拼到 URL 查询串里
- `--data-urlencode` 适合带空格、中文、特殊字符的参数

### 5.3 提交表单

```bash
curl -X POST https://httpbin.org/post \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=admin&password=<example-password>"
```

### 5.4 发送 JSON

```bash
curl -X POST https://httpbin.org/post \
  -H "Content-Type: application/json" \
  -d '{"name":"alice","role":"admin"}'
```

### 5.5 测试 Basic Auth

```bash
curl -u user:passwd https://httpbin.org/basic-auth/user/passwd
```

### 5.6 使用 Cookie

先写入 Cookie：

```bash
curl -c cookies.txt "https://httpbin.org/cookies/set?theme=dark"
```

再带着 Cookie 访问：

```bash
curl -b cookies.txt https://httpbin.org/cookies
```

### 5.7 上传文件

```bash
curl -F "file=@./output.txt" https://httpbin.org/post
```

适合测试：

- 表单上传
- `multipart/form-data`
- 文件字段名是否正确

---

## 6. 用于下载文件的常见操作

### 6.1 下载到当前目录并沿用远端文件名

```bash
curl -O https://ftp.gnu.org/gnu/wget/README
```

### 6.2 下载并指定文件名

```bash
curl -o wget-readme.txt https://ftp.gnu.org/gnu/wget/README
```

区别：

- `-O`：保留原文件名
- `-o`：自定义本地文件名

### 6.3 下载时跟随重定向

```bash
curl -L -o github.html https://github.com
```

很多下载链接实际会先跳转到：

- 对象存储
- CDN
- 临时签名链接

下载类 URL 很常要配合 `-L`。

### 6.4 断点续传

```bash
curl -C - -O https://ftp.gnu.org/gnu/wget/wget-latest.tar.gz
```

适合：

- 下载大文件中断后继续
- 网络不稳定环境

### 6.5 限速下载

```bash
curl --limit-rate 1M -O https://ftp.gnu.org/gnu/wget/wget-latest.tar.gz
```

说明：

- `1M` 表示每秒约 1 MB
- 可写成 `500K`、`2M`、`10M`

### 6.6 下载失败自动重试

```bash
curl -L --retry 5 --retry-delay 3 -O https://ftp.gnu.org/gnu/wget/README
```

### 6.7 只在远端文件较新时才下载

```bash
curl -z wget-readme.txt -o wget-readme.txt https://ftp.gnu.org/gnu/wget/README
```

适合做简单同步。

### 6.8 带用户名密码下载

```bash
curl -u user:passwd -O https://httpbin.org/basic-auth/user/passwd
```

说明：

- 这个例子主要演示“带认证访问”
- 真正下载私有文件时，把 URL 换成你的实际下载地址

### 6.9 使用 Cookie 下载

```bash
curl -b cookies.txt -c cookies.txt -L -o cookies.json https://httpbin.org/cookies
```

说明：

- `-b`：读取 Cookie
- `-c`：保存 Cookie

适合有登录态的下载请求。

---

## 7. 实战示例

### 7.1 快速检查一个网站是否正常

```bash
curl -I -L --connect-timeout 5 -m 15 https://www.baidu.com
```

你可以从结果里重点看：

- 状态码是不是 `200`
- 有没有异常跳转
- 证书和响应头是否正常

### 7.2 输出状态码和总耗时

```bash
curl -o /dev/null -s -w "code=%{http_code} total=%{time_total}s\n" https://www.baidu.com
```

### 7.3 测试 JSON API

```bash
curl -s https://httpbin.org/get \
  -H "Accept: application/json"
```

提交 JSON：

```bash
curl -s https://httpbin.org/post \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{"name":"alice","role":"admin"}'
```

### 7.4 下载一个真实文件

```bash
curl -L -o wget-readme.txt https://ftp.gnu.org/gnu/wget/README
```

### 7.5 下载大文件并支持中断恢复

```bash
curl -L -C - --retry 5 -O https://ftp.gnu.org/gnu/wget/wget-latest.tar.gz
```

### 7.6 测试某个 IP 上的 HTTPS 站点

```bash
curl --resolve www.baidu.com:443:39.156.66.10 https://www.baidu.com -I
```

---

## 8. Windows 使用注意事项

### 8.1 建议显式使用 `curl.exe`

在部分 PowerShell 环境中，`curl` 可能被映射到 `Invoke-WebRequest` 别名。为避免混淆，建议写成：

```powershell
curl.exe -I https://www.baidu.com
```

### 8.2 丢弃输出的设备名不同

- Linux/macOS：`/dev/null`
- Windows：`NUL`

示例：

```powershell
curl.exe -o NUL -s -w "%{http_code}`n" https://www.baidu.com
```

### 8.3 JSON 引号问题

PowerShell 下建议优先用单引号包裹 JSON：

```powershell
curl.exe -X POST https://httpbin.org/post `
  -H "Content-Type: application/json" `
  -d '{"username":"admin","password":"<example-password>"}'
```

---

## 9. 常见问题

### 9.1 看到 `301/302`，但没拿到内容

原因通常是目标地址有重定向。加上：

```bash
curl -L URL
```

### 9.2 HTTPS 报证书错误

测试时可临时加：

```bash
curl -k URL
```

但如果是正式环境，应该优先修复证书链问题，而不是长期忽略校验。

### 9.3 接口返回 `403`

常见原因：

- 缺少认证头
- User-Agent 被拦截
- Referer / Cookie 缺失
- 源站或 WAF 限制

可先用：

```bash
curl -v URL
```

再按需补上 `-H`、`-A`、`-b`。

### 9.4 下载链接打不开

优先排查：

- 是否需要 `-L`
- 是否需要登录态 Cookie
- 是否需要认证头
- 是否被代理或防火墙拦截

---

## 10. 常用速查

```bash
# 看响应头
curl -I https://www.baidu.com

# 看详细过程
curl -v https://www.baidu.com

# 只看状态码
curl -o /dev/null -s -w "%{http_code}\n" https://www.baidu.com

# 下载文件
curl -O https://ftp.gnu.org/gnu/wget/README

# 指定文件名下载
curl -L -o wget-readme.txt https://ftp.gnu.org/gnu/wget/README

# 断点续传
curl -C - -O https://ftp.gnu.org/gnu/wget/wget-latest.tar.gz

# 发 JSON POST
curl -X POST https://httpbin.org/post \
  -H "Content-Type: application/json" \
  -d '{"name":"test"}'

# 忽略证书错误
curl -k https://self-signed.badssl.com
```

---

## 11. 一句话总结

如果你把 `curl` 先当成三个工具来记，会最好上手：

- 网页测试工具：看状态码、响应头、重定向、耗时、证书
- API 调试工具：发 GET / POST / JSON / 表单 / Cookie / 文件上传
- 下载工具：保存文件、跟随跳转、断点续传、限速、重试、带认证下载

熟练之后，它就是排查 HTTP/HTTPS 问题时最好用的命令之一。

---

## 12. wget 是什么

`wget` 也是一个命令行下载工具，但和 `curl` 的侧重点不一样：

- `curl`：更适合接口调试、构造请求、看响应细节
- `wget`：更适合下载文件、断点续传、批量抓取、递归镜像

如果你只是想“稳稳地把文件下下来”，很多时候 `wget` 会更顺手。

---

## 13. wget 基本语法

```bash
wget [选项] URL
```

最简单的例子：

```bash
wget https://ftp.gnu.org/gnu/wget/README
```

默认行为：

- 把远端文件下载到当前目录
- 默认沿用远端文件名
- 自动显示下载进度

---

## 14. wget 最常用参数

```bash
-O FILE                 保存为指定文件名
-c                      断点续传
-q                      静默模式
--show-progress         显示进度条
--limit-rate=1m         限速
--tries=5               失败重试次数
--timeout=15            超时时间
--no-check-certificate  忽略 HTTPS 证书校验
--user=NAME             指定用户名
--password=PASS         指定密码
--header="K: V"         添加请求头
--post-data='a=1&b=2'   发送 POST 表单数据
--mirror                镜像抓取
--convert-links         转换页面中的链接，方便离线查看
--no-parent             不回溯到父目录
-r                      递归下载
-l N                    递归深度
-b                      后台下载
```

---

## 15. wget 常见下载操作

### 15.1 直接下载文件

```bash
wget https://ftp.gnu.org/gnu/wget/README
```

### 15.2 指定本地文件名

```bash
wget -O wget-readme.txt https://ftp.gnu.org/gnu/wget/README
```

### 15.3 断点续传

```bash
wget -c https://ftp.gnu.org/gnu/wget/wget-latest.tar.gz
```

适合：

- 下载大文件
- 网络偶发中断后继续

### 15.4 限速下载

```bash
wget --limit-rate=1m https://ftp.gnu.org/gnu/wget/wget-latest.tar.gz
```

### 15.5 失败自动重试

```bash
wget --tries=5 --timeout=15 https://ftp.gnu.org/gnu/wget/README
```

### 15.6 后台下载

```bash
wget -b https://ftp.gnu.org/gnu/wget/wget-latest.tar.gz
```

说明：

- 下载会放到后台执行
- 日志默认写入当前目录的 `wget-log`

### 15.7 按时间戳更新下载

```bash
wget -N https://ftp.gnu.org/gnu/wget/README
```

适合：

- 只在远端文件更新后重新拉取
- 做简单的文件同步

### 15.8 忽略证书错误

```bash
wget --no-check-certificate https://self-signed.badssl.com/
```

注意：这只适合测试环境。

---

## 16. wget 访问控制与认证

### 16.1 带请求头下载

```bash
wget --header="Authorization: Bearer YOUR_TOKEN" https://httpbin.org/bearer -O bearer.json
```

### 16.2 Basic Auth

```bash
wget --user=user --password=passwd https://httpbin.org/basic-auth/user/passwd -O auth.json
```

### 16.3 发送 POST 表单

```bash
wget --post-data="username=admin&password=<example-password>" https://httpbin.org/post -O post-result.json
```

### 16.4 保存响应到标准输出

有时候你不是想下载文件，而是想快速看返回内容：

```bash
wget -qO- https://httpbin.org/get
```

这里的 `-O-` 表示输出到标准输出而不是文件。

---

## 17. wget 递归下载与镜像

### 17.1 抓取单个页面引用的资源

```bash
wget -r -l1 -H -k -p https://www.gnu.org/gnu/gnu.html
```

常见参数含义：

- `-r`：递归下载
- `-l1`：递归深度为 1，避免抓太多
- `-H`：允许跨主机抓取资源
- `-k`：把页面里的链接转换成本地可访问形式
- `-p`：把页面显示需要的资源一起抓下来

### 17.2 小范围镜像一个目录

```bash
wget --mirror --convert-links --no-parent https://ftp.gnu.org/gnu/wget/
```

适合：

- 离线保存文档目录
- 备份公开文件目录

注意：

- `--mirror` 很强，不要对大型站点随便执行
- 开始前最好确认目录范围、磁盘空间和目标站点的抓取政策

---

## 18. Windows 下使用 wget 的注意事项

如果你在 Windows PowerShell 里直接输入 `wget`，拿到的未必是 GNU `wget`。

常见情况：

- PowerShell 里的 `wget` 可能是 `Invoke-WebRequest` 的别名
- WSL、Git Bash、MSYS2、Cygwin 里的 `wget` 才更接近 Linux 上常见的 GNU `wget`

建议先确认你实际调用的是谁：

```powershell
Get-Command wget
```

如果你想用真正的 GNU `wget`，更稳妥的方式通常是：

- 在 `WSL` 里使用
- 在 `Git Bash` 或 `MSYS2` 里使用
- 明确写出实际可执行文件路径

---

## 19. curl 和 wget 怎么选

可以先用这个简单判断：

- 要测接口、看状态码、加 Header、发 JSON：优先 `curl`
- 要下载大文件、自动重试、断点续传：优先 `wget`
- 要递归抓页面、做离线镜像：优先 `wget`
- 要看 TLS、重定向、请求细节：优先 `curl`

一个很实用的习惯是：

- 先用 `curl` 排查请求是否正确
- 再用 `wget` 执行长时间下载

---

## 20. wget 常用速查

```bash
# 下载文件
wget https://ftp.gnu.org/gnu/wget/README

# 指定文件名
wget -O wget-readme.txt https://ftp.gnu.org/gnu/wget/README

# 断点续传
wget -c https://ftp.gnu.org/gnu/wget/wget-latest.tar.gz

# 限速下载
wget --limit-rate=1m https://ftp.gnu.org/gnu/wget/wget-latest.tar.gz

# 后台下载
wget -b https://ftp.gnu.org/gnu/wget/wget-latest.tar.gz

# 带认证访问
wget --user=user --password=passwd https://httpbin.org/basic-auth/user/passwd -O auth.json

# 发 POST 请求并保存结果
wget --post-data="username=admin&password=123456" https://httpbin.org/post -O post-result.json

# 直接把响应输出到终端
wget -qO- https://httpbin.org/get

# 小范围镜像目录
wget --mirror --convert-links --no-parent https://ftp.gnu.org/gnu/wget/
```

---

## 21. 最后一句

把它们配合起来会最舒服：

- `curl` 负责“测”
- `wget` 负责“下”

这样日常排查网站、测试接口、下载文件、保存页面基本就够用了。
