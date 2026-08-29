# Elasticsearch 安装

## 1. 文档说明

| 属性 | 说明 |
|------|------|
| 文档版本 | V1.0 |
| 生效日期 | 2026-08-15 |
| 适用范围 | 公司所有后端研发团队与运维（Elasticsearch 8.x） |
| 作者 | 架构组 |

> **用途**：Elasticsearch 8.x 的单机/集群安装、JVM 堆配置、安全认证、IK 分词器安装，以及常用 REST API 速查，用于全文检索与日志检索场景。

---

## 2. 版本约定

| 项 | 约定 |
|----|------|
| **版本** | Elasticsearch 8.x |
| **端口** | 9200（HTTP）/ 9300（传输） |
| **安全** | 8.x 默认开启 xpack.security |
| **分词** | IK 中文分词器 |
| **不兼容提醒** | ES 禁止以 root 运行，JVM 堆内存需手动配置 |

> 【强制】ES 8.x 禁止以 root 用户启动；`max_map_count` 等内核参数必须调优，否则启动报错。

---

## 3. 安装方式一：Linux 单机安装

### 3.1 系统调优

```bash
# 修改内核参数（永久生效）
echo 'vm.max_map_count = 262144' >> /etc/sysctl.conf
sysctl -p

# 关闭 swap 或降低 swappiness
echo 'vm.swappiness = 1' >> /etc/sysctl.conf
sysctl -p

# 文件句柄限制
echo 'elasticsearch - nofile 65535' >> /etc/security/limits.conf
```

### 3.2 下载安装

```bash
# 下载解压（版本号以官方最新 8.x 为准）
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-8.15.0-linux-x86_64.tar.gz
tar -zxvf elasticsearch-8.15.0-linux-x86_64.tar.gz
mv elasticsearch-8.15.0 /usr/local/elasticsearch

# 创建专用用户并授权（禁止 root 运行）
useradd -m elasticsearch
chown -R elasticsearch:elasticsearch /usr/local/elasticsearch
```

### 3.3 配置 JVM 堆内存（jvm.options）

```
# 堆内存建议为物理内存的一半，且不超过 32G（Xms 与 Xmx 保持一致）
-Xms4g
-Xmx4g
```

> 【强制】`Xms` 与 `Xmx` 必须一致，避免运行时堆扩容造成停顿；避免超过 32G（JVM 压缩指针失效）。

### 3.4 启动

```bash
su - elasticsearch
/usr/local/elasticsearch/bin/elasticsearch -d

# 验证
curl http://localhost:9200
```

---

## 4. 安装方式二：Docker（单机）

```bash
docker run -d \
  --name es8 \
  -p 9200:9200 -p 9300:9300 \
  -e "discovery.type=single-node" \
  -e "xpack.security.enabled=true" \
  -e "ELASTIC_PASSWORD=Pangpang@ES2026" \
  -e "ES_JAVA_OPTS=-Xms4g -Xmx4g" \
  -v es-data:/usr/share/elasticsearch/data \
  docker.elastic.co/elasticsearch/elasticsearch:8.15.0
```

---

## 5. 集群配置（elasticsearch.yml）

```yaml
cluster.name: pangpang-es
node.name: es-node-1
path.data: /var/lib/elasticsearch
path.logs: /var/log/elasticsearch
network.host: 0.0.0.0
http.port: 9200
discovery.seed_hosts: ["es-node-1", "es-node-2", "es-node-3"]
cluster.initial_master_nodes: ["es-node-1"]
xpack.security.enabled: true
```

> 【推荐】生产部署至少 3 节点（含专用主节点），启用安全认证并配置 Kibana。

---

## 6. 安全认证配置

### 6.1 生成内置用户密码

```bash
/usr/local/elasticsearch/bin/elasticsearch-setup-passwords auto
# 输出 elastic / kibana_system 等账号密码，请妥善保存
```

### 6.2 验证带认证访问

```bash
curl -u elastic:password http://localhost:9200
```

> 【强制】生产必须启用 `xpack.security`，禁止匿名访问 9200 端口。

---

## 7. IK 中文分词器

```bash
# 安装 IK 插件（版本必须与 ES 完全一致）
/usr/local/elasticsearch/bin/elasticsearch-plugin install \
  https://github.com/infinilabs/analysis-ik/releases/download/v8.15.0/elasticsearch-analysis-ik-8.15.0.zip

# 重启 ES 后验证
curl -X POST http://localhost:9200/_analyze -H 'Content-Type: application/json' \
  -d '{"analyzer": "ik_max_word", "text": "中华人民共和国国歌"}'
```

---

## 8. 常用 REST API 速查

| 场景 | 请求 |
|------|------|
| 查看集群健康 | `GET /_cluster/health` |
| 查看节点 | `GET /_cat/nodes?v` |
| 创建索引 | `PUT /products` |
| 写入文档 | `POST /products/_doc` |
| 全文检索 | `GET /products/_search?q=keyword` |
| 查看索引分片 | `GET /_cat/shards?v` |
| 删除索引 | `DELETE /products` |

---

## 9. 常见问题排查

| 问题现象 | 原因 | 解决方案 |
|---------|------|---------|
| `bootstrap check failure` | 内核参数不足 | 设置 `max_map_count`、关闭 swap |
| 无法以 root 启动 | 安全限制 | 使用 `elasticsearch` 用户运行 |
| `OutOfMemoryError` | 堆内存不足 | 调大 `Xms/Xmx` 并保持一致 |
| 集群状态 yellow | 主分片未分配/磁盘满 | `GET /_cluster/allocation/explain` 定位 |
| IK 分词不生效 | 插件版本不匹配 | 插件版本必须与 ES 一致 |

---

## 10. 落地检查清单

| 序号 | 检查项 | 检查方式 | 责任人 |
|------|--------|---------|--------|
| 1 | 版本为 8.x | `GET /` 返回 version | 后端开发/运维 |
| 2 | 非 root 用户运行 | `ps -ef \| grep elastic` | 运维 |
| 3 | JVM 堆已配置且 Xms=Xmx | 查看 jvm.options | 运维 |
| 4 | 安全认证已开启 | 未带账号访问返回 401 | 运维 |
| 5 | IK 分词器可用 | `_analyze` 验证 | 后端开发 |

---

**文档结束**

*本文档由 pangpang-doc 维护*
