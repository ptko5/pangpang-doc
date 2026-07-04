# 接口封装与请求规范

## 1. 文档说明

| 属性 | 说明 |
|------|------|
| 文档版本 | V1.0 |
| 生效日期 | 2026-06-30 |
| 适用范围 | 公司所有前端开发团队 |
| 作者 | 架构组 |

---

## 2. 请求工具封装

### 2.1 axios封装

#### 【强制】请求工具基础配置

```typescript
import axios, { AxiosInstance, AxiosRequestConfig, AxiosResponse } from 'axios';

const BASE_URL = import.meta.env.VITE_API_BASE_URL || '/api';

const request: AxiosInstance = axios.create({
  baseURL: BASE_URL,
  timeout: 30000,
  headers: {
    'Content-Type': 'application/json',
  },
});

export default request;
```

### 2.2 请求拦截器

#### 【强制】请求拦截器配置

```typescript
request.interceptors.request.use(
  (config: AxiosRequestConfig) => {
    const token = storage.getToken();
    if (token && config.headers) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  (error) => {
    return Promise.reject(error);
  }
);
```

### 2.3 响应拦截器

#### 【强制】响应拦截器配置

```typescript
import { HTTP_STATUS_UNAUTHORIZED } from '@/constants/http';
import { useAuthStore } from '@/stores/auth.store';

request.interceptors.response.use(
  (response: AxiosResponse) => {
    const { code, message, data } = response.data;
    
    if (code !== 10000) {
      message.error(message);
      return Promise.reject(new Error(message));
    }
    
    return data;
  },
  (error) => {
    if (error.response) {
      const { status, data } = error.response;
      
      if (status === HTTP_STATUS_UNAUTHORIZED) {
        useAuthStore.getState().logout();
        window.location.href = '/login';
      } else {
        const message = data?.message || '请求失败';
        message.error(message);
      }
    } else if (error.request) {
      message.error('网络请求失败，请检查网络连接');
    } else {
      message.error('请求配置错误');
    }
    
    return Promise.reject(error);
  }
);
```

---

## 3. API接口封装

### 3.1 API文件组织

#### 【强制】API文件命名规范

- API文件使用 `.api.ts` 后缀
- 按业务模块划分API文件

```
src/api/
├── user.api.ts       # 用户模块API
├── order.api.ts      # 订单模块API
├── auth.api.ts       # 认证模块API
└── index.ts          # 统一导出
```

### 3.2 API接口定义

#### 【强制】API接口封装规范

```typescript
import request from '@/utils/request';
import type { User, CreateUserRequest, UpdateUserRequest, UserQuery, PageResult } from '@/types';

export const getUserList = (params: UserQuery): Promise<PageResult<User>> => {
  return request.get('/users', { params });
};

export const getUserById = (id: number): Promise<User> => {
  return request.get(`/users/${id}`);
};

export const createUser = (data: CreateUserRequest): Promise<User> => {
  return request.post('/users', data);
};

export const updateUser = (id: number, data: UpdateUserRequest): Promise<User> => {
  return request.put(`/users/${id}`, data);
};

export const deleteUser = (id: number): Promise<void> => {
  return request.delete(`/users/${id}`);
};
```

### 3.3 API统一导出

#### 【强制】API统一导出

```typescript
// api/index.ts
export * from './user.api';
export * from './order.api';
export * from './auth.api';
```

---

## 4. 请求规范

### 4.1 请求方法使用

#### 【强制】HTTP方法使用规范

| 方法 | 用途 | 示例 |
|------|------|------|
| **GET** | 查询数据 | `getUserList`, `getUserById` |
| **POST** | 创建数据 | `createUser`, `login` |
| **PUT** | 更新数据（全量） | `updateUser` |
| **PATCH** | 更新数据（部分） | `updateUserStatus` |
| **DELETE** | 删除数据 | `deleteUser` |

### 4.2 参数传递

#### 【强制】参数传递规范

```typescript
// GET请求 - 参数放在query中
export const getUserList = (params: UserQuery): Promise<PageResult<User>> => {
  return request.get('/users', { params });
};

// POST请求 - 参数放在body中
export const createUser = (data: CreateUserRequest): Promise<User> => {
  return request.post('/users', data);
};

// PUT请求 - 参数放在body中，ID放在URL中
export const updateUser = (id: number, data: UpdateUserRequest): Promise<User> => {
  return request.put(`/users/${id}`, data);
};

// DELETE请求 - ID放在URL中
export const deleteUser = (id: number): Promise<void> => {
  return request.delete(`/users/${id}`);
};

// 批量删除 - 参数放在query中
export const batchDeleteUsers = (ids: number[]): Promise<void> => {
  return request.delete('/users', { params: { ids } });
};
```

### 4.3 文件上传

#### 【推荐】文件上传封装

```typescript
export const uploadFile = (file: File, onProgress?: (progress: number) => void): Promise<{ url: string }> => {
  const formData = new FormData();
  formData.append('file', file);
  
  return request.post('/upload', formData, {
    headers: {
      'Content-Type': 'multipart/form-data',
    },
    onUploadProgress: (progressEvent) => {
      if (onProgress && progressEvent.total) {
        const progress = Math.round((progressEvent.loaded / progressEvent.total) * 100);
        onProgress(progress);
      }
    },
  });
};

export const uploadFiles = (files: File[]): Promise<{ urls: string[] }> => {
  const formData = new FormData();
  files.forEach(file => formData.append('files', file));
  
  return request.post('/upload/batch', formData, {
    headers: {
      'Content-Type': 'multipart/form-data',
    },
  });
};
```

---

## 5. 错误处理规范

### 5.1 错误分类

#### 【强制】错误类型定义

| 错误类型 | HTTP状态码 | 处理方式 |
|---------|-----------|---------|
| **业务错误** | 200（code !== 10000） | 显示错误消息 |
| **认证错误** | 401 | 跳转登录页 |
| **授权错误** | 403 | 显示无权限提示 |
| **资源不存在** | 404 | 显示资源不存在 |
| **系统错误** | 500 | 显示系统繁忙 |
| **网络错误** | - | 显示网络异常 |

### 5.2 错误处理示例

#### 【推荐】组件内错误处理

```tsx
function UserList() {
  const [users, setUsers] = useState<User[]>([]);
  const [isLoading, setIsLoading] = useState<boolean>(false);
  const [error, setError] = useState<string | null>(null);

  const fetchUsers = async () => {
    setIsLoading(true);
    setError(null);
    try {
      const result = await getUserList({});
      setUsers(result.data);
    } catch (err) {
      setError('获取用户列表失败');
    } finally {
      setIsLoading(false);
    }
  };

  useEffect(() => {
    fetchUsers();
  }, []);

  if (error) {
    return <div>{error}</div>;
  }

  return (
    <div>
      {isLoading ? 'Loading...' : users.map(user => <div key={user.id}>{user.username}</div>)}
    </div>
  );
}
```

---

## 6. 请求优化

### 6.1 请求防抖

#### 【推荐】搜索防抖

```typescript
import { debounce } from 'lodash';

function SearchInput() {
  const [value, setValue] = useState<string>('');
  const [results, setResults] = useState<string[]>([]);

  const search = debounce(async (keyword: string) => {
    if (!keyword.trim()) {
      setResults([]);
      return;
    }
    const response = await searchUsers({ keyword });
    setResults(response.data.map(user => user.username));
  }, 300);

  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    setValue(e.target.value);
    search(e.target.value);
  };

  return (
    <div>
      <input type="text" value={value} onChange={handleChange} />
      <ul>
        {results.map(result => <li key={result}>{result}</li>)}
      </ul>
    </div>
  );
}
```

### 6.2 请求节流

#### 【推荐】滚动加载节流

```typescript
import { throttle } from 'lodash';

function InfiniteScroll() {
  const [items, setItems] = useState<Item[]>([]);
  const [page, setPage] = useState<number>(1);
  const [isLoading, setIsLoading] = useState<boolean>(false);

  const loadMore = throttle(async () => {
    if (isLoading) return;
    setIsLoading(true);
    try {
      const response = await getItems({ page: page + 1 });
      setItems(prev => [...prev, ...response.data]);
      setPage(prev => prev + 1);
    } catch (error) {
      console.error('Failed to load more items:', error);
    } finally {
      setIsLoading(false);
    }
  }, 500);

  useEffect(() => {
    window.addEventListener('scroll', loadMore);
    return () => window.removeEventListener('scroll', loadMore);
  }, [loadMore]);

  return <div>{items.map(item => <div key={item.id}>{item.name}</div>)}</div>;
}
```

### 6.3 请求取消

#### 【推荐】取消重复请求

```typescript
import { CancelTokenSource } from 'axios';

function SearchInput() {
  const [value, setValue] = useState<string>('');
  const [results, setResults] = useState<string[]>([]);
  const [cancelToken, setCancelToken] = useState<CancelTokenSource | null>(null);

  const search = async (keyword: string) => {
    if (cancelToken) {
      cancelToken.cancel('Request cancelled');
    }

    const source = axios.CancelToken.source();
    setCancelToken(source);

    try {
      const response = await request.get('/search', {
        params: { keyword },
        cancelToken: source.token,
      });
      setResults(response.data);
    } catch (error) {
      if (!axios.isCancel(error)) {
        console.error('Search failed:', error);
      }
    }
  };

  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    setValue(e.target.value);
    search(e.target.value);
  };

  return (
    <div>
      <input type="text" value={value} onChange={handleChange} />
    </div>
  );
}
```

---

## 7. 落地检查清单

| 序号 | 检查项 | 检查方式 | 责任人 |
|------|--------|---------|--------|
| 1 | 是否封装统一的request工具 | 检查代码 | 架构师 |
| 2 | 是否配置请求拦截器（添加Token） | 检查代码 | 开发人员 |
| 3 | 是否配置响应拦截器（统一错误处理） | 检查代码 | 开发人员 |
| 4 | API文件是否使用.api.ts后缀 | 检查文件 | 开发人员 |
| 5 | API是否按业务模块划分 | 检查目录 | 开发人员 |
| 6 | 是否使用正确的HTTP方法 | 检查代码 | 开发人员 |
| 7 | 参数传递是否符合规范（query/body/URL） | 检查代码 | 开发人员 |
| 8 | 是否处理401认证过期（跳转登录） | 检查代码 | 开发人员 |
| 9 | 搜索等高频操作是否添加防抖 | 检查代码 | 开发人员 |
| 10 | 是否处理请求取消（避免重复请求） | 检查代码 | 开发人员 |

---

**文档结束**

*本规范由架构组制定，解释权归架构组所有。*