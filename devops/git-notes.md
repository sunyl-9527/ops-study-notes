# Git 常用命令与核心知识

## 目录

- [1. Git 是什么](#1-git-是什么)
- [2. 核心概念](#2-核心概念)
  - [2.1 三个区域](#21-三个区域)
  - [2.2 文件状态流转](#22-文件状态流转)
  - [2.3 分支与指针](#23-分支与指针)
- [3. 初始配置](#3-初始配置)
- [4. 仓库操作](#4-仓库操作)
- [5. 日常工作流命令](#5-日常工作流命令)
  - [5.1 查看状态与历史](#51-查看状态与历史)
  - [5.2 暂存与提交](#52-暂存与提交)
  - [5.3 撤销与回退](#53-撤销与回退)
- [6. 分支操作](#6-分支操作)
- [7. 远程仓库](#7-远程仓库)
- [8. 合并与变基](#8-合并与变基)
  - [8.1 merge](#81-merge)
  - [8.2 rebase](#82-rebase)
  - [8.3 merge vs rebase 怎么选](#83-merge-vs-rebase-怎么选)
- [9. stash 暂存工作现场](#9-stash-暂存工作现场)
- [10. tag 标签](#10-tag-标签)
- [11. .gitignore](#11-gitignore)
- [12. 常见场景解决方案](#12-常见场景解决方案)
- [13. 命令速查表](#13-命令速查表)
- [14. GitHub CLI（gh）使用教程](#14-github-cligh使用教程)
  - [14.1 安装与登录](#141-安装与登录)
  - [14.2 仓库操作](#142-仓库操作)
  - [14.3 Pull Request](#143-pull-request)
  - [14.4 Issue](#144-issue)
  - [14.5 其他常用命令](#145-其他常用命令)
  - [14.6 gh 速查表](#146-gh-速查表)

---

## 1. Git 是什么

Git 是目前最流行的**分布式版本控制系统**，由 Linus Torvalds（Linux 作者）于 2005 年创建。

**分布式**的意思是：每个开发者本地都有完整的仓库历史，不依赖中央服务器也能工作。GitHub / GitLab / Gitee 只是托管远程仓库的平台，不是 Git 本身。

---

## 2. 核心概念

### 2.1 三个区域

理解这三个区域是学好 Git 的基础：

```
工作区 (Working Directory)
  │  git add
  ▼
暂存区 (Staging Area / Index)
  │  git commit
  ▼
本地仓库 (Repository / .git 目录)
  │  git push
  ▼
远程仓库 (Remote)
```

| 区域 | 说明 |
|------|------|
| 工作区 | 你实际编辑文件的目录 |
| 暂存区 | 准备纳入下次提交的文件快照，`git add` 后进入这里 |
| 本地仓库 | `git commit` 后永久保存的历史记录 |
| 远程仓库 | GitHub 等平台上的仓库，通过 `push/pull` 同步 |

### 2.2 文件状态流转

```
未跟踪 (Untracked)
  │  git add
  ▼
已暂存 (Staged)
  │  git commit
  ▼
已提交 (Committed) ── 修改文件 ──▶ 已修改 (Modified)
                                        │  git add
                                        ▼
                                    已暂存 (Staged)
```

### 2.3 分支与指针

Git 的分支本质是一个指向某次提交的**可移动指针**，`HEAD` 则是指向当前所在分支的指针。

```
A ── B ── C  ← main ← HEAD
          └── D ── E  ← feature
```

创建分支几乎是零成本的（只是新建一个指针文件），切换分支也极快。

---

## 3. 初始配置

安装 Git 后第一件事，设置身份信息（会写入每次提交记录）：

```bash
git config --global user.name "你的名字"
git config --global user.email "your@email.com"

# 设置默认编辑器（可选，默认是 vim）
git config --global core.editor "code --wait"   # VS Code
git config --global core.editor "nano"          # nano

# 设置默认分支名为 main（较新的习惯）
git config --global init.defaultBranch main

# 查看所有配置
git config --list

# 查看某一项
git config user.name
```

配置文件位置：
- 全局：`~/.gitconfig`
- 项目级：`.git/config`（优先级更高，会覆盖全局）

---

## 4. 仓库操作

```bash
# 在当前目录初始化新仓库
git init

# 克隆远程仓库到本地（会自动创建同名目录）
git clone https://github.com/user/repo.git

# 克隆到指定目录名
git clone https://github.com/user/repo.git my-folder

# 克隆时只取最近 1 次提交历史（大仓库提速）
git clone --depth 1 https://github.com/user/repo.git
```

---

## 5. 日常工作流命令

### 5.1 查看状态与历史

```bash
# 查看工作区和暂存区状态
git status

# 简洁版状态
git status -s

# 查看工作区相对于暂存区的改动（还没 add 的）
git diff

# 查看暂存区相对于上次提交的改动（已 add 的）
git diff --staged

# 查看提交历史
git log

# 单行显示
git log --oneline

# 带分支图的历史
git log --oneline --graph --all

# 查看某个文件的修改历史
git log --oneline -- path/to/file

# 查看某次提交的详情
git show <commit-hash>
git show HEAD          # 最新提交
git show HEAD~1        # 上一次提交
```

### 5.2 暂存与提交

```bash
# 暂存指定文件
git add file.txt

# 暂存当前目录所有变更
git add .

# 交互式选择部分变更暂存（精细控制）
git add -p

# 提交
git commit -m "提交信息"

# 暂存所有已跟踪文件并提交（跳过 git add，不包含新文件）
git commit -am "提交信息"

# 修改最近一次提交的信息（只用于未推送的提交）
git commit --amend -m "新的提交信息"

# 把新的暂存内容并入最近一次提交（不改变信息）
git commit --amend --no-edit
```

### 5.3 撤销与回退

这是最容易混淆的部分，按场景记：

**场景一：文件改了，还没 add，想恢复**

```bash
git restore file.txt         # 丢弃工作区的修改（危险，不可恢复）
git checkout -- file.txt     # 旧写法，效果相同
```

**场景二：已经 add 了，想从暂存区移除（文件改动保留）**

```bash
git restore --staged file.txt
git reset HEAD file.txt      # 旧写法
```

**场景三：提交错了，想撤销提交（安全方式，生成反向提交）**

```bash
git revert <commit-hash>     # 新增一次"反操作"提交，历史不变
git revert HEAD              # 撤销最新提交
```

**场景四：回退到某次提交（会修改历史，谨慎用于已推送的提交）**

```bash
# 软回退：提交撤销，改动保留在暂存区
git reset --soft HEAD~1

# 混合回退（默认）：提交撤销，改动保留在工作区
git reset --mixed HEAD~1
git reset HEAD~1

# 硬回退：提交撤销，改动全部丢弃（危险）
git reset --hard HEAD~1
git reset --hard <commit-hash>
```

> 原则：已推送到远程的提交用 `revert`，本地未推送的提交可以用 `reset`。

---

## 6. 分支操作

```bash
# 查看所有本地分支
git branch

# 查看所有分支（含远程）
git branch -a

# 创建新分支
git branch feature-login

# 切换分支
git switch feature-login
git checkout feature-login   # 旧写法

# 创建并切换（最常用）
git switch -c feature-login
git checkout -b feature-login  # 旧写法

# 重命名当前分支
git branch -m new-name

# 删除分支（已合并才允许删）
git branch -d feature-login

# 强制删除（未合并也删）
git branch -D feature-login

# 查看各分支最后一次提交
git branch -v

# 查看已合并到当前分支的分支
git branch --merged

# 查看未合并的分支
git branch --no-merged
```

---

## 7. 远程仓库

```bash
# 查看远程仓库
git remote -v

# 添加远程仓库（origin 是约定俗成的默认名）
git remote add origin https://github.com/user/repo.git

# 修改远程地址
git remote set-url origin https://github.com/user/new-repo.git

# 删除远程仓库关联
git remote remove origin

# 拉取远程变更（默认 fetch + merge，若配置了 pull.rebase 则改为 fetch + rebase）
git pull

# 只拉取不合并
git fetch origin

# 把本地分支推送到远程
git push origin main

# 首次推送并建立追踪关系（之后可以直接 git push）
git push -u origin main

# 删除远程分支
git push origin --delete feature-login

# 推送所有标签
git push origin --tags
```

**fetch vs pull**：

```
git fetch  →  只把远程变更下载到本地，不动你的工作区
git pull   →  默认等于 git fetch && git merge
```

`git pull` 的合并行为可以通过 `pull.rebase` 配置改成 rebase（`git pull --rebase` 或 `git config pull.rebase true`），此时等价于 `git fetch && git rebase`。团队协作前建议先确认仓库或个人的 `pull.rebase` 配置，避免合并策略和预期不一致。

推荐先 `fetch` 看看有什么变更，再决定要不要 `merge`。

---

## 8. 合并与变基

### 8.1 merge

把两个分支的历史合并，生成一个新的合并提交（Merge Commit）。

```bash
# 将 feature 分支合并到当前分支
git merge feature-login

# 禁止快进合并，强制生成 merge commit（保留分支历史）
git merge --no-ff feature-login

# 合并冲突后，解决冲突文件，然后
git add <冲突文件>
git commit
```

合并后的历史：

```
A ── B ── C ──────── M  ← main
          └── D ── E ┘  ← feature（merge commit M 保留了两条线）
```

### 8.2 rebase

把当前分支的提交"搬移"到目标分支的最新提交之后，历史变成一条直线。

```bash
# 在 feature 分支上，把提交变基到 main 最新位置
git switch feature-login
git rebase main

# 变基冲突解决后
git add <冲突文件>
git rebase --continue

# 放弃变基
git rebase --abort
```

变基后的历史：

```
A ── B ── C ── D' ── E'  ← feature（D、E 被重新写在 C 之后）
               ↑
              main
```

### 8.3 merge vs rebase 怎么选

| | merge | rebase |
|---|---|---|
| 历史形态 | 保留真实分叉，有 merge commit | 线性，干净整洁 |
| 适用场景 | 公共分支合并、多人协作 | 个人特性分支整理 |
| 安全性 | 安全，不改写历史 | 会改写提交 hash，**不要对已推送的提交 rebase** |
| 追溯性 | 能看出来自哪个分支 | 看不出原始分支 |

> 黄金法则：**永远不要 rebase 已经推送到远程的公共分支**（会让其他人的历史错乱）。

---

## 9. stash 暂存工作现场

场景：正在开发功能，突然要切换分支修 bug，但当前改动还不想提交。

```bash
# 把工作区和暂存区的改动临时存起来
git stash

# 存储时加描述
git stash push -m "登录功能开发到一半"

# 查看所有 stash
git stash list

# 恢复最近一次 stash（stash 保留）
git stash apply

# 恢复并删除最近一次 stash
git stash pop

# 恢复指定 stash
git stash apply stash@{2}

# 删除指定 stash
git stash drop stash@{0}

# 清空所有 stash
git stash clear
```

---

## 10. tag 标签

标签用于标记发布版本，与分支不同，标签不会移动。

```bash
# 查看所有标签
git tag

# 创建轻量标签（只是指针）
git tag v1.0.0

# 创建附注标签（推荐，包含创建者信息和说明）
git tag -a v1.0.0 -m "正式发布 1.0.0 版本"

# 给某次历史提交打标签
git tag -a v0.9.0 <commit-hash>

# 查看标签详情
git show v1.0.0

# 推送标签到远程（push 默认不推送标签）
git push origin v1.0.0
git push origin --tags   # 推送所有标签

# 删除本地标签
git tag -d v1.0.0

# 删除远程标签
git push origin --delete v1.0.0

# 切换到某个标签（会进入"分离 HEAD"状态）
git checkout v1.0.0
```

---

## 11. .gitignore

`.gitignore` 文件告诉 Git 哪些文件不需要跟踪。

```gitignore
# 忽略单个文件
secret.key

# 忽略某个目录
node_modules/
dist/
.venv/

# 忽略所有 .log 文件
*.log

# 忽略某目录下所有 .txt，但不忽略根目录的
logs/*.txt

# 用 ! 取消忽略（例外规则）
*.env
!.env.example

# 忽略所有隐藏文件（. 开头）
.*
# 但不忽略 .gitignore 本身
!.gitignore
```

**注意**：`.gitignore` 只对**未跟踪**的文件生效。如果文件已经被 `git add` 过，需要先从跟踪中移除：

```bash
# 停止跟踪某文件（文件本身保留在磁盘）
git rm --cached file.txt

# 停止跟踪整个目录
git rm --cached -r node_modules/
```

常用 `.gitignore` 模板：[github.com/github/gitignore](https://github.com/github/gitignore)

---

## 12. 常见场景解决方案

### 提交了敏感信息（密码/密钥）怎么办

```bash
# 1. 立即修改/吊销泄露的密钥，历史清理不能替代吊销
# 2. 用 git-filter-repo 从历史中删除文件（比 filter-branch 更快也更安全，官方已不推荐 filter-branch）
pip install git-filter-repo   # 或用系统包管理器安装
git filter-repo --path secret.key --invert-paths

# 3. 强制推送改写后的历史
git push --force --all
git push --force --tags
```

`git filter-repo` 默认会移除 `origin` 远程配置作为安全保护，执行后需要重新 `git remote add origin ...`。历史改写会影响所有协作者，操作前需通知团队重新克隆或执行 `git pull --rebase`，避免旧历史被重新推回。

### 找回误删的提交

```bash
# 查看所有操作记录（包括已删除的提交）
git reflog

# 恢复到某个状态
git reset --hard HEAD@{3}
git checkout -b recovered-branch HEAD@{5}
```

`reflog` 默认保留 90 天，是你的后悔药。

### 合并时只想要某一次提交

```bash
# cherry-pick：把指定提交"摘"到当前分支
git cherry-pick <commit-hash>

# 摘多个提交
git cherry-pick A B C

# 摘一段范围（不含 A，含 B）
git cherry-pick A..B
```

### 查找是哪次提交引入了 bug

```bash
# 二分查找：Git 自动帮你定位出问题的提交
git bisect start
git bisect bad                  # 当前版本有 bug
git bisect good v1.0.0          # 这个版本没 bug
# Git 会自动 checkout 中间版本，你测试后告诉它 good 或 bad
git bisect good / git bisect bad
# 找到后
git bisect reset                # 结束，回到原位置
```

### 修改历史提交（本地未推送）

```bash
# 交互式变基，修改最近 3 次提交
git rebase -i HEAD~3
# 编辑器里把想修改的行从 pick 改为：
# reword  → 只改提交信息
# edit    → 修改内容
# squash  → 合并到上一个提交
# drop    → 删除这次提交
```

---

## 13. 命令速查表

### 基础操作

| 命令 | 说明 |
|------|------|
| `git init` | 初始化仓库 |
| `git clone <url>` | 克隆仓库 |
| `git status` | 查看状态 |
| `git add <file>` | 暂存文件 |
| `git add .` | 暂存所有变更 |
| `git commit -m "msg"` | 提交 |
| `git log --oneline` | 查看简洁历史 |
| `git diff` | 查看未暂存的改动 |
| `git diff --staged` | 查看已暂存的改动 |

### 分支

| 命令 | 说明 |
|------|------|
| `git branch` | 列出本地分支 |
| `git switch -c <name>` | 创建并切换分支 |
| `git switch <name>` | 切换分支 |
| `git merge <branch>` | 合并分支 |
| `git rebase <branch>` | 变基 |
| `git branch -d <name>` | 删除分支 |

### 远程

| 命令 | 说明 |
|------|------|
| `git remote -v` | 查看远程地址 |
| `git fetch` | 拉取远程变更（不合并） |
| `git pull` | 拉取并合并 |
| `git push -u origin main` | 推送并设置追踪 |
| `git push` | 推送到已追踪的远程分支 |

### 撤销

| 命令 | 说明 |
|------|------|
| `git restore <file>` | 丢弃工作区修改 |
| `git restore --staged <file>` | 取消暂存 |
| `git reset --soft HEAD~1` | 撤销提交，改动留在暂存区 |
| `git reset --hard HEAD~1` | 撤销提交并丢弃改动 |
| `git revert <hash>` | 生成反向提交（安全撤销） |
| `git reflog` | 查看操作历史，用于找回 |

### 其他

| 命令 | 说明 |
|------|------|
| `git stash` | 临时保存工作现场 |
| `git stash pop` | 恢复工作现场 |
| `git cherry-pick <hash>` | 应用指定提交 |
| `git tag -a v1.0 -m "msg"` | 创建标签 |
| `git bisect start` | 二分法定位 bug |

---

*参考资料：git-scm.com 官方文档 / Pro Git（免费电子书）*

---

## 14. GitHub CLI（gh）使用教程

`gh` 是 GitHub 官方命令行工具，可以直接在终端完成创建仓库、管理 PR、操作 Issue 等工作，无需打开浏览器。

### 14.1 安装与登录

**安装（Windows）**

```powershell
winget install --id GitHub.cli
```

**安装（Ubuntu/Debian）**

```bash
sudo apt install gh -y
```

**登录**

```bash
# 浏览器授权登录（推荐）
gh auth login

# 按提示选择：
# ? What account do you want to log into? GitHub.com
# ? What is your preferred protocol for Git operations? HTTPS
# ? How would you like to authenticate? Login with a web browser
# 复制终端显示的一次性验证码，在浏览器完成授权
```

**查看登录状态**

```bash
gh auth status

# 输出示例：
# ✓ Logged in to github.com account yourname (keyring)
# - Token scopes: 'repo', 'read:org'
```

**登出**

```bash
gh auth logout
```

---

### 14.2 仓库操作

**创建仓库**

```bash
# 在当前目录创建公开仓库，并推送
gh repo create my-repo --public --source . --remote origin --push

# 创建私有仓库
gh repo create my-repo --private --source . --remote origin --push

# 只创建远程仓库（不关联本地目录）
gh repo create my-repo --public

# 带描述
gh repo create my-repo --public --description "这是我的项目"
```

**克隆仓库**

```bash
# 克隆自己的仓库（无需完整 URL）
gh repo clone my-repo

# 克隆别人的仓库
gh repo clone username/repo-name
```

**查看仓库**

```bash
# 查看当前仓库信息
gh repo view

# 在浏览器打开当前仓库
gh repo view --web

# 查看指定仓库
gh repo view username/repo-name
```

**修改仓库设置**

```bash
# 改为私有
gh repo edit --visibility private --accept-visibility-change-consequences

# 改为公开
gh repo edit --visibility public --accept-visibility-change-consequences

# 修改描述
gh repo edit --description "新的描述"
```

**列出自己的仓库**

```bash
gh repo list

# 只看私有仓库
gh repo list --visibility private

# 只看公开仓库
gh repo list --visibility public
```

**删除仓库**

```bash
gh repo delete my-repo --yes
```

**Fork 仓库**

```bash
gh repo fork username/repo-name

# fork 并克隆到本地
gh repo fork username/repo-name --clone
```

---

### 14.3 Pull Request

**创建 PR**

```bash
# 交互式创建（推荐）
gh pr create

# 指定标题和正文
gh pr create --title "修复登录 bug" --body "修复了用户无法登录的问题"

# 创建草稿 PR
gh pr create --draft

# 指定目标分支
gh pr create --base main --head feature-login
```

**查看 PR**

```bash
# 列出所有 PR
gh pr list

# 查看当前分支的 PR
gh pr view

# 在浏览器打开
gh pr view --web

# 查看指定编号的 PR
gh pr view 42
```

**检查 PR 状态（CI 检查）**

```bash
gh pr checks
```

**审查与合并**

```bash
# 合并当前 PR
gh pr merge

# 指定合并方式
gh pr merge --merge     # 普通 merge commit
gh pr merge --squash    # squash 合并（压缩为一个提交）
gh pr merge --rebase    # rebase 合并

# 审批 PR
gh pr review --approve
gh pr review --request-changes --body "需要修改 X"
gh pr review --comment --body "有个问题想问一下"
```

**切换到 PR 对应的分支**

```bash
# 检出 PR 42 的代码到本地
gh pr checkout 42
```

**关闭 PR**

```bash
gh pr close 42
```

---

### 14.4 Issue

**创建 Issue**

```bash
# 交互式创建
gh issue create

# 指定标题和正文
gh issue create --title "发现一个 bug" --body "步骤：1. 打开首页 2. 点击登录 3. 报错"

# 指定标签
gh issue create --title "新功能" --label "enhancement"
```

**查看 Issue**

```bash
# 列出所有 open 的 Issue
gh issue list

# 列出所有（含已关闭）
gh issue list --state all

# 查看指定 Issue
gh issue view 10

# 在浏览器打开
gh issue view 10 --web
```

**关闭 / 重开 Issue**

```bash
gh issue close 10
gh issue reopen 10
```

**评论 Issue**

```bash
gh issue comment 10 --body "我也遇到了这个问题"
```

---

### 14.5 其他常用命令

**查看 CI/CD 工作流运行状态**

```bash
# 列出最近的 workflow 运行
gh run list

# 查看某次运行的详情
gh run view 123456

# 实时查看日志
gh run watch
```

**管理 Gist**

```bash
# 创建公开 gist
gh gist create file.txt --public

# 创建私有 gist
gh gist create file.txt

# 列出自己的 gist
gh gist list
```

**API 直接调用（高级用法）**

```bash
# 调用 GitHub REST API
gh api repos/sunyl-9527/work-notes

# 用 jq 过滤结果
gh api repos/sunyl-9527/work-notes --jq '.visibility'

# 调用 GraphQL
gh api graphql -f query='{ viewer { login } }'
```

**别名（简化常用命令）**

```bash
# 创建别名
gh alias set prc 'pr create'
gh alias set prl 'pr list'

# 之后就可以用
gh prc
gh prl

# 查看所有别名
gh alias list
```

---

### 14.6 gh 速查表

| 命令 | 说明 |
|------|------|
| `gh auth login` | 登录 GitHub |
| `gh auth status` | 查看登录状态 |
| `gh repo create <name>` | 创建仓库 |
| `gh repo clone <name>` | 克隆仓库 |
| `gh repo view --web` | 浏览器打开仓库 |
| `gh repo edit --visibility private` | 设为私有 |
| `gh repo list` | 列出自己的仓库 |
| `gh pr create` | 创建 PR |
| `gh pr list` | 列出 PR |
| `gh pr merge` | 合并 PR |
| `gh pr checkout <number>` | 检出 PR 分支 |
| `gh pr checks` | 查看 CI 检查状态 |
| `gh issue create` | 创建 Issue |
| `gh issue list` | 列出 Issue |
| `gh issue close <number>` | 关闭 Issue |
| `gh run list` | 查看 workflow 运行列表 |
| `gh api <path>` | 直接调用 GitHub API |
| `gh alias set <name> <cmd>` | 创建命令别名 |
