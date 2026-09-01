# React 开发规范

## 1. 文档说明

| 属性 | 说明 |
|------|------|
| 文档版本 | V1.0 |
| 来源 | 飞书云文档 |
| 生效日期 | 2026-08-29 |
| 适用范围 | 公司所有前端研发团队（React 19.x） |
| 作者 | ptko |

> **用途**：涵盖 React 组件设计、Hooks 使用、状态管理、性能优化、测试规范、TypeScript 结合及工程化实践。

---

## 2. 组件设计模式

### 2.1 组件拆分原则

| 类型 | 职责 | 特征 |
|-|-|-|
| UI 组件 | 纯展示，无业务逻辑 | 接收 props 渲染，可复用性强 |
| 业务组件 | 封装业务逻辑和数据处理 | 连接状态管理，包含副作用 |
| 页面组件 | 组合子组件，定义页面结构 | 路由级别，通常对应单一页面 |

### 2.2 函数组件 vs Class 组件

统一推荐使用函数组件 + Hooks，Class 组件不再推荐使用。

```tsx
interface Props {
  title: string;
  onClick?: () => void;
}

const Button: React.FC<Props> = ({ title, onClick }) => {
  return <button onClick={onClick}>{title}</button>;
};
```

### 2.3 受控组件 vs 非受控组件

| 类型 | 适用场景 |
|-|-|
| 受控组件 | 需要实时校验、联动、提交到服务端 |
| 非受控组件 | 仅需获取最终值、文件上传等场景 |

```tsx
// 受控组件
const ControlledInput = () => {
  const [value, setValue] = useState('');
  return <input value={value} onChange={e => setValue(e.target.value)} />;
};

// 非受控组件
const UncontrolledInput = () => {
  const inputRef = useRef<HTMLInputElement>(null);
  return <><input ref={inputRef} /><button onClick={() => console.log(inputRef.current?.value)}>提交</button></>;
};
```

### 2.4 Compound Components（组合组件）模式

适用于需要共享隐式状态且子组件有多种组合方式的场景。

```tsx
const TabsContext = React.createContext<TabsContextValue | null>(null);

const Tabs = ({ children, defaultTab }: { children: React.ReactNode; defaultTab: string }) => {
  const [activeTab, setActiveTab] = useState(defaultTab);
  return (
    <TabsContext.Provider value={{ activeTab, setActiveTab }}>
      <div className="tabs">{children}</div>
    </TabsContext.Provider>
  );
};
```

---

## 3. Hooks 使用规范

> **Hooks 两条核心规则**：只在顶层调用 Hooks，不要在循环、条件或嵌套函数中调用。

### 3.1 useState

```tsx
// 推荐：函数式更新（当新状态依赖旧状态时）
setCount(prev => prev + 1);

// 对象形式更新需合并整个对象
setState(prev => ({ ...prev, name: 'Alice' }));
```

### 3.2 useEffect

```tsx
// 正确：包含所有依赖
useEffect(() => {
  document.title = count + '条消息';
}, [count]);

// 正确：清理副作用
useEffect(() => {
  const subscription = api.subscribe(handleData);
  return () => subscription.unsubscribe();
}, []);
```

### 3.3 useMemo / useCallback

| Hook | 适用场景 | 滥用防护 |
|-|-|-|
| useMemo | 复杂计算结果、引用类型常量 | 简单计算不值得缓存 |
| useCallback | 作为 prop 传递给子组件的函数 | 未作为 props 传递的函数无需 useCallback |

```tsx
const sortedList = useMemo(() => {
  return items.slice().sort((a, b) => a.name.localeCompare(b.name));
}, [items]);

const handleSubmit = useCallback((data: FormData) => {
  dispatch({ type: 'SUBMIT', payload: data });
}, [dispatch]);
```

### 3.4 自定义 Hooks

```tsx
const useUserProfile = (userId: string) => {
  const [user, setUser] = useState<User | null>(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchUser(userId).then(setUser).finally(() => setLoading(false));
  }, [userId]);

  return { user, loading };
};
```

---

## 4. 状态管理方案

### 4.1 状态层级划分

| 层级 | 推荐方案 | 判断依据 |
|-|-|-|
| 组件局部状态 | useState / useReducer | 仅当前组件使用 |
| 跨组件共享状态 | Context | 父子/兄弟组件共享 |
| 全局状态 | Redux / Zustand / Jotai | 跨页面/模块共享 |
| 服务端数据 | React Query / SWR | 异步数据获取与缓存 |

### 4.2 Context 使用规范

```tsx
// 推荐：按业务域拆分
const UserContext = createContext<UserContextValue | null>(null);
const ThemeContext = createContext<ThemeContextValue | null>(null);
```

### 4.3 全局状态库选型建议

| 库 | 特点 | 适用场景 |
|-|-|-|
| Redux Toolkit | 功能完善，生态丰富 | 大型项目 |
| Zustand | 轻量，API 简洁 | 中型项目 |
| Jotai | 原子化状态 | 需要细粒度状态订阅 |

### 4.4 服务端状态管理（React Query）

```tsx
const { data, isLoading, error } = useQuery({
  queryKey: ['user', userId],
  queryFn: () => fetchUser(userId),
  staleTime: 5 * 60 * 1000,
});
```

---

## 5. 性能优化实践

> **性能优化的优先级**：先测量再优化，避免过度优化。

### 5.1 React.memo 使用场景

```tsx
const ExpensiveList = React.memo(({ items }: { items: Item[] }) => {
  return items.map(item => <ListItem key={item.id} {...item} />);
});
```

### 5.2 列表渲染优化

对于大数据量列表（大于 100 项），使用虚拟列表。

```tsx
import { FixedSizeList } from 'react-window';

const VirtualizedList = ({ items }: { items: Item[] }) => (
  <FixedSizeList height={400} itemCount={items.length} itemSize={50} width="100%">
    {({ index, style }) => <div style={style}>{items[index].name}</div>}
  </FixedSizeList>
);
```

### 5.3 懒加载（React.lazy + Suspense）

```tsx
const Dashboard = lazy(() => import('./pages/Dashboard'));

const App = () => (
  <Suspense fallback={<Spinner />}>
    <Routes>
      <Route path="/dashboard" element={<Dashboard />} />
    </Routes>
  </Suspense>
);
```

---

## 6. 测试规范

### 6.1 单元测试框架选型

| 工具 | 职责 |
|-|-|
| Jest | 测试运行器、断言库、Mock 能力 |
| React Testing Library | 基于用户行为的组件测试 |

### 6.2 测试覆盖率要求

| 类型 | 覆盖率建议 |
|-|-|
| 关键业务组件 | ≥ 80% |
| 工具函数 / 纯函数 | ≥ 90% |
| UI 展示组件 | ≥ 60% |

---

## 7. 代码分割策略

### 7.1 动态导入

```tsx
const loadUtils = async () => {
  const { formatDate } = await import('./utils/date');
  return formatDate(new Date());
};
```

### 7.2 预加载策略

```tsx
const Link = ({ to, children }) => {
  const handleMouseEnter = () => {
    import('./pages/' + to);
  };
  return <a href={to} onMouseEnter={handleMouseEnter}>{children}</a>;
};
```

---

**文档结束**

*本文档由飞书云文档导入，pangpang-doc 维护*
