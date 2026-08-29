# Helm Chart 使用

## 1. 文档说明

| 属性 | 说明 |
|------|------|
| 文档版本 | V1.0 |
| 生效日期 | 2026-08-15 |
| 适用范围 | 公司所有使用 K8s 的研发团队与运维（Helm 3） |
| 作者 | 架构组 |

> **用途**：Helm 3 基础、Chart 目录结构、自定义 Spring Boot Chart、values 模板化以及发布与回滚，实现 K8s 应用的可复用打包管理。

---

## 2. Helm 是什么

Helm 是 Kubernetes 的包管理工具，将一组 K8s 资源（Deployment/Service/ConfigMap 等）打包为 **Chart**，通过 **values** 参数化实现一次编写、多环境复用。

```text
Chart（模板 + values）
   └─> helm install
         └─> Release（一个已部署的实例）
```

> Helm 3 无需 Tiller，直接与 kube-apiserver 通信，更安全简单。

---

## 3. Chart 目录结构

```text
order-service/
├── Chart.yaml          # Chart 元数据
├── values.yaml         # 默认参数
├── values-prod.yaml    # 生产覆盖参数
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    ├── configmap.yaml
    └── _helpers.tpl     # 公共模板函数
```

### 3.1 Chart.yaml

```yaml
apiVersion: v2
name: order-service
description: Order Service Helm Chart
type: application
version: 1.2.0
appVersion: "1.2.0"
```

---

## 4. values 模板化

### 4.1 默认 values.yaml

```yaml
replicaCount: 3
image:
  repository: registry.pangpang.com/order-service
  tag: "1.2.0-45"
  pullPolicy: IfNotPresent
service:
  type: ClusterIP
  port: 80
resources:
  requests:
    cpu: 500m
    memory: 512Mi
  limits:
    cpu: "1"
    memory: 1Gi
env:
  SPRING_PROFILES_ACTIVE: prod
```

### 4.2 模板文件（templates/deployment.yaml）

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Values.nameOverride | default .Chart.Name }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Chart.Name }}
  template:
    metadata:
      labels:
        app: {{ .Chart.Name }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: 8080
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: {{ .Values.env.SPRING_PROFILES_ACTIVE | quote }}
```

> 【强制】模板中禁止硬编码环境差异值，一律通过 `values` 注入。

---

## 5. 安装与发布

```bash
# 1. 语法与渲染检查
helm lint ./order-service
helm template ./order-service   # 本地渲染预览

# 2. 按环境安装
helm install order-service ./order-service -f values-prod.yaml -n production

# 3. 升级（滚动更新）
helm upgrade order-service ./order-service -f values-prod.yaml -n production

# 4. 查看 Release
helm list -n production
```

---

## 6. 版本管理与回滚

```bash
# 查看发布历史
helm history order-service -n production

# 回滚到指定版本
helm rollback order-service 3 -n production
```

> 【推荐】每次升级前记录 `REVISION`，故障时一条命令回滚到上一版本。

---

## 7. 仓库管理（Chart Repository / OCI）

```bash
# 打包并推送 Chart（OCI 方式）
helm package ./order-service
helm push order-service-1.2.0.tgz oci://registry.pangpang.com/charts

# 拉取使用
helm pull oci://registry.pangpang.com/charts/order-service --version 1.2.0
```

> 【推荐】Chart 纳入仓库版本管理，多环境复用同一 Chart 不同 values。

---

## 8. 常见问题排查

| 问题现象 | 原因 | 解决方案 |
|---------|------|---------|
| `lint` 报错 | 模板语法错误 | 修复 YAML/模板语法 |
| 渲染值不对 | 变量名/优先级错误 | `helm template` 本地预览 |
| 安装失败 | values 缺 key | 核对 values 文件与模板引用 |
| 升级后异常 | 配置变更 | `helm rollback` 回滚 |
| 权限不足 | RBAC 受限 | 检查发布账号权限 |

---

## 9. 落地检查清单

| 序号 | 检查项 | 检查方式 | 责任人 |
|------|--------|---------|--------|
| 1 | `helm lint` 通过 | 命令验证 | 后端开发 |
| 2 | 多环境 values 分离 | 文件核对 | 后端开发 |
| 3 | 可安装/升级/回滚 | 发布演练 | 运维 |
| 4 | 模板无硬编码环境值 | 模板审查 | 后端开发 |
| 5 | Chart 纳入仓库管理 | 仓库核对 | 运维 |

---

**文档结束**

*本文档由 pangpang-doc 维护*
