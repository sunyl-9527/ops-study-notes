# Docker 学习文档

> 适用对象：Docker 初学者、开发环境搭建、单机服务部署入门  
> 使用建议：先掌握镜像、容器、卷、网络四个核心概念，再开始写 `Dockerfile` 和 `compose.yaml`

---

## 目录

1. [Docker 是什么](#1-docker-是什么)
2. [安装与基础配置](#2-安装与基础配置)
3. [核心概念](#3-核心概念)
4. [镜像操作](#4-镜像操作)
5. [容器操作](#5-容器操作)
6. [数据持久化](#6-数据持久化)
7. [网络](#7-网络)
8. [Dockerfile](#8-dockerfile)
9. [Docker Compose](#9-docker-compose)
10. [常见实践](#10-常见实践)
11. [排错与清理](#11-排错与清理)
12. [快速参考](#12-快速参考)

---

## 1. Docker 是什么

Docker 是一个容器化平台，用来把应用及其运行环境一起打包，做到“在我机器上能跑，换台机器通常也能跑”。

### 容器和虚拟机的区别

| 对比项 | 容器 | 虚拟机 |
|---|---|---|
| 启动速度 | 秒级 | 分钟级 |
| 资源占用 | 共享宿主机内核，较轻 | 自带完整系统，较重 |
| 隔离方式 | 进程级隔离 | 硬件级 / 系统级隔离 |
| 典型用途 | 应用打包、开发部署 | 多系统隔离、强隔离场景 |

### Docker 的基本链路

```text
docker CLI -> dockerd -> image registry
                      -> image -> container
```

### 一句话理解

- 镜像是“模板”
- 容器是“运行中的实例”
- 仓库是“镜像存放地”

---

## 2. 安装与基础配置

### 常见安装方式

- `Linux`：通常安装 `Docker Engine`
- `Windows / macOS`：通常安装 `Docker Desktop`

### Ubuntu 快速安装

```bash
curl -fsSL https://get.docker.com | sh

sudo systemctl enable --now docker

sudo usermod -aG docker $USER
newgrp docker
```

### 验证安装

```bash
docker version
docker info
docker run hello-world
```

### 国内网络常见配置

编辑 `/etc/docker/daemon.json`：

```json
{
  "registry-mirrors": [
    "https://mirror.ccs.tencentyun.com"
  ]
}
```

重启 Docker：

```bash
sudo systemctl daemon-reload
sudo systemctl restart docker
```

### 常见注意事项

- 把用户加入 `docker` 组后，相当于获得了较高宿主机权限，生产环境要谨慎。
- `hello-world` 能运行，只代表 Docker 基本可用，不代表网络、挂载、端口都配置正确。
- Windows 上路径挂载语法与 Linux 不同，示例命令不要直接照搬。

---

## 3. 核心概念

| 概念 | 说明 |
|---|---|
| `Image` | 只读模板，用来创建容器 |
| `Container` | 镜像的运行实例 |
| `Registry` | 镜像仓库，如 Docker Hub |
| `Dockerfile` | 构建镜像的说明文件 |
| `Volume` | Docker 管理的数据卷 |
| `Bind Mount` | 宿主机目录直接挂载进容器 |
| `Network` | 容器之间通信的网络 |

### 推荐的理解方式

- `Image` 像安装包或项目快照
- `Container` 像启动后的程序进程
- `Volume` 像独立的数据盘

### 新手最容易混淆的点

- 删除容器不等于删除镜像
- 删除容器时，容器内未持久化的数据通常也会消失
- `EXPOSE` 只是声明端口，不会自动映射到宿主机

---

## 4. 镜像操作

```bash
# 搜索镜像
docker search nginx

# 拉取镜像
docker pull nginx
docker pull nginx:1.25

# 查看本地镜像
docker images
docker image ls

# 查看镜像详情
docker inspect nginx:latest

# 打标签
docker tag nginx:latest myrepo/nginx:v1.0

# 推送到仓库
docker login
docker push myrepo/nginx:v1.0

# 删除镜像
docker rmi nginx
docker image rm nginx:1.25

# 导出 / 导入镜像
docker save -o nginx.tar nginx:latest
docker load -i nginx.tar
```

### 常见提醒

- `latest` 不是“最新版”的保证，只是一个普通标签。
- 删除镜像前，如果仍有容器依赖它，通常会失败。
- 离线传输镜像时优先使用 `save/load`，不要依赖 `commit` 生成临时镜像。

### 仓库认证

```bash
# 登录私有仓库，交互式输入密码
docker login registry.example.com

# CI/CD 等非交互场景，避免密码出现在命令历史和进程列表中
echo "$REGISTRY_TOKEN" | docker login registry.example.com -u ci-bot --password-stdin

# 登出，清理本地缓存的凭据
docker logout registry.example.com
```

- 优先使用有效期可控的访问 Token，而不是账号主密码。
- 登录凭据默认写入 `~/.docker/config.json`，生产环境建议配置 [credential helper](https://docs.docker.com/engine/reference/commandline/login/#credential-stores)（如 `docker-credential-pass`、`docker-credential-secretservice`），避免明文落盘。
- 使用自建仓库且证书为私有 CA 签发时，需要把 CA 证书放入 Docker 信任目录，否则 `pull/push` 会报证书校验失败。

### 镜像安全基线

- 优先使用固定 `digest`（如 `nginx@sha256:...`）而不是可变标签，避免同一 tag 背后镜像内容被悄悄替换。
- 用 [Trivy](https://github.com/aquasecurity/trivy) 或 `docker scout` 扫描已知漏洞：

```bash
trivy image myrepo/nginx:v1.0
docker scout cves myrepo/nginx:v1.0
```

- 基础镜像和依赖需要定期重新构建以获取安全补丁，不要长期固定在某个旧 tag 不更新。
- 生产镜像尽量以非 root 用户运行，必要时用 `--read-only` 和 `--cap-drop=ALL` 收紧运行时权限。

---

## 5. 容器操作

### 基本生命周期

```bash
# 运行容器
docker run -d -p 8080:80 --name my-nginx nginx

# 查看容器
docker ps
docker ps -a

# 停止 / 启动 / 重启
docker stop my-nginx
docker start my-nginx
docker restart my-nginx

# 删除容器
docker rm my-nginx
docker rm -f my-nginx

# 删除所有已停止容器
docker container prune
```

### 进入容器与查看日志

```bash
docker exec -it my-nginx bash
docker exec -it my-nginx sh

docker logs my-nginx
docker logs -f my-nginx
docker logs --tail 100 my-nginx

docker stats
docker stats my-nginx
```

### 常用 `run` 参数

| 参数 | 说明 |
|---|---|
| `-d` | 后台运行 |
| `-it` | 交互式终端 |
| `--name` | 容器名称 |
| `-p 8080:80` | 宿主机端口映射到容器端口 |
| `-v src:dst` | 挂载卷或目录 |
| `-e KEY=VALUE` | 设置环境变量 |
| `--rm` | 退出后自动删除 |
| `--restart unless-stopped` | 异常退出后自动重启 |
| `--network` | 指定网络 |
| `--cpus` | 限制 CPU |
| `-m` | 限制内存 |

### 运行容器时的常见坑

- 容器启动后立刻退出，通常是主进程结束了。
- 端口映射成功不代表服务可访问，也可能是应用只监听 `127.0.0.1`。
- 容器里未必有 `bash`，很多轻量镜像只能用 `sh`。

---

## 6. 数据持久化

容器适合运行程序，不适合直接存放关键业务数据。容器删掉后，未持久化的数据通常会丢失。

### Volume

```bash
docker volume create mydata

docker run -d -v mydata:/app/data nginx

docker volume ls
docker volume inspect mydata
docker volume rm mydata
docker volume prune
```

### Bind Mount

```bash
docker run -d -v /host/path:/container/path nginx
```

Windows 示例：

```bash
docker run -d -v C:\Users\data:/app/data nginx
```

### 两者区别

| 对比项 | Volume | Bind Mount |
|---|---|---|
| 管理方式 | Docker 管理 | 宿主机自己管理 |
| 适合场景 | 数据库、持久化数据 | 本地开发、热更新代码 |
| 可移植性 | 更好 | 依赖宿主机路径 |

### 建议

- 生产环境优先用 `volume`
- 开发环境改代码频繁时优先用 `bind mount`
- 数据库目录不要随手挂到不稳定路径

---

## 7. 网络

### 常见网络类型

| 类型 | 说明 |
|---|---|
| `bridge` | 默认网络，最常用 |
| `host` | 直接使用宿主机网络 |
| `none` | 不启用网络 |
| 自定义网络 | 推荐，多容器通信更清晰 |

### 常用命令

```bash
docker network create mynet

docker run -d --network mynet --name web nginx
docker run -d --network mynet --name db mysql

docker network ls
docker network inspect mynet
docker network connect mynet my-nginx
docker network rm mynet
```

### 实战建议

- 多容器应用尽量放到同一个自定义网络里。
- 容器间访问优先使用容器名或服务名，不要写死 IP。
- `host` 模式性能高但隔离弱，排查端口冲突也更难。

---

## 8. Dockerfile

### 基本结构

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

ENV PYTHONUNBUFFERED=1

EXPOSE 8000

CMD ["python", "app.py"]
```

### 常用指令

| 指令 | 说明 |
|---|---|
| `FROM` | 基础镜像 |
| `WORKDIR` | 工作目录 |
| `COPY` | 复制文件 |
| `ADD` | 类似 `COPY`，但功能更多 |
| `RUN` | 构建时执行命令 |
| `CMD` | 容器默认启动命令 |
| `ENTRYPOINT` | 容器入口点 |
| `ENV` | 环境变量 |
| `EXPOSE` | 声明端口 |
| `ARG` | 构建参数 |
| `USER` | 指定运行用户 |

### 构建镜像

```bash
docker build -t myapp:v1.0 .
docker build -f ./docker/Dockerfile -t myapp:v1.0 .
docker history myapp:v1.0
```

### 推荐写法

```dockerfile
FROM node:20-alpine

WORKDIR /app

COPY package.json package-lock.json ./
RUN npm ci

COPY . .

RUN addgroup -S app && adduser -S app -G app
USER app

CMD ["npm", "start"]
```

### 最佳实践

- 优先选官方、精简、稳定的基础镜像。
- 先复制依赖文件，再安装依赖，最后复制源码，提升缓存命中率。
- 用 `.dockerignore` 排除 `.git`、日志、依赖目录等无关内容。
- 尽量不要以 `root` 用户运行应用。
- 能用 `npm ci` 就尽量别用 `npm install`。

### 多阶段构建

```dockerfile
FROM golang:1.21 AS builder
WORKDIR /app
COPY . .
RUN go build -o main .

FROM alpine:latest
WORKDIR /app
COPY --from=builder /app/main .
CMD ["./main"]
```

适合用来减小最终镜像体积，并避免把编译工具一起带入运行环境。

---

## 9. Docker Compose

`Docker Compose` 用一个配置文件管理多个容器，适合本地开发、测试环境和单机服务编排。

### 示例：Web + Postgres + Redis

```yaml
services:
  web:
    build: .
    ports:
      - "8000:8000"
    environment:
      DATABASE_URL: postgresql://user:<db-password>@db:5432/mydb
      REDIS_URL: redis://redis:6379
    depends_on:
      - db
      - redis
    volumes:
      - .:/app
    restart: unless-stopped

  db:
    image: postgres:15
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: <db-password>
      POSTGRES_DB: mydb
    volumes:
      - pgdata:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    volumes:
      - redisdata:/data

volumes:
  pgdata:
  redisdata:
```

### 常用命令

```bash
docker compose up -d
docker compose down
docker compose down -v
docker compose ps
docker compose logs -f web
docker compose build
docker compose up -d --build
docker compose exec web sh
docker compose up -d --scale web=3
```

### 使用建议

- 新版 Compose 通常直接写 `services:` 即可，不一定要保留 `version` 字段。
- 服务之间互相访问时，用服务名，例如 `db`、`redis`。
- `depends_on` 只能控制启动顺序，不能保证服务已经“可用”。
- 上面 `web` 服务固定绑定了宿主机 `8000` 端口，多个副本会抢占同一端口导致 `--scale web=3` 启动失败。需要横向扩容时，去掉 `ports` 里的固定映射（或改成端口范围），改由前面的 Nginx/负载均衡容器统一对外暴露端口。

---

## 10. 常见实践

### 运行 Nginx 静态站点

```bash
docker run -d \
  -p 80:80 \
  --name my-site \
  -v $(pwd)/html:/usr/share/nginx/html:ro \
  nginx
```

### 运行 MySQL

```bash
docker run -d \
  -p 3306:3306 \
  --name mysql \
  -e MYSQL_ROOT_PASSWORD='<strong-password>' \
  -e MYSQL_DATABASE=mydb \
  -v mysql-data:/var/lib/mysql \
  mysql:8.0
```

### 运行 Redis

```bash
docker run -d \
  -p 6379:6379 \
  --name redis \
  -v redis-data:/data \
  redis:7-alpine \
  redis-server --appendonly yes
```

### 开发环境常用模式

- 代码目录挂载到容器里，便于热更新
- 数据库使用 `volume` 持久化
- 应用、数据库、缓存放同一个 Compose 项目中统一管理

---

## 11. 排错与清理

### 常见排错命令

```bash
docker logs my-nginx
docker inspect my-nginx
docker exec -it my-nginx sh
docker top my-nginx
docker stats
docker events
```

### 常见问题排查思路

1. 容器有没有成功启动：`docker ps -a`
2. 主进程有没有报错：`docker logs`
3. 端口有没有映射：`docker port 容器名`
4. 网络能不能互通：进入容器后检查 DNS、端口、服务监听地址
5. 数据有没有正确挂载：`docker inspect`

### 清理命令

```bash
# 删除停止的容器、未使用的网络、悬空镜像、构建缓存
docker system prune

# 连未使用的数据卷也一起删除
docker system prune --volumes

# 查看空间占用
docker system df
```

### 风险提醒

- `docker system prune` 不会删除“正在被使用”的容器，但会清理未使用资源。
- `docker system prune --volumes` 可能删除未挂载的数据卷，执行前一定确认。
- `docker rm -f`、`image prune -a`、`volume prune` 都属于高风险清理命令。

---

## 12. 快速参考

```text
镜像: pull / push / build / images / rmi / tag / save / load
容器: run / ps / stop / start / restart / rm / exec / logs / stats
卷:   volume create / ls / inspect / rm / prune
网络: network create / ls / inspect / connect / rm
系统: info / version / system df / system prune
编排: docker compose up / down / ps / logs / exec / build
```

### 学习顺序建议

1. 先会跑现成镜像
2. 再学挂载、网络、日志
3. 再写 `Dockerfile`
4. 最后上 `Docker Compose`

> 进一步学习方向：镜像优化、容器安全、CI/CD 集成、Kubernetes、镜像仓库管理。
