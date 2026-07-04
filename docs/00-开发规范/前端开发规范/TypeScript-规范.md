# TypeScript类型规范

## 1. 文档说明

| 属性 | 说明 |
|------|------|
| 文档版本 | V1.0 |
| 生效日期 | 2026-06-30 |
| 适用范围 | 公司所有前端开发团队 |
| 作者 | 架构组 |

---

## 2. 基础类型规范

### 2.1 类型声明

#### 【强制】使用明确类型

- 禁止使用 `any` 类型
- 禁止使用 `var` 声明变量
- 使用 `const` 声明常量，使用 `let` 声明变量

#### ✅ 正确示例

```typescript
const userName: string = '张三';
const userId: number = 1;
const isActive: boolean = true;
const tags: string[] = ['admin', 'user'];
const userIds: number[] = [1, 2, 3];
```

#### ❌ 错误示例

```typescript
const userName: any = '张三';        // 错误：使用any
var userId = 1;                      // 错误：使用var
let tags = ['admin', 'user'];        // 错误：缺少类型声明
```

### 2.2 对象类型声明

#### 【强制】使用interface或type

- 使用 `interface` 定义对象类型（推荐）
- 使用 `type` 定义联合类型、交叉类型等

#### ✅ 正确示例

```typescript
interface User {
  id: number;
  username: string;
  email: string;
  status: UserStatus;
  createdAt: string;
}

interface CreateUserRequest {
  username: string;
  email: string;
  password: string;
}

interface UpdateUserRequest {
  username?: string;
  email?: string;
}

type UserQuery = {
  username?: string;
  email?: string;
  status?: UserStatus;
} & PaginationQuery;
```

#### ❌ 错误示例

```typescript
const user: { id: number; name: string } = { id: 1, name: '张三' };  // 错误：内联类型
```

### 2.3 数组类型声明

#### 【强制】数组类型定义

- 使用 `Type[]` 或 `Array<Type>` 定义数组类型

#### ✅ 正确示例

```typescript
const users: User[] = [];
const userIds: number[] = [1, 2, 3];
const names: Array<string> = ['张三', '李四'];

interface PageResult<T> {
  data: T[];
  total: number;
  pageNum: number;
  pageSize: number;
}
```

---

## 3. 泛型规范

### 3.1 泛型使用

#### 【强制】泛型命名

- 泛型参数使用单个大写字母
- 常用泛型参数命名：`T`, `U`, `V`, `K`, `V`

#### ✅ 正确示例

```typescript
function identity<T>(arg: T): T {
  return arg;
}

interface ResponseData<T> {
  code: number;
  message: string;
  data: T;
}

interface PageResult<T> {
  data: T[];
  total: number;
  pageNum: number;
  pageSize: number;
}

class DataService<T> {
  private data: T[] = [];
  
  add(item: T): void {
    this.data.push(item);
  }
  
  get(index: number): T | undefined {
    return this.data[index];
  }
}
```

#### ❌ 错误示例

```typescript
function identity<Type>(arg: Type): Type { }  // 错误：使用单词作为泛型参数
```

### 3.2 泛型约束

#### 【推荐】泛型约束

```typescript
interface Identifiable {
  id: number;
}

function findById<T extends Identifiable>(items: T[], id: number): T | undefined {
  return items.find(item => item.id === id);
}

interface ResponseData<T extends object = object> {
  code: number;
  message: string;
  data: T;
}
```

---

## 4. 联合类型与交叉类型

### 4.1 联合类型

#### 【强制】联合类型使用

```typescript
type Status = 'active' | 'inactive' | 'locked';
type HttpMethod = 'GET' | 'POST' | 'PUT' | 'DELETE';

function setStatus(status: Status): void {
  // ...
}

function request(method: HttpMethod, url: string): Promise<ResponseData> {
  // ...
}
```

### 4.2 交叉类型

#### 【推荐】交叉类型使用

```typescript
interface BaseEntity {
  createdAt: string;
  updatedAt: string;
}

interface User {
  id: number;
  username: string;
}

type UserResponse = User & BaseEntity;

function getUser(): UserResponse {
  return {
    id: 1,
    username: '张三',
    createdAt: '2026-06-30',
    updatedAt: '2026-06-30',
  };
}
```

---

## 5. 类型别名

### 5.1 类型别名定义

#### 【推荐】类型别名使用

```typescript
type ID = number | string;
type Email = string;
type Phone = string;
type Timestamp = string;
type JSONValue = string | number | boolean | object | null;

type ApiResponse<T> = {
  code: number;
  message: string;
  data: T;
};

type ApiError = {
  code: number;
  message: string;
  errors?: string[];
};
```

---

## 6. 枚举规范

### 6.1 枚举定义

#### 【强制】枚举使用

```typescript
export enum UserStatus {
  ACTIVE = 'ACTIVE',
  INACTIVE = 'INACTIVE',
  LOCKED = 'LOCKED',
}

export enum OrderStatus {
  PENDING = 'PENDING',
  PAID = 'PAID',
  SHIPPED = 'SHIPPED',
  COMPLETED = 'COMPLETED',
  CANCELLED = 'CANCELLED',
}

export enum HttpMethod {
  GET = 'GET',
  POST = 'POST',
  PUT = 'PUT',
  DELETE = 'DELETE',
}
```

### 6.2 枚举值获取

#### 【推荐】枚举值遍历

```typescript
const statusOptions = Object.entries(UserStatus).map(([key, value]) => ({
  label: key,
  value,
}));
```

---

## 7. 类型断言

### 7.1 类型断言规范

#### 【强制】类型断言使用

- 使用 `as` 语法进行类型断言
- 禁止使用尖括号语法

#### ✅ 正确示例

```typescript
const user = response.data as User;
const input = document.getElementById('username') as HTMLInputElement;
```

#### ❌ 错误示例

```typescript
const user = <User>response.data;        // 错误：使用尖括号语法
```

### 7.2 非空断言

#### 【推荐】非空断言使用

```typescript
const userId = user?.id!;
const username = getUsername()!;
```

---

## 8. 类型守卫

### 8.1 类型守卫使用

#### 【推荐】类型守卫

```typescript
function isString(value: unknown): value is string {
  return typeof value === 'string';
}

function isNumber(value: unknown): value is number {
  return typeof value === 'number';
}

function isUser(value: unknown): value is User {
  return typeof value === 'object' && value !== null && 'id' in value && 'username' in value;
}

function processValue(value: unknown) {
  if (isString(value)) {
    console.log(value.toUpperCase());
  } else if (isNumber(value)) {
    console.log(value + 1);
  }
}
```

---

## 9. TypeScript配置规范

### 9.1 tsconfig.json

#### 【强制】基础配置

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "useDefineForClassFields": true,
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "skipLibCheck": true,
    "moduleResolution": "bundler",
    "allowImportingTsExtensions": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noEmit": true,
    "jsx": "react-jsx",
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noFallthroughCasesInSwitch": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "strictFunctionTypes": true,
    "strictPropertyInitialization": true,
    "alwaysStrict": true,
    "esModuleInterop": true,
    "forceConsistentCasingInFileNames": true,
    "resolvePackageJsonExports": true
  },
  "include": ["src"],
  "references": [{ "path": "./tsconfig.node.json" }]
}
```

### 9.2 严格模式

#### 【强制】启用严格模式

| 选项 | 说明 |
|------|------|
| `strict: true` | 启用所有严格类型检查选项 |
| `noImplicitAny` | 禁止隐式any类型 |
| `strictNullChecks` | 严格的null检查 |
| `strictFunctionTypes` | 严格的函数类型检查 |
| `strictPropertyInitialization` | 严格的属性初始化检查 |
| `alwaysStrict` | 在严格模式下解析代码 |

---

## 10. 类型定义组织

### 10.1 类型定义目录

#### 【强制】类型定义组织

```
src/types/
├── api.ts           # API响应类型
├── user.ts          # 用户相关类型
├── order.ts         # 订单相关类型
├── pagination.ts    # 分页相关类型
└── index.ts         # 统一导出
```

### 10.2 类型导出

#### 【强制】统一导出

```typescript
// types/index.ts
export * from './api';
export * from './user';
export * from './order';
export * from './pagination';
```

---

## 11. 落地检查清单

| 序号 | 检查项 | 检查方式 | 责任人 |
|------|--------|---------|--------|
| 1 | 是否使用明确类型，禁止any | ESLint检查 | 开发人员 |
| 2 | 是否使用const/let，禁止var | ESLint检查 | 开发人员 |
| 3 | 对象类型是否使用interface/type定义 | 检查代码 | 开发人员 |
| 4 | 数组类型是否使用Type[]或Array<Type> | 检查代码 | 开发人员 |
| 5 | 泛型参数是否使用单个大写字母 | 检查代码 | 开发人员 |
| 6 | 是否使用as语法进行类型断言 | 检查代码 | 开发人员 |
| 7 | 是否启用严格模式（strict: true） | 检查配置 | 架构师 |
| 8 | 是否定义类型守卫函数 | 检查代码 | 开发人员 |
| 9 | 类型定义是否统一放在types目录 | 检查目录 | 开发人员 |
| 10 | 是否有统一的类型导出文件 | 检查文件 | 开发人员 |

---

**文档结束**

*本规范由架构组制定，解释权归架构组所有。*