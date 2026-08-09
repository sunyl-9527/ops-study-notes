# ops-study-notes

个人运维学习笔记，覆盖 Linux、DevOps、Windows、Python 自动化和远程运维实践。

> 仓库中的命令以学习和实验为目的。执行修改网络、防火墙、磁盘、用户或服务配置的命令前，请先确认运行环境并准备回滚方案。

## 从哪里开始

| 目标 | 推荐入口 | 说明 |
|---|---|---|
| 系统学习云计算运维 | [云计算运维学习路线](./devops/cloud-ops-roadmap.md) | 从基础、服务部署到容器与自动化 |
| 查询 Linux 命令 | [Linux Shell 命令手册](./linux/linux-shell-command-handbook.md) | 文件、网络、用户、进程、存储和脚本 |
| 排查云服务器故障 | [云服务器运维常用命令](./linux/cloud-server-ops-commands.md) | 按系统、网络、磁盘、服务和日志定位问题 |
| 学习容器与服务入口 | [Docker 学习文档](./devops/docker-notes.md) / [Nginx 指南](./devops/nginx-notes.md) | 部署、配置和排错实践 |
| 搭建自动化实验环境 | [Ansible + VMware 实验](./devops/ansible-vmware-lab.md) | 1 个控制节点和 4 个被控节点 |
| 使用 Python 做运维 | [Python 运维工程师学习路线](./python/学习路线.md) | 系统操作、网络、数据处理和工具开发 |

## 分类导航

| 分类 | 内容 |
|---|---|
| [DevOps](./devops/README.md) | Git、Nginx、Docker、中间件、Ansible 和学习路线 |
| [Linux](./linux/README.md) | Shell 命令、云服务器排障、curl、Vim 和 Arch Linux |
| [Windows](./windows/README.md) | PowerShell 与 WSL |
| [Python](./python/README.md) | 面向运维场景的 Python 学习路线 |
| [远程运维](./remote/README.md) | FRP 部署和远程环境恢复 Runbook |
| [提示词](./prompts/README.md) | DevOps、Linux 与 Python 学习提示词模板 |

## 建议学习顺序

```text
PowerShell / WSL -> Linux 基础 -> Git -> Nginx -> Docker
       -> 中间件 -> Ansible -> Python 自动化 -> Kubernetes / 云平台（规划中）
```

每个阶段建议保留三类产出：可复现的实验步骤、故障现象与处理记录、验证结果。仅收藏命令而不做实验，很难形成可迁移的运维能力。

## 内容约定

- 文档优先说明适用环境、前置条件、操作步骤、验证方式和回滚思路。
- 命令中的域名、IP、用户名、密码和 token 使用占位符，不记录真实生产信息。
- 凭据使用密码管理器、密钥管理服务或私有配置保存，不提交到 Git。
- 新文件使用小写英文 `kebab-case`；已有中文文件名暂时保留，避免破坏外部链接。
- 新增文档后同步更新对应分类的 `README.md`。

## 安全提醒

公开仓库不应包含真实资产信息、访问凭据、私钥或内部下载地址。若敏感值曾进入 Git 历史，仅删除当前文件并不够：应先吊销或轮换凭据，再根据需要清理 Git 历史。
