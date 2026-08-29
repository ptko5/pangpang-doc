# pangpang-doc Skills 体系

> **铁律**：A user-invoked skill may invoke model-invoked skills, but never another user-invoked one.
>
> 翻译：用户编排技能可以调用纪律检查技能，但绝不可以调用另一个用户编排技能。

## 两层架构

| 层级 | 目录 | 触发方式 | 职责 | 数量 |
|------|------|---------|------|------|
| **User-invoked（编排层）** | `user-invoked/` | 用户打 `/xxx` 命令 | 编排工作流、串联纪律检查 | 6 个 |
| **Model-invoked（纪律层）** | `model-invoked/` | 模型自动触发 | 执行单一维度的规范检查约束 | 10 个 |

## 快速导航

### User-invoked Skills（编排层）

| 技能 | 命令 | 用途 |
|------|------|------|
| [create-doc-from-link](user-invoked/create-doc-from-link.md) | `/create-doc-from-link <url>` | 根据外部链接创建结构化文档 |
| [create-research-doc](user-invoked/create-research-doc.md) | `/create-research-doc <主题>` | 创建技术调研文档 |
| [create-arch-design-doc](user-invoked/create-arch-design-doc.md) | `/create-arch-design-doc <主题>` | 创建架构设计文档 |
| [submit-docs-to-git](user-invoked/submit-docs-to-git.md) | `/submit-docs` | 提交文档到远程仓库 |
| [audit-all-docs](user-invoked/audit-all-docs.md) | `/audit-all-docs` | 全库文档健康度审计 |
| [generate-toc](user-invoked/generate-toc.md) | `/generate-toc` | 生成全库目录索引 |

### Model-invoked Skills（纪律层）

| 技能 | 检查维度 | 对应规范 | 自动修复 | 级别 |
|------|---------|---------|---------|------|
| [check-doc-structure](model-invoked/check-doc-structure.md) | 文档头部、标题层级、结尾标记 | 文档书写规范 §3, §5 | ✅ | 强制 |
| [check-markdown-format](model-invoked/check-markdown-format.md) | 标题递增、代码块语言、表格格式 | 文档书写规范 §5 | ✅ | 强制 |
| [check-chinese-typography](model-invoked/check-chinese-typography.md) | 中英文空格、全角标点、反引号 | 文档书写规范 §5.2 | ✅ | 强制 |
| [check-doc-placement](model-invoked/check-doc-placement.md) | 文档目录归属正确性 | 文档书写规范 §4 | ❌ | 强制 |
| [check-content-quality](model-invoked/check-content-quality.md) | 占位符、空章节、TODO | 文档书写规范 §6.1 | ❌ | 禁止 |
| [check-cross-references](model-invoked/check-cross-references.md) | 内部引用路径、链接有效性 | 文档书写规范 §6.2 | ✅ | 推荐 |
| [check-image-resources](model-invoked/check-image-resources.md) | 图片存放路径、外链图片 | 文档书写规范 §6.3 | ❌ | 强制 |
| [check-git-commit-format](model-invoked/check-git-commit-format.md) | 提交信息格式、类型模块 | Git工作流规范 §3 | ✅ | 强制 |
| [check-branch-naming](model-invoked/check-branch-naming.md) | 分支命名前缀、描述规范 | Git工作流规范 §2.2 | ❌ | 强制 |
| [generate-checklist](model-invoked/generate-checklist.md) | 生成本次修改的检查清单 | 所有规范 §落地检查 | ✅ | 辅助 |

## 设计文档

完整的设计理念与体系架构请参阅：
👉 [文档项目规则与技能体系设计](../docs/03-技术笔记/新技术调研/文档项目规则与技能体系设计.md)

---

**技能体系版本：V1.0**
