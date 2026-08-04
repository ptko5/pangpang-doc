# 异常与安全规范

## 1. 文档说明

| 属性 | 说明 |
|------|------|
| 文档版本 | V1.0 |
| 生效日期 | 2026-06-30 |
| 适用范围 | 公司所有后端开发团队 |
| 作者 | 架构组 |

---

## 2. 全局异常处理规范

### 2.1 异常分类体系

#### 【强制】异常类型定义

| 异常类型 | 父类 | HTTP状态码 | 用途 |
|---------|------|-----------|------|
| **BusinessException** | RuntimeException | 400 / 404 | 业务逻辑异常 |
| **ValidationException** | RuntimeException | 400 | 参数校验异常 |
| **AuthenticationException** | RuntimeException | 401 | 认证异常 |
| **AuthorizationException** | RuntimeException | 403 | 授权异常 |
| **ResourceNotFoundException** | RuntimeException | 404 | 资源不存在 |
| **SystemException** | RuntimeException | 500 | 系统内部异常 |

#### ✅ 正确示例

```java
public class BusinessException extends RuntimeException {
    private final ErrorCode errorCode;
    
    public BusinessException(ErrorCode errorCode) {
        super(errorCode.getMessage());
        this.errorCode = errorCode;
    }
    
    public BusinessException(ErrorCode errorCode, String message) {
        super(message);
        this.errorCode = errorCode;
    }
    
    public BusinessException(ErrorCode errorCode, Throwable cause) {
        super(errorCode.getMessage(), cause);
        this.errorCode = errorCode;
    }
    
    public ErrorCode getErrorCode() {
        return errorCode;
    }
}

public class ValidationException extends RuntimeException {
    private final List<String> errors;
    
    public ValidationException(String message) {
        super(message);
        this.errors = List.of(message);
    }
    
    public ValidationException(List<String> errors) {
        super(String.join("; ", errors));
        this.errors = errors;
    }
    
    public List<String> getErrors() {
        return errors;
    }
}
```

### 2.2 错误码规范

#### 【强制】错误码定义

错误码采用 **5位数字** 编码：

| 编码范围 | 含义 | 示例 |
|---------|------|------|
| `10000-19999` | 系统通用错误 | `10000`-成功, `10001`-系统繁忙 |
| `20000-29999` | 用户模块错误 | `20001`-用户不存在 |
| `30000-39999` | 订单模块错误 | `30001`-订单不存在 |
| `40000-49999` | 支付模块错误 | `40001`-支付失败 |
| `50000-59999` | 权限模块错误 | `50001`-权限不足 |

#### ✅ 正确示例

```java
public enum ErrorCode {
    SUCCESS(10000, "success"),
    SYSTEM_ERROR(10001, "系统繁忙，请稍后重试"),
    PARAM_ERROR(10002, "参数校验失败"),
    
    USER_NOT_FOUND(20001, "用户不存在"),
    USER_EXISTS(20002, "用户已存在"),
    USER_PASSWORD_ERROR(20003, "密码错误"),
    
    ORDER_NOT_FOUND(30001, "订单不存在"),
    ORDER_CLOSED(30002, "订单已关闭"),
    
    PAYMENT_FAILED(40001, "支付失败"),
    PAYMENT_TIMEOUT(40002, "支付超时"),
    
    AUTHENTICATION_FAILED(50001, "认证失败"),
    AUTHORIZATION_FAILED(50002, "权限不足");
    
    private final int code;
    private final String message;
    
    ErrorCode(int code, String message) {
        this.code = code;
        this.message = message;
    }
    
    public int getCode() {
        return code;
    }
    
    public String getMessage() {
        return message;
    }
}
```

### 2.3 全局异常处理器

#### 【强制】全局异常处理器实现

```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    
    private static final Logger log = LoggerFactory.getLogger(GlobalExceptionHandler.class);
    
    @ExceptionHandler(BusinessException.class)
    public ApiResponse<Void> handleBusinessException(BusinessException e) {
        log.warn("业务异常: code={}, message={}", e.getErrorCode().getCode(), e.getMessage());
        return ApiResponse.fail(e.getErrorCode());
    }
    
    @ExceptionHandler(ValidationException.class)
    public ApiResponse<Void> handleValidationException(ValidationException e) {
        log.warn("参数校验异常: {}", e.getMessage());
        return ApiResponse.fail(ErrorCode.PARAM_ERROR, e.getMessage());
    }
    
    @ExceptionHandler(AuthenticationException.class)
    public ApiResponse<Void> handleAuthenticationException(AuthenticationException e) {
        log.warn("认证异常: {}", e.getMessage());
        return ApiResponse.fail(ErrorCode.AUTHENTICATION_FAILED, e.getMessage());
    }
    
    @ExceptionHandler(AuthorizationException.class)
    public ApiResponse<Void> handleAuthorizationException(AuthorizationException e) {
        log.warn("授权异常: {}", e.getMessage());
        return ApiResponse.fail(ErrorCode.AUTHORIZATION_FAILED, e.getMessage());
    }
    
    @ExceptionHandler(ResourceNotFoundException.class)
    public ApiResponse<Void> handleResourceNotFoundException(ResourceNotFoundException e) {
        log.warn("资源不存在: {}", e.getMessage());
        return ApiResponse.fail(10003, e.getMessage());
    }
    
    @ExceptionHandler(Exception.class)
    public ApiResponse<Void> handleException(Exception e) {
        log.error("系统异常", e);
        return ApiResponse.fail(ErrorCode.SYSTEM_ERROR);
    }
    
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ApiResponse<Void> handleMethodArgumentNotValid(MethodArgumentNotValidException e) {
        List<String> errors = e.getBindingResult().getFieldErrors().stream()
            .map(error -> error.getField() + ": " + error.getDefaultMessage())
            .toList();
        log.warn("参数校验失败: {}", errors);
        return ApiResponse.fail(ErrorCode.PARAM_ERROR, String.join("; ", errors));
    }
}
```

---

## 3. 安全编码规范

### 3.1 XSS防护

#### 【强制】XSS防护措施

| 措施 | 说明 | 实现方式 |
|------|------|---------|
| **输入校验** | 对用户输入进行严格校验 | 使用 `@Valid` + 正则表达式 |
| **输出转义** | 对输出内容进行HTML转义 | 使用 `HtmlUtils.htmlEscape()` |
| **内容安全策略(CSP)** | 设置HTTP响应头 | `Content-Security-Policy: default-src 'self'` |
| **富文本过滤** | 使用白名单过滤富文本 | 使用 `Jsoup.clean()` |

#### ✅ 正确示例

```java
import org.springframework.web.util.HtmlUtils;
import org.jsoup.Jsoup;
import org.jsoup.safety.Safelist;

public class XssUtils {
    
    public static String escapeHtml(String input) {
        if (input == null) {
            return null;
        }
        return HtmlUtils.htmlEscape(input);
    }
    
    public static String cleanHtml(String input) {
        if (input == null) {
            return null;
        }
        return Jsoup.clean(input, Safelist.relaxed());
    }
    
    public static String cleanText(String input) {
        if (input == null) {
            return null;
        }
        return Jsoup.clean(input, Safelist.none());
    }
}
```

### 3.2 CSRF防护

#### 【强制】CSRF防护措施

| 措施 | 说明 | 适用场景 |
|------|------|---------|
| **SameSite Cookie** | 设置Cookie的SameSite属性 | 所有Cookie |
| **Token验证** | 请求中携带CSRF Token | 表单提交 |
| **Referer验证** | 验证请求来源 | API接口 |
| **双重提交Cookie** | 比较Cookie和请求参数中的Token | 表单提交 |

#### ✅ 正确示例

```java
@Configuration
public class SecurityConfig {
    
    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf
                .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
                .csrfTokenRequestHandler(new CsrfTokenRequestAttributeHandler())
            )
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            );
        return http.build();
    }
}
```

### 3.3 密码安全

#### 【强制】密码加密规范

| 算法 | 说明 | 推荐 |
|------|------|------|
| **BCrypt** | 自适应哈希算法，自带盐值 | ✅ 推荐 |
| **Argon2** | 内存密集型哈希算法 | ✅ 推荐 |
| **MD5** | 单向哈希，已不安全 | ❌ 禁止 |
| **SHA-1** | 单向哈希，已不安全 | ❌ 禁止 |

#### ✅ 正确示例

```java
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;

@Configuration
public class SecurityConfig {
    
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}

@Service
public class UserServiceImpl implements UserService {
    
    @Autowired
    private PasswordEncoder passwordEncoder;
    
    @Override
    public Long createUser(CreateUserRequest request) {
        UserEntity user = new UserEntity();
        user.setUsername(request.username());
        user.setEmail(request.email());
        user.setPassword(passwordEncoder.encode(request.password()));
        user.setStatus(1);
        userMapper.insert(user);
        return user.getId();
    }
    
    @Override
    public boolean verifyPassword(String rawPassword, String encodedPassword) {
        return passwordEncoder.matches(rawPassword, encodedPassword);
    }
}
```

### 3.4 敏感信息保护

#### 【强制】敏感信息处理

| 类型 | 处理方式 | 说明 |
|------|---------|------|
| **密码** | 加密存储 | 使用BCrypt/Argon2 |
| **Token** | 安全传输 | 使用HTTPS |
| **日志** | 脱敏处理 | 禁止打印密码、Token |
| **配置** | 加密存储 | 使用Jasypt加密配置 |
| **传输** | 加密传输 | 使用HTTPS |

#### ✅ 正确示例

```java
// 日志脱敏 - 使用Logback Pattern
// logback-spring.xml
<pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>

// 自定义日志脱敏转换器
public class MaskConverter extends ClassicConverter {
    @Override
    public String convert(ILoggingEvent event) {
        String message = event.getMessage();
        message = message.replaceAll("password=[^&]*", "password=***");
        message = message.replaceAll("token=[^&]*", "token=***");
        return message;
    }
}

// 配置文件加密 - 使用Jasypt
@Configuration
public class JasyptConfig {
    
    @Bean("jasyptStringEncryptor")
    public StringEncryptor stringEncryptor() {
        PooledPBEStringEncryptor encryptor = new PooledPBEStringEncryptor();
        SimpleStringPBEConfig config = new SimpleStringPBEConfig();
        config.setPassword(System.getenv("JASYPT_ENCRYPTOR_PASSWORD"));
        config.setAlgorithm("PBEWITHHMACSHA512ANDAES_256");
        encryptor.setConfig(config);
        return encryptor;
    }
}
```

---

## 4. 限流熔断规范

### 4.1 Sentinel限流配置

#### 【强制】限流规则

| 规则类型 | 说明 | 使用场景 |
|---------|------|---------|
| **限流规则** | 限制QPS | 接口流量控制 |
| **熔断规则** | 降级处理 | 服务不可用时 |
| **热点规则** | 热点参数限流 | 特定参数限流 |
| **系统规则** | 系统保护 | 全局流量控制 |

#### ✅ 正确示例

```java
@Configuration
public class SentinelConfig {
    
    @PostConstruct
    public void initRules() {
        initFlowRules();
        initDegradeRules();
    }
    
    private void initFlowRules() {
        List<FlowRule> rules = new ArrayList<>();
        
        FlowRule userApiRule = new FlowRule();
        userApiRule.setResource("user-api");
        userApiRule.setGrade(RuleConstant.FLOW_GRADE_QPS);
        userApiRule.setCount(1000);
        userApiRule.setControlBehavior(RuleConstant.CONTROL_BEHAVIOR_WARM_UP);
        rules.add(userApiRule);
        
        FlowRule orderApiRule = new FlowRule();
        orderApiRule.setResource("order-api");
        orderApiRule.setGrade(RuleConstant.FLOW_GRADE_QPS);
        orderApiRule.setCount(500);
        rules.add(orderApiRule);
        
        FlowRuleManager.loadRules(rules);
    }
    
    private void initDegradeRules() {
        List<DegradeRule> rules = new ArrayList<>();
        
        DegradeRule remoteServiceRule = new DegradeRule();
        remoteServiceRule.setResource("remote-service");
        remoteServiceRule.setGrade(RuleConstant.DEGRADE_GRADE_RT);
        remoteServiceRule.setCount(500);
        remoteServiceRule.setTimeWindow(10);
        rules.add(remoteServiceRule);
        
        DegradeRuleManager.loadRules(rules);
    }
}
```

### 4.2 限流注解使用

#### 【推荐】限流注解

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface RateLimit {
    
    String resource() default "";
    
    int qps() default 100;
    
    String fallback() default "";
}

@Aspect
@Component
public class RateLimitAspect {
    
    @Around("@annotation(rateLimit)")
    public Object around(ProceedingJoinPoint joinPoint, RateLimit rateLimit) throws Throwable {
        String resource = rateLimit.resource();
        if (resource.isEmpty()) {
            resource = joinPoint.getSignature().getName();
        }
        
        Entry entry = null;
        try {
            entry = SphU.entry(resource, EntryType.IN);
            return joinPoint.proceed();
        } catch (BlockException e) {
            log.warn("接口限流: {}", resource);
            return ApiResponse.fail(10004, "请求过于频繁，请稍后重试");
        } finally {
            if (entry != null) {
                entry.exit();
            }
        }
    }
}

@RestController
@RequestMapping("/api/users")
public class UserController {
    
    @Autowired
    private UserService userService;
    
    @GetMapping("/{id}")
    @RateLimit(resource = "user-detail", qps = 500)
    public ApiResponse<UserResponse> getUserById(@PathVariable Long id) {
        UserResponse user = userService.getUserById(id);
        return ApiResponse.success(user);
    }
}
```

---

## 5. API安全规范

### 5.1 接口鉴权

#### 【强制】JWT鉴权

```java
@Component
public class JwtTokenFilter extends OncePerRequestFilter {
    
    private static final Logger log = LoggerFactory.getLogger(JwtTokenFilter.class);
    private static final String AUTHORIZATION_HEADER = "Authorization";
    private static final String BEARER_PREFIX = "Bearer ";
    
    @Autowired
    private JwtTokenProvider jwtTokenProvider;
    
    @Override
    protected void doFilterInternal(HttpServletRequest request, 
                                    HttpServletResponse response, 
                                    FilterChain filterChain) throws ServletException, IOException {
        String token = extractToken(request);
        
        if (token != null && jwtTokenProvider.validateToken(token)) {
            String username = jwtTokenProvider.getUsername(token);
            UserDetails userDetails = userDetailsService.loadUserByUsername(username);
            Authentication authentication = new UsernamePasswordAuthenticationToken(
                userDetails, null, userDetails.getAuthorities());
            SecurityContextHolder.getContext().setAuthentication(authentication);
        }
        
        filterChain.doFilter(request, response);
    }
    
    private String extractToken(HttpServletRequest request) {
        String bearerToken = request.getHeader(AUTHORIZATION_HEADER);
        if (bearerToken != null && bearerToken.startsWith(BEARER_PREFIX)) {
            return bearerToken.substring(BEARER_PREFIX.length());
        }
        return null;
    }
}
```

### 5.2 接口签名验证

#### 【推荐】API签名

```java
public class ApiSignUtils {
    
    private static final String SIGN_KEY = "your-secret-key";
    private static final String TIMESTAMP_KEY = "timestamp";
    private static final String SIGN_KEY_KEY = "sign";
    private static final long EXPIRE_TIME = 5 * 60 * 1000;
    
    public static String generateSign(Map<String, String> params) {
        params.put(TIMESTAMP_KEY, String.valueOf(System.currentTimeMillis()));
        List<String> keys = new ArrayList<>(params.keySet());
        keys.sort(String::compareTo);
        
        StringBuilder sb = new StringBuilder();
        for (String key : keys) {
            sb.append(key).append("=").append(params.get(key)).append("&");
        }
        sb.append("key=").append(SIGN_KEY);
        
        return DigestUtils.md5DigestAsHex(sb.toString().getBytes()).toUpperCase();
    }
    
    public static boolean verifySign(Map<String, String> params, String sign) {
        String timestamp = params.get(TIMESTAMP_KEY);
        if (timestamp == null) {
            return false;
        }
        
        long time = Long.parseLong(timestamp);
        if (System.currentTimeMillis() - time > EXPIRE_TIME) {
            return false;
        }
        
        String expectedSign = generateSign(params);
        return expectedSign.equals(sign);
    }
}
```

---

## 6. 落地检查清单

| 序号 | 检查项 | 检查方式 | 责任人 |
|------|--------|---------|--------|
| 1 | 是否定义统一异常体系（BusinessException等） | 检查代码 | 架构师 |
| 2 | 是否使用全局异常处理器 | 检查代码 | 架构师 |
| 3 | 错误码是否符合5位数字规范 | 检查代码 | 开发人员 |
| 4 | 是否使用BCrypt/Argon2加密密码 | 检查代码 | 安全工程师 |
| 5 | 日志中是否打印敏感信息（密码、Token） | SonarQube扫描 | 代码评审人 |
| 6 | 是否启用XSS防护 | 检查代码 | 安全工程师 |
| 7 | 是否启用CSRF防护 | 检查代码 | 安全工程师 |
| 8 | 是否配置Sentinel限流熔断 | 检查代码 | 架构师 |
| 9 | 接口是否启用JWT鉴权 | 检查代码 | 安全工程师 |
| 10 | 是否使用HTTPS传输 | 检查配置 | 运维工程师 |

---

**文档结束**

*本规范由架构组制定，解释权归架构组所有。*