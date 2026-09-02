# AI 前端项目搭建指南

## 1. 文档说明

| 属性 | 说明 |
|------|------|
| 文档版本 | V1.0 |
| 生效日期 | 2026-09-02 |
| 适用范围 | AI 编程助手（Trae 等）用于独立搭建前端项目 |
| 作者 | 架构组 |

> **用途**：本文档是 AI 搭建前端项目的**唯一输入依据**。AI 仅凭本文档即可独立完成一个可运行、可验收的前端工程（管理后台/业务系统）。文档中的【强制】项必须逐条满足，【推荐】项在条件允许时优先满足。

---

## 2. 项目背景说明

公司要建设一套 B 端管理系统（管理后台），面向运营与内部人员，包含登录鉴权、工作台、业务列表、详情、表单、图表等典型页面。项目需满足：

- **多人并行开发**：代码结构清晰、模块解耦，支持多成员同时开发不同业务模块。
- **快速迭代**：路由与菜单可配置化，新增页面不修改框架代码。
- **中后台组件化**：基于成熟 UI 组件库快速搭建统一风格页面。
- **可维护性**：强类型、统一规范、可测试、可审计。
- **性能与体验**：首屏加载快、代码按需加载、交互流畅。

> **交付形态**：本文档要求产出一个**可直接 `npm run dev` 运行**的前端工程，包含登录页与鉴权、可配置路由菜单框架、若干示例业务页面、统一的请求封装与错误处理、样式体系、单元测试与构建配置，以及 README 说明如何启动与打包。

---

## 3. 开发目标

### 3.1 总目标

AI 依据本文档产出一个**前端工程**，满足：

1. 工程可 `npm install` + `npm run dev` 直接运行，`npm run build` 可产出生产包。
2. 登录 + Token 持久化 + 路由鉴权（未登录跳登录页，无权限页 403）。
3. 统一请求封装（Axios）：BaseURL、拦截器、统一错误提示、Loading、Token 注入。
4. 可配置的侧边菜单 + 路由，新增页面只需新增路由与菜单配置。
5. 提供 2~3 个示例业务页（列表 + 表单 + 详情）作为模板。
6. 配置 TypeScript 严格模式、ESLint + Prettier、单元测试用例。

### 3.2 验收性目标（可直接核验）

| 目标 | 验收方式 |
|------|---------|
| `npm run dev` 启动无报错 | 浏览器打开首页正常渲染 |
| `npm run build` 打包成功 | 产物产出且无 TS/构建错误 |
| 登录流程闭环 | 输入账号密码 → 获取 Token → 跳转首页 → 刷新不掉登录态 |
| 路由鉴权生效 | 未登录访问受保护路由自动跳登录 |
| 请求统一封装生效 | 接口 401 自动跳登录、业务错误统一 toast |
| 代码规范通过 | `npm run lint` 无 error |

---

## 4. 技术栈选择

### 4.1 技术栈（【强制】对齐公司标准）

| 组件 | 版本 | 用途 |
|------|------|------|
| React | 19.x | UI 框架 |
| TypeScript | 5.x | 类型系统（严格模式） |
| Vite | 6.x | 构建工具（开发服务器 + 打包） |
| Ant Design | 5.x | UI 组件库 |
| Zustand | 4.x | 状态管理 |
| React Router | 6.x | 路由 |
| Axios | 1.x | HTTP 客户端 |
| Day.js | 1.x | 日期处理（AntD 内置依赖） |
| Vite | — | 亦支持 ESLint 9 + Prettier + Vitest + Testing Library |

> 【推荐】脚手架基于 Vite 官方 React-TS 模板初始化，再叠加上述依赖；【禁止】引入未选型的重量级框架（如 Umi、Next.js 服务端渲染），除非需求明确。

### 4.2 目录选型说明

- 组件库统一使用 Ant Design 5.x，配合其 `theme` token 定制主题，不额外引入 CSS 框架（如 Tailwind），保持一致性（若团队已标准化 Tailwind 可另行确认）。

---

## 5. 项目结构规划

### 5.1 顶层结构（【强制】交付物）

```text
frontend/
├── index.html
├── package.json
├── vite.config.ts
├── tsconfig.json
├── tsconfig.node.json
├── .eslintrc / eslint.config.js
├── .prettierrc
├── .env.development
├── .env.production
├── public/
├── src/
│   ├── main.tsx                 # 入口：挂载、全局 Provider
│   ├── App.tsx                  # 根组件：路由出口 + 布局
│   ├── router/                  # 路由配置 + 守卫
│   ├── layouts/                 # 主布局（侧边栏/顶栏/内容区）
│   ├── pages/                   # 页面（按业务模块分目录）
│   ├── components/              # 通用组件（跨页面复用）
│   ├── stores/                  # Zustand store
│   ├── api/                     # 接口定义（按模块）
│   ├── hooks/                   # 自定义 hooks
│   ├── utils/                   # 工具函数（请求封装、鉴权、格式化）
│   ├── types/                   # 全局类型定义
│   ├── styles/                  # 全局样式与主题变量
│   └── test/                    # 测试工具与 setup
└── README.md                    # 启动/打包/目录说明
```

### 5.2 模块划分约定（【强制】）

- `pages/` 按业务模块分目录：`pages/{module}/{Feature}.tsx`。
- `api/` 按业务模块分文件：`api/{module}.ts`，一个模块一个文件，函数返回类型明确。
- 组件分类：`components/` 放**跨页面复用**组件；页面独有组件放页面目录下的 `components/`。
- 新增业务模块时：在 `pages/`、`api/`、`router` 菜单配置三处各加一处，**不改框架代码**。

---

## 6. 组件设计规范

### 6.1 组件编写原则（【强制】）

1. **函数组件优先**：一律使用函数组件 + Hooks，禁止 class 组件。
2. **props 类型化**：所有 props 用 TypeScript 定义 `interface`，禁止 `any`。
3. **组件单一职责**：一个组件只做一件事；超过 200 行的组件应拆分。
4. **受控与解耦**：表单类组件受控；业务组件从父级注入数据，不直接调用 store/API（数据层由页面层负责）。
5. **命名**：组件文件大驼峰 `OrderList.tsx`；通用组件以语义命名，`index.tsx` 仅用于目录入口。

### 6.2 通用组件要求

- 基于 AntD 封装少量业务通用组件（如 `ProTable` 风格列表容器、`PageContainer` 页面容器、`EmptyState`），保持封装简单、可透传。
- 禁止为"封装而封装"：AntD 已满足的直接用 AntD，避免重复包装。

---

## 7. 状态管理方案

### 7.1 选型与原则（【强制】）

- 统一使用 **Zustand** 管理全局状态；局部状态用组件内 `useState` / `useReducer`。
- 全局状态只放**跨页面共享且会被修改**的数据：登录用户信息、权限/角色、全局配置（主题、布局偏好）。
- **服务端数据不放入全局 store**：列表/详情等接口数据由页面/请求层管理，使用 React Query 或自定义 hook + 本地状态。
- 一个 store 一个 `create` 文件，`stores/{name}.ts`，导出 hook 使用。

### 7.2 示例：用户 store（交付物）

```ts
import { create } from 'zustand';

interface UserState {
  user: UserInfo | null;
  token: string | null;
  setLogin: (token: string, user: UserInfo) => void;
  logout: () => void;
}

export const useUserStore = create<UserState>((set) => ({
  user: null,
  token: localStorage.getItem('token'),
  setLogin: (token, user) => {
    localStorage.setItem('token', token);
    set({ token, user });
  },
  logout: () => {
    localStorage.removeItem('token');
    set({ user: null, token: null });
  },
}));
```

---

## 8. 路由设计

### 8.1 路由结构（【强制】）

- 使用 React Router 6.x，`createBrowserRouter`。
- 路由配置与菜单配置**同源**：定义一份 `routes` 配置（含 `path`、`title`、`icon`、`component`、`roles`），同时驱动路由与侧边菜单渲染。

### 8.2 路由守卫（【强制】）

- 前置校验：未登录（无有效 Token）访问受保护路由 → 跳转 `/login`，并携带 `redirect` 回跳。
- 权限校验：路由配置 `roles`，用户无对应角色 → 渲染 403 页面。
- 登录后登录页已访问 → 跳回首页。
- 全局 404 兜底页面。

### 8.3 懒加载（【强制】）

- 页面组件统一 `React.lazy(() => import('@/pages/...'))` 按需加载。
- 顶层 Layout 不懒加载；`Suspense` 包裹路由出口并显示全局 Loading。

---

## 9. 接口对接规范

### 9.1 统一请求封装（【强制】交付 `utils/request.ts`）

- 基于 Axios 实例：`baseURL` 从环境变量读取（`.env.development` / `.env.production`）。
- 请求拦截器：注入 `Authorization: Bearer {token}`。
- 响应拦截器：
  - 业务成功（`code === 0`）→ 返回 `data`。
  - 业务错误 → 统一 `message` 提示，`reject`。
  - HTTP 401 → 清除登录态并跳转登录页。
  - HTTP 5xx/网络错误 → 统一错误提示。
- 导出类型化方法：`get<T>` / `post<T>` / `put<T>` / `delete<T>`，返回 `Promise<T>`。

### 9.2 API 定义规范（【强制】）

- `api/{module}.ts` 每个接口一个函数，入参/返回值类型化，禁止散落的裸 `axios` 调用。

```ts
// api/order.ts
import { request } from '@/utils/request';

export interface OrderPageParams { page: number; size: number; status?: number }
export interface OrderItem { id: number; orderNo: string; amount: number; status: number }

export const fetchOrders = (params: OrderPageParams) =>
  request.get<{ list: OrderItem[]; total: number }>('/api/v1/orders', { params });
```

### 9.3 联调约定

- 环境变量区分 dev/prod 的 `VITE_API_BASE_URL`。
- 开发期支持 mock（如 MSW 或 vite 插件），便于无后端并行开发；mock 开关通过环境变量控制。

---

## 10. 样式体系

### 10.1 全局样式（【强制】）

- 使用 **CSS 变量**定义主题 token：颜色、圆角、间距、字体，统一在 `styles/theme.css` 声明，并在 `:root` 上生效。
- 布局与间距禁止魔法数字散落，使用 token 变量。
- 全局样式（`styles/global.css`）只放 reset 与公共类；业务样式使用 **CSS Modules** 或 scoped 命名，避免全局污染。

### 10.2 组件样式

- 组件级样式用 CSS Modules（`xxx.module.css`）。
- 基于 AntD 主题定制：通过 `ConfigProvider` 的 `theme.token` 统一品牌色、圆角。
- 深色模式/主题切换（【推荐】）：预留 CSS 变量 + AntD 算法切换（`defaultAlgorithm` / `darkAlgorithm`）。

---

## 11. 构建配置

### 11.1 Vite 配置（【强制】交付 `vite.config.ts`）

- 别名：`@` → `src`（tsconfig 同步 `paths`）。
- 开发服务器：`proxy` 将 `/api` 代理到后端网关，避免跨域。
- 构建：`build.outDir`、`chunkSizeWarningLimit`、`sourcemap`（生产关闭）。
- 分包策略：`manualChunks` 将 `react`、`antd`、`axios` 等公共依赖拆成独立 chunk，配合 `react-router`、`zustand` 等。
- 环境变量：`vite-env.d.ts` 定义 `ImportMetaEnv` 类型。

### 11.2 质量工具（【强制】）

- ESLint：React Hooks 规则、TypeScript 严格规则，`npm run lint` 无 error。
- Prettier：统一格式化，提交前自动格式化。
- 提交规范：【推荐】引入 husky + lint-staged，commit 前跑 lint + 单测。

### 11.3 环境配置

- `.env.development`：`VITE_API_BASE_URL=/api`（走 proxy）。
- `.env.production`：`VITE_API_BASE_URL=<生产网关地址>`。
- 所有配置项以 `VITE_` 前缀暴露，禁止在代码里硬编码环境地址。

---

## 12. 性能优化

### 12.1 首屏与加载（【强制】）

- 路由懒加载 + 分包，首屏仅加载登录/首页所需 chunk。
- 图片懒加载、图标按需引入（AntD 图标 tree-shaking）。
- 生产包 gzip/brotli 压缩（Vite 插件或 Nginx 层开启）。

### 12.2 运行时性能（【强制】）

- 列表分页 + 虚拟滚动（超 1000 行时用虚拟滚动组件）。
- 大组件用 `React.memo` 包裹（props 稳定时），`useMemo`/`useCallback` 仅在必要时使用，避免滥用。
- 事件防抖/节流：搜索、滚动、resize 等高频事件。
- 【推荐】接口数据缓存：用 React Query 的 `staleTime` 缓存列表，减少重复请求。

### 12.3 打包体积监控

- CI 中检查产物体积，超过阈值（如 gzip 后单 chunk > 300KB）告警提示拆分。

---

## 13. 可访问性要求（A11y）

### 13.1 基础要求（【强制】）

- 语义化标签：`<button>`、`<a>`、`<nav>`、`<header>`、`<main>`，禁止用 `div` 模拟可点击元素而不加语义与键盘支持。
- 表单控件必须有 `label`（`htmlFor` 关联 `id`）。
- 图片必须有 `alt` 文本；图标按钮必须有 `aria-label`。
- 键盘可操作：所有交互元素可 `Tab` 聚焦、`Enter`/`Space` 触发；焦点顺序合理。

### 13.2 增强要求（【推荐】）

- 色彩对比度满足 WCAG AA（正文 ≥ 4.5:1）。
- 页面标题随路由更新（`document.title`），利于屏幕阅读器。
- 加载/错误/空态有文字提示，不依赖纯视觉。
- 动画尊重 `prefers-reduced-motion`（减少动效）。

---

## 14. 验收标准

### 14.1 功能验收（全部通过才算完成）

| 序号 | 验收项 | 判定 |
|------|--------|------|
| 1 | `npm install && npm run dev` 无报错 | 首页正常渲染 |
| 2 | `npm run build` 成功 | 产物产出、无 TS 错误 |
| 3 | 登录 → 跳首页 → 刷新保持登录态 | 全流程闭环 |
| 4 | 未登录访问受保护页跳登录 | 守卫生效 |
| 5 | 无权限访问 403 页 | 权限校验生效 |
| 6 | 接口 401 自动跳登录、业务错误统一提示 | 拦截器生效 |
| 7 | 路由懒加载生效 | Network 中页面 chunk 按需加载 |
| 8 | 新增一个示例页面仅改路由+菜单+页面三处 | 骨架可扩展验证 |
| 9 | `npm run lint` 无 error | 规范校验通过 |
| 10 | 单测用例通过 | `npm run test` 通过 |

### 14.2 质量验收

- 核心逻辑（登录鉴权、请求封装）有单元测试，关键分支覆盖。
- 无 `TODO`/`待补充`/`console.log` 残留（调试日志清理）。
- 无硬编码环境地址与密钥；类型无 `any` 滥用（除边界）。

---

## 15. 注意事项

1. **版本对齐**：依赖版本以本文档第 4 节为准，禁止自行升级大版本。
2. **不重复封装**：AntD 已提供的功能直接用，避免过度封装导致维护成本。
3. **数据流清晰**：服务端数据不入全局 store，避免状态不同步；变更走 API 后刷新。
4. **安全**：不在前端存放密钥；Token 存 `localStorage`（短期）需结合后端短时效；敏感操作二次确认。
5. **可访问性不是可选项**：按钮/表单/图片的语义要求按第 13 节执行。
6. **可复现**：README 必须让新成员从零环境按步骤复现启动与打包。
7. **本指南输入输出**：AI 应将本指南视为"需求 + 约束"，逐章实现，并在 README 或验收清单中标注每条验收项落实情况。

---

## 16. 落地检查清单

| 序号 | 检查项 | 责任人 |
|------|--------|--------|
| 1 | 目录结构按第 5 节落地，新增模块三处扩展 | AI |
| 2 | 请求封装 + 拦截器 + 401/错误处理生效 | AI |
| 3 | 路由守卫（登录/权限/404）生效 | AI |
| 4 | 全局样式用 CSS 变量 + 主题 token | AI |
| 5 | 路由懒加载 + 分包 + 压缩 | AI |
| 6 | 表单/图片/按钮语义化与键盘可操作 | AI |
| 7 | lint 与单测通过 | AI |
| 8 | 全部验收项通过（第 14 节） | 架构评审 |

---

**文档结束**

*本文档由 pangpang-doc 维护*
