# Notes 知识库导航

> 这是 `Notes` 目录的总入口，只负责说明目录结构、学习入口和维护规则。
>
> 具体文档清单放在各分类目录的 `README.md` 中，避免同一份索引在多个地方重复维护。

---

## 目录结构

```text
Notes/
  DevOps/    云计算运维路线、Git、Docker、Nginx、中间件
  Linux/     Linux 命令、云服务器运维、curl、Arch 实验
  Windows/   PowerShell、WSL
  Prompts/   学习与工作场景提示词模板
  Private/   本机环境、凭据、敏感记录
```

---

## 分类入口

| 分类 | 入口 | 内容 |
|---|---|---|
| DevOps | [DevOps/README.md](./DevOps/README.md) | 云计算运维、Git、Nginx、Docker、中间件 |
| Linux | [Linux/README.md](./Linux/README.md) | Linux 命令、云服务器运维、curl、Arch 实验 |
| Windows | [Windows/README.md](./Windows/README.md) | PowerShell、WSL、Windows/Linux 混合环境 |
| Prompts | [Prompts/README.md](./Prompts/README.md) | DevOps、Linux、Python 提示词模板 |
| Private | [Private/README.md](./Private/README.md) | 本机环境与凭据记录，不建议公开同步 |

---

## 推荐学习入口

如果目标是系统学习云计算运维，优先从这里开始：

- 总路线：[DevOps/cloud-ops-roadmap.md](./DevOps/cloud-ops-roadmap.md)
- Windows 基础：[Windows/powershell-notes.md](./Windows/powershell-notes.md)
- WSL 环境：[Windows/wsl-notes.md](./Windows/wsl-notes.md)
- Linux 命令：[Linux/linux-shell-command-handbook.md](./Linux/linux-shell-command-handbook.md)
- 云服务器排障：[Linux/cloud-server-ops-commands.md](./Linux/cloud-server-ops-commands.md)

如果只是查命令：

- Git：[DevOps/git-notes.md](./DevOps/git-notes.md)
- Nginx：[DevOps/nginx-notes.md](./DevOps/nginx-notes.md)
- Docker：[DevOps/docker-notes.md](./DevOps/docker-notes.md)
- curl：[Linux/curl-command-guide.md](./Linux/curl-command-guide.md)

---

## 维护规则

- 根 README 只维护分类入口和关键学习入口。
- 分类 README 维护本分类下的完整文档清单。
- 单篇笔记只写具体知识点、命令、实践和排障，不再重复全局路线。
- 文件名统一使用小写英文 `kebab-case`，例如 `docker-notes.md`。
- 涉及密码、token、密钥、账号信息的内容放入 `Private/`。

---

## 敏感信息提醒

`Private/` 目录用于放置本机环境、凭据、初始化状态等敏感记录。

建议：

- 不要提交到公开仓库。
- 不要截图公开分享。
- 如果需要分享经验，另写一份脱敏版学习笔记。
