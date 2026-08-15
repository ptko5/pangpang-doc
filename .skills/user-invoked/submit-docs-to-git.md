---
name: submit-docs-to-git
trigger: /submit-docs
layer: user-invoked
description: 执行完整的 Git 提交与推送流程：规范检查 → 格式化提交信息 → 推送远程仓库
invokes: [check-doc-structure, check-markdown-format, check-chinese-typography, check-content-quality, check-cross-references, check-image-resources, check-git-commit-format, check-branch-naming, generate-checklist]
---

## 执行步骤

### Step 1: 获取变更列表

执行以下命令获取变更：
```bash
git status --porcelain
```

解析输出为三类：
- **新增（A）**：新创建的文档
- **修改（M）**：已有文档的修改
- **删除（D）**：已删除的文档

如果变更列表为空 → 输出：「没有检测到待提交的变更」→ 终止流程。

### Step 2: 针对变更文档执行纪律检查

对每个状态为 A 或 M 的 `.md` 文件，串行执行：
1. `check-doc-structure`
2. `check-markdown-format`
3. `check-chinese-typography`
4. `check-content-quality`
5. `check-cross-references`
6. `check-image-resources`

**处理规则：**
- 所有检查通过（含自动修复）→ 继续
- 存在不可自动修复的违规：
  - 输出每个文件的违规明细（文件 + 行号 + 规则 + 描述）
  - 终止流程，提示：「请修复以上违规后重新执行」
  - 不执行 Git 操作

### Step 3: 检查分支规范

调用 `check-branch-naming`：
- 当前分支名需符合以下前缀之一：
  - `feature/` - 新增文档/功能
  - `fix/` - 修复文档问题
  - `docs/` - 纯文档维护（本项目的常用前缀）
  - `hotfix/` - 紧急修复
- 不符合 → 提示用户是否切换分支，终止流程

### Step 4: 生成规范的提交信息

根据变更文件推断：
- **提交类型**：`docs`（本项目 99% 情况为文档变更）
- **模块**：从文件路径提取
  - `docs/00-开发规范/xxx.md` → 模块=`规范`
  - `docs/01-环境搭建/xxx.md` → 模块=`环境`
  - `docs/02-部署运维/xxx.md` → 模块=`运维`
  - `docs/03-技术笔记/架构设计/xxx.md` → 模块=`架构设计`
  - `docs/03-技术笔记/新技术调研/xxx.md` → 模块=`调研`
  - `docs/03-技术笔记/问题排查/xxx.md` → 模块=`排查`
  - `docs/03-技术笔记/源码阅读/xxx.md` → 模块=`源码`
  - `docs/04-项目模板/xxx.md` → 模块=`模板`
  - `docs/05-知识管理/xxx.md` → 模块=`知识管理`
- **简短描述**：从变更的文件名推断，如「新增国际化方案选型文档」

提交信息格式：
```
docs(<模块>): <简短描述>

- 变更文件1
- 变更文件2
- ...
```

### Step 5: 验证提交格式

调用 `check-git-commit-format` 验证生成的提交信息，不符合则修正。

### Step 6: 执行 Git 提交

```bash
git add <所有变更文件>
git commit -m "<提交信息>"
```

### Step 7: 推送到远程仓库

```bash
git push -u origin <当前分支>
```

推送失败处理：
- **认证失败**（could not read Username / Authentication failed）：
  输出：「GitHub 认证失败，请在本地终端执行以下命令完成推送：」
  输出完整的 `git push -u origin <分支名>` 命令
  提示用户可以通过 Personal Access Token 或 SSH Key 解决
- **冲突**：提示拉取远程分支并解决冲突后重试
- **分支保护**：提示当前分支禁止直接推送，需要通过 MR

### Step 8: 生成检查清单

调用 `generate-checklist`，基于本次提交的内容生成落地检查清单，输出到终端。

### Step 9: 输出最终结果

```
✅ 文档提交成功
📝 提交信息：docs(调研): 新增国际化方案选型文档
🔗 分支：trae/agent-xxx
📊 纪律检查：18 项通过，2 项自动修复，0 项违规
📋 检查清单：（输出检查清单摘要）

下一步：
1. 如未推送成功，在本地终端执行：git push -u origin <分支名>
2. 登录 GitHub 创建 MR
3. 指定评审人进行文档评审
```

## 失败处理

| 失败场景 | 处理方式 |
|---------|---------|
| 纪律检查不通过（不可自动修复） | 输出违规明细，终止流程，用户修复后重试 |
| 分支命名不规范 | 提供建议分支名，询问是否切换 |
| 推送认证失败 | 输出手动推送命令 + 认证解决指引 |
| 提交已存在（无新变更） | 提示「工作区干净，无需提交」 |
