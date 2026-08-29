---
name: check-cross-references
layer: model-invoked
description: 检查文档间交叉引用是否使用相对路径（禁止外链），以及引用文件是否真实存在
corresponds: ["文档书写规范 §6.2 交叉引用规则"]
can-autofix: true
severity: advisory
---

## 检查规则

### R1: 内部文档引用必须使用相对路径（推荐级，实际按强制检查）

**违规模式**：
Markdown 链接 `[文本](URL)` 中，URL 匹配以下任一模式：
- 以 `http://` 或 `https://` 开头，且 URL 中包含以下 **内部文档域名特征**：
  - `github.com` / `raw.githubusercontent.com`（GitHub 仓库地址）
  - 公司内部 GitLab / Gitea 域名（如 `git.xxx.com`）
  - 本地绝对路径：`file:///workspace/docs/...` 或 `/workspace/docs/...`

**正确模式**：
使用从当前文档所在目录出发的 **相对路径**，以 `.md` 结尾。

```markdown
✅ 正确：
参考 [Git-工作流规范](../00-开发规范/Git-工作流规范.md)
详见 [微服务架构设计](架构设计/微服务架构设计.md)（同级子目录）
参考 [根 README](../../README.md)

❌ 错误：
参考 https://github.com/org/repo/blob/main/docs/00-开发规范/Git-工作流规范.md
参考 file:///workspace/docs/00-开发规范/Git-工作流规范.md
参考 /workspace/docs/00-开发规范/Git-工作流规范.md
```

### R2: 引用的目标文件必须真实存在（强制）

对于所有 Markdown 链接：
- 如果 URL 是相对路径且以 `.md` 结尾（指向同仓库内的文档）
- 拼接 `当前文档所在目录 + 相对路径` → 解析为绝对路径
- 检查该文件是否存在

**不存在即违规**，输出死链明细。

### R3: 锚点引用完整性校验（推荐）

形如 `[文本](path/to/doc.md#章节锚点)` 的链接，额外检查：
1. 目标文档存在
2. 读取目标文档，检查是否存在能生成该锚点的标题
   （Markdown 锚点规则：标题文本转小写、空格变 `-`、移除非字母数字字符）

### R4: 禁止引用本地图片使用绝对路径（强制，配合 check-image-resources）

图片链接 `![alt](URL)` 的 URL 不得使用 `/workspace/assets/...` 或 `file:///...` 的绝对路径。
必须使用相对路径指向 `assets/` 目录。

---

## 自动修复逻辑

### R1 自动修复：内部绝对 URL → 相对路径

**算法**（针对 GitHub 内部链接）：

1. 从 URL 中提取 `/blob/<branch>/docs/` 之后的路径部分：
   ```
   原始URL: https://github.com/org/repo/blob/main/docs/00-开发规范/Git-工作流规范.md
                                                  ↑ 从这里截断
   提取到: 00-开发规范/Git-工作流规范.md
   ```
2. 基于 **当前文档所在目录**，计算该 `docs/` 内路径的相对路径：
   ```
   当前文档: docs/03-技术笔记/架构设计/微服务架构设计.md
   当前目录: docs/03-技术笔记/架构设计/
   目标:     docs/00-开发规范/Git-工作流规范.md
   → 相对路径: ../../00-开发规范/Git-工作流规范.md
   ```
3. 替换链接中的 URL 为相对路径
4. 标记 `autofixed: true`

**本地绝对路径修复（`/workspace/docs/...`）同理**：截断到 `docs/` 之后，计算相对路径。

### R2 不自动修复（死链）

文件不存在时，仅输出明细，需人工处理（删除链接或创建目标文件）。

### R3 不自动修复（锚点缺失）

锚点不存在时，需人工确认目标章节标题拼写。

### R4 自动修复：图片绝对路径 → 相对路径

同 R1 算法，目标路径基于 `assets/` 目录计算。

---

## 输出格式

```json
{
  "passed": false,
  "total_violations": 4,
  "autofixed_count": 2,
  "violations": [
    {
      "rule": "R1",
      "line": 56,
      "message": "内部文档引用使用了 GitHub 绝对 URL，应使用相对路径",
      "context_before": "参考 [Git规范](https://github.com/org/repo/blob/main/docs/00-开发规范/Git-工作流规范.md)",
      "context_after": "参考 [Git规范](../../00-开发规范/Git-工作流规范.md)",
      "autofixed": true
    },
    {
      "rule": "R1",
      "line": 78,
      "message": "内部文档引用使用了本地绝对路径",
      "context_before": "详见 [架构图](/workspace/docs/03-技术笔记/架构设计/微服务架构设计.md)",
      "context_after": "详见 [架构图](架构设计/微服务架构设计.md)",
      "autofixed": true
    },
    {
      "rule": "R2",
      "line": 102,
      "message": "交叉引用死链：目标文件不存在",
      "context": "[某文档](./不存在的文档.md)",
      "resolved_path": "/workspace/docs/03-技术笔记/不存在的文档.md",
      "autofixed": false,
      "suggestion": "请创建该文档或修正链接路径"
    },
    {
      "rule": "R3",
      "line": 120,
      "message": "锚点引用可能失效",
      "context": "[2.1 背景](微服务架构设计.md#2-1-背景与动机)",
      "target_doc": "docs/03-技术笔记/架构设计/微服务架构设计.md",
      "anchor": "#2-1-背景与动机",
      "available_anchors": ["#2-背景与目标", "#2-1-业务背景", "#2-2-现存问题"],
      "autofixed": false,
      "suggestion": "请确认锚点是否正确，候选锚点见 available_anchors"
    }
  ],
  "summary": "共发现4项交叉引用违规，2项已自动修复（绝对URL→相对路径），2项需人工处理"
}
```
