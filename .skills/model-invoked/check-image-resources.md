***

name: check-image-resources
layer: model-invoked
description: 检查文档中引用的图片资源是否存放于标准 assets/images/ 目录，禁止引用外部图片 URL
corresponds: \["文档书写规范 §6.3 图片与资源"]
can-autofix: false
severity: mandatory
-------------------

## 检查规则

### R1: 禁止引用外部图片 URL（强制）

Markdown 图片语法 `![alt](URL)` 中，URL 不允许是外部网络地址：

| ❌ 禁止类型  | 示例                                                      |
| ------- | ------------------------------------------------------- |
| HTTP 图片 | `![图](https://example.com/a.png)`                       |
| 图床 CDN  | `![图](https://cdn.jsdelivr.net/gh/user/repo/img/a.png)` |
| 微信公众号图  | `![图](https://mmbiz.qpic.cn/...)`                       |
| 协议相对路径  | `![图](//xxx.com/a.png)`                                 |

**为什么禁止？**

- 外部图片随时可能 404（防盗链、图床倒闭、域名过期）

- 离线阅读时无法查看

- 合规风险（无法审查外部图片内容）

### R2: 图片必须引用 assets/images/ 目录下的资源（强制）

图片 URL 必须是 **相对路径**，且最终解析到的路径位于 `/workspace/assets/images/` 下（或其子目录）。

```
✅ 正确（从 docs/03-技术笔记/架构设计/ 某文档出发）：
![架构图](../../assets/images/arch-001.png)
![流程图](../../assets/images/microservice/flow-001.svg)

❌ 错误：
![架构图](./本地相对路径下的图.png)（不在 assets/images/ 中）
![架构图](/workspace/assets/images/arch-001.png)（使用了绝对路径）
```

### R3: 引用的图片文件必须真实存在（强制）

对每个通过 R2 的图片引用：

1. 基于当前文档目录 + 图片相对路径 → 解析绝对路径
2. 检查文件是否存在
3. 不存在 → 违规（输出「缺失图片文件」明细）

### R4: 推荐图片使用 PNG / SVG 格式（推荐）

非强制，但输出建议：

- **架构图/流程图**：优先 SVG（矢量，缩放不模糊，可 Git diff）

- **截图/照片**：PNG（无损）或 WebP（体积小）

- 禁止 BMP、TIFF 等未压缩大体积格式

- 单张图片建议 ≤ 500KB（过大建议压缩）

***

## 自动修复逻辑

**can-autofix = false**

原因：

- R1：自动下载外链图片可能失败、可能涉及版权、图片文件名需人工命名

- R2：移动图片文件涉及跨目录操作，需确认不破坏其他文档引用

- R3/R4：需人工补充或压缩

**违规后给出的行动建议模板**：

```
📋 R1 外链图片修复指引：
   1. 手动下载：https://example.com/arch.png
   2. 重命名为：有意义的英文名（如 microservice-arch-v1.png）
   3. 移动到：/workspace/assets/images/
   4. 替换文档中的链接为相对路径：
      ![架构图](../../assets/images/microservice-arch-v1.png)
```

***

## 输出格式

```json
{
  "passed": false,
  "total_violations": 3,
  "autofixed_count": 0,
  "violations": [
    {
      "rule": "R1",
      "line": 45,
      "message": "引用了外部图片 URL（随时可能失效）",
      "url": "https://mmbiz.qpic.cn/mmbiz_png/xxx/0?wx_fmt=png",
      "alt_text": "架构总览图",
      "autofixed": false,
      "suggestion": "请下载图片到 assets/images/ 目录，并替换为相对路径引用"
    },
    {
      "rule": "R2",
      "line": 78,
      "message": "图片引用路径不规范：未指向 assets/images/ 目录",
      "url": "./screenshot-2026-08-15.png",
      "resolved_path": "/workspace/docs/03-技术笔记/架构设计/screenshot-2026-08-15.png",
      "expected_path_pattern": "assets/images/**",
      "autofixed": false,
      "suggestion": "移动文件到 /workspace/assets/images/ 并更新引用路径"
    },
    {
      "rule": "R3",
      "line": 112,
      "message": "图片文件不存在（死链）",
      "url": "../../assets/images/missing-diagram.png",
      "resolved_path": "/workspace/assets/images/missing-diagram.png",
      "autofixed": false,
      "suggestion": "请确认文件是否已放入正确目录，或修正文件名拼写"
    }
  ],
  "summary": "共发现3项图片资源违规，均需人工处理（下载/移动/补充图片）"
}
```

