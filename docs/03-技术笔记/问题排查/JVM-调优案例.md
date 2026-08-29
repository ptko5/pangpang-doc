# JVM 调优案例

## 1. 文档说明

| 属性 | 说明 |
|------|------|
| 文档版本 | V1.0 |
| 生效日期 | 2026-08-15 |
| 适用范围 | 公司所有后端研发团队与运维（JDK 25） |
| 作者 | 架构组 |

> **用途**：JVM 调优的方法论、工具链与案例模板，通过完整案例（现象→排查→根因→方案→预防）指导线上 JVM 问题的处理。

---

## 2. 案例模板（统一结构）

| 章节 | 要求 |
|------|------|
| 现象描述 | 异常表现、影响范围、时间线 |
| 排查过程 | 使用的工具与数据 |
| 根因分析 | 定位到具体根因 |
| 解决方案 | 具体修复与参数 |
| 预防措施 | 避免复发的手段 |

---

## 3. JVM 工具链

| 工具 | 用途 |
|------|------|
| `jps` | 查看 Java 进程 |
| `jstat` | 查看 GC 与类加载 |
| `jmap` | 堆转储（heap dump） |
| `jstack` | 线程转储（thread dump） |
| `jcmd` | 综合诊断命令 |
| Arthas | 在线诊断（dashboard/trace/thread） |

```bash
# 常用组合
jps -l
jstat -gcutil <pid> 1000      # GC 每 1s 采样
jstack <pid> > thread.dump    # 线程快照
jmap -dump:format=b,file=heap.hprof <pid>   # 堆快照
```

> 【强制】生产抓取 dump 前确认影响（堆 dump 可能暂停应用），优先使用 Arthas 在线诊断。

---

## 4. 案例一：内存溢出（OOM）

### 4.1 现象描述

- 服务运行数日后出现 `java.lang.OutOfMemoryError: Java heap space`
- 接口响应变慢，最终实例被 OOMKill 重启

### 4.2 排查过程

```bash
# 1. 启动参数开启堆转储
java -Xmx4g -XX:+HeapDumpOnOutOfMemoryError \
     -XX:HeapDumpPath=/opt/dumps -jar app.jar

# 2. 复现后分析 dump（使用 MAT/Eclipse MAT）
jmap -dump:format=b,file=heap.hprof <pid>
```

### 4.3 根因分析

- 通过 MAT 分析：`order_list` 缓存集合无限增长（`ConcurrentHashMap` 未设置容量上限），导致堆被业务数据占满

### 4.4 解决方案

```java
// 限制缓存容量，使用带淘汰的缓存
private final Cache<String, Order> cache =
    Caffeine.newBuilder().maximumSize(10_000)
            .expireAfterWrite(Duration.ofMinutes(10))
            .build();
```

### 4.5 预防措施

- 所有缓存必须设容量上限与过期策略
- 监控堆使用率，设置 GC 与 OOM 告警
- 压测覆盖长稳运行场景

---

## 5. 案例二：CPU 飙升

### 5.1 现象描述

- 监控显示某实例 CPU 持续 100%，接口大量超时

### 5.2 排查过程

```bash
# 1. 定位 CPU 最高的进程
top

# 2. 查看进程内 CPU 最高线程
top -H -p <pid>

# 3. 线程 id 转十六进制
printf "%x\n" <threadId>

# 4. 抓线程栈定位
jstack <pid> | grep -A 30 "0x<hex>"
```

### 5.3 根因分析

- 线程栈显示某方法存在 `while(true)` 自旋，因缓存未命中反复查库无退出条件

### 5.4 解决方案

- 修复自旋逻辑，增加重试上限与超时；为热点查询增加缓存

### 5.5 预防措施

- 代码审查关注自旋与死循环
- 配置 CPU 使用率告警与自动 dump 脚本

---

## 6. 案例三：频繁 Full GC

### 6.1 现象描述

- 接口间歇性卡顿，`jstat` 显示 Full GC 频繁（每分钟多次）

### 6.2 排查过程

```bash
jstat -gcutil <pid> 1000
# FGC 列快速增长，FCT 高
```

### 6.3 根因分析

- 堆中大量短命大对象（批量查询加载全表），导致老年代快速膨胀
- 或 `Xmx` 过小，内存分配压力大

### 6.4 解决方案

```bash
# 调整堆大小与 GC 参数
java -Xms4g -Xmx4g -XX:+UseG1GC \
     -XX:MaxGCPauseMillis=200 -jar app.jar
```

- 优化 SQL，避免一次性加载全表（分页/投影）
- 排查大对象与未关闭的流

### 6.5 预防措施

- 建立 GC 指标大盘（Full GC 次数/耗时）
- 大查询限制返回条数，分批处理

---

## 7. 落地检查清单

| 序号 | 检查项 | 检查方式 | 责任人 |
|------|--------|---------|--------|
| 1 | 工具链熟练使用 | 模拟演练 | 后端开发 |
| 2 | 生产已开启 OOM 堆转储 | 启动参数核对 | 运维 |
| 3 | 堆/GC 监控接入 | 大盘核对 | 运维 |
| 4 | 案例复盘沉淀 | 笔记核对 | 后端开发 |
| 5 | 告警与自动 dump 就绪 | 脚本验证 | 运维 |

---

**文档结束**

*本文档由 pangpang-doc 维护*
