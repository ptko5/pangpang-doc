# AI 后端微服务项目搭建指南

## 1. 文档说明

| 属性 | 说明 |
|------|------|
| 文档版本 | V1.0 |
| 生效日期 | 2026-09-02 |
| 适用范围 | AI 编程助手（Trae 等）用于独立搭建后端微服务项目 |
| 作者 | 架构组 |

> **用途**：本文档是 AI 搭建后端微服务项目的**唯一输入依据**。AI 仅凭本文档即可独立完成一个可运行、可验收的微服务项目骨架。文档中的【强制】项必须逐条满足，【推荐】项在条件允许时优先满足。

---

## 2. 项目背景说明

公司要建设一套电商类后端服务，支持商品、订单、用户、支付等核心业务，需要拆分为多个微服务独立开发、独立部署、独立扩容。系统需支撑如下业务特征：

- **高并发**：核心接口（下单、扣库存）峰值 QPS ≥ 2000，读多写少。
- **高可用**：核心服务可用性 ≥ 99.95%，不允许单点故障，支持水平扩容。
- **数据一致性**：订单与库存、支付与订单之间存在跨服务事务需求，需采用最终一致性方案。
- **快速迭代**：各服务由不同小组并行开发，接口契约先行，前后端并行。
- **多云/私有化部署**：需支持 Docker 容器化 + 可选的 Kubernetes 编排。

> **交付形态**：本文档要求产出一个**可直接编译运行**的微服务项目骨架，包含网关、注册中心、配置中心接入、业务示例服务（订单 + 库存 + 用户）、通用中间件封装（Redis/MySQL/MQ）、CI 流水线与 Docker 部署文件，以及一份 README 说明如何启动。

---

## 3. 开发目标

### 3.1 总目标

AI 依据本文档产出一个**后端微服务项目**，满足：

1. 多服务可独立启动、独立打包、独立部署（docker 镜像）。
2. 服务间通过注册中心（Nacos）发现与调用，支持负载均衡。
3. 具备统一的 API 规范、统一异常处理、统一返回结构。
4. 具备配置中心（Nacos Config）、分布式日志、基础监控埋点。
5. 核心业务（订单/库存/用户）给出可运行的示例实现，含数据库脚本。
6. 提供一键本地启动（docker-compose）与生产部署（K8s manifest 或 Helm）两种方式。

### 3.2 验收性目标（可直接核验）

| 目标 | 验收方式 |
|------|---------|
| 项目可 `gradle build` 成功 | 执行构建命令通过 |
| 全部服务可本地一键启动 | 运行 docker-compose 后各服务健康检查通过 |
| 网关路由到业务服务成功 | 通过网关调用订单示例接口返回 200 |
| 服务间调用成功 | 订单服务调用库存服务返回正确结果 |
| 配置文件外部化 | 修改 Nacos 配置后服务热刷新生效 |

---

## 4. 技术选型建议

### 4.1 技术栈（【强制】对齐公司标准）

| 组件 | 版本 | 用途 |
|------|------|------|
| JDK | 25 | 核心语言环境（使用虚拟线程/记录类等新特性） |
| Spring Boot | 4.x | 应用框架 |
| Spring Cloud | 2024.x | 微服务框架（服务发现/负载均衡/网关） |
| Spring Data JPA | 3.x | ORM（配合 Hibernate 6.x） |
| Gradle | 8.x | 构建工具 |
| MySQL | 8.x | 主数据库 |
| Redis | 7.x | 缓存/分布式锁 |
| Nacos | 2.x | 注册中心 + 配置中心 |
| Sentinel | 2.x | 熔断/限流/降级 |
| RabbitMQ | 3.x | 异步消息（订单事件/库存扣减） |
| OpenFeign | 4.x | 声明式服务间调用 |
| Resilience4j 或 Sentinel | 2.x | 容错 |

> 【推荐】脚手架优先基于 Spring Initializr 生成，保持版本对齐；禁止引入未经公司选型的重框架（如 Dubbo、自研 RPC）。

### 4.2 服务划分建议

```text
gateway            # 网关：统一入口、鉴权、路由、限流
user-service       # 用户服务：注册/登录/用户信息
product-service    # 商品服务：商品/库存查询与扣减
order-service      # 订单服务：下单/订单查询
payment-service    # 支付服务：支付/回调（示例可简化）
common/            # 公共模块：统一返回、异常、工具、基础实体
```

---

## 5. 服务拆分原则

### 5.1 拆分依据（【强制】）

1. **按业务能力拆**：一个服务只负责一个高内聚业务域（用户/商品/订单/支付），禁止按"控制器/服务/DAO 层"拆服务。
2. **按数据边界拆**：每个服务独占自己的数据库（或 schema），禁止跨服务直连对方数据库表。
3. **按团队/变更频率拆**：变更频率高、独立迭代的模块拆成独立服务。
4. **服务粒度评估**：当一个服务承担超过 3 个不相关业务能力、或团队无法独立上线时，应继续拆分；当拆分后服务间调用链 > 3 跳时，评估是否合并。

### 5.2 拆分禁忌（【禁止】）

- 【禁止】为"分层"而拆（controller/dao 各一个服务）。
- 【禁止】为"一张表"拆一个服务。
- 【禁止】循环依赖：A 调 B、B 又调 A（必须消除或引入消息/事件）。
- 【禁止】共享数据库表导致隐性耦合。

### 5.3 拆分清单（交付物）

产出 `docs/split.md`，包含：服务清单、每个服务的职责边界、对外提供的接口清单、拥有的数据表清单、服务间依赖图（Mermaid）。

---

## 6. API 设计规范

### 6.1 统一返回结构（【强制】）

所有 HTTP 接口统一返回：

```json
{
  "code": 0,
  "message": "success",
  "data": {},
  "traceId": "xxxx"
}
```

- `code == 0` 表示成功；非 0 为业务/系统错误码。
- 错误码规划：`1xxxx` 通用系统错误；`2xxxx` 用户域；`3xxxx` 商品域；`4xxxx` 订单域；`5xxxx` 支付域。每域从 00 开始递增。

### 6.2 RESTful 约定（【强制】）

- 使用名词复数作为资源路径：`GET /api/v1/orders`、`POST /api/v1/orders`、`PUT /api/v1/orders/{id}`、`DELETE /api/v1/orders/{id}`。
- 版本号放路径：`/api/v1/`，不向后兼容时升大版本。
- 分页统一参数：`page`（从 0 起）、`size`、`sort`，返回 `{ list, total, page, size }`。
- 查询用 GET 参数；写操作用 JSON body（`Content-Type: application/json`）。
- 禁止在 URL 中暴露敏感信息（用户密码、Token、身份证号）。

### 6.3 统一异常处理（【强制】）

- 全局 `@RestControllerAdvice` 捕获异常，映射为统一返回结构。
- 自定义业务异常 `BizException(code, message)`；参数校验异常、兜底异常分别映射错误码。
- 接口文档：使用 springdoc-openapi 生成 Swagger UI，并随包输出 `docs/api-*.md` 接口清单。

### 6.4 幂等与重试（【强制】）

- 写接口（下单、支付、扣库存）必须支持幂等：客户端传 `Idempotency-Key` 头或请求幂等号，服务端去重并返回原结果。
- 支付回调等外部回调接口必须幂等，且校验签名。

---

## 7. 数据库设计

### 7.1 通用规范（【强制】）

- 每服务独立数据库（或 schema），命名 `{service}_db`。
- 表名小写下划线复数（`order_item`），字段小写下划线（`order_status`）。
- 主键统一 `bigint` 自增或雪花 ID；统一包含 `created_at`、`updated_at`、`deleted`（逻辑删除 0/1）。
- 所有表使用 `utf8mb4`，InnoDB 引擎。
- 关键查询字段必须建索引，禁止全表扫描；联合索引遵循最左前缀。

### 7.2 示例：订单表 DDL（交付物）

```sql
CREATE TABLE `t_order` (
  `id`          BIGINT       NOT NULL AUTO_INCREMENT COMMENT '主键',
  `order_no`    VARCHAR(64)  NOT NULL COMMENT '订单号',
  `user_id`     BIGINT       NOT NULL COMMENT '用户ID',
  `amount`      DECIMAL(10,2) NOT NULL COMMENT '订单金额',
  `status`      TINYINT      NOT NULL DEFAULT 0 COMMENT '0待支付 1已支付 2已取消',
  `created_at`  DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
  `updated_at`  DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  `deleted`     TINYINT      NOT NULL DEFAULT 0,
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_order_no` (`order_no`),
  KEY `idx_user_id` (`user_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='订单表';
```

### 7.3 交付物

- 每个服务提供 `db/schema.sql`（建表）与 `db/data.sql`（种子数据）。
- 跨服务数据一致性：禁止跨服务 join；需要聚合时通过接口调用或事件同步。
- 金额一律使用 `DECIMAL`，禁止 `float/double`。

---

## 8. 服务间通信方式

### 8.1 同步调用（【强制】）

- 服务发现与调用：注册到 Nacos，使用 **OpenFeign** + 负载均衡调用。
- 统一在 `common` 模块定义 Feign 接口与 DTO，契约先行。
- 同步调用必须设置超时（连接/读超时 3s/5s 默认），必须配置熔断降级（Sentinel 或 Resilience4j），降级返回兜底或友好提示。

### 8.2 异步事件（【推荐】核心链路强制）

- 订单状态流转、库存扣减等**强一致不需要、最终一致即可**的场景，使用 **RabbitMQ** 发布事件（`order.created`、`stock.deducted` 等）。
- 事件消息必须包含 `eventId`（幂等去重键）、`traceId`、`timestamp`、`sourceService`、`payload`。
- 消费者必须幂等消费，失败重试 + 死信队列。
- 【禁止】跨服务同步调用链超过 3 跳。

### 8.3 分布式事务

- 优先通过**事件驱动 + 本地消息表**实现最终一致性，不引入强一致分布式事务框架。
- 若确需强一致（如支付扣款），采用 TCC 或 Seata AT 模式，但必须评估成本；示例项目默认采用事件 + 对账补偿方案。

---

## 9. 配置管理

### 9.1 原则（【强制】）

- **配置外部化**：所有环境相关配置（数据库地址、Redis、密钥、开关）放 Nacos 配置中心，禁止硬编码进代码。
- 区分 `application.yml`（本地默认）与 Nacos 中的 `{service}-{env}.yml`（dev/prod）。
- 敏感配置（密码、Token）必须加密存储或引用环境变量，禁止明文提交到 git。

### 9.2 结构（交付物）

```text
{service}/src/main/resources/
  application.yml            # 本地默认，含 Nacos 地址
  bootstrap.yml              # 引入 Nacos 配置中心
```

- 配置分组：`{namespace}/{group: DEFAULT_GROUP}/{dataId: {service}.yml}`。
- 支持配置热刷新：使用 `@RefreshScope` 或 Spring Cloud 原生刷新机制。
- 本地开发不依赖 Nacos 也能启动（提供 `application-local.yml` 缺省值）。

---

## 10. 日志与监控

### 10.1 日志规范（【强制】）

- 使用 **SLF4J + Logback**，统一日志格式：
  `时间 | 级别 | traceId | 服务名 | 类名:行号 | 消息`。
- 日志必须携带 `traceId`，跨服务通过 MQ/HTTP Header 透传，便于全链路追踪。
- 禁止在生产打印敏感信息（密码、Token、完整身份证号）；异常必须打印完整堆栈。
- 分级：ERROR 记录异常、WARN 记录降级/重试、INFO 记录关键业务事件（下单成功等）。

### 10.2 监控（【强制】）

- 引入 **Micrometer + Actuator**，暴露 `/actuator/prometheus` 指标。
- 关键指标：QPS、RT（P99）、错误率、线程池活跃度、GC、连接池使用率。
- 可选接入 **Prometheus + Grafana** 采集展示（提供 `docker-compose` 中的监控编排）。
- 提供 `docs/alert-rules.md`：核心接口错误率 > 1%、P99 > 500ms 触发告警。

---

## 11. 部署方案

### 11.1 本地一键启动（【强制】交付）

提供 `docker-compose.yml`，一键拉起：

```yaml
services:
  mysql:      # 8.x，初始化脚本挂载 db/
  redis:      # 7.x
  nacos:      # 2.x 单机模式
  rabbitmq:   # 3.x，含 management
  gateway:    # 构建产物镜像
  user-service:
  product-service:
  order-service:
  payment-service:
```

### 11.2 生产部署（【推荐】交付）

- 每个服务提供 `Dockerfile`（多阶段构建：Gradle 编译 → JRE 运行，非 root 用户运行）。
- 提供 `deploy/k8s/*.yaml` 或 Helm Chart：Deployment（含存活/就绪探针、资源 limits）、Service、ConfigMap/Secret。
- 网关暴露为唯一外部入口，业务服务仅集群内访问。

### 11.3 CI/CD（【推荐】）

- 提供 GitHub Actions：代码推送 → 单元测试 → 构建镜像 → 推送仓库。
- 环境变量与密钥使用仓库 Secrets，禁止明文。

---

## 12. 安全策略

### 12.1 认证与鉴权（【强制】）

- 网关统一鉴权：登录发放 JWT（或 Token），网关校验签名后向下游透传 `X-User-Id`。
- 内部服务间调用校验来源（内部 Header/Token），禁止直接暴露公网。
- 敏感接口（支付、改密）二次校验。

### 12.2 数据安全（【强制】）

- 密码存储：`BCrypt` 加盐哈希，禁止明文/可逆加密。
- 传输：生产环境全站 HTTPS；接口参数校验防止 SQL 注入（JPA 参数化）与 XSS。
- 日志与响应中脱敏手机号、身份证号、卡号。

### 12.3 其他（【强制】）

- 依赖漏洞：构建时跑 `gradle dependencyCheck` 或 CI 中集成安全扫描。
- 限流防刷：网关对接口做 Sentinel 限流（默认 QPS 阈值可配置）。
- 【禁止】在代码、配置、README 中出现真实密钥/密码（使用占位符 `${DB_PASSWORD}`）。

---

## 13. 项目结构（交付物骨架）

```text
backend-microservices/
├── settings.gradle
├── build.gradle                 # 统一依赖版本管理（BOM）
├── docker-compose.yml
├── README.md                    # 启动说明
├── common/                      # 公共模块（统一返回/异常/Feign DTO/工具）
├── gateway/                     # 网关服务
├── user-service/
├── product-service/
├── order-service/
├── payment-service/
└── deploy/
    └── k8s/
```

【强制】所有服务模块结构统一：

```text
{service}/
├── build.gradle
├── src/main/java/.../{controller,service,repository,entity,dto,config,feign}
├── src/main/resources/{application.yml, bootstrap.yml}
├── src/test/java/...            # 单元测试
└── db/{schema.sql, data.sql}
```

---

## 14. 验收标准

### 14.1 功能验收（全部通过才算完成）

| 序号 | 验收项 | 判定 |
|------|--------|------|
| 1 | `gradle build` 全项目通过 | 命令执行成功 |
| 2 | docker-compose 一键启动成功 | 各服务健康检查通过 |
| 3 | 网关路由 `/api/v1/**` 到对应服务 | 调用返回 200 |
| 4 | 用户服务注册/登录可用 | 注册后可用 token 调通鉴权接口 |
| 5 | 下单调用扣库存（跨服务同步调用） | 返回正确并幂等 |
| 6 | 通过 MQ 异步发布订单事件并消费 | 消费者日志出现消费成功 |
| 7 | Nacos 修改配置后热刷新 | 无需重启配置生效 |
| 8 | 日志包含 traceId 且可跨服务追踪 | 同 traceId 贯穿调用链 |
| 9 | Actuator 指标可被 Prometheus 抓取 | `/actuator/prometheus` 返回指标 |
| 10 | README 可指导新成员本地启动 | 按文档步骤可复现 |

### 14.2 代码质量验收

- 单元测试覆盖核心业务（下单/扣库存/登录）关键分支，行覆盖率 ≥ 70%。
- 无 `TODO`/`待补充` 占位符残留；无硬编码密钥。
- 符合公司编码规范（见 [00-开发规范](../00-开发规范/README.md)）。

---

## 15. 注意事项

1. **版本对齐**：所有依赖版本以本文档第 4 节为准，禁止自行升级大版本导致不兼容。
2. **契约先行**：先定义好 Feign 接口与 DTO 再实现，服务间改动需同步评估影响。
3. **禁止跨库 join**：跨服务数据只能通过接口或事件获取。
4. **幂等是底线**：所有写接口、MQ 消费者必须幂等，否则高并发下必出数据问题。
5. **敏感信息**：任何交付物（含 README、配置示例）不得包含真实密钥。
6. **最终一致性**：不要为了让"看起来一致"而引入强一致分布式事务，优先事件 + 对账补偿。
7. **可复现**：README 必须让新人从零环境按步骤复现启动，缺失一步即为不合格。
8. **本指南输入输出**：AI 应将本指南视为"需求 + 约束"，逐章实现并在 README 或验收清单中标注每条验收项的落实情况。

---

## 16. 落地检查清单

| 序号 | 检查项 | 责任人 |
|------|--------|--------|
| 1 | 服务拆分清单 `docs/split.md` 已产出 | AI + 架构评审 |
| 2 | 统一返回/异常/错误码全局生效 | AI |
| 3 | 各服务 db/schema.sql 已产出且无跨库引用 | AI |
| 4 | 同步调用带超时与熔断降级 | AI |
| 5 | MQ 事件含幂等键与 traceId | AI |
| 6 | 配置全部外部化到 Nacos，无明文密钥 | AI |
| 7 | docker-compose 与 Dockerfile 可运行 | AI |
| 8 | 监控指标可采集 | AI |
| 9 | 全部验收项通过（第 14 节） | 架构评审 |

---

**文档结束**

*本文档由 pangpang-doc 维护*
