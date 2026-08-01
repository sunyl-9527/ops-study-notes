# DevOps 笔记导航

> 本目录用于整理云计算运维、DevOps 工具链、容器化、中间件和服务入口相关笔记。

---

## 推荐阅读顺序

1. [cloud-ops-roadmap.md](./cloud-ops-roadmap.md)
2. [git-notes.md](./git-notes.md)
3. [nginx-notes.md](./nginx-notes.md)
4. [docker-notes.md](./docker-notes.md)
5. [cloud-ops-middleware-notes.md](./cloud-ops-middleware-notes.md)

---

## 文档说明

| 文档 | 定位 | 适合场景 |
|---|---|---|
| [cloud-ops-roadmap.md](./cloud-ops-roadmap.md) | 总览路线 | 想系统学习云计算运维时先读，作为唯一路线入口 |
| [cloud-ops-middleware-notes.md](./cloud-ops-middleware-notes.md) | 中间件专题笔记 | 学 Nginx、MySQL、Redis、MQ、ELK、监控等组件 |
| [git-notes.md](./git-notes.md) | Git 手册 | 查分支、提交、回退、远程、stash、tag、GitHub CLI |
| [nginx-notes.md](./nginx-notes.md) | Nginx 入门 | 学反向代理、负载均衡、HTTPS、日志排查 |
| [docker-notes.md](./docker-notes.md) | Docker 手册 | 学镜像、容器、网络、数据卷、Dockerfile、Compose |

---

## 学习主线

```text
路线总览 -> Git -> Nginx -> Docker -> 中间件 -> 监控日志 -> 自动化 -> Kubernetes -> 云平台
```

---

## 实践建议

- 每学一个工具，都做一次从零部署。
- 每个实验都用 Git 记录配置和文档变更。
- Docker 与 Nginx 可以结合起来做 Web 服务入口实验。
- 中间件学习要同时关注部署、日志、备份、监控和故障排查。
- 学到 Kubernetes 前，先把 Docker Compose 的多服务编排练熟。
