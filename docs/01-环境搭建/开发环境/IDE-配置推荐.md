# IDE 配置推荐

## 1. 文档说明

| 属性 | 说明 |
|------|------|
| 文档版本 | V1.0 |
| 生效日期 | 2026-08-15 |
| 适用范围 | 公司所有研发团队（后端 IntelliJ IDEA / 前端 WebStorm 或 VS Code） |
| 作者 | 架构组 |

> **用途**：统一 IDE 配置基线，确保团队成员在 JDK 25 + Spring Boot 4.x 与 React 19 + TypeScript 5 + Vite 6 环境下拥有一致的开发体验（代码风格、快捷键、插件、调试配置）。

---

## 2. IDE 选型

| 角色 | 推荐 IDE | 备选 | 说明 |
|------|---------|------|------|
| 后端开发 | IntelliJ IDEA（Ultimate 或 Community） | Eclipse | Ultimate 对 Spring Boot 4.x 支持最佳 |
| 前端开发 | WebStorm / VS Code | — | VS Code 轻量免费，WebStorm 调试更专业 |
| 全栈 | IntelliJ IDEA Ultimate | — | 前后端一体化开发 |

> 【推荐】后端统一使用 IntelliJ IDEA，前端以 VS Code 为主（成本低），项目根目录提交 `.editorconfig` 与格式化配置，保证跨 IDE 风格一致。

---

## 3. 后端 IDE 配置（IntelliJ IDEA）

### 3.1 JDK 25 配置

1. 打开 `File → Project Structure → SDK`，点击 `Add SDK → Download JDK`
2. 选择 **Version: 25**、**Vendor: Eclipse Temurin**，点击 Download
3. 在 `Project Settings → Project` 中将 `SDK` 与 `Language Level` 均设为 **25**

> 【强制】项目 Language Level 必须为 25，避免误用旧语法兼容配置。

### 3.2 构建工具集成（Gradle 8）

1. `Settings → Build Tools → Gradle`：
   - `Gradle JVM` 选择 JDK 25
   - `Build and run using` 选择 `Gradle`
2. 打开 Gradle 项目后，等待 IDEA 自动同步依赖

### 3.3 Spring Boot 4 支持

- Ultimate 版本对 Spring Boot 4.x 提供自动补全、`spring-boot:run` 运行、热部署支持
- Community 版本缺少 Spring 支持，需安装 `Spring Assistant` 插件（社区版建议直接升级 Ultimate）

### 3.4 代码风格导入

> 团队统一使用 `.editorconfig` + Checkstyle 风格配置，禁止各自为政。

1. 仓库根目录提交 `.editorconfig`：

```ini
root = true

[*]
charset = utf-8
indent_style = space
indent_size = 4
trim_trailing_whitespace = true
insert_final_newline = true
end_of_line = lf

[*.{yml,yaml,json}]
indent_size = 2
```

2. IDEA 中安装 `CheckStyle-IDEA` 插件并导入团队 `checkstyle.xml`
3. 使用 `Code Style → Import Scheme` 导入团队导出的 `scheme.jar`

---

## 4. 前端 IDE 配置（VS Code）

### 4.1 必装插件清单

| 插件 | 用途 |
|------|------|
| `ESLint` | 代码规范检查 |
| `Prettier - Code formatter` | 统一格式化 |
| `TypeScript Vue Plugin (Volar)` | TypeScript 5 支持 |
| `Vite` | Vite 6 开发服务器支持 |
| `GitLens` | Git 增强 |
| `EditorConfig for VS Code` | 编辑器配置 |
| `Error Lens` | 行内错误提示 |

> 注：Volar 为 Vue 项目所需，纯 React 项目可替换为 `TypeScript` + `React Native Tools` 插件。

### 4.2 统一配置 `.vscode/settings.json`

```json
{
  "editor.formatOnSave": true,
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": "explicit"
  },
  "typescript.tsdk": "node_modules/typescript/lib",
  "files.eol": "\n"
}
```

---

## 5. 通用推荐配置

| 配置项 | 推荐值 | 说明 |
|--------|--------|------|
| 自动保存 | 开启（延迟 1000ms） | 减少 Ctrl+S 干扰 |
| 代码补全大小写敏感 | 不敏感 | 提升补全命中率 |
| 终端 Shell | Windows 用 Git Bash | 统一 Unix 命令习惯 |
| Git 提交 | 配置 GPG 签名 | 追溯安全 |
| 主题字体 | JetBrains Mono / Fira Code | 支持连字，提升可读性 |

> 【推荐】开启"非空文件自动保存"与"代码补全不区分大小写"，新成员可明显降低上手摩擦。

---

## 6. 快捷键速查（IDEA）

| 操作 | Windows/Linux | macOS |
|------|---------------|-------|
| 全局搜索 | `Ctrl + Shift + F` | `Cmd + Shift + F` |
| 跳转实现 | `Ctrl + Alt + B` | `Cmd + Option + B` |
| 重命名 | `Shift + F6` | `Shift + F6` |
| 快速修复 | `Alt + Enter` | `Option + Enter` |
| 运行当前类 | `Ctrl + Shift + F10` | `Ctrl + Shift + R` |
| 生成代码 | `Alt + Insert` | `Cmd + N` |

---

## 7. 落地检查清单

| 序号 | 检查项 | 检查方式 | 责任人 |
|------|--------|---------|--------|
| 1 | IDEA SDK 与 Language Level 为 25 | 项目结构核对 | 后端开发 |
| 2 | Gradle 8 + JDK 25 同步成功 | 构建面板无报错 | 后端开发 |
| 3 | `.editorconfig` 已生效 | 格式化行为验证 | 全栈 |
| 4 | ESLint/Prettier 配置生效 | 保存后自动格式化 | 前端开发 |
| 5 | 团队代码风格已导入 | 与 CI 检查结果一致 | 全栈 |

---

**文档结束**

*本文档由 pangpang-doc 维护*
