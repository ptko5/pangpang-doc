# Kubernetes 入门

## 1. 文档说明

| 属性 | 说明 |
|------|------|
| 文档版本 | V1.0 |
| 生效日期 | 2026-08-15 |
| 适用范围 | 公司所有研发团队与运维（K8s 1.30+） |
| 作者 | 架构组 |

> **用途**：掌握 Kubernetes 核心概念、核心资源 YAML、kubectl 速查，并通过 Spring Boot（JDK 25 镜像）部署实战快速上手。

---

## 2. 核心概念

| 概念 | 说明 |
|------|------|
| **Pod** | 最小调度单元，内含一个或多个容器 |
| **Deployment** | 管理无状态应用副本与滚动更新 |
| **Service** | 提供稳定的访问入口（ClusterIP/NodePort/LoadBalancer） |
| **ConfigMap / Secret** | 配置与敏感信息管理 |
| **Ingress** | 七层入口，路由外部流量 |
| **Namespace** | 资源隔离（dev/test/prod） |

```text
Ingress ──> Service ──> Pod(Deployment 管理)
                           └─> 容器(镜像)
```

> 核心思想：**声明式管理**——你描述「期望状态」，K8s 控制平面持续调和到该状态。

---

## 3. 架构总览

```text
┌────────────── 控制平面（Master）──────────────┐
│  kube-apiserver │ etcd │ controller-manager │  scheduler │
└───────────────────────────────────────────────┘
                      │
┌─────────────────────┴─────────────────────────┐
│  工作节点 1             工作节点 2               │
│  kubelet │ kube-proxy   kubelet │ kube-proxy    │
│  Pod 容器运行时          Pod 容器运行时           │
└───────────────────────────────────────────────┘
```

---

## 4. 核心资源 YAML

### 4.1 Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
    spec:
      containers:
        - name: order-service
          image: registry.pangpang.com/order-service:1.2.0-45
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: 500m
              memory: 512Mi
            limits:
              cpu: "1"
              memory: 1Gi
          readinessProbe:
            httpGet:
              path: /actuator/health
              port: 8080
            initialDelaySeconds: 20
```

> 【强制】生产 Deployment 必须配置资源 `requests/limits` 与健康检查探针。

### 4.2 Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: order-service
  namespace: production
spec:
  selector:
    app: order-service
  ports:
    - port: 80
      targetPort: 8080
  type: ClusterIP
```

### 4.3 ConfigMap / Secret

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: order-service-config
data:
  application-prod.yml: |
    server:
      port: 8080
---
apiVersion: v1
kind: Secret
metadata:
  name: order-service-secret
type: Opaque
data:
  db-password: cGFuZ3Bhbmc=   # base64 编码，禁止明文
```

---

## 5. kubectl 速查

| 操作 | 命令 |
|------|------|
| 查看资源 | `kubectl get pods -n production` |
| 查看详情 | `kubectl describe pod <name> -n production` |
| 查看日志 | `kubectl logs -f <pod> -n production` |
| 进入容器 | `kubectl exec -it <pod> -- bash` |
| 应用配置 | `kubectl apply -f deployment.yaml` |
| 滚动更新 | `kubectl rollout restart deployment/order-service` |
| 回滚 | `kubectl rollout undo deployment/order-service` |
| 端口转发 | `kubectl port-forward svc/order-service 8080:80` |

---

## 6. Spring Boot 部署实战（JDK 25 镜像）

```bash
# 1. 构建并推送镜像（见 Dockerfile-编写规范）
docker build -t registry.pangpang.com/order-service:1.2.0-45 .
docker push registry.pangpang.com/order-service:1.2.0-45

# 2. 应用资源配置
kubectl apply -f configmap.yaml
kubectl apply -f secret.yaml
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml

# 3. 验证部署
kubectl get pods -n production
kubectl rollout status deployment/order-service -n production
```

> 【推荐】生产使用 [Helm-Chart-使用.md](./Helm-Chart-使用.md) 打包管理以上资源，支持参数化与版本回滚。

---

## 7. 常见问题排查

| 问题现象 | 原因 | 解决方案 |
|---------|------|---------|
| Pod 一直 `Pending` | 资源不足/调度失败 | `describe pod` 查看事件 |
| Pod 一直 `CrashLoopBackOff` | 启动失败/配置错误 | 查看日志与探针配置 |
| 镜像拉取失败 | 私有仓库未认证 | 配置 `imagePullSecrets` |
| 服务无法访问 | Service selector 不匹配 | 核对标签与探针 |
| 集群节点 NotReady | 节点资源耗尽/宕机 | `kubectl describe node` 排查 |

---

## 8. 落地检查清单

| 序号 | 检查项 | 检查方式 | 责任人 |
|------|--------|---------|--------|
| 1 | 集群节点 Ready | `kubectl get nodes` | 运维 |
| 2 | 应用可滚动发布与回滚 | 发布演练 | 后端开发 |
| 3 | 资源限额与探针已配置 | YAML 审查 | 后端开发 |
| 4 | 敏感信息使用 Secret | 审查 ConfigMap/Secret | 运维 |
| 5 | 日志与监控接入 | 查看 Pod 日志 | 运维 |

---

**文档结束**

*本文档由 pangpang-doc 维护*
