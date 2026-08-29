# GitHub Actions 配置

## 1. 文档说明

| 属性 | 说明 |
|------|------|
| 文档版本 | V1.0 |
| 生效日期 | 2026-08-15 |
| 适用范围 | 公司所有使用 GitHub 仓库的研发团队 |
| 作者 | 架构组 |

> **用途**：GitHub Actions 的 workflow 结构、事件触发、环境变量与 Secrets、Docker 构建推送，以及前后端部署示例，实现仓库内 CI/CD 自动化。整体流程见 [自动化发布流程.md](./自动化发布流程.md)。

---

## 2. 核心概念

| 概念 | 说明 |
|------|------|
| Workflow | 一个 `.github/workflows/*.yml` 自动化流程 |
| Job | 工作流中的一个任务（可在不同 Runner 执行） |
| Step | Job 中的一步（运行命令或使用 Action） |
| Runner | 执行任务的机器（GitHub 托管或自托管） |
| Event | 触发工作流的事件（push / pull_request 等） |

```text
Event(push) ──> Workflow ──> Job1 ──> Step1/Step2
                    └──────> Job2 ──> Step1/Step2
```

---

## 3. Workflow 文件结构

```yaml
name: CI Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Setup JDK 25
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '25'
      - name: Build with Gradle
        run: ./gradlew clean build
```

> 【强制】workflow 文件必须放在 `.github/workflows/` 目录，并以 `on` 定义明确的触发事件。

---

## 4. 环境变量与 Secrets

```yaml
env:
  REGISTRY: ghcr.io

jobs:
  deploy:
    runs-on: ubuntu-latest
    env:
      IMAGE_NAME: pangpang/order-service
    steps:
      - name: Login to Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - name: Print config
        run: echo "Registry=${{ env.REGISTRY }}"
```

> 【强制】密码、Token 等敏感信息必须存入仓库 `Settings → Secrets and variables → Actions`，通过 `secrets.XXX` 引用，禁止明文写入 yml。

---

## 5. 前后端构建示例

### 5.1 后端（Gradle + Docker）

```yaml
name: Backend Build & Push

on:
  push:
    branches: [main]
    paths: ['backend/**']

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '25'
      - name: Build
        working-directory: backend
        run: ./gradlew bootJar
      - name: Build & Push Image
        run: |
          docker build -t ghcr.io/pangpang/order-service:${{ github.sha }} backend/
          echo "${{ secrets.GITHUB_TOKEN }}" | docker login ghcr.io -u ${{ github.actor }} --password-stdin
          docker push ghcr.io/pangpang/order-service:${{ github.sha }}
```

### 5.2 前端（Node.js + Vite）

```yaml
name: Frontend Build

on:
  push:
    branches: [main]
    paths: ['frontend/**']

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '22'
          cache: pnpm
      - name: Install & Build
        working-directory: frontend
        run: |
          corepack enable
          pnpm install --frozen-lockfile
          pnpm build
      - name: Deploy to Pages
        uses: peaceiris/actions-gh-pages@v4
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: frontend/dist
```

---

## 6. 常用 Action 速查

| Action | 用途 |
|--------|------|
| `actions/checkout` | 拉取代码 |
| `actions/setup-java` | 配置 JDK |
| `actions/setup-node` | 配置 Node.js |
| `docker/build-push-action` | 构建并推送镜像 |
| `actions/cache` | 依赖缓存加速 |
| `actions/upload-artifact` | 上传构建产物 |

---

## 7. 常见问题排查

| 问题现象 | 原因 | 解决方案 |
|---------|------|---------|
| workflow 未触发 | 事件/路径过滤不符 | 核对 `on` 与 `paths` |
| Secrets 为空 | 未在仓库配置 | 仓库 Settings 添加 |
| 构建慢 | 未开缓存 | 添加 `actions/cache` |
| 权限不足 | GITHUB_TOKEN 权限限制 | 配置 `permissions` 与仓库设置 |
| 前端构建失败 | pnpm 版本不一致 | 使用 `corepack` + lockfile |

---

## 8. 落地检查清单

| 序号 | 检查项 | 检查方式 | 责任人 |
|------|--------|---------|--------|
| 1 | workflow 文件语法正确 | Actions 页面无报错 | 全栈 |
| 2 | 敏感信息使用 Secrets | 检查 yml 无明文 | 全栈 |
| 3 | 前后端可自动构建 | 提交触发验证 | 全栈 |
| 4 | 镜像可推送至仓库 | Actions 日志确认 | 后端开发 |
| 5 | 部署步骤可回滚 | 发布演练 | 运维 |

---

**文档结束**

*本文档由 pangpang-doc 维护*
