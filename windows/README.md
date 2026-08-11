# Windows 笔记

Windows 既是多数学习者的宿主环境，也是运维工作中绕不开的管理对象。本目录覆盖这两个视角：用 PowerShell 管理 Windows，用 WSL 在 Windows 上获得 Linux 实验环境。

## 文档索引

| 文档 | 内容 | 推荐场景 |
|---|---|---|
| [powershell-notes.md](./powershell-notes.md) | PowerShell 语法、对象管道、系统管理、计划任务、防火墙/Defender 和脚本实践 | Windows 自动化与日常运维 |
| [wsl-notes.md](./wsl-notes.md) | WSL 安装、发行版管理、文件互通、网络模式、systemd 与 Docker | Windows/Linux 混合开发环境 |

## 推荐顺序

1. 先掌握 PowerShell 的对象与管道模型——它和 Linux 管道传文本的思路不同，先建立对象思维再学命令事半功倍。
2. 配置 WSL 2，建立随时可以做实验的 Linux 环境，并学会用 `wsl --export` 备份。
3. 之后进入 [Linux 笔记](../linux/README.md) 学习 Linux 基础，WSL 就是你的练习场。

## 注意事项

- 涉及发行版注销、磁盘回收或网络重置时，先导出备份并确认命令影响范围。
- PowerShell 脚本落地生产前，先在测试机验证；`Remove-Item -Recurse -Force` 一类命令要格外确认路径。
- 完整学习路径中本目录属于阶段 0（环境搭建），见[从运维小白到运维开发工程师学习路线](../ops-engineer-roadmap.md)。
