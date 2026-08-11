# archlinux-lab 初始化记录

WSL 发行版：`archlinux-lab`，WSL 版本 2，用户 `sunyl`

---

## 问题修复

### NSS 登录失败

**症状**：`wsl -d archlinux-lab` 报 `getpwnam(sunyl) failed`，无法登录。

**原因**：`/etc/nsswitch.conf` 中 `passwd`、`group`、`shadow` 条目包含 `systemd` 解析器，但 WSL 启动时 systemd 未就绪，导致用户查找失败。

**修复**：移除三行中的 `systemd` 后缀：

先备份原文件，避免改错后无法恢复：

```bash
sudo cp /etc/nsswitch.conf /etc/nsswitch.conf.bak
```

```diff
-passwd: files systemd
+passwd: files

-group:  files [SUCCESS=merge] systemd
+group:  files

-shadow: files systemd
+shadow: files
```

文件路径：`/etc/nsswitch.conf`

**回滚**：如果修改后用户查找、权限或其他登录相关行为出现异常，可用备份文件恢复：

```bash
sudo cp /etc/nsswitch.conf.bak /etc/nsswitch.conf
```

若已经因为改坏配置导致 `wsl -d archlinux-lab` 完全无法登录、连 `sudo` 都进不去，可以在 Windows 端以 root 身份直接进入该发行版修复：

```powershell
wsl -d archlinux-lab -u root
```

进入后用上面的备份文件恢复，或重新编辑 `/etc/nsswitch.conf`。

---

## 系统更新

```
pacman -Syu
```

所有已安装包升级到最新版本。

---

## 新增工具

| 工具 | 用途 |
|------|------|
| `neovim` | 现代 vim，替代编辑器 |
| `tree` | 目录树展示 |
| `htop` | 交互式进程监控 |
| `zsh` | 备用 shell |
| `fzf` | 模糊搜索（Ctrl-R 历史、Ctrl-T 文件） |
| `ripgrep` (`rg`) | 快速代码搜索，替代 grep |
| `fd` | 快速文件查找，替代 find |
| `bat` | 带语法高亮的 cat 替代品 |

已有工具（保留）：`git`、`vim`、`base-devel`、`man-db`、`curl`、`wget`、`tmux`、`yay`、`conda`

---

## 系统配置

### Locale

```
/etc/locale.conf → LANG=en_US.UTF-8
/etc/locale.gen  → en_US.UTF-8, zh_CN.UTF-8 均已生成
```

### 时区

```
/etc/localtime → Asia/Shanghai (CST UTC+8)
```

---

## 用户环境 (`~/.bashrc`)

### 环境变量

```bash
LANG=en_US.UTF-8
EDITOR=nvim
VISUAL=nvim
HISTSIZE=10000          # 命令历史 10000 条
HISTCONTROL=ignoredups  # 不记录重复命令
```

### 提示符

显示用户、主机、当前目录，以及当前 git 分支（如果在 git 仓库内）：

```
sunyl@hostname:~/projects/myrepo (main)$
```

### 别名

| 别名 | 实际命令 | 说明 |
|------|----------|------|
| `ll` | `ls -lh` | 详细列表 |
| `la` | `ls -lAh` | 含隐藏文件 |
| `cat` | `bat --style=plain` | 语法高亮 |
| `grep` | `rg` | 快速搜索 |
| `..` / `...` | `cd ..` / `cd ../..` | 快速跳目录 |
| `pacs` | `sudo pacman -S` | 安装包 |
| `pacr` | `sudo pacman -Rns` | 卸载包 |
| `pacu` | `sudo pacman -Syu` | 更新系统 |
| `ys` | `yay -S` | 从 AUR 安装 |
| `rm/cp/mv` | 加 `-i` | 操作前确认，防误删 |

### fzf 快捷键

- `Ctrl-R`：模糊搜索命令历史
- `Ctrl-T`：模糊搜索当前目录下的文件

---

## Git 全局配置

```ini
core.editor   = nvim
defaultBranch = main
pull.rebase   = false
color.ui      = auto
```

---

## 目录结构

```
~/lab/
├── scratch/    # 临时测试代码
├── notes/      # 笔记
└── projects/   # 正式项目
```

---

## 快速验证

在 Windows 端（PowerShell / CMD）执行以下命令进入该发行版，再在里面确认环境正常：

```powershell
wsl -d archlinux-lab
```

进入后在 archlinux-lab 内部执行：

```bash
id                    # 应显示 uid=1000(sunyl)
git --version
nvim --version
bat --version
rg --version
fzf --version
date                  # 应显示 CST 时间
```
