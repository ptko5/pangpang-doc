# API接口统一规范

## 1. 文档说明

| 属性 | 说明 |
|------|------|
| 文档版本 | V1.0 |
| 生效日期 | 2026-06-30 |
| 适用范围 | 公司所有前后端开发团队 |
| 作者 | 架构组 |

---

## 2. 接口设计原则

### 2.1 RESTful设计原则

#### 【强制】RESTful规范

| HTTP方法 | 操作类型 | 说明 |
|---------|---------|------|
| **GET** | 查询 | 获取资源列表或单个资源 |
| **POST** | 创建 | 创建新资源 |
| **PUT** | 更新 | 全量更新资源 |
| **PATCH** | 部分更新 | 部分更新资源 |
| **DELETE** | 删除 | 删除资源 |

### 2.2 资源命名规范

#### 【强制】URL命名规则

- 使用 **小写英文**
- 使用 **下划线** 分隔单词
- 使用 **复数形式** 表示资源集合
- 禁止使用动词

#### ✅ 正确示例

```
GET    /api/v1/users                  # 获取用户列表
GET    /api/v1/users/{id}             # 获取单个用户
POST   /api/v1/users                  # 创建用户
PUT    /api/v1/users/{id}             # 更新用户
DELETE /api/v1/users/{id}             # 删除用户
GET    /api/v1/users/{id}/orders      # 获取用户订单列表
POST   /api/v1/users/{id}/orders      # 创建用户订单
```

#### ❌ 错误示例

```
GET    /api/v1/getUserList            # 错误：使用动词
POST   /api/v1/createUser             # 错误：使用动词
GET    /api/v1/user/getById?id=1      # 错误：使用动词和查询参数
```

### 2.3 版本管理

#### 【强制】版本号规范

- 版本号放在URL路径中
- 使用 `v{数字}` 格式
- 主版本升级时更新版本号

#### ✅ 正确示例

```
/api/v1/users
/api/v2/users
/api/v1/orders
/api/v2/orders
```

---

## 3. 请求规范

### 3.1 请求头规范

#### 【强制】标准请求头

| 请求头 | 说明 | 必填 | 示例 |
|--------|------|------|------|
| `Content-Type` | 请求内容类型 | ✅ | `application/json` |
| `Authorization` | 认证令牌 | ✅ | `Bearer {token}` |
| `Accept` | 接受类型 | ✅ | `application/json` |
| `X-Request-Id` | 请求ID | ✅ | `uuid` |
| `X-Tenant-Id` | 租户ID | ❌ | `tenant-001` |

#### ✅ 正确示例

```http
POST /api/v1/users HTTP/1.1
Content-Type: application/json
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Accept: application/json
X-Request-Id: 550e8400-e29b-41d4-a716-446655440000
```

### 3.2 请求参数规范

#### 【强制】参数传递方式

| 参数类型 | 传递方式 | 示例 |
|---------|---------|------|
| **路径参数** | URL路径 | `/api/v1/users/{id}` |
| **查询参数** | URL查询字符串 | `/api/v1/users?page=1&size=10` |
| **请求体** | JSON格式 | `{"username": "test", "email": "test@example.com"}` |

#### 【强制】查询参数命名

- 使用 **小驼峰** 命名
- 使用 **filter** 前缀表示过滤条件
- 使用 **sort** 表示排序
- 使用 **page/size** 表示分页

#### ✅ 正确示例

```
GET /api/v1/users?page=1&size=10&status=1&sort=createdAt,desc
GET /api/v1/orders?filter=status:1,type:2&page=1&size=20
```

### 3.3 请求体规范

#### 【强制】请求体格式

- 使用 **JSON** 格式
- 使用 **小驼峰** 命名
- 嵌套对象使用 **小驼峰** 命名

#### ✅ 正确示例

```json
{
  "username": "test",
  "email": "test@example.com",
  "password": "password123",
  "profile": {
    "realName": "张三",
    "phone": "13800138000",
    "address": "北京市朝阳区"
  }
}
```

---

## 4. 响应规范

### 4.1 统一响应格式

#### 【强制】响应结构

```json
{
  "code": 200,
  "message": "success",
  "data": {
    "id": 1,
    "username": "test",
    "email": "test@example.com"
  },
  "timestamp": 1625097600000,
  "requestId": "550e8400-e29b-41d4-a716-446655440000"
}
```

| 字段 | 类型 | 说明 | 必填 |
|------|------|------|------|
| `code` | Integer | 业务状态码 | ✅ |
| `message` | String | 响应消息 | ✅ |
| `data` | Object/Array | 响应数据 | ✅ |
| `timestamp` | Long | 响应时间戳 | ✅ |
| `requestId` | String | 请求ID | ✅ |

### 4.2 状态码规范

#### 【强制】HTTP状态码

| 状态码 | 说明 | 示例 |
|--------|------|------|
| **200** | 请求成功 | 查询、更新成功 |
| **201** | 创建成功 | 创建资源成功 |
| **204** | 无内容 | 删除成功 |
| **400** | 请求参数错误 | 参数校验失败 |
| **401** | 未授权 | Token无效或过期 |
| **403** | 禁止访问 | 权限不足 |
| **404** | 资源不存在 | 未找到资源 |
| **409** | 冲突 | 资源重复或状态冲突 |
| **500** | 服务器错误 | 系统内部错误 |
| **503** | 服务不可用 | 服务降级或熔断 |

#### 【强制】业务状态码

| 状态码 | 说明 |
|--------|------|
| **200** | 成功 |
| **40001** | 参数校验失败 |
| **40002** | 请求格式错误 |
| **40101** | Token无效 |
| **40102** | Token过期 |
| **40103** | 未登录 |
| **40301** | 权限不足 |
| **40401** | 资源不存在 |
| **40901** | 资源重复 |
| **50001** | 系统错误 |
| **50002** | 数据库错误 |

### 4.3 分页响应格式

#### 【强制】分页响应结构

```json
{
  "code": 200,
  "message": "success",
  "data": {
    "list": [
      { "id": 1, "username": "test1" },
      { "id": 2, "username": "test2" }
    ],
    "total": 100,
    "page": 1,
    "size": 10,
    "pages": 10
  },
  "timestamp": 1625097600000,
  "requestId": "550e8400-e29b-41d4-a716-446655440000"
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `list` | Array | 数据列表 |
| `total` | Long | 总记录数 |
| `page` | Integer | 当前页码 |
| `size` | Integer | 每页大小 |
| `pages` | Integer | 总页数 |

---

## 5. 错误响应规范

### 5.1 错误响应格式

#### 【强制】错误响应结构

```json
{
  "code": 40001,
  "message": "参数校验失败",
  "data": {
    "errors": [
      { "field": "username", "message": "用户名不能为空" },
      { "field": "email", "message": "邮箱格式不正确" }
    ]
  },
  "timestamp": 1625097600000,
  "requestId": "550e8400-e29b-41d4-a716-446655440000"
}
```

#### 【强制】通用错误响应

```json
{
  "code": 50001,
  "message": "系统内部错误，请稍后重试",
  "data": null,
  "timestamp": 1625097600000,
  "requestId": "550e8400-e29b-41d4-a716-446655440000"
}
```

---

## 6. 接口示例

### 6.1 用户管理接口

#### 【示例】创建用户

**请求：**

```http
POST /api/v1/users HTTP/1.1
Content-Type: application/json
Authorization: Bearer {token}
X-Request-Id: 550e8400-e29b-41d4-a716-446655440000

{
  "username": "test",
  "email": "test@example.com",
  "password": "password123",
  "status": 1
}
```

**响应：**

```json
{
  "code": 200,
  "message": "创建成功",
  "data": {
    "id": 1,
    "username": "test",
    "email": "test@example.com",
    "status": 1,
    "createdAt": "2026-06-30T10:00:00",
    "updatedAt": "2026-06-30T10:00:00"
  },
  "timestamp": 1625097600000,
  "requestId": "550e8400-e29b-41d4-a716-446655440000"
}
```

#### 【示例】查询用户列表

**请求：**

```http
GET /api/v1/users?page=1&size=10&status=1&sort=createdAt,desc HTTP/1.1
Authorization: Bearer {token}
X-Request-Id: 550e8400-e29b-41d4-a716-446655440001
```

**响应：**

```json
{
  "code": 200,
  "message": "success",
  "data": {
    "list": [
      {
        "id": 2,
        "username": "test2",
        "email": "test2@example.com",
        "status": 1,
        "createdAt": "2026-06-30T10:05:00",
        "updatedAt": "2026-06-30T10:05:00"
      },
      {
        "id": 1,
        "username": "test",
        "email": "test@example.com",
        "status": 1,
        "createdAt": "2026-06-30T10:00:00",
        "updatedAt": "2026-06-30T10:00:00"
      }
    ],
    "total": 100,
    "page": 1,
    "size": 10,
    "pages": 10
  },
  "timestamp": 1625097600000,
  "requestId": "550e8400-e29b-41d4-a716-446655440001"
}
```

#### 【示例】查询单个用户

**请求：**

```http
GET /api/v1/users/1 HTTP/1.1
Authorization: Bearer {token}
X-Request-Id: 550e8400-e29b-41d4-a716-446655440002
```

**响应：**

```json
{
  "code": 200,
  "message": "success",
  "data": {
    "id": 1,
    "username": "test",
    "email": "test@example.com",
    "status": 1,
    "createdAt": "2026-06-30T10:00:00",
    "updatedAt": "2026-06-30T10:00:00"
  },
  "timestamp": 1625097600000,
  "requestId": "550e8400-e29b-41d4-a716-446655440002"
}
```

#### 【示例】更新用户

**请求：**

```http
PUT /api/v1/users/1 HTTP/1.1
Content-Type: application/json
Authorization: Bearer {token}
X-Request-Id: 550e8400-e29b-41d4-a716-446655440003

{
  "username": "test_update",
  "email": "test_update@example.com",
  "status": 1
}
```

**响应：**

```json
{
  "code": 200,
  "message": "更新成功",
  "data": {
    "id": 1,
    "username": "test_update",
    "email": "test_update@example.com",
    "status": 1,
    "createdAt": "2026-06-30T10:00:00",
    "updatedAt": "2026-06-30T11:00:00"
  },
  "timestamp": 1625097600000,
  "requestId": "550e8400-e29b-41d4-a716-446655440003"
}
```

#### 【示例】删除用户

**请求：**

```http
DELETE /api/v1/users/1 HTTP/1.1
Authorization: Bearer {token}
X-Request-Id: 550e8400-e29b-41d4-a716-446655440004
```

**响应：**

```json
{
  "code": 200,
  "message": "删除成功",
  "data": null,
  "timestamp": 1625097600000,
  "requestId": "550e8400-e29b-41d4-a716-446655440004"
}
```

---

## 7. 接口文档规范

### 7.1 文档格式

#### 【强制】接口文档要求

每个接口必须包含以下内容：

| 项目 | 说明 |
|------|------|
| **接口名称** | 清晰描述接口功能 |
| **接口路径** | 完整URL路径 |
| **HTTP方法** | GET/POST/PUT/PATCH/DELETE |
| **所属模块** | 接口所属业务模块 |
| **接口描述** | 详细描述接口功能 |
| **请求头** | 请求头参数列表 |
| **请求参数** | 路径参数、查询参数、请求体 |
| **响应参数** | 响应数据结构 |
| **错误码** | 可能返回的错误码 |
| **示例** | 请求和响应示例 |

### 7.2 文档工具

#### 【推荐】文档工具选择

| 工具 | 说明 |
|------|------|
| **Swagger/OpenAPI** | 自动生成接口文档 |
| **Postman** | API测试和文档 |
| **Apifox** | 接口管理和测试 |

---

## 8. 落地检查清单

| 序号 | 检查项 | 检查方式 | 责任人 |
|------|--------|---------|--------|
| 1 | URL是否使用小写+下划线 | 检查接口文档 | 开发人员 |
| 2 | URL是否使用复数形式 | 检查接口文档 | 开发人员 |
| 3 | 是否使用HTTP方法表示操作类型 | 检查接口文档 | 开发人员 |
| 4 | 是否包含版本号 | 检查接口文档 | 开发人员 |
| 5 | 请求头是否完整 | 检查代码 | 开发人员 |
| 6 | 响应格式是否统一 | 检查代码 | 开发人员 |
| 7 | 状态码是否符合规范 | 检查代码 | 开发人员 |
| 8 | 是否包含分页响应 | 检查列表接口 | 开发人员 |
| 9 | 是否包含错误码定义 | 检查代码 | 开发人员 |
| 10 | 是否有完整的接口文档 | 检查文档 | 开发人员 |

---

**文档结束**

*本规范由架构组制定，解释权归架构组所有。*