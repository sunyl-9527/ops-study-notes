# CI/CD 持续集成与持续部署笔记

> 适用对象：已掌握 Git 和 Docker 基础，希望继续学习自动化交付的运维学习者  
> 定位：从手工部署走向自动化交付，为运维开发和 DevOps 工程实践打基础

---

## 目录

1. [CI/CD 是什么](#1-cicd-是什么)
2. [核心概念](#2-核心概念)
3. [GitHub Actions 入门](#3-github-actions-入门)
4. [构建并推送 Docker 镜像](#4-构建并推送-docker-镜像)
5. [通过 SSH 自动部署](#5-通过-ssh-自动部署)
6. [GitLab CI 简介](#6-gitlab-ci-简介)
7. [Jenkins 简介](#7-jenkins-简介)
8. [流水线设计实践](#8-流水线设计实践)
9. [安全实践](#9-安全实践)
10. [回滚与发布策略](#10-回滚与发布策略)
11. [常见问题排查](#11-常见问题排查)
12. [学习路线与实践任务](#12-学习路线与实践任务)
13. [总结](#13-总结)

---

## 1. CI/CD 是什么

CI/CD 是把代码检查、测试、构建、发布和部署自动串联起来的工程方法。它不是某一个具体工具，也不只是几段脚本；GitHub Actions、GitLab CI 和 Jenkins 都是实现它的平台。

### CI、持续交付与持续部署

CI 是 `Continuous Integration`，即持续集成。开发者频繁把小批量代码合并到共享仓库，平台自动执行格式检查、静态分析、测试、编译、打包等任务。CI 主要回答：这次变更能否安全集成到主干？

持续交付是 `Continuous Delivery`：通过 CI 后自动产生随时可部署的版本，生产部署前通常保留人工审批。

持续部署是 `Continuous Deployment`：通过全部检查后自动进入生产环境，不再要求人工批准。它对测试覆盖、监控、告警、回滚和发布策略要求更高。日常交流中的“CD”可能指二者之一，需要结合团队流程确认。

| 概念 | 自动化终点 | 人工批准 | 主要目标 |
|---|---|---|---|
| 持续集成 | 测试、构建完成 | 不涉及生产批准 | 尽早发现集成问题 |
| 持续交付 | 生成可部署版本 | 生产前通常需要 | 保持随时可发布 |
| 持续部署 | 自动部署生产 | 通常不需要 | 快速、频繁上线 |

### 解决的运维痛点

传统手工部署常见问题：
- 操作步骤只在某个人的记忆里。
- 测试和生产执行的命令不一致。
- 容易漏传文件、传错版本或忘记某一步。
- 生产服务器直接拉源码，缺少明确产物。
- 发布失败后不知道上一个可用版本。
- 多人操作时记录、权限和责任不清晰。
- 密码散落在脚本、聊天记录或命令历史中。

CI/CD 把步骤写成版本化配置，由受控环境自动执行，从而让相同代码接受相同检查、每次运行关联提交和日志、产物可以追踪与回滚、部署流程可以审查和重复执行。

### 流水线示意图

```text
开发者提交代码
      |
      v
Git 仓库触发流水线
      |
      v
代码检查 -> 单元测试 -> 构建应用 -> 构建镜像 -> 安全扫描
      |          |           |           |           |
      +----------+-----------+-----------+-----------+
                             |
                    任一步失败则停止
                             v
                    推送版本化构建产物
                             |
                             v
             部署 dev -> 验证 -> 审批 -> 部署 prod
                             |
                             v
                    健康检查、监控、回滚
```

CI/CD 不替代 Git 和 Docker，而是把它们连接起来：Git 提供提交、分支、标签和触发事件；Docker 提供一致的构建与运行载体；CI/CD 平台负责按规则执行测试、构建和部署。

如对分支和标签不熟悉，先复习 [Git 常用命令与核心知识](./git-notes.md)。如对镜像、`Dockerfile` 和 Compose 不熟悉，先复习 [Docker 学习文档](./docker-notes.md)。

---

## 2. 核心概念

### 流水线、阶段、作业和步骤

- 流水线：称为 `pipeline` 或 `workflow`，表示一次完整的自动化过程。
- 阶段：称为 `stage`，表达“检查 -> 测试 -> 构建 -> 发布 -> 部署”等逻辑顺序。
- 作业：称为 `job`，是一组在同一运行环境中执行的步骤。
- 步骤：称为 `step`，可以运行 Shell 命令或调用封装好的 Action。

GitLab CI 直接使用 `stages`。GitHub Actions 通常用多个有依赖关系的 `job` 表达阶段。同一作业的步骤依次运行并共享工作目录，不同作业默认可以并行。

### Runner

`runner` 是实际执行作业的计算环境，可以是平台托管的临时 Linux、Windows、macOS 虚拟机，也可以是自建物理机、虚拟机或 Kubernetes 节点。

托管 Runner 每次通常从干净环境开始，使用简单。自托管 Runner 能访问内网或特殊硬件，但团队要负责补丁、隔离、清理和容量。

### Artifact

`artifact` 是流水线产物，例如二进制文件、前端静态资源、Python wheel、测试报告、覆盖率报告和容器镜像。普通产物通常由 CI 平台短期保存；镜像一般推送到 GHCR、Docker Hub 或私有仓库。产物应能追踪到具体提交，同一版本尽量只构建一次。

### 环境变量和 Secret

环境变量适合保存非敏感配置，如环境名、镜像地址、区域、日志级别和构建参数：

```yaml
name: 环境变量示例
on:
  workflow_dispatch:
jobs:
  show-env:
    runs-on: ubuntu-latest
    env:
      APP_ENV: staging
    steps:
      - name: 显示环境
        run: echo "当前环境为 ${APP_ENV}"
```

Secret 保存 API 令牌、仓库密码、SSH 私钥和云平台凭据。它不能进入仓库，也不应输出到日志：

```yaml
- name: 使用密钥
  env:
    API_TOKEN: ${{ secrets.API_TOKEN }}
  run: ./scripts/publish.sh
```

平台会尝试遮盖日志中的 Secret，但仍要避免打印、变形拼接、写入产物或传给不可信脚本。

---

## 3. GitHub Actions 入门

工作流文件保存在 `.github/workflows/<工作流名称>.yml`。一个仓库可以拆成 `lint.yml`、`test.yml`、`build-image.yml` 和 `deploy.yml` 等多个工作流。

### 工作流结构

```yaml
name: 示例工作流
on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main
permissions:
  contents: read
jobs:
  example:
    runs-on: ubuntu-latest
    steps:
      - name: 检出代码
        uses: actions/checkout@v4
      - name: 执行命令
        run: echo "开始检查"
```

逐段理解：
- `name`：Actions 页面显示的名称。
- `on`：触发工作流的事件。
- `permissions`：内置 `GITHUB_TOKEN` 的权限。
- `jobs`：一个或多个作业。
- `runs-on`：作业使用的 Runner。
- `steps`：顺序执行的步骤。
- `uses`：调用可复用 Action。
- `run`：运行 Shell 命令。

### 常见触发条件

指定分支推送：

```yaml
on:
  push:
    branches:
      - main
      - "release/**"
```

拉取请求：

```yaml
on:
  pull_request:
    branches:
      - main
```

只在指定路径变化时触发：

```yaml
on:
  push:
    paths:
      - "src/**"
      - "tests/**"
      - "pyproject.toml"
```

手动触发并接收参数：

```yaml
on:
  workflow_dispatch:
    inputs:
      environment:
        description: 选择部署环境
        required: true
        type: choice
        options:
          - dev
          - staging
          - prod
```

定时触发使用 UTC 时间：

```yaml
on:
  schedule:
    - cron: "0 2 * * 1"
```

标签发布：

```yaml
on:
  push:
    tags:
      - "v*"
```

### 常用 Actions

| Action | 用途 |
|---|---|
| `actions/checkout@v4` | 检出仓库代码 |
| `actions/setup-python@v5` | 安装并缓存 Python |
| `actions/setup-node@v4` | 安装并缓存 Node.js |
| `actions/cache@v4` | 缓存工具或依赖目录 |
| `actions/upload-artifact@v4` | 上传报告或构建文件 |
| `actions/download-artifact@v4` | 下载前序作业产物 |
| `docker/login-action@v3` | 登录镜像仓库 |
| `docker/setup-buildx-action@v3` | 配置 Buildx |
| `docker/metadata-action@v5` | 生成镜像标签与元数据 |
| `docker/build-push-action@v6` | 构建并推送镜像 |

使用第三方 Action 前要检查来源、维护状态和权限；关键生产流程可固定到可信提交摘要。

### 完整示例：Python lint 与 test

假设项目有 `requirements.txt`、`src/` 和 `tests/`。创建 `.github/workflows/python-ci.yml`：

```yaml
name: Python 持续集成
on:
  push:
    branches:
      - main
      - dev
  pull_request:
    branches:
      - main
permissions:
  contents: read
concurrency:
  group: python-ci-${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
jobs:
  lint-and-test:
    name: Python ${{ matrix.python-version }} 检查与测试
    runs-on: ubuntu-latest
    timeout-minutes: 15
    strategy:
      fail-fast: false
      matrix:
        python-version:
          - "3.11"
          - "3.12"
    steps:
      - name: 检出代码
        uses: actions/checkout@v4
      - name: 设置 Python
        uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
          cache: pip
      - name: 安装依赖
        run: |
          python -m pip install --upgrade pip
          python -m pip install -r requirements.txt
          python -m pip install ruff pytest pytest-cov
      - name: 执行代码检查
        run: python -m ruff check .
      - name: 执行单元测试
        run: python -m pytest --cov=src --cov-report=term-missing --cov-report=xml
      - name: 上传覆盖率报告
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: coverage-python-${{ matrix.python-version }}
          path: coverage.xml
          if-no-files-found: warn
          retention-days: 7
```

设计要点：
- `permissions` 只允许读取源码。
- `concurrency` 取消同一分支上已经过时的运行。
- `matrix` 验证两个 Python 版本。
- `timeout-minutes` 防止异常命令长期占用额度。
- `cache: pip` 缓存依赖下载。
- `if: always()` 在测试失败后仍尝试保存报告。

正式项目应把开发依赖固定在依赖文件中，而不是长期临时安装最新版。默认分支若为 `master`，需把示例中的 `main` 改为实际名称。

多个作业用 `needs` 建立顺序，没有 `needs` 时默认并行：

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - run: echo "执行测试"
  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - run: echo "测试通过后构建"
```

---

## 4. 构建并推送 Docker 镜像

目标链路是“提交代码 -> 测试 -> 构建镜像 -> 生成标签 -> 推送 GHCR”。GHCR 地址通常为 `ghcr.io/<用户或组织>/<仓库>:<标签>`。

`${{ secrets.GITHUB_TOKEN }}` 由 GitHub 自动创建，不需要手工录入；权限由当前仓库和 `permissions` 控制。创建 `.github/workflows/docker-image.yml`：

```yaml
name: 构建并推送 Docker 镜像
on:
  push:
    branches:
      - main
    tags:
      - "v*"
  pull_request:
    branches:
      - main
  workflow_dispatch:
permissions:
  contents: read
  packages: write
concurrency:
  group: docker-${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
jobs:
  build-image:
    runs-on: ubuntu-latest
    timeout-minutes: 30
    steps:
      - name: 检出代码
        uses: actions/checkout@v4
      - name: 生成小写镜像名
        id: image
        shell: bash
        run: echo "name=ghcr.io/${GITHUB_REPOSITORY,,}" >> "$GITHUB_OUTPUT"
      - name: 设置 Docker Buildx
        uses: docker/setup-buildx-action@v3
      - name: 登录 GHCR
        if: github.event_name != 'pull_request'
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - name: 生成镜像元数据
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ steps.image.outputs.name }}
          tags: |
            type=ref,event=branch
            type=ref,event=tag
            type=sha,prefix=sha-
            type=raw,value=latest,enable={{is_default_branch}}
      - name: 构建并按条件推送镜像
        uses: docker/build-push-action@v6
        with:
          context: .
          file: ./Dockerfile
          push: ${{ github.event_name != 'pull_request' }}
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          provenance: true
```

- `setup-buildx-action` 配置支持缓存、多平台等能力的 Buildx。
- `login-action` 传递认证信息；拉取请求不登录也不推送。
- `metadata-action` 可生成 `main`、`v1.2.0`、`sha-a1b2c3d`、`latest` 等标签。
- `build-push-action` 负责构建，并用 `push` 判断是否上传。
- `type=gha` 缓存 BuildKit 层；缓存丢失时仍必须能从零构建。

Docker Hub 可配置 `DOCKERHUB_USERNAME` 和 `DOCKERHUB_TOKEN`：

```yaml
- name: 登录 Docker Hub
  uses: docker/login-action@v3
  with:
    username: ${{ secrets.DOCKERHUB_USERNAME }}
    password: ${{ secrets.DOCKERHUB_TOKEN }}
```

应使用专用访问令牌，不要使用账户密码。企业仓库也应使用权限有限、可轮换的机器人账户。

需要多平台镜像时增加 `docker/setup-qemu-action@v3`，并给构建步骤设置：

```yaml
platforms: linux/amd64,linux/arm64
```

多平台构建更慢，只在确有需求时启用。

---

## 5. 通过 SSH 自动部署

SSH 部署适合学习、小型服务和已有单机 Docker 环境。服务器应提前创建专用部署用户，安装 Docker，准备 `/srv/demo/compose.yaml`，并配置健康检查。

Compose 可通过变量指定镜像：

```yaml
services:
  app:
    image: ${APP_IMAGE}
    restart: unless-stopped
    ports:
      - "127.0.0.1:8000:8000"
```

配置 `DEPLOY_USER`、`DEPLOY_SSH_KEY`、`SSH_KNOWN_HOSTS` 和 `DEPLOY_IMAGE` 四个 Secrets，分别表示部署用户、专用私钥、核验过的主机公钥记录和不带标签的镜像名。服务器地址使用 `<your-server>` 占位符。

创建 `.github/workflows/deploy.yml`：

```yaml
name: 部署到服务器
on:
  workflow_dispatch:
    inputs:
      image_tag:
        description: 要部署的镜像标签
        required: true
        type: string
env:
  DEPLOY_HOST: "<your-server>"
permissions:
  contents: read
concurrency:
  group: deploy-production
  cancel-in-progress: false
jobs:
  deploy:
    name: 部署生产环境
    runs-on: ubuntu-latest
    timeout-minutes: 15
    environment: production
    steps:
      - name: 配置 SSH
        shell: bash
        env:
          SSH_PRIVATE_KEY: ${{ secrets.DEPLOY_SSH_KEY }}
          SSH_KNOWN_HOSTS: ${{ secrets.SSH_KNOWN_HOSTS }}
        run: |
          install -m 700 -d ~/.ssh
          printf '%s\n' "$SSH_PRIVATE_KEY" > ~/.ssh/id_ed25519
          chmod 600 ~/.ssh/id_ed25519
          printf '%s\n' "$SSH_KNOWN_HOSTS" > ~/.ssh/known_hosts
          chmod 600 ~/.ssh/known_hosts
      - name: 部署指定镜像
        shell: bash
        env:
          DEPLOY_USER: ${{ secrets.DEPLOY_USER }}
          DEPLOY_IMAGE: ${{ secrets.DEPLOY_IMAGE }}
          IMAGE_TAG: ${{ inputs.image_tag }}
        run: |
          ssh -i ~/.ssh/id_ed25519 \
            "${DEPLOY_USER}@${DEPLOY_HOST}" \
            bash -s -- "$DEPLOY_IMAGE" "$IMAGE_TAG" <<'REMOTE'
          set -euo pipefail
          IMAGE_NAME="$1"
          IMAGE_TAG="$2"
          cd /srv/demo
          export APP_IMAGE="${IMAGE_NAME}:${IMAGE_TAG}"
          docker compose config --quiet
          docker compose pull app
          docker compose up -d --no-deps app
          curl --fail --retry 10 --retry-delay 3 http://127.0.0.1:8000/health
          docker compose ps
          REMOTE
```

密钥管理重点：
- 私钥不能写入工作流或仓库。
- 不要使用 `StrictHostKeyChecking=no` 绕过主机公钥检查。
- 在可信网络获取并核验主机公钥，再保存到 `SSH_KNOWN_HOSTS`。
- 私钥只供部署使用，权限要小，定期轮换，泄露后立即吊销。
- 部署用户不要直接使用 `root`，只授予所需目录和 Docker 操作权限。

`environment: production` 可关联指定人员审批、部署分支限制和环境专属 Secrets。也可使用 `appleboy/ssh-action` 简化配置，但第三方 Action 会接触认证信息，必须审查来源、版本、权限和维护状态。先掌握原生 `ssh` 更有助于排查连接、主机公钥、认证和远程脚本问题。

---

## 6. GitLab CI 简介

GitLab CI/CD 默认读取仓库根目录的 `.gitlab-ci.yml`，由 GitLab Runner 执行：

```yaml
stages:
  - lint
  - test
variables:
  PIP_CACHE_DIR: "$CI_PROJECT_DIR/.cache/pip"
cache:
  paths:
    - .cache/pip/
lint:
  stage: lint
  image: python:3.12-slim
  script:
    - python -m pip install ruff
    - python -m ruff check .
unit-test:
  stage: test
  image: python:3.12-slim
  script:
    - python -m pip install -r requirements.txt
    - python -m pip install pytest
    - python -m pytest
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH'
```

Runner 可由平台托管，也可部署在自己的服务器或 Kubernetes 中。

| GitHub Actions | GitLab CI/CD | 含义 |
|---|---|---|
| workflow | pipeline | 一次完整运行 |
| job | job | 一组任务 |
| `steps` | `script` | 作业内命令 |
| `runs-on` | Runner 与 `tags` | 执行环境 |
| Action | `include`、组件、脚本 | 复用逻辑 |
| Artifact | Artifact | 构建文件和报告 |
| Secret | CI/CD Variable | 敏感变量 |
| Environment | Environment | 部署环境 |
| `needs` | `needs` | 作业依赖 |

两者语法不同，但测试、构建、产物、环境和部署的设计原则可以迁移。

---

## 7. Jenkins 简介

Jenkins 是历史较久、插件生态庞大的自动化服务器，通常需要团队自行安装、升级、备份和维护节点。在已有大量 Jenkins 流水线、内网代码仓库、特殊硬件、专有工具或旧发布平台的企业中仍很常见。

仓库根目录可保存声明式 `Jenkinsfile`：

```groovy
pipeline {
    agent any
    stages {
        stage('检出') {
            steps {
                checkout scm
            }
        }
        stage('测试') {
            steps {
                sh 'python -m pytest'
            }
        }
        stage('构建镜像') {
            steps {
                sh 'docker build -t demo:${BUILD_NUMBER} .'
            }
        }
    }
    post {
        always {
            echo '流水线执行结束'
        }
    }
}
```

Jenkins 灵活，但插件升级、凭据治理、权限和节点维护会增加运维成本。实际环境还要配置 Agent 标签、超时、产物、凭据和工作目录清理。

---

## 8. 流水线设计实践

### 分支策略与保护

简单的主干开发流程：

```text
功能分支 -> 拉取请求 -> CI -> 审查 -> 合并 main -> 构建并部署
```

建议让功能分支执行快速检查，拉取请求执行完整测试但不推送正式镜像，`main` 合并后构建不可变镜像并部署 dev，`v*` 标签触发正式发布，生产环境通过审批部署。

主分支可禁止直接推送，要求拉取请求、代码审查和指定 CI 检查通过。分支越多，合并与流水线规则越复杂；小团队不要盲目引入长期分支。

### 测试分层

| 层级 | 特点 | 建议时机 |
|---|---|---|
| 格式与静态检查 | 最快 | 每次提交 |
| 单元测试 | 快、定位清晰 | 提交和拉取请求 |
| 集成测试 | 依赖数据库或中间件 | 拉取请求、主分支 |
| 端到端测试 | 慢、覆盖完整链路 | 主分支、预发布 |
| 性能与安全测试 | 成本高 | 定时或发布前 |

快速稳定的检查放在前面，让明显问题尽早失败。慢测试可并行或只在关键事件触发；不稳定测试应修复根因，不要长期依赖重跑。

### 构建一次，多环境部署

```text
同一提交 -> 构建一次镜像 -> dev -> staging -> prod 部署同一镜像
```

不要按环境重新构建，否则依赖变化可能让各环境验证的不是同一产物。环境差异应通过运行时配置、变量或外部配置管理。

### 版本号与环境

常见版本标识有语义化版本 `v1.4.2`、提交摘要 `sha-a1b2c3d`、构建号和分支标签。建议同时保留人类友好版本和不可变提交版本。`latest` 和分支标签会移动，不能单独作为审计与回滚依据；生产记录最好保存镜像摘要 `sha256:...`。

| 环境 | 用途 | 建议 |
|---|---|---|
| dev | 开发联调 | 主分支合并后自动部署 |
| staging | 模拟生产与验收 | 自动或手动部署 |
| prod | 正式服务 | 审批后部署或成熟后的自动部署 |

各环境应使用独立配置和凭据。存在不可逆数据库变更、合规窗口或自动回滚尚不成熟时，应保留人工审批。审批者应看到版本、测试结果、风险和回滚方式。

同一环境不要同时执行两个部署：

```yaml
concurrency:
  group: deploy-production
  cancel-in-progress: false
```

测试工作流可取消旧运行；正在执行的生产部署通常应完成或按明确流程终止。

---

## 9. 安全实践

CI/CD 通常拥有源码、产物仓库和服务器权限，是重要安全边界。

### Secret 与最小权限

禁止提交真实 `.env`、云平台密钥、SSH 私钥、仓库令牌、生产连接串和证书私钥。`.gitignore` 不能清除已经进入 Git 历史的密钥；一旦提交，应视为泄露并立即轮换。

普通检查工作流只需：

```yaml
permissions:
  contents: read
```

只有推送镜像时才增加 `packages: write`。不要为解决权限错误直接授予管理员权限；不同环境应使用不同凭据，并限制资源范围和有效期。

项目依赖应使用锁文件或固定版本。第三方 Action 至少固定主版本，关键流程可固定完整提交摘要，同时定期检查更新和安全公告。

### Trivy 镜像扫描

```yaml
- name: 构建本地扫描镜像
  uses: docker/build-push-action@v6
  with:
    context: .
    load: true
    tags: demo:scan
    cache-from: type=gha
    cache-to: type=gha,mode=max
- name: 使用 Trivy 扫描高危漏洞
  uses: aquasecurity/trivy-action@0.28.0
  with:
    image-ref: demo:scan
    format: table
    exit-code: "1"
    ignore-unfixed: true
    severity: HIGH,CRITICAL
```

`exit-code: "1"` 让高危结果阻止流水线。是否忽略暂无修复的问题要结合团队策略；扫描失败必须有人跟进。还可增加源码分析、依赖扫描、Secret 扫描、软件物料清单和镜像签名。

### fork 拉取请求风险

来自 fork 的拉取请求默认无法读取仓库 Secrets，这是保护机制，不要绕过。`pull_request_target` 可能在高权限目标仓库上下文运行；若同时执行不可信代码，可能泄露令牌。

基本原则：
- 不可信代码只运行测试，不接触生产 Secret。
- 需要 Secret 的作业只执行受信任分支代码。
- 外部贡献经审查后，由维护者触发受控流程。
- 不在高权限上下文执行拉取请求提供的脚本。

自托管 Runner 可能残留进程、文件、容器和凭据。生产 Runner 不应同时执行任意外部拉取请求，应按信任等级隔离并自动清理。不要用 `printenv`、`set` 打印全部环境变量；上传产物前排除 `.env`、私钥、认证配置、调试转储和用户敏感数据。

---

## 10. 回滚与发布策略

### 重新部署旧镜像

最简单的回滚方式是保留旧版本镜像，重新部署旧标签或摘要：

```bash
export APP_IMAGE=ghcr.io/example/demo:sha-a1b2c3d
docker compose pull app
docker compose up -d --no-deps app
curl --fail http://127.0.0.1:8000/health
```

不要在回滚时从旧提交临时重建；直接使用已验证且未覆盖的旧产物更快、更可追踪。

应用回滚不一定能撤销数据库变化。数据库变更宜采用“添加新结构 -> 应用兼容新旧结构 -> 迁移并验证数据 -> 后续版本清理旧结构”的分阶段方式，关键变更前还要验证备份确实可以恢复。

### 蓝绿、金丝雀和滚动发布

- 蓝绿发布：维护两套环境，新版本验证后切流，失败时快速切回；回退快但资源成本高。
- 金丝雀发布：先让少量流量访问新版本，观察错误率、延迟和业务指标后逐步扩大。
- 滚动发布：逐批替换实例，资源开销较低，但新旧版本会短暂共存，接口和数据格式必须兼容。

部署命令成功不等于业务可用。发布后应检查容器状态、健康端点、错误日志、成功率、延迟、资源使用、关键业务操作和版本号。回滚条件要明确，例如健康检查连续失败或错误率超过阈值。

---

## 11. 常见问题排查

先判断失败在哪一层：触发、平台、Runner、依赖、测试、构建、镜像仓库还是目标服务器。

### 工作流不触发

1. 文件是否位于 `.github/workflows/`。
2. YAML 是否可解析，缩进是否正确。
3. `on` 中分支名是否与实际一致。
4. `paths` 或 `paths-ignore` 是否过滤了变更。
5. 标签是否匹配规则。
6. 仓库或组织是否禁用了 Actions。
7. 工作流是否存在于能够接收事件的分支。

可暂时加入 `workflow_dispatch` 手动验证。

### 权限、Secret 和缓存

遇到 `Resource not accessible by integration` 或 `unauthorized` 时，检查 `permissions`、fork 事件、Secret 名称与作用范围、`environment` 和外部令牌权限。Secret 不存在时通常得到空字符串，可以检查是否为空，但不要打印值。

缓存不命中常见原因是锁文件变化、键包含易变值、系统或工具版本不同、路径错误或缓存过期。Dockerfile 应先复制依赖清单并安装依赖，再复制频繁变化的源码。缓存只是优化，没有缓存时构建仍须正确完成。

### Runner 环境差异

本地成功而 Runner 失败时检查：
- 操作系统和 CPU 架构。
- 文件名大小写、路径分隔符和执行权限。
- `CRLF` 与 `LF` 行尾。
- 未写入依赖清单的全局工具。
- Python、Node.js、Java 等版本。
- 时区、编码、端口和网络条件。

应显式固定工具版本，并从干净环境验证安装流程。

### Docker 与 SSH

Docker 推送失败时检查登录的 registry、镜像小写名称、命名空间、令牌权限、`push` 条件和标签合法性。拉取请求按设计可能只构建不推送。

```text
SSH 连接超时       -> 地址、端口、防火墙、路由
SSH 主机公钥失败   -> known_hosts 或服务器公钥变化
SSH 认证失败       -> 用户名、私钥、公钥授权
远程命令不存在     -> PATH 或 Docker 未安装
远程权限不足       -> 部署用户权限
健康检查失败       -> 应用、端口、依赖和配置
```

不要关闭安全检查掩盖问题；在服务器查看 `docker compose ps`、`docker compose logs` 和系统日志。

作业一直排队时，检查托管 Runner 状态、组织并发额度、自托管 Runner 是否在线、`runs-on` 标签和环境审批。调试日志可临时开启，但不能打印 Secret；解决后删除临时调试命令。

---

## 12. 学习路线与实践任务

按“持续集成 -> 构建产物 -> 自动部署 -> 回滚”逐步练习，不要一开始就接入生产服务器和高权限密钥。

### 第一阶段：笔记仓库

- [ ] 阅读一个开源项目的 `.github/workflows/`，找出触发条件、作业和 Runner。
- [ ] 给自己的笔记仓库添加 Markdown lint 工作流。
- [ ] 在拉取请求和主分支推送时运行，并设为合并前必须通过。
- [ ] 故意制造检查错误，再修复并阅读日志。

### 第二阶段：演示项目

- [ ] 创建有少量单元测试的 Python 或 Node.js 项目。
- [ ] 添加 `lint + test` 流水线并固定运行时和依赖版本。
- [ ] 添加 `test + build` 流水线并上传报告或构建产物。
- [ ] 拆分并行作业，比较缓存前后的耗时。

### 第三阶段：构建和推送镜像

- [ ] 编写可用的 `Dockerfile`。
- [ ] 拉取请求只构建，主分支合并后构建并推送镜像。
- [ ] 同时生成提交摘要和版本标签。
- [ ] 使用 `docker/build-push-action@v6` 缓存并接入 Trivy。
- [ ] 在全新环境拉取并启动镜像。

### 第四阶段：部署与回滚

- [ ] 准备隔离虚拟机、低权限部署用户和专用 SSH 密钥。
- [ ] 核验 `known_hosts`，不关闭主机公钥检查。
- [ ] 用 Compose 启动演示服务并增加健康检查。
- [ ] 手动触发工作流部署指定镜像标签。
- [ ] 故意部署错误版本，模拟一次部署失败和旧镜像回滚。
- [ ] 记录版本、现象、回滚命令和恢复时间。

### 第五阶段：工程控制

- [ ] 设置主分支保护和必要检查。
- [ ] 建立 dev、staging、prod 环境并隔离 Secrets。
- [ ] 为 production 配置人工审批。
- [ ] 添加部署并发控制和镜像保留策略。
- [ ] 记录部署成功率、耗时和失败原因。
- [ ] 写出不依赖个人记忆的回滚步骤。

完成后应能回答：什么事件触发流水线、作业在哪里运行、产物如何对应提交、谁能使用 Secret、当前部署哪个镜像、怎样快速回滚，以及如何证明服务真正可用。

---

## 13. 总结

CI/CD 的核心不是追求所有操作无人值守，而是建立可靠、可重复、可审计的交付链路。持续集成让每次变更尽早接受检查；持续交付保持随时可发布；持续部署在测试、监控和回滚成熟后自动进入生产。

```text
Git 提交与分支 -> 自动检查和测试 -> 版本化产物 -> Docker 镜像
              -> 分级环境部署 -> 健康检查和监控 -> 回滚与改进
```

GitHub Actions 适合从代码仓库直接实践。GitLab CI 和 Jenkins 语法不同，但流水线、作业、Runner、产物、环境和 Secret 等概念可以迁移。对运维学习者而言，掌握 CI/CD 是从执行手工命令走向设计自动化交付系统的重要一步。