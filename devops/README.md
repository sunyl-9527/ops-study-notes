# DevOps 笔记

本目录整理运维学习路线、工程协作、服务入口、容器、中间件与自动化实践。

## 推荐顺序

1. [云计算运维学习路线](./cloud-ops-roadmap.md)：先建立完整知识地图。
2. [Git 常用命令与核心知识](./git-notes.md)：掌握版本管理和协作基础。
3. [Nginx 入门学习指南](./nginx-notes.md)：理解 Web 服务入口、代理和负载均衡。
4. [Docker 学习文档](./docker-notes.md)：学习镜像、容器、网络、存储与 Compose。
5. [CI/CD 持续集成与持续部署笔记](./cicd-notes.md)：把 Git、测试、镜像构建和自动部署连接成流水线。
6. [云计算运维中间件学习笔记](./cloud-ops-middleware-notes.md)：扩展数据库、缓存、消息队列、日志和监控。
7. [监控与告警实战笔记](./monitoring-notes.md)：动手搭建 Prometheus、Grafana、Alertmanager 与日志实验环境。
8. [Kubernetes 入门学习笔记](./k8s-notes.md)：进入容器编排，学习核心对象、应用发布、配置存储与排障。
9. [Ansible + VMware 实验](./ansible-vmware-lab.md)：搭建自动化配置管理实验环境。
10. [Ansible 核心知识笔记](./ansible-notes.md)：在实验环境上系统学习模块、playbook、变量模板与 roles。
11. [Arch Linux 虚拟机优化](./archlinux-vm-optimization.md)：为 Ansible 实验节点或本地实验环境调优虚拟机。

## 文档定位

| 文档 | 适用场景 |
|---|---|
| [cloud-ops-roadmap.md](./cloud-ops-roadmap.md) | 制定阶段目标和检查学习进度 |
| [git-notes.md](./git-notes.md) | 查询分支、提交、撤销、远程协作和 GitHub CLI |
| [nginx-notes.md](./nginx-notes.md) | 部署静态站点、反向代理、HTTPS 和排查日志 |
| [docker-notes.md](./docker-notes.md) | 查询容器日常操作并完成 Compose 实践 |
| [cicd-notes.md](./cicd-notes.md) | 学习代码检查、镜像构建、自动部署、安全和回滚流水线 |
| [cloud-ops-middleware-notes.md](./cloud-ops-middleware-notes.md) | 学习常见中间件的部署与运维关注点 |
| [monitoring-notes.md](./monitoring-notes.md) | 实践指标采集、PromQL、Grafana 看板、告警通知与日志排障 |
| [k8s-notes.md](./k8s-notes.md) | 学习 Kubernetes 核心对象、发布更新、配置存储与 Pod 排障 |
| [ansible-vmware-lab.md](./ansible-vmware-lab.md) | 在隔离网络中完成多节点 Ansible 实验 |
| [ansible-notes.md](./ansible-notes.md) | 查询 Ansible 模块、playbook 语法、roles 与调试排错 |
| [archlinux-vm-optimization.md](./archlinux-vm-optimization.md) | 优化 VMware 中 Arch Linux 实验节点的性能与资源占用 |

Kubernetes 入门笔记现已建立；Terraform 等基础设施即代码相关笔记仍属于规划中内容，见 [cloud-ops-roadmap.md](./cloud-ops-roadmap.md) 中的第七、八阶段说明。

完成每个专题后，至少做一次从零部署、一次故障注入和一次清理恢复，并把验证命令写进实验记录。
