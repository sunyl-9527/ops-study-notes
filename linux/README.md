# Linux 笔记

本目录同时提供系统学习材料和故障排查速查手册。

## 学习与查询入口

| 文档 | 定位 | 建议用法 |
|---|---|---|
| [linux-shell-command-handbook.md](./linux-shell-command-handbook.md) | Linux Shell 综合手册 | 系统学习或按目录查命令 |
| [cloud-server-ops-commands.md](./cloud-server-ops-commands.md) | 云服务器排障清单 | 从现象出发检查资源、网络、服务和日志 |
| [curl-command-guide.md](./curl-command-guide.md) | curl 与 wget 指南 | 调试 HTTP、下载文件和调用 API |
| [vim-usage-guide.md](./vim-usage-guide.md) | Vim 使用手册 | 学习移动、编辑、搜索和配置 |
| [neovim-usage-guide.md](./neovim-usage-guide.md) | Neovim 补充手册 | 在 Vim 基础上了解 Neovim 特性 |
| [archlinux-lab-setup.md](./archlinux-lab-setup.md) | Arch Linux 实验记录 | 复现 WSL/Arch 环境问题与修复 |

## 推荐顺序

1. 先通过 Shell 手册掌握文件、权限、进程、网络和 systemd。
2. 用云服务器手册练习“观察 -> 假设 -> 验证 -> 恢复”的排障流程。
3. 补充 curl、Vim/Neovim 等日常工具。
4. 在虚拟机或 WSL 实验环境中验证命令，不直接对生产主机试验破坏性操作。
