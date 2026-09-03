***

name: check-branch-naming
layer: model-invoked
description: 检查 Git 分支命名是否符合规范（前缀、描述、格式）
corresponds: \["Git-工作流规范 §2.1 分支类型定义", "Git-工作流规范 §2.3 分支操作规范"]
can-autofix: false
severity: mandatory
-------------------

## 检查规则

### R1: 永久分支必须严格命名（强制）

仓库的永久分支只能是以下两个：

| 分支名       | 用途       | 保护要求            |
| --------- | -------- | --------------- |
| `main`    | 生产环境代码基线 | 必须开启分支保护，禁止直接推送 |
| `develop` | 日常开发集成分支 | 必须开启分支保护，禁止直接推送 |

违规场景：

- 使用 `master`（应改 `main`）

- 使用 `dev`（应改 `develop`）

- 出现其他疑似永久分支的命名（如 `prod`、`release` 无版本号）

### R2: 临时分支必须使用规范前缀（强制）

临时分支必须以 **`<前缀>/<描述>`** 格式命名，前缀只能是下表之一：

| 前缀         | 用途                        | 创建源     | 合并目标           | 示例                         |
| ---------- | ------------------------- | ------- | -------------- | -------------------------- |
| `feature/` | 新功能开发（文档库：新增完整模块、大量新文档）   | develop | develop        | `feature/user-auth-module` |
| `fix/`     | Bug 修复（文档库：修正已有文档错误）      | develop | develop        | `fix/login-error-doc`      |
| `docs/`    | 纯文档维护（**本项目最常用前缀**）       | develop | develop        | `docs/i18n-solution-doc`   |
| `release/` | 预发布准备（文档库：版本迭代时的文档收尾）     | develop | main + develop | `release/v1.2.0`           |
| `hotfix/`  | 生产紧急修复（文档库：紧急修正线上文档错误）    | main    | main + develop | `hotfix/typo-in-readme`    |
| `trae/`    | AI Agent 创建的分支（**本项目专用**） | develop | develop → MR   | `trae/agent-SYkgLN`        |

**描述部分规则**：

- 全部使用 **小写字母**，单词之间用 `-`（kebab-case）

- 禁止使用中文、空格、下划线（`_`）、点号（`.` 用于分隔版本号除外）

- 描述长度 ≥ 3 个字符，禁止无意义描述如 `fix/fix`、`docs/xxx`

- 推荐格式：`前缀/模块-动作` 或 `前缀/功能名`

  - `docs/规范-补全文档书写`

  - `docs/调研-新增国际化方案选型`

  - `fix/架构设计-修正错别字`

### R3: 发布分支必须带语义化版本号（强制）

`release/` 分支必须符合：

```
release/v<主>.<次>.<修订>
```

示例：

- ✅ `release/v1.0.0`、`release/v2.1.3`

- ❌ `release/1.0.0`（缺 `v` 前缀）

- ❌ `release/v1.0`（缺修订号）

- ❌ `release/may-release`（非版本号）

### R4: trae/ 分支格式（强制，本项目专用）

AI Agent 创建的分支格式：

```
trae/agent-<随机ID>
```

- ID 为 6\~8 位大写字母+数字混合（由 Trae 自动生成）

- 禁止：`trae/test`、`trae/manual-branch` 等人工创建的 trae 前缀分支（人工创建请用 `docs/` 或 `fix/`）

### R5: 禁止在永久分支直接提交（强制，仅检查当前分支状态）

如果当前分支是 `main` 或 `develop`，且检测到：

- 有未提交的变更，且即将执行 `git commit`

则输出 **强制阻止**：请切换到临时分支（如 `docs/xxx` 或 `trae/agent-xxx`）后再提交。

***

## 自动修复逻辑

**can-autofix = false**

原因：

- 重命名分支是破坏性操作（可能影响已推送的远程分支、MR链接）

- 需要用户确认后手动执行：

  ```bash
  # 重命名本地分支
  git branch -m <旧名> <新名>

  # 如果已推送旧分支到远程
  git push origin --delete <旧名>
  git push -u origin <新名>
  ```

***

## 输出格式

```json
{
  "passed": false,
  "total_violations": 2,
  "autofixed_count": 0,
  "current_branch": "doc-updates",
  "violations": [
    {
      "rule": "R2",
      "message": "分支缺少规范前缀（feature/fix/docs/release/hotfix/trae）",
      "current_name": "doc-updates",
      "suggested_names": [
        "docs/调研-新增国际化方案选型",
        "docs/书写规范v2"
      ],
      "based_on_changed_files": [
        "docs/03-技术笔记/新技术调研/国际化方案选型.md",
        "docs/00-开发规范/文档书写规范.md"
      ],
      "autofixed": false,
      "how_to_fix": "git branch -m doc-updates docs/调研-新增国际化方案选型"
    },
    {
      "rule": "R5",
      "message": "检测到当前位于永久分支 develop，且有未提交变更",
      "autofixed": false,
      "severity": "blocking",
      "how_to_fix": "git checkout -b docs/xxx 后再进行提交操作"
    }
  ],
  "summary": "共发现2项分支命名违规，需人工重命名分支后再继续流程"
}
```

