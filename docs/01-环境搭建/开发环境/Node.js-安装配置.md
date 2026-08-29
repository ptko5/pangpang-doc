# Node.js 安装配置

## 1. 文档说明

| 属性 | 说明 |
|------|------|
| 文档版本 | V1.0 |
| 生效日期 | 2026-08-15 |
| 适用范围 | 公司所有前端研发团队（Node.js 22 LTS） |
| 作者 | 架构组 |

> **用途**：安装 Node.js 22 LTS、使用 nvm 管理多版本、配置 npm/yarn/pnpm 包管理器与国内镜像源，并完成前端工程初始化配置。

---

## 2. 版本约定

| 项 | 约定 |
|----|------|
| **版本** | Node.js 22 LTS（奇数版本为实验版，禁止用于生产） |
| **包管理器** | pnpm（默认）+ npm/yarn 备选 |
| **前端技术栈** | React 19.x、TypeScript 5.x、Vite 6.x |

> 选择 pnpm 作为默认包管理器：磁盘占用低（硬链接）、安装快、依赖隔离严格，适合 React 19 + Vite 6 工程。

---

## 3. 安装 Node.js

### 3.1 nvm 安装（Linux/macOS，推荐）

> nvm 可自由切换多个 Node 版本，是前端开发的标准选择。

```bash
# 安装 nvm（版本号以官方最新为准）
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash

# 使配置生效
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"

# 安装 Node.js 22 LTS
nvm install 22
nvm alias default 22
nvm use 22

# 验证
node -v   # v22.x.x
npm -v
```

### 3.2 Windows 安装（nvm-windows）

1. 下载 [nvm-windows](https://github.com/coreybutler/nvm-windows/releases) 的 `nvm-setup.exe` 安装
2. 以管理员身份打开 PowerShell：

```powershell
nvm install 22
nvm use 22
node -v
```

> 【推荐】macOS 也可使用 `brew install node@22`，但建议统一采用 nvm 以便多版本共存。

---

## 4. 包管理器配置

### 4.1 npm 镜像源

```bash
# 使用 npmmirror 国内镜像
npm config set registry https://registry.npmmirror.com

# 验证
npm config get registry
```

### 4.2 安装 pnpm（默认包管理器）

```bash
# 通过 corepack 启用（Node 自带）
corepack enable
corepack prepare pnpm@latest --activate

# 或全局安装
npm install -g pnpm

# 配置 pnpm 镜像源
pnpm config set registry https://registry.npmmirror.com
```

### 4.3 配置工程脚本（package.json）

```json
{
  "name": "pangpang-frontend",
  "private": true,
  "version": "1.0.0",
  "type": "module",
  "packageManager": "pnpm@9.0.0",
  "engines": {
    "node": ">=22.0.0",
    "pnpm": ">=9.0.0"
  },
  "scripts": {
    "dev": "vite",
    "build": "tsc -b && vite build",
    "preview": "vite preview",
    "lint": "eslint ."
  },
  "dependencies": {
    "react": "^19.0.0"
  },
  "devDependencies": {
    "typescript": "^5.0.0",
    "vite": "^6.0.0"
  }
}
```

> 【强制】`package.json` 必须声明 `packageManager` 与 `engines` 字段，确保团队成员使用一致的 Node 与包管理器版本。

---

## 5. 常用命令速查

| 场景 | 命令 |
|------|------|
| 安装项目依赖 | `pnpm install` |
| 安装依赖到 dependencies | `pnpm add react-router-dom` |
| 安装依赖到 devDependencies | `pnpm add -D vite` |
| 全局安装 | `pnpm add -g pnpm` |
| 移除依赖 | `pnpm remove axios` |
| 更新依赖 | `pnpm update` |
| 查看依赖树 | `pnpm why axios` |

---

## 6. 常见问题排查

| 问题现象 | 原因 | 解决方案 |
|---------|------|---------|
| 安装依赖缓慢/超时 | 默认官方源网络慢 | 配置 `npmmirror` 镜像源 |
| `node -v` 版本不符 | 未切换到目标版本 | `nvm use 22` / `nvm alias default 22` |
| `EACCES` 权限错误 | 全局安装权限不足 | 使用 nvm 安装，避免 `sudo npm` |
| `corepack` 不可用 | 未启用或版本过低 | `corepack enable` 或升级 Node |
| `lockfile` 版本冲突 | 团队包管理器不一致 | 统一 pnpm，提交 `pnpm-lock.yaml` |

---

## 7. 落地检查清单

| 序号 | 检查项 | 检查方式 | 责任人 |
|------|--------|---------|--------|
| 1 | `node -v` 为 v22.x | 命令验证 | 前端开发 |
| 2 | `nvm` 默认版本为 22 | `nvm alias` | 前端开发 |
| 3 | 镜像源指向 `npmmirror` | `pnpm config get registry` | 前端开发 |
| 4 | `pnpm install` 可正常安装 | 空项目验证 | 前端开发 |
| 5 | `package.json` 含 `packageManager`/`engines` | 代码审查 | 前端开发 |

---

**文档结束**

*本文档由 pangpang-doc 维护*
