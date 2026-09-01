# Java 开发规范

## 1. 文档说明

| 属性 | 说明 |
|------|------|
| 文档版本 | V1.0 |
| 来源 | 飞书云文档 |
| 生效日期 | 2026-08-29 |
| 适用范围 | 公司所有 Java 后端研发团队 |
| 作者 | ptko |

> **用途**：统一团队 Java 开发标准，提升代码质量、可读性和可维护性。适用于所有使用 Java 8+ 的项目。

---

## 2. 编码规范

### 2.1 代码格式

统一采用 **Google Java Style Guide** 作为代码格式化标准。

| 规则 | 示例 |
|-|-|
| 使用 4 个空格缩进 | ✅ 使用空格，禁止 Tab |
| 二元运算符两侧 | `a + b`（✅）而非 `a+b`（❌） |
| 逗号后面 | `method(a, b)`（✅） |
| 花括号 | 与语句开始处在同一行 |

- 单行不超过 **120 个字符**
- 超长方法调用/声明需换行时，缩进 8 个空格

### 2.2 命名规范

| 类型 | 规范 | 示例 |
|-|-|-|
| **类名** | UpperCamelCase，名词/名词短语 | `UserService`, `OrderController` |
| **方法名** | lowerCamelCase，动词/动词短语 | `getUserById`, `calculateTotal` |
| **变量名** | lowerCamelCase，见名知意 | `userName`, `orderList` |
| **常量** | UPPER_SNAKE_CASE | `MAX_RETRY_COUNT`, `DEFAULT_TIMEOUT` |
| **包名** | 全小写，com.company.module | `com.example.service` |

### 2.3 关键字与保留字

- 不使用 Java 保留字或关键字作为标识符
- 不使用 `l`（L 的小写）作为变量名，易与 `1` 混淆

---

## 3. 类与接口设计原则

### 3.1 类的设计原则

| 原则 | 说明 |
|-|-|
| 单一职责（SRP） | 一个类只负责一项职责 |
| 开闭原则（OCP） | 对扩展开放，对修改封闭 |
| 里氏替换（LSP） | 子类可以扩展父类功能，但不应改变父类行为 |

### 3.2 接口 vs 抽象类选择

| 场景 | 选择 |
|-|-|
| 多实现、需要解耦 | 接口（Interface） |
| 需要共享代码、状态 | 抽象类（Abstract Class） |
| 需要构造函数 | 抽象类 |
| Java 8+ 需要默认方法 | 两者皆可，接口更轻量 |

### 3.3 继承与组合

优先使用组合（has-a）而非继承（is-a），除非满足以下条件：
- 继承层级明确，不超过 3 层
- 子类是父类的"真子类型"
- 需要重写父类方法

---

## 4. 异常处理机制

### 4.1 异常分类

| 类型 | 特点 | 示例 |
|-|-|-|
| **Checked Exception** | 编译器强制处理 | `IOException`, `SQLException` |
| **Unchecked Exception** | 运行时异常 | `NullPointerException`, `IllegalArgumentException` |
| **Error** | JVM 错误，通常不可恢复 | `OutOfMemoryError` |

> 【推荐】统一使用 Unchecked Exception（RuntimeException 及其子类），便于调用方灵活处理。

### 4.2 异常抛出规范

- 异常应精确描述问题
- 不要抛出 `Exception`、`RuntimeException` 等宽泛类型
- 优先抛出自定义业务异常
- 异常链保留原始异常信息

---

## 5. 集合框架使用规范

### 5.1 List/Set/Map 选用原则

| 集合类型 | 适用场景 | 不适用场景 |
|-|-|-|
| **ArrayList** | 随机访问多，插入/删除少 | 频繁中间插入/删除 |
| **LinkedList** | 频繁在中间插入/删除 | 随机访问多 |
| **HashSet** | 需要去重、不关心顺序 | 需要保持插入顺序 |
| **TreeSet** | 需要自然排序或自定义排序 | 只需要去重 |
| **HashMap** | 键值对，快速查找 | 需要按键排序 |
| **TreeMap** | 需要按键排序 | 只需要快速查找 |
| **ConcurrentHashMap** | 并发环境下的键值存储 | 单线程环境 |

### 5.2 空集合返回

> 【强制】返回空集合而非 null，使用 `Collections.emptyList()` 或 `List.of()`。

---

## 6. 并发编程规范

### 6.1 基本原则

> 共享可变状态是万恶之源。尽量设计无状态的组件，或使用不可变对象。

### 6.2 synchronized 与 Lock

| 特性 | synchronized | ReentrantLock |
|-|-|-|
| 锁获取 | 隐式，自动释放 | 显式，需手动释放 |
| 超时控制 | 不支持 | 支持 `tryLock(timeout)` |
| 公平锁 | 非公平 | 可配置 |
| 多条件 | 不支持 | 支持 `Condition` |
| 性能 | JDK 6+ 已优化 | 高并发下略优 |

### 6.3 线程池参数配置建议

| 参数 | 建议 |
|-|-|
| 核心线程数 | CPU 密集型：CPU 核数 + 1；IO 密集型：CPU 核数 × 2 |
| 最大线程数 | 核心线程数 ~ 核心线程数 × 2 |
| 队列容量 | 避免无界队列，防止 OOM |
| 拒绝策略 | CallerRunsPolicy 或自定义日志记录 |

---

## 7. JVM 调优建议

### 7.1 内存区域与垃圾回收

| 区域 | 用途 | 特点 |
|-|-|-|
| **堆（Heap）** | 对象实例、数组 | GC 管理，分代收集 |
| **方法区** | 类信息、常量、静态变量 | JDK 8+ 为元空间 |
| **虚拟机栈** | 方法调用、局部变量 | 线程私有 |
| **程序计数器** | 字节码行号 | 线程私有 |

### 7.2 GC 算法选择

| 收集器 | 特点 |
|-|-|
| Serial GC | 单线程，适用于小内存 |
| Parallel GC | 多线程，追求吞吐量 |
| G1 GC | 分区式回收，适用于大内存（推荐） |
| ZGC / Shenandoah | 低延迟，TB 级内存 |

### 7.3 性能问题排查工具

| 工具 | 用途 |
|-|-|
| `jstat` | 查看 GC 统计、类加载信息 |
| `jmap` | 生成堆转储（heap dump） |
| `jstack` | 打印线程堆栈，排查死锁 |
| `jinfo` | 查看/修改 JVM 参数 |
| Arthas | 线上诊断利器（推荐） |

> 详细 JVM 调优案例见 [JVM-调优案例](../../03-技术笔记/问题排查/JVM-调优案例.md)。

### 7.4 监控指标建议

- **GC 频率**：年轻代 GC 频率、老年代 GC 频率
- **GC 耗时**：总 GC 时间占比（应 < 10%）
- **内存使用率**：堆使用率、老年代使用率
- **OOM 次数**：监控是否发生内存溢出
- **线程数**：活跃线程数、峰值线程数

---

## 8. 常用工具类与 API 规范

### 8.1 String 处理

- 使用 `String.isEmpty()` 而非 `length() == 0`
- 字符串拼接使用 `StringBuilder`（循环场景）
- 使用 `String.join()` 或 `String.format()` 做格式化

### 8.2 日期时间处理

> 【强制】统一使用 `java.time` API（Java 8+），废弃 `java.util.Date` 和 `java.util.Calendar`。

### 8.3 Stream API 常用收集器

| 场景 | 收集器 |
|-|-|
| 转 List | `Collectors.toList()` |
| 转 Set | `Collectors.toSet()` |
| 转 Map | `Collectors.toMap(k, v)` |
| 分组 | `Collectors.groupingBy()` |
| 统计 | `Collectors.summarizingInt()` |

---

## 附录：IDE 格式化配置

- **IntelliJ IDEA**：使用 Google Java Style 格式化模板
- **Eclipse**：导入团队统一的 formatter XML
- **Checkstyle**：CI 阶段自动检查代码格式

| 版本 | 日期 | 变更说明 |
|-|-|-|
| v1.0 | 2026-05-19 | 初始版本 |

---

**文档结束**

*本文档由飞书云文档导入，pangpang-doc 维护*
