***

name: check-markdown-format
layer: model-invoked
description: 检查 Markdown 格式规范（标题层级、代码块语言、表格格式、列表格式）
corresponds: \["文档书写规范 §5 Markdown 格式纪律"]
can-autofix: true
severity: mandatory
-------------------

## 检查规则

### R1: 代码块必须指定语言类型（强制）

检查所有代码块围栏（\`\`\`），不允许无语言类型的代码块：

````
✅ 正确：```java / ```sql / ```bash / ```json / ```yaml / ```xml / ```markdown
❌ 错误：```（无语言标识）
````

**豁免**：以下情况可无语言（仍建议补）：

- 少于 3 行的短文本

- 纯错误日志堆栈（自动识别 `at xxx.xxx.Xxx` 模式 → 指定为 `log`）

**常见语言映射建议**：

| 内容特征                       | 推荐语言标识               |
| -------------------------- | -------------------- |
| Java 类、Spring 注解           | `java`               |
| SQL 查询                     | `sql`                |
| Shell 命令、git 命令            | `bash`               |
| JSON 数据                    | `json`               |
| YAML / yml 配置              | `yaml`               |
| XML、pom.xml、MyBatis mapper | `xml`                |
| TypeScript / React TSX     | `typescript` / `tsx` |
| HTTP 请求示例、curl             | `http` / `bash`      |
| Markdown 示例（含在文档内）         | `markdown`           |
| 正则表达式                      | `regex`              |
| Mermaid 图                  | `mermaid`            |
| 错误堆栈、日志输出                  | `log`                |

### R2: 表格格式规范（强制）

所有 Markdown 表格（以 `|` 开头的行）：

1. 必须包含 **表头分隔行**（第二行为 `|---|---|` 或带对齐标记 `|:---|---:|`）
2. 每行的列数必须一致（与表头列数相同）
3. 表格前后必须有 **空行**（与正文段落分隔）

```
✅ 正确：
（空行）
| 列 A | 列 B |
|------|------|
| 数据 | 数据 |
（空行）

❌ 错误：缺少分隔行
| 列 A | 列 B |
| 数据 | 数据 |
```

### R3: 列表格式规范（推荐）

检查无序列表 `-` 和有序列表 `1. 2. 3.`：

1. 同级别列表的缩进必须一致（2 空格或 4 空格，通篇统一）
2. 有序列表推荐使用 `1. 1. 1.`（Markdown 自动递增）而非硬编码 `1. 2. 3.`
3. 列表项内容超过 1 行时，续行缩进必须对齐

### R4: 反引号术语规范（强制）

以下场景必须使用反引号 `` ` `` 包裹：

1. 文件名：`application.yml`、`UserController.java`
2. 命令：`git commit`、`npm install`
3. 代码标识符：类名、方法名、变量名、注解名（`@RestController`）
4. 路径：`docs/00-开发规范/`
5. HTTP 状态码和方法：`GET`、`POST`、`200 OK`、`404`
6. 版本号：`JDK 25`、`React 19.x`（反引号包裹术语部分，空格前后规则见中文排版检查）

***

## 自动修复逻辑

### R1 自动修复（代码块语言）

对无语言的代码块，根据首行内容特征推断并补全：

| 内容特征                                                   | 推断语言           |
| ------------------------------------------------------ | -------------- |
| 包含 `public class` / `@RestController` / `package com.` | `java`         |
| 包含 `SELECT / INSERT / UPDATE / CREATE TABLE`（大小写不敏感）   | `sql`          |
| 首字符是 `$` 或包含 `git / npm / mvn / gradle / java -jar`    | `bash`         |
| 首字符是 `{` 或 `[` 且是合法 JSON                               | `json`         |
| 包含 `server:` / `spring:` / 冒号缩进结构                      | `yaml`         |
| 包含 `<?xml` 或 `</` 标签                                   | `xml`          |
| 包含 `import React` 或 `const ...: React.FC`              | `tsx`          |
| 包含 `graph TD` / `sequenceDiagram`（Mermaid）             | `mermaid`      |
| 匹配 `at xxx.xxx.ClassName.method`（Java堆栈）               | `log`          |
| 以上均不匹配                                                 | 标记为需人工判断，不自动修复 |

自动修复标记：`autofixed: true`，`autofix_note: 已自动推断语言为 xxx`

### R2 自动修复（表格格式）

1. 缺少分隔行：自动插入 `|------|------|...`（列数与表头对齐）
2. 列数不一致：无法自动修复，需人工处理
3. 表格前后缺空行：自动插入空行

### R3 不自动修复

推荐级违规，仅输出提示。

### R4 不自动修复

需结合语义判断，仅输出可疑项供人工确认。

***

## 输出格式

````json
{
  "passed": false,
  "total_violations": 4,
  "autofixed_count": 2,
  "violations": [
    {
      "rule": "R1",
      "line": 45,
      "message": "代码块未指定语言类型",
      "code_context": "```\nSELECT * FROM user\n```",
      "autofixed": true,
      "autofix_note": "已推断为 SQL，自动补全语言标识 ```sql"
    },
    {
      "rule": "R1",
      "line": 120,
      "message": "代码块未指定语言类型（无法自动推断）",
      "code_context": "```\n<3行自定义文本>\n```",
      "autofixed": false
    },
    {
      "rule": "R2",
      "line": 78,
      "message": "表格缺少表头分隔行",
      "autofixed": true,
      "autofix_note": "已自动插入分隔线"
    },
    {
      "rule": "R4",
      "line": 33,
      "message": "疑似缺少反引号的术语：application.yml（文件名）",
      "autofixed": false
    }
  ],
  "summary": "共发现4项违规，2项已自动修复，2项需人工处理"
}
````

