# Java 日志规范

## 1. 文档说明

| 属性 | 说明 |
|------|------|
| 文档版本 | V1.0 |
| 生效日期 | 2026-08-15 |
| 适用范围 | 公司所有 Java 后端开发团队 |
| 作者 | 架构组 |
| 技术栈 | JDK 25、Spring Boot 4.x、SLF4J + Logback |

---

## 2. 日志框架选型

### 2.1 【强制】统一使用 SLF4J + Logback

| 组件 | 说明 |
|------|------|
| **SLF4J** | 日志门面，代码中只依赖 SLF4J API |
| **Logback** | 底层实现（Spring Boot 4.x 默认集成） |

#### ✅ 正确：通过 SLF4J 获取 Logger

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

@Slf4j  // 使用 Lombok 注解（项目默认）
public class OrderService {
    // 或手动声明：
    // private static final Logger log = LoggerFactory.getLogger(OrderService.class);
}
```

#### ❌ 禁止

```java
// 禁止直接使用 Log4j / JUL / System.out
System.out.println("order created: " + orderId);
```

> 【强制】禁止 `System.out.println` / `printStackTrace()` 打印日志，统一走 SLF4J。

---

## 3. 日志级别规则

### 3.1 【强制】级别使用规范

| 级别 | 使用场景 | 示例 |
|------|---------|------|
| **TRACE** | 细粒度调试（仅本地开发开启） | 方法入参出参、循环内追踪 |
| **DEBUG** | 开发/排查时使用的调试信息 | 请求参数、中间计算值 |
| **INFO** | 关键业务节点、状态变更 | 订单创建成功、用户注册 |
| **WARN** | 潜在风险但不影响当前流程 | 重试、缓存降级、参数不规范 |
| **ERROR** | 异常、失败、影响功能 | 业务异常、系统异常、调用失败 |

### 3.2 【强制】生产环境日志级别

| 环境 | 推荐级别 |
|------|---------|
| 开发环境 | DEBUG |
| 测试环境 | INFO（可临时 DEBUG） |
| 预发/生产环境 | INFO（业务包可 WARN） |

> 生产环境禁止长期输出 DEBUG 级别，避免海量日志拖垮磁盘与采集链路。

### 3.3 【强制】级别使用铁律

- **禁止** 用 ERROR 打印「正常业务分支」（如用户取消订单用 INFO）
- **禁止** 用 `log.info(e)` 打印异常却不带堆栈
- **必须** 在 catch 块使用 ERROR 记录完整异常堆栈

```java
// ✅ 正确：记录异常时传最后一个参数为异常对象
try {
    orderService.createOrder(request);
} catch (BizException e) {
    log.error("创建订单失败, orderNo={}", request.orderNo(), e);
}

// ❌ 错误：异常对象被拼接成字符串，丢失堆栈
catch (Exception e) {
    log.error("创建订单失败: " + e.getMessage());
}
```

---

## 4. 日志格式规范

### 4.1 【强制】统一日志格式

```text
时间 | 级别 | TraceId | 线程 | Logger | 消息
```

日志配置文件（`logback-spring.xml`）标准片段：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <property name="LOG_PATTERN"
              value="%d{yyyy-MM-dd HH:mm:ss.SSS} | %-5level | %X{traceId:-} | %thread | %logger{40} | %msg%n"/>

    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>${LOG_PATTERN}</pattern>
            <charset>UTF-8</charset>
        </encoder>
    </appender>

    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${LOG_PATH:-/var/log/app}/app.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <fileNamePattern>${LOG_PATH:-/var/log/app}/app.%d{yyyy-MM-dd}.%i.log</fileNamePattern>
            <maxFileSize>100MB</maxFileSize>
            <maxHistory>30</maxHistory>
            <totalSizeCap>5GB</totalSizeCap>
        </rollingPolicy>
        <encoder>
            <pattern>${LOG_PATTERN}</pattern>
            <charset>UTF-8</charset>
        </encoder>
    </appender>

    <root level="INFO">
        <appender-ref ref="CONSOLE"/>
        <appender-ref ref="FILE"/>
    </root>
</configuration>
```

### 4.2 【强制】占位符打印，禁止字符串拼接

```java
// ✅ 正确：占位符 + 惰性求值（级别不满足时不执行 toString）
log.info("用户登录成功, userId={}, ip={}", userId, clientIp);

// ❌ 错误：字符串拼接，即使级别是 DEBUG 也先执行拼接
log.info("用户登录成功, userId=" + userId + ", ip=" + clientIp);
```

### 4.3 【推荐】日志内容要素

- 关键日志必须携带 **业务主键**（orderId、userId、requestId）便于检索
- 中文消息 + 英文参数名混排时注意中英文空格（见 `../00-开发规范/文档书写规范.md`）

---

## 5. TraceId 链路追踪

### 5.1 【强制】全链路 TraceId 透传

微服务场景下，同一个请求跨服务必须有相同的 TraceId 才能串联日志。项目已配置 Spring Cloud Sleuth / Micrometer Tracing，接入方式：

```yaml
# application.yml
management:
  tracing:
    sampling:
      probability: 1.0
```

> 生产环境抽样率建议 1.0（100%），TraceId 占日志空间很小，保证可排查性。

### 5.2 MDC 手动注入

在过滤器/切面中把 TraceId 放入 MDC，日志 pattern 用 `%X{traceId}` 输出：

```java
import org.slf4j.MDC;

public class TraceIdFilter implements Filter {
    @Override
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) throws IOException, ServletException {
        String traceId = Optional.ofNullable(((HttpServletRequest) req).getHeader("X-Trace-Id"))
                .orElseGet(() -> UUID.randomUUID().toString().replace("-", ""));
        MDC.put("traceId", traceId);
        try {
            chain.doFilter(req, res);
        } finally {
            MDC.remove("traceId");  // 必须清理，避免线程池复用导致串号
        }
    }
}
```

> 【强制】使用 MDC 后必须在 finally 中 `MDC.remove`，否则线程池复用时 TraceId 会串到下一个请求。

---

## 6. 敏感信息脱敏

### 6.1 【强制】禁止打印的敏感字段

| 字段 | 说明 |
|------|------|
| 密码 / 密码哈希 | 一律不打印 |
| Token / Session / 密钥 | 一律不打印 |
| 身份证号、银行卡号 | 打印前脱敏 |
| 手机号、邮箱 | 打印前脱敏（可部分遮蔽） |

### 6.2 脱敏示例

```java
// 手机号 138****1234
log.info("用户手机号: {}", maskPhone(phone));

// 使用工具类统一脱敏（参考 SQL与数据访问规范 §6 数据脱敏规范）
```

> 可选用 Logback 的 `Converter` 或日志框架的敏感词过滤插件，统一在输出层脱敏，避免到处手写。

---

## 7. 日志落地检查清单

| 序号 | 检查项 | 级别 | 检查方式 |
|------|--------|------|---------|
| 1 | 统一使用 SLF4J，无 `System.out` / `printStackTrace` | 【强制】 | 代码扫描 |
| 2 | 生产环境无 DEBUG 长期输出 | 【强制】 | 配置文件检查 |
| 3 | 异常日志传异常对象（保留堆栈） | 【强制】 | 代码审查 |
| 4 | 日志使用占位符，无字符串拼接 | 【强制】 | 代码扫描 |
| 5 | 关键日志携带业务主键 | 【推荐】 | 抽查 |
| 6 | TraceId 全链路透传，MDC 正确清理 | 【强制】 | 跨服务联调验证 |
| 7 | 敏感字段已脱敏 / 不打印 | 【强制】 | 日志抽检 |
| 8 | 日志文件滚动策略（按天+大小）与留存期已配置 | 【推荐】 | 配置文件检查 |
| 9 | 日志采集链路已打通（配合 `../02-部署运维/服务器运维/日志管理.md`） | 【推荐】 | ELK 检索验证 |

---

**文档结束**

*本文档由 pangpang-doc 维护*
