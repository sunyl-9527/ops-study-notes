# 从运维小白到运维开发工程师：完整学习路线

> 适用对象：零基础或刚入门、目标是成为运维工程师并进一步走向运维开发（DevOps / SRE 方向）的学习者
>
> 文档定位：本仓库的总路线图。它把所有分类笔记串成一条从小白到运维开发的成长路径。各阶段的详细知识点在对应的独立笔记中，本文只负责"先学什么、后学什么、学到什么程度算过关"。

---

## 一、先想清楚：运维、运维开发到底是什么

| 角色 | 日常在做什么 | 核心能力 |
|---|---|---|
| 传统运维 | 装系统、部署服务、处理告警、备份恢复 | Linux、网络、中间件、排障 |
| 云计算运维 | 管理云主机、安全组、负载均衡、云数据库 | 传统运维能力 + 云平台资源管理 |
| 运维开发（DevOps 方向） | 写自动化工具、搭建 CI/CD 流水线、维护监控平台 | 运维能力 + 编程能力 + 工程化思维 |
| SRE | 用软件工程方法保障服务可靠性，定义 SLO、做容量规划 | 运维开发能力 + 可靠性工程方法论 |

成长路径不是跳跃式的，而是叠加式的：**运维开发 = 扎实的运维基础 + 编程与自动化能力**。跳过基础直接学工具，遇到故障时会寸步难行。

```text
阶段 0        阶段 1-2       阶段 3-4        阶段 5-6         阶段 7-8
本机环境  ->  Linux/网络  ->  服务部署     ->  自动化/监控  ->  容器编排/运维开发
（小白）      （地基）        （能干活）       （高效干活）     （运维开发工程师）
```

---

## 二、路线总览与对应笔记

| 阶段 | 主题 | 对应笔记 | 预估周期 |
|---|---|---|---|
| 0 | 搭好学习环境 | [wsl-notes.md](./windows/wsl-notes.md)、[powershell-notes.md](./windows/powershell-notes.md) | 1 周 |
| 1 | Linux 基础 | [linux-shell-command-handbook.md](./linux/linux-shell-command-handbook.md)、[vim-usage-guide.md](./linux/vim-usage-guide.md) | 3-4 周 |
| 2 | 网络基础与排障 | [network-basics-notes.md](./linux/network-basics-notes.md)、[curl-command-guide.md](./linux/curl-command-guide.md)、[cloud-server-ops-commands.md](./linux/cloud-server-ops-commands.md) | 2-3 周 |
| 3 | Git 与工程协作 | [git-notes.md](./devops/git-notes.md) | 1-2 周 |
| 4 | 服务部署：Nginx / Docker / 中间件 | [nginx-notes.md](./devops/nginx-notes.md)、[docker-notes.md](./devops/docker-notes.md)、[cloud-ops-middleware-notes.md](./devops/cloud-ops-middleware-notes.md) | 4-6 周 |
| 5 | 脚本与自动化 | [shell-scripting-notes.md](./linux/shell-scripting-notes.md)、[ansible-vmware-lab.md](./devops/ansible-vmware-lab.md) | 3-4 周 |
| 6 | 监控与告警 | [monitoring-notes.md](./devops/monitoring-notes.md) | 2-3 周 |
| 7 | 容器编排与持续交付 | [k8s-notes.md](./devops/k8s-notes.md)、[cicd-notes.md](./devops/cicd-notes.md) | 4-6 周 |
| 8 | 运维开发：Python 工具化 | [python-ops-notes.md](./python/python-ops-notes.md)、[学习路线.md](./python/学习路线.md) | 持续进行 |

> 周期按每天 1-2 小时学习估算，仅供参考。重要的不是速度，而是每个阶段都完成"动手实验 + 故障排查 + 文档记录"三件事。
>
> 云计算运维方向的更细化路线见 [cloud-ops-roadmap.md](./devops/cloud-ops-roadmap.md)，两者互补：本文管全局成长路径，那份文档管云运维专题的深度。

---

## 三、阶段 0：搭好学习环境（小白起点）

**目标**：在自己的 Windows 电脑上拥有一个随时可以做实验、搞坏了能快速重建的 Linux 环境。

- 学会 PowerShell 基本操作，能管理文件、服务、查看系统信息 → [powershell-notes.md](./windows/powershell-notes.md)
- 安装 WSL 2，理解发行版管理、文件互通、网络模式 → [wsl-notes.md](./windows/wsl-notes.md)
- 学会导出/导入发行版做备份，敢于折腾

**过关标准**：

- [ ] 能用 `wsl --export` 备份发行版，删掉后能用 `--import` 恢复
- [ ] 知道 Windows 和 WSL 之间怎么互访文件，以及为什么跨文件系统 I/O 慢
- [ ] 有一个专门用于实验的发行版，坏了不心疼

---

## 四、阶段 1：Linux 基础（一切的地基）

**目标**：把 Linux 当成自己的主力工作环境，而不是"偶尔登上去看看的黑框框"。

- 文件与目录、权限、用户与组、进程管理、软件包管理 → [linux-shell-command-handbook.md](./linux/linux-shell-command-handbook.md)
- systemd 服务管理：start/stop/enable、看日志 `journalctl`
- 至少熟练一个终端编辑器 → [vim-usage-guide.md](./linux/vim-usage-guide.md) / [neovim-usage-guide.md](./linux/neovim-usage-guide.md)
- 建议装一个 Arch 或其他发行版从头配置一遍 → [archlinux-lab-setup.md](./linux/archlinux-lab-setup.md)

**过关标准**：

- [ ] 不查资料能完成：建用户、改权限、找出占用磁盘最大的目录、杀掉失控进程
- [ ] 能说清楚 `ls -l` 输出里每一列的含义
- [ ] 能用 `systemctl` 和 `journalctl` 定位一个服务为什么起不来

---

## 五、阶段 2：网络基础与排障（运维的看家本领）

**目标**：面对"服务连不上"这类最常见的问题，有一套自己的分层排查思路。

- TCP/IP 分层、IP 与子网、TCP 握手、DNS、HTTP/HTTPS → [network-basics-notes.md](./linux/network-basics-notes.md)
- curl 调试接口与下载 → [curl-command-guide.md](./linux/curl-command-guide.md)
- 云服务器视角的系统化排障 → [cloud-server-ops-commands.md](./linux/cloud-server-ops-commands.md)

**过关标准**：

- [ ] 给一个"网站打不开"的现象，能按 ping → 路由 → 端口 → 服务 → 应用 的顺序逐层缩小范围
- [ ] 能用 `ss`、`dig`、`curl -v`、`tcpdump` 各自回答一类问题
- [ ] 能解释 502 和 504 的区别，并知道分别去哪里查

---

## 六、阶段 3：Git 与工程协作（进入工程化世界的门票）

**目标**：所有脚本、配置、文档从此进 Git，养成"变更可追溯"的习惯。这是运维和运维开发共同的底层习惯。

- 三区模型、分支、合并、远程协作、回滚 → [git-notes.md](./devops/git-notes.md)
- 建立自己的笔记/脚本仓库，每次实验都提交

**过关标准**：

- [ ] 能独立完成：开分支 → 修改 → 合并 → 解决一次冲突
- [ ] 提交了敏感信息知道怎么正确处理（先轮换凭据，再清理历史）
- [ ] 自己的实验环境配置全部在 Git 里，重建环境不靠回忆

---

## 七、阶段 4：服务部署（从"会用 Linux"到"能干运维的活"）

**目标**：能独立把一套 Web + 数据库 + 缓存的服务从零部署起来，并处理常见故障。

- Nginx：静态站点、反向代理、负载均衡、HTTPS、限流 → [nginx-notes.md](./devops/nginx-notes.md)
- Docker：镜像、容器、卷、网络、Dockerfile、Compose → [docker-notes.md](./devops/docker-notes.md)
- 中间件：MySQL/PostgreSQL、Redis、消息队列的部署与运维关注点 → [cloud-ops-middleware-notes.md](./devops/cloud-ops-middleware-notes.md)

**过关标准**：

- [ ] 用 Docker Compose 部署 Nginx + 应用 + MySQL + Redis 全套并能互通
- [ ] 故意制造一次 502，通过日志完成排查
- [ ] 完成一次 MySQL 备份恢复演练
- [ ] 每个部署都写了文档：步骤、验证方式、回滚方案

---

## 八、阶段 5：脚本与自动化（告别重复劳动）

**目标**：把阶段 4 里手工做过两遍以上的事情全部自动化。这是从"操作员"到"工程师"的分水岭。

- Shell 脚本：健壮性（`set -euo pipefail`）、函数、参数解析、实战脚本 → [shell-scripting-notes.md](./linux/shell-scripting-notes.md)
- Ansible：inventory、ad-hoc、playbook、幂等性 → [ansible-vmware-lab.md](./devops/ansible-vmware-lab.md)
- 实验环境调优 → [archlinux-vm-optimization.md](./devops/archlinux-vm-optimization.md)

**过关标准**：

- [ ] 写出一个带错误处理、日志输出的备份脚本，敢放进 cron 跑
- [ ] 用 Ansible 把一台裸机初始化成可用节点（装包、配置、起服务），且重复执行结果一致
- [ ] 能解释什么是幂等性，以及为什么自动化脚本必须幂等

---

## 九、阶段 6：监控与告警（让系统"会说话"）

**目标**：从"用户报障才知道挂了"变成"告警比用户先到"。

- 指标/日志/链路三支柱、Prometheus + Grafana + Alertmanager 实战 → [monitoring-notes.md](./devops/monitoring-notes.md)
- 为阶段 4 部署的每个服务设计核心指标和告警规则

**过关标准**：

- [ ] 用 Compose 起一套监控栈，接入主机和至少一个中间件的指标
- [ ] 写出 5 条以上能解释清楚含义的 PromQL
- [ ] 配置一条磁盘告警并故意触发它，验证通知链路
- [ ] 能说清楚"告警太多"为什么和"没有告警"一样危险

---

## 十、阶段 7：容器编排与持续交付（现代运维的标配）

**目标**：理解生产环境为什么需要 Kubernetes，并能把"提交代码 → 自动测试 → 自动构建 → 自动部署"的流水线搭起来。

- Kubernetes：核心对象、部署、排障、Helm → [k8s-notes.md](./devops/k8s-notes.md)
- CI/CD：GitHub Actions 流水线、镜像构建推送、自动部署、回滚策略 → [cicd-notes.md](./devops/cicd-notes.md)

**过关标准**：

- [ ] 在本地集群完成：部署 → Service 暴露 → 滚动更新 → 回滚 的完整流程
- [ ] 能排查 Pending / ImagePullBackOff / CrashLoopBackOff 三类典型故障
- [ ] 给自己的项目配置一条从 push 到部署的完整流水线
- [ ] 流水线中没有任何明文凭据

---

## 十一、阶段 8：运维开发（用代码解决运维问题）

**目标**：当现成工具满足不了需求时，能自己写。这是"运维开发工程师"区别于"会用工具的运维"的核心。

- Python 运维开发：子进程、SSH 自动化、API 调用、日志、CLI 工具化 → [python-ops-notes.md](./python/python-ops-notes.md)
- Python 语言基础（如果还不牢） → [学习路线.md](./python/学习路线.md)
- 把前面阶段的 Shell 脚本挑复杂的用 Python 重写，体会两者的分工

**过关标准**：

- [ ] 写出一个批量巡检工具：并发连接多台机器、汇总输出报告、异常告警
- [ ] 会用 argparse 把脚本做成规范的命令行工具
- [ ] 给自己的工具写了测试，并挂上 CI 自动运行
- [ ] 能封装一个第三方 API（如告警 webhook）供其他脚本复用

**进阶方向（按兴趣选择）**：

- 平台方向：FastAPI 写内部运维平台、开发 Ansible 自定义模块
- 云原生方向：Kubernetes Operator、Terraform（规划中，见 [cloud-ops-roadmap.md](./devops/cloud-ops-roadmap.md) 第八阶段）
- SRE 方向：SLO/SLI 设计、混沌工程、容量规划

---

## 十二、贯穿全程的三个习惯

1. **一切进 Git**：脚本、配置、文档、实验记录。运维开发的第一步是把自己的工作工程化。
2. **故障是最好的老师**：每个阶段都要故意搞坏一次再修复。只见过正常状态的人不会排障。
3. **写文档**：每个实验留下三样东西——可复现的步骤、遇到的故障与处理过程、验证结果。这些积累就是你面试和晋升时的作品集。

---

## 十三、常见问题

**Q：要先学编程还是先学运维？**
先学运维基础（阶段 0-4），期间自然会接触 Shell；到阶段 5 系统学脚本，阶段 8 上 Python。带着"我想自动化什么"的问题去学编程，比空学语法有效得多。

**Q：没有服务器怎么练？**
WSL + VMware 虚拟机足够覆盖阶段 0-7 的全部实验（本仓库的实验笔记都基于这套环境）。阶段 8 之后可以考虑一台低配云服务器体验真实公网环境。

**Q：每个阶段都要学到精通再往下走吗？**
不用。达到各阶段的"过关标准"就前进，后面的阶段会反复用到并加深前面的技能。卡在单点追求完美是最常见的弃学原因。

**Q：Kubernetes 和 Python 哪个优先？**
看目标岗位：传统企业运维岗 K8s 优先；互联网运维开发岗 Python 优先。两者最终都要会，顺序不影响结局。
