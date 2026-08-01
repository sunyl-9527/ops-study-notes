# 云计算运维学习笔记与路线

> 适用对象：希望系统学习云计算运维、DevOps、容器化与中间件运维的学习者
>
> 文档定位：作为 `devops` 目录下的总览笔记，用来串联 Git、Nginx、Docker 和中间件学习路线

---

## 一、学习目标

云计算运维的核心目标不是“会装几个软件”，而是具备一套完整的系统维护能力：

- 能理解业务服务如何运行、发布和访问
- 能部署和维护常见中间件
- 能使用 Git 管理配置、脚本和运维文档
- 能使用 Nginx 管理入口流量、代理和负载均衡
- 能使用 Docker 管理应用和中间件运行环境
- 能逐步进入 Kubernetes、监控、日志、自动化和云平台运维体系

最终要形成的能力链是：

```text
版本管理 -> 服务入口 -> 容器化部署 -> 中间件运维 -> 监控日志 -> 自动化 -> 云原生
```

---

## 二、学习主线

建议按照下面这条主线学习：

1. Git 与工程协作
2. Nginx 与服务入口
3. Docker 与容器化
4. 核心中间件运维
5. 监控、日志与排障
6. 自动化运维
7. Kubernetes 与云原生
8. 云平台资源与生产化能力

这条路线的好处是：先掌握日常工作最常用的工具，再进入中间件和云原生体系，学习过程更容易形成闭环。

---

## 三、配套学习笔记

本文件作为唯一的云计算运维学习路线入口。具体知识点放在独立学习笔记中，阅读时可以按路线跳转：

| 阶段 | 学习笔记 |
|---|---|
| Git 与工程协作 | [git-notes.md](./git-notes.md) |
| Nginx 与服务入口 | [nginx-notes.md](./nginx-notes.md) |
| Docker 与容器化 | [docker-notes.md](./docker-notes.md) |
| 核心中间件运维 | [cloud-ops-middleware-notes.md](./cloud-ops-middleware-notes.md) |

---

## 四、第一阶段：Git 与工程协作

### 学习重点

- Git 三个区域：工作区、暂存区、本地仓库
- 文件状态流转：未跟踪、已修改、已暂存、已提交
- 分支管理：创建、切换、合并、删除
- 远程仓库：clone、pull、fetch、push
- 合并与变基：merge、rebase
- 暂存现场：stash
- 标签管理：tag
- `.gitignore` 规则
- GitHub CLI 基础使用

### 运维场景中的 Git

Git 在云计算运维中常用于：

- 管理部署脚本
- 管理配置文件
- 管理 Dockerfile 和 Compose 文件
- 管理 Kubernetes YAML
- 管理 Ansible Playbook
- 记录变更历史，方便回滚和追踪问题

### 实践任务

- 创建一个自己的运维笔记仓库
- 新建 `scripts`、`docker`、`nginx`、`k8s` 目录
- 每完成一个实验都提交一次 Git 记录
- 为每个阶段打一个 tag，例如 `docker-basic-v1`

验收标准：

- 能清楚说明 `git add`、`git commit`、`git push` 的区别
- 能独立创建分支并完成合并
- 能通过 `git log` 和 `git diff` 追踪变更
- 能写出适合运维项目的 `.gitignore`

---

## 五、第二阶段：Nginx 与服务入口

### 学习重点

- Nginx 的三种常见角色：Web 服务器、反向代理、负载均衡器
- 配置文件结构：全局块、events、http、server、location
- 静态站点托管
- 反向代理后端服务
- 负载均衡 upstream
- HTTPS 基础配置
- 访问日志与错误日志
- 常见问题：404、403、502、504、连接超时

### 运维场景中的 Nginx

Nginx 常处在业务入口位置，用来处理：

- 用户请求入口
- 静态资源服务
- API 反向代理
- 多后端负载均衡
- 域名与路径转发
- HTTPS 证书接入

### 实践任务

- 部署一个静态站点
- 使用 Nginx 代理一个本地后端服务
- 配置两个后端服务做负载均衡
- 修改日志格式并观察请求链路
- 故意制造一次 502，并完成排查

验收标准：

- 能看懂 `server` 和 `location` 的匹配关系
- 能配置基础反向代理
- 能通过日志定位一次访问失败
- 能解释 Nginx 在业务架构中的入口作用

---

## 六、第三阶段：Docker 与容器化

### 学习重点

- 镜像、容器、仓库的关系
- 容器生命周期管理
- 数据卷与目录挂载
- Docker 网络
- Dockerfile 构建镜像
- Docker Compose 多服务编排
- 容器日志与资源排查
- 镜像清理与磁盘空间管理

### 运维场景中的 Docker

Docker 在运维中的核心价值是：

- 统一运行环境
- 降低部署复杂度
- 快速启动实验环境
- 封装应用依赖
- 方便迁移到 Kubernetes

### 实践任务

- 使用 Docker 运行 Nginx、MySQL、Redis
- 为容器配置数据卷，验证重启后数据不丢失
- 编写一个简单应用的 Dockerfile
- 使用 Docker Compose 编排 Web + MySQL + Redis
- 查看容器日志、网络和磁盘占用

验收标准：

- 能解释镜像和容器的区别
- 能独立编写基础 Dockerfile
- 能使用 Compose 管理多个服务
- 能处理端口冲突、挂载失败、容器无法启动等问题

---

## 七、第四阶段：核心中间件运维

### 推荐学习顺序

1. Nginx
2. MySQL
3. Redis
4. Docker Compose
5. Prometheus + Grafana
6. RabbitMQ 或 Kafka
7. Elasticsearch / ELK
8. Kubernetes
9. Ansible
10. Nacos / Consul / etcd
11. Terraform

### 每个中间件都要掌握的问题

- 它解决什么问题
- 核心架构是什么
- 如何单机部署
- 配置文件在哪里
- 日志在哪里看
- 如何备份恢复
- 如何做高可用
- 如何接入监控
- 常见故障如何排查
- 如何做基础性能调优

### 实践任务

- MySQL：完成备份恢复和主从复制实验
- Redis：完成持久化、主从和哨兵实验
- RabbitMQ / Kafka：完成消息生产、消费和积压排查实验
- Elasticsearch：完成日志写入、索引查询和简单检索实验
- MinIO：完成对象上传、下载和访问权限实验

验收标准：

- 能从零部署一个中间件
- 能解释它在业务架构中的位置
- 能配置持久化或备份
- 能接入基础监控
- 能写出一份部署文档和一份故障排查文档

---

## 八、第五阶段：监控、日志与排障

### 学习重点

- 主机指标：CPU、内存、磁盘、网络
- 服务指标：连接数、请求数、错误率、延迟
- 中间件指标：QPS、缓存命中率、队列积压、慢查询
- 日志分类：系统日志、应用日志、访问日志、错误日志
- 告警规则：阈值、持续时间、告警等级

### 推荐技术栈

- 指标监控：Prometheus
- 图表展示：Grafana
- 告警通知：Alertmanager
- 日志采集：Filebeat / Fluent Bit
- 日志检索：Elasticsearch + Kibana

### 实践任务

- 使用 Prometheus 采集主机指标
- 使用 Grafana 做一个中间件仪表盘
- 为 Nginx、MySQL、Redis 分别设计核心指标
- 配置一次磁盘空间告警
- 使用日志定位一次 502 或连接失败问题

验收标准：

- 能说清楚“服务正常”和“服务健康”的区别
- 能通过指标判断资源瓶颈
- 能通过日志还原请求失败过程
- 能设计基础告警规则

---

## 九、第六阶段：自动化运维

### 学习重点

- Shell 脚本基础
- Ansible 主机清单、模块、Playbook
- 批量部署和批量配置
- 幂等性概念
- 配置模板化
- 部署回滚思路

### 运维场景

自动化运维主要解决：

- 重复部署成本高
- 多台机器配置不一致
- 手工操作容易遗漏
- 新环境搭建慢
- 故障恢复依赖人工经验

### 实践任务

- 编写脚本检查 Nginx、Docker、Redis 运行状态
- 使用 Ansible 批量安装 Nginx
- 使用 Ansible 分发配置文件并重载服务
- 使用 Ansible 一键部署一个 Docker Compose 项目

验收标准：

- 能把手工步骤整理成脚本
- 能用 Ansible 管理多台主机
- 能做到配置变更可重复执行
- 能把部署过程纳入 Git 管理

---

## 十、第七阶段：Kubernetes 与云原生

### 学习重点

- Pod、Deployment、StatefulSet
- Service、Ingress
- ConfigMap、Secret
- PV、PVC、StorageClass
- 滚动更新与回滚
- 健康检查
- Helm Chart

### 学习路径

1. 先理解 Docker 和 Compose
2. 再理解 Kubernetes 的对象模型
3. 用 Deployment 部署无状态服务
4. 用 StatefulSet 理解有状态中间件
5. 用 Ingress 暴露入口
6. 用 Helm 管理复杂应用

### 实践任务

- 在 K8s 中部署一个 Nginx 应用
- 使用 Service 暴露内部访问
- 使用 Ingress 暴露外部访问
- 使用 ConfigMap 管理配置
- 使用 Secret 管理敏感配置
- 使用 PV/PVC 给 Redis 或 MySQL 做持久化

验收标准：

- 能解释 Pod 和容器的关系
- 能看懂 Deployment、Service、Ingress 的职责
- 能排查 Pod 启动失败
- 能完成一次滚动更新和回滚

---

## 十一、第八阶段：云平台资源与生产化能力

### 云平台基础资源

- 云服务器
- VPC、子网、路由表
- 安全组
- 负载均衡
- 云数据库
- 对象存储
- 云监控

### 生产化能力

- 权限最小化
- HTTPS 与证书管理
- 备份恢复
- 高可用设计
- 容量规划
- 变更管理
- 故障复盘
- 成本优化

### 实践任务

- 在云服务器上部署一套 Web + MySQL + Redis 架构
- 使用云负载均衡暴露服务
- 使用对象存储保存静态资源或备份文件
- 配置安全组，只开放必要端口
- 制定一份备份和恢复流程

验收标准：

- 能画出一套基础云上业务架构图
- 能说明每个云资源的作用
- 能从安全、成本、可用性三个角度审视架构
- 能写出一次变更前检查清单

---

## 十二、综合实践项目

### 项目一：基础 Web 运维架构

架构：

```text
User -> Nginx -> App -> MySQL
                     -> Redis
```

目标：

- 建立服务入口、应用、数据库、缓存之间的基础关系
- 掌握 Nginx 代理、Docker 部署和日志排查

### 项目二：Docker Compose 中间件实验环境

架构：

```text
Nginx + App + MySQL + Redis + RabbitMQ
```

目标：

- 使用 Compose 管理多服务
- 统一网络、数据卷和配置
- 为后续 Kubernetes 学习做准备

### 项目三：可观测性平台

架构：

```text
Exporter -> Prometheus -> Grafana
Logs     -> Elasticsearch -> Kibana
```

目标：

- 建立监控指标和日志检索能力
- 让服务故障可以被发现、定位和复盘

### 项目四：Kubernetes 云原生部署

架构：

```text
Ingress -> Service -> Deployment / StatefulSet -> PVC
```

目标：

- 把容器化服务迁移到 K8s
- 理解声明式部署、服务发现和持久化

### 项目五：自动化交付与恢复

目标：

- 使用 Git 管理所有配置和脚本
- 使用 Ansible 自动化部署基础环境
- 使用 Docker Compose 或 K8s 完成应用发布
- 制定故障恢复和回滚流程

---

## 十三、个人知识库沉淀方式

建议在学习过程中持续沉淀下面这些文档：

- `部署手册`：每个组件如何从零部署
- `配置说明`：关键配置项的作用
- `故障排查手册`：现象、原因、定位命令、解决办法
- `常用命令清单`：高频命令和使用场景
- `架构图`：每个项目的组件关系
- `复盘记录`：每次故障或实验问题的总结

推荐目录结构：

```text
DevOps/
  git-notes.md
  nginx-notes.md
  docker-notes.md
  cloud-ops-middleware-notes.md
  cloud-ops-roadmap.md
  projects/
    web-basic/
    docker-compose-lab/
    monitor-lab/
    k8s-lab/
```

---

## 十四、学习检查清单

完成下面这些能力点，说明已经具备比较完整的云计算运维入门能力：

- 能使用 Git 管理脚本、配置和文档
- 能使用 Nginx 配置静态站点、反向代理和负载均衡
- 能使用 Docker 部署应用和中间件
- 能使用 Docker Compose 编排多服务环境
- 能部署并维护 MySQL、Redis、消息队列等常见中间件
- 能通过日志和监控定位常见故障
- 能使用 Ansible 做基础自动化部署
- 能理解 Kubernetes 的核心对象和部署方式
- 能在云平台上规划基础网络、主机、存储和负载均衡
- 能写出部署、排障、备份和恢复文档

---

## 十五、总结

云计算运维的学习重点，是把零散工具串成一套完整工作流：

```text
用 Git 管理变更
用 Nginx 承接流量
用 Docker 封装环境
用中间件支撑业务
用监控和日志发现问题
用自动化减少重复操作
用 Kubernetes 和云平台承载生产系统
```

当你能把这些环节串起来，并且能用文档记录部署、排障和恢复过程，就已经从“会使用工具”进入到“能维护系统”的阶段。
