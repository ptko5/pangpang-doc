# RabbitMQ / Kafka 安装

## 1. 文档说明

| 属性 | 说明 |
|------|------|
| 文档版本 | V1.0 |
| 生效日期 | 2026-08-15 |
| 适用范围 | 公司所有后端研发团队（RabbitMQ 3.x / Kafka 3.x） |
| 作者 | 架构组 |

> **用途**：介绍 RabbitMQ 与 Kafka 的选择依据，分别给出安装方式、核心概念与 Spring Boot 集成要点，作为消息队列选型与部署的标准参考。

---

## 2. 选型对比

| 维度 | RabbitMQ 3.x | Kafka 3.x |
|------|-------------|-----------|
| 定位 | 消息代理（AMQP 协议） | 分布式流平台 |
| 吞吐量 | 万级/秒 | 百万级/秒 |
| 消息可靠性 | 确认机制完善 | 副本机制 |
| 使用场景 | 业务解耦、任务队列、RPC | 日志收集、事件流、大数据 |
| 消费方式 | 推/拉 | 仅拉取 + offset 管理 |

> **选型建议**：业务消息（订单、通知）用 RabbitMQ；高吞吐日志/事件流用 Kafka。两者可并存，不互相替代。

---

## 3. RabbitMQ 安装（Docker 推荐）

### 3.1 安装

```bash
docker run -d \
  --name rabbitmq \
  -p 5672:5672 -p 15672:15672 \
  -e RABBITMQ_DEFAULT_USER=admin \
  -e RABBITMQ_DEFAULT_PASS=Pangpang@MQ2026 \
  rabbitmq:3.13-management
```

> `-management` 镜像自带 Web 管理控制台（http://localhost:15672）。

### 3.2 核心概念

| 概念 | 说明 |
|------|------|
| Producer / Consumer | 生产者 / 消费者 |
| Exchange | 交换机（direct / topic / fanout / headers） |
| Queue | 消息队列 |
| Binding | 交换机与队列的绑定关系 |
| vhost | 虚拟主机（隔离环境） |

### 3.3 Spring Boot 集成要点

```yaml
spring:
  rabbitmq:
    host: localhost
    port: 5672
    username: admin
    password: Pangpang@MQ2026
    publisher-confirm-type: correlated
    listener:
      simple:
        acknowledge-mode: manual
```

> 【推荐】生产开启 `publisher-confirm-type`（发送确认）与消费端手动 ACK，避免消息丢失与重复消费。

---

## 4. Kafka 安装（Docker 推荐）

### 4.1 安装（KRaft 模式，3.x 无需 ZooKeeper）

```bash
docker run -d \
  --name kafka \
  -p 9092:9092 \
  -e KAFKA_NODE_ID=1 \
  -e KAFKA_PROCESS_ROLES=broker,controller \
  -e KAFKA_LISTENERS=PLAINTEXT://:9092,CONTROLLER://:9093 \
  -e KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://localhost:9092 \
  -e KAFKA_CONTROLLER_LISTENER_NAMES=CONTROLLER \
  -e KAFKA_LISTENER_SECURITY_PROTOCOL_MAP=CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT \
  -e KAFKA_CONTROLLER_QUORUM_VOTERS=1@localhost:9093 \
  -e KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR=1 \
  apache/kafka:3.8.0
```

### 4.2 核心概念

| 概念 | 说明 |
|------|------|
| Topic | 消息主题 |
| Partition | 分区（并行度单元） |
| Replica | 分区副本（高可用） |
| Producer / Consumer | 生产 / 消费 |
| Consumer Group | 消费组（组内分摊分区） |
| Offset | 消费位点 |

### 4.3 常用命令行

```bash
# 进入容器
docker exec -it kafka /opt/kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --list

# 创建主题
kafka-topics.sh --bootstrap-server localhost:9092 --create --topic orders --partitions 3 --replication-factor 1

# 控制台生产/消费
kafka-console-producer.sh --bootstrap-server localhost:9092 --topic orders
kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic orders --from-beginning
```

### 4.4 Spring Boot 集成要点

```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.apache.kafka.common.serialization.StringSerializer
    consumer:
      group-id: order-service-group
      auto-offset-reset: earliest
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.apache.kafka.common.serialization.StringDeserializer
```

```java
@Service
public class OrderMessageService {

    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;

    public void send(String topic, String payload) {
        kafkaTemplate.send(topic, payload);
    }

    @KafkaListener(topics = "orders", groupId = "order-service-group")
    public void onMessage(String message) {
        // 消费业务处理（保证幂等）
    }
}
```

> 【强制】Kafka 消费者业务必须做幂等处理（消息可能重复投递）；topic 的 `partitions` 与 `replication-factor` 依据数据量提前规划。

---

## 5. 常见问题排查

| 问题现象 | 原因 | 解决方案 |
|---------|------|---------|
| RabbitMQ 连接失败 | 未开放 5672 / 密码错误 | 核对端口与账号 |
| 消息堆积 | 消费慢/异常 | 检查消费者线程与 ACK 配置 |
| Kafka 消费重复 | 重平衡/未提交 offset | 幂等 + 手动提交 |
| Kafka `UNKNOWN_TOPIC` | 自动创建未开 | `auto.create.topics.enable=true` 或预创建 |
| Kafka 集群脑裂 | controller 配置错误 | 核对 KRaft 配置 |

---

## 6. 落地检查清单

| 序号 | 检查项 | 检查方式 | 责任人 |
|------|--------|---------|--------|
| 1 | RabbitMQ 管理台可访问 | http://localhost:15672 | 后端开发 |
| 2 | Kafka 可创建主题并收发 | 命令行验证 | 后端开发 |
| 3 | 生产者发送确认已开启 | 配置核对 | 后端开发 |
| 4 | 消费者幂等处理 | 代码审查 | 后端开发 |
| 5 | 生产环境集群高可用 | 节点数与副本核对 | 运维 |

---

**文档结束**

*本文档由 pangpang-doc 维护*
