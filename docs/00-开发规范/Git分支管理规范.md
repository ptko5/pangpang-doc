<!--
  Project: pangpang-doc
  Module: 开发规范
  Document: Git 分支管理规范
  Version: V1.0
  Author: ptko
  Created: 2026-08-29
  Source: 飞书云文档导入
  Maintainer: pangpang-doc
-->

# Git 分支管理规范

## 1. 文档说明

| 属性 | 说明 |
|------|------|
| 文档版本 | V1.0 |
| 来源 | 飞书云文档 |
| 生效日期 | 2026-08-29 |
| 适用范围 | 公司所有研发团队 |
| 作者 | ptko |

> **用途**：定义 Git 分支命名规范、工作流程、合并策略，确保团队协作高效、代码质量可控。

---

## 2. 分支命名规范

### 2.1 分支类型

| 分支类型 | 命名格式 | 示例 | 说明 |
|-|-|-|-|
| 主分支 | `master` | `master` | 生产环境代码（始终保持稳定） |
| 开发分支 | `develop` | `develop` | 开发主分支（集成分支） |
| 功能分支 | `feature/` | `feature/user-login` | 新功能开发 |
| 发布分支 | `release/` | `release/v1.2.0` | 正式发布准备 |
| 热修复分支 | `hotfix/` | `hotfix/fix-login-bug` | 紧急修复（线上问题） |
| 缺陷修复 | `fixbug/` | `fixbug/order-display` | 非紧急缺陷修复 |

### 2.2 命名规则

```text
# 功能分支
feature/<ticket-id>-<简短描述>
feature/T001-user-login

# 发布分支
release/<版本号>
release/v1.0.0

# 热修复分支（线上紧急修复）
hotfix/<ticket-id>-<简短描述>
hotfix/T101-fix-login-500

# 缺陷修复（非紧急）
fixbug/<ticket-id>-<简短描述>
fixbug/T201-order-list-empty
```

> - 使用**小写字母 + 中划线**，禁止驼峰、空格、大写
> - 描述使用**英文**，简洁明了（不超过 5 个单词）
> - 包含**任务编号**，便于关联项目管理工具

---

## 3. 分支职责说明

### 3.1 各分支职责

| 分支 | 职责 | 命名 | 保护策略 |
|-|-|-|-|
| `master` | 生产环境运行的稳定代码 | 固定 | 必须通过 PR 合并，禁止 force push |
| `develop` | 开发主分支，集成了所有已完成功能 | 固定 | 必须通过 PR 合并，禁止 force push |
| `feature/*` | 开发新功能，功能完成后合并回 develop | feature/功能描述 | 可直接 push |
| `release/*` | 发布准备，修复最后问题 | release/版本号 | 必须通过 PR 合并 |
| `hotfix/*` | 线上紧急修复，修复后同步到 develop | hotfix/问题描述 | 必须通过 PR 合并 |
| `fixbug/*` | 非紧急缺陷修复 | fixbug/缺陷描述 | 可直接 push |

### 3.2 代码流转方向

```text
feature/* ──────────▶ develop
                        │
                        ▼
fixbug/*  ──────────▶  develop ──────────▶ release/*
                                          │
                                          ▼
hotfix/* ────────────────────────────────▶ master
                        │                    │
                        ▼                    ▼
                    develop              develop
```

---

## 4. 工作流程详解

### 4.1 项目初始化

```bash
git init
git checkout -b master
git add .
git commit -m "chore: initial project structure"
git checkout -b develop
git push -u origin master
git push -u origin develop
```

### 4.2 日常功能开发流程

```bash
# Step 1: 创建功能分支
git checkout develop
git pull origin develop
git checkout -b feature/T001-user-login

# Step 2: 开发并提交代码
git add src/user/
git commit -m "feat(user): add user entity and repository"

# Step 3: 推送功能分支
git push origin feature/T001-user-login

# Step 4: 同步 develop 最新代码
git checkout develop
git pull origin develop
git checkout feature/T001-user-login
git rebase develop
git push origin feature/T001-user-login --force-with-lease

# Step 5: 创建 Pull Request（目标分支 develop）
# Step 6: 合并后清理
git checkout develop
git pull origin develop
git branch -d feature/T001-user-login
git push origin --delete feature/T001-user-login
```

### 4.3 版本发布流程

```bash
# Step 1: 准备发布分支
git checkout develop && git pull origin develop
git checkout -b release/v1.2.0

# Step 2: 修复发布问题
git add . && git commit -m "fix: update version to 1.2.0"

# Step 3: 合并到 master
git checkout master && git pull origin master
git merge --no-ff release/v1.2.0 -m "Merge release/v1.2.0"
git tag -a v1.2.0 -m "Release version 1.2.0"
git push origin master && git push origin v1.2.0

# Step 4: 合并回 develop
git checkout develop
git merge --no-ff release/v1.2.0 -m "Merge release/v1.2.0 into develop"
git push origin develop

# Step 5: 清理
git branch -d release/v1.2.0
git push origin --delete release/v1.2.0
```

### 4.4 热修复流程

```bash
# 从 master 创建热修复分支
git checkout master && git pull origin master
git checkout -b hotfix/T101-fix-critical-login-bug

# 修复问题
git add . && git commit -m "hotfix: fix login 500 error"

# 合并到 master
git checkout master
git merge --no-ff hotfix/T101-fix-critical-login-bug -m "Merge hotfix/T101"
git tag -a v1.2.1 -m "Hotfix v1.2.1"
git push origin master && git push origin v1.2.1

# 同步到 develop
git checkout develop
git merge --no-ff hotfix/T101-fix-critical-login-bug -m "Merge hotfix/T101"
git push origin develop

# 清理
git branch -d hotfix/T101-fix-critical-login-bug
git push origin --delete hotfix/T101-fix-critical-login-bug
```

---

## 5. Commit 规范

### 5.1 提交信息格式

```text
<type>(<scope>): <subject>

<body>

<footer>
```

### 5.2 Type 类型

| 类型 | 说明 | 示例 |
|-|-|-|
| feat | 新功能 | `feat(user): add phone login feature` |
| fix | 修复 Bug | `fix(order): fix order list empty bug` |
| docs | 文档更新 | `docs: update API documentation` |
| style | 代码格式 | `style: format code` |
| refactor | 重构 | `refactor: extract user service` |
| perf | 性能优化 | `perf: optimize query performance` |
| test | 测试相关 | `test: add user service unit test` |
| chore | 构建/工具 | `chore: upgrade dependencies` |
| ci | CI/CD 配置 | `ci: add github actions workflow` |
| revert | 回滚提交 | `revert: revert commit abc123` |

### 5.3 Commit 检查清单

- [ ] 第一行不超过 50 个字符
- [ ] 使用祈使语气（add, fix, update）
- [ ] type 使用小写
- [ ] scope 使用小写
- [ ] 句末不加句号

---

## 6. Pull Request 规范

### 6.1 PR 描述模板

```markdown
## 变更信息

| 项目 | 内容 |
|------|------|
| 关联任务 | T001 - 用户登录功能 |
| 分支 | feature/T001-user-login → develop |

## 变更描述

- 做了什么
- 为什么做
- 怎么做的

## 影响范围

- **影响的模块**：user-center
- **数据库变更**：新增 user_login_log 表
- **API 变更**：新增 POST /api/v1/users/login

## 测试情况

- [ ] 本地测试通过
- [ ] 单元测试通过（覆盖率 80%）
- [ ] 集成测试通过
```

### 6.2 Code Review 检查清单

| 检查项 | 标准 | 优先级 |
|-|-|-|
| 代码风格 | 符合团队代码规范 | P0 |
| 业务逻辑 | 逻辑正确，边界条件处理 | P0 |
| 安全性 | 无 SQL 注入、XSS、敏感信息泄露 | P0 |
| 性能 | 无 N+1 查询、无明显性能问题 | P1 |
| 测试覆盖 | 必要的单元测试 | P1 |
| 命名规范 | 变量/函数/类命名清晰易懂 | P2 |
| 代码复用 | 无重复代码 | P2 |

---

## 7. 分支保护规则

### 7.1 Master 分支保护

```yaml
# Settings → Branches → Branch protection rules
Branch name pattern: master

# Require pull request reviews before merging
# Required number of approvals: 1
# Require review from Code Owners: true

# Require status checks to pass before merging
# 勾选 CI 检查（如 github-actions）
# 勾选 coverage 检查

# Require branches to be up to date before merging
# Do not allow bypassing the above settings
# Do not allow force pushes
# Do not allow branch deletion
```

### 7.2 Develop 分支保护

```yaml
Branch name pattern: develop

# Require pull request reviews before merging
# Required number of approvals: 1
# Require status checks to pass before merging
# Do not allow force pushes
# Do not allow branch deletion
```

---

**文档结束**

*本文档由飞书云文档导入，pangpang-doc 维护*

---
