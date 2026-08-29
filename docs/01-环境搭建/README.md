# 环境搭建

## 1. 文档说明

| 属性 | 说明 |
|------|------|
| 文档版本 | V1.0 |
| 生效日期 | 2026-08-15 |
| 适用范围 | 公司所有研发团队的开发环境与中间件搭建 |
| 作者 | 架构组 |

> **模块定位**：本模块是「从一台新机器到能跑起整个项目」的完整指南。覆盖开发环境（JDK/Node.js/IDE/构建工具）与中间件（MySQL/Redis/Nginx 等）的安装配置。

---

## 2. 模块目录导航

### 2.1 树形结构

```text
01-环境搭建/
├── README.md                              ← 你正在看的模块总览
│
├── 开发环境/                              🖥️ 本地开发必备
│   ├── JDK-安装配置.md                     ☕ JDK 25（Temurin）安装与多版本管理
│   ├── Node.js-安装配置.md                 🟢 Node.js 22 LTS + nvm + 包管理器
│   ├── IDE-配置推荐.md                     💡 IntelliJ IDEA 配置与插件清单
│   └── Maven-Gradle-配置.md                🛠️ Gradle 8 构建配置与仓库镜像
│
└── 中间件安装/                            🧰 常用中间件
    ├── MySQL-安装与配置.md                 🐬 MySQL 8.x 安装与初始化
    ├── Redis-安装与配置.md                 🚀 Redis 7.x 安装与高可用
    ├── Nginx-安装与配置.md                 🌐 Nginx 反向代理与 HTTPS
    ├── RabbitMQ-Kafka-安装.md              🐰 Kafka / RabbitMQ 消息队列
    └── Elasticsearch-安装.md               🔍 Elasticsearch 8.x 安装与集群
```

### 2.2 每份文档的一句话用途

| 文档名 | 一句话用途 | 谁必须读 |
|--------|-----------|---------|
| `JDK-安装配置.md` | 安装 JDK 25、配置 JAVA_HOME、多版本切换 | 所有后端开发 |
| `Node.js-安装配置.md` | 安装 Node.js 22 LTS、nvm 管理、配置 npm/pnpm 镜像 | 前端开发 |
| `IDE-配置推荐.md` | IDEA 配置 JDK25/Spring Boot 4 支持、插件、代码风格 | 后端开发 |
| `Maven-Gradle-配置.md` | Gradle 8 配置 JDK 25 编译、镜像仓库、多环境 profile | 后端开发 |
| `MySQL-安装与配置.md` | MySQL 8.x 安装、utf8mb4、用户权限、备份 | 后端开发/DBA |
| `Redis-安装与配置.md` | Redis 7.x 安装、持久化、主从哨兵、常用命令 | 后端开发/运维 |
| `Nginx-安装与配置.md` | Nginx 安装、反向代理、负载均衡、HTTPS | 后端开发/运维 |
| `RabbitMQ-Kafka-安装.md` | 消息队列选型、Kafka/RabbitMQ 安装与 Spring Boot 集成 | 后端开发 |
| `Elasticsearch-安装.md` | ES 8.x 安装、JVM 配置、IK 分词器、REST API | 后端开发 |

---

## 3. 技术栈版本对照表

> 与 [docs/README.md](../README.md) 保持一致，安装时请使用以下版本，禁止自行升级到不兼容版本。

| 组件 | 版本 | 安装方式（本模块） |
|------|------|-------------------|
| **JDK** | 25（Temurin） | `JDK-安装配置.md` |
| **Node.js** | 22 LTS | `Node.js-安装配置.md` |
| **Gradle** | 8.x | `Maven-Gradle-配置.md` |
| **Maven** | 3.9.x（备选） | `Maven-Gradle-配置.md` |
| **MySQL** | 8.x | `MySQL-安装与配置.md` |
| **Redis** | 7.x | `Redis-安装与配置.md` |
| **Nginx** | 1.27.x | `Nginx-安装与配置.md` |
| **Elasticsearch** | 8.x | `Elasticsearch-安装.md` |
| **Kafka** | 3.x | `RabbitMQ-Kafka-安装.md` |
| **RabbitMQ** | 3.x | `RabbitMQ-Kafka-安装.md` |

---

## 4. 快速上手路径（新机器三步走）

```mermaid
flowchart LR
    A[新机器] --> B[安装 JDK 25<br/>JDK-安装配置]
    B --> C[安装 IDE<br/>IDE-配置推荐]
    C --> D[安装构建工具<br/>Maven-Gradle-配置]
    D --> E[安装中间件<br/>MySQL / Redis 等]
    E --> F[跑通项目<br/>参考 02-部署运维]
```

1. **第 1 步**：安装 JDK 25（见 `开发环境/JDK-安装配置.md`）
2. **第 2 步**：配置 IDE 与构建工具（见 `开发环境/IDE-配置推荐.md` + `开发环境/Maven-Gradle-配置.md`）
3. **第 3 步**：按项目依赖安装中间件（见 `中间件安装/` 对应文档）

> 前后端都参与的项目，前端另需 Node.js（见 `开发环境/Node.js-安装配置.md`）。

---

## 5. 与其他模块的关联

| 上下游 | 关联点 |
|--------|--------|
| **上游** | 安装版本严格遵循 `../README.md` 技术栈表 |
| **下游（部署运维）** | 单机安装配置可直接复用为 `../02-部署运维/` 中 Docker/K8s 部署的参数依据（如 MySQL 字符集、Redis 持久化、Nginx 代理配置） |
| **规范** | 安装后的中间件使用必须遵守 `../00-开发规范/后端开发规范/SQL与数据访问规范.md`（如 MySQL 字符集、索引）与 `../00-开发规范/安全开发规范.md` |

---

## 6. 落地检查清单

| 序号 | 检查项 | 检查方式 | 责任人 |
|------|--------|---------|--------|
| 1 | JDK 版本是否为 25，`java -version` 正常 | 命令验证 | 开发人员 |
| 2 | Node.js 是否 22 LTS，包管理器镜像配置完成 | 命令验证 | 前端开发 |
| 3 | 构建工具能否编译通过（Gradle 8 + JDK 25） | 本地构建 | 开发人员 |
| 4 | 中间件版本是否与技术栈表一致 | 版本命令核对 | 开发人员/运维 |
| 5 | MySQL 是否 utf8mb4 字符集 | `SHOW VARIABLES LIKE 'character_set%'` | DBA |
| 6 | Redis 持久化与备份策略是否配置 | 配置文件核对 | 运维 |

---

**文档结束**

*本文档由 pangpang-doc 维护*
