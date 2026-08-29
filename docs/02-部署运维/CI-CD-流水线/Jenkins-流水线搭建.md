# Jenkins 流水线搭建

## 1. 文档说明

| 属性 | 说明 |
|------|------|
| 文档版本 | V1.0 |
| 生效日期 | 2026-08-15 |
| 适用范围 | 公司所有研发团队与运维（Jenkins 2.x） |
| 作者 | 架构组 |

> **用途**：Jenkins 的安装、凭据管理、Pipeline as Code 编写，以及与 Docker/K8s 的集成，实现从代码提交到镜像构建的自动化。整体发布流程见 [自动化发布流程.md](./自动化发布流程.md)。

---

## 2. 架构与角色

```text
代码仓库(Git) ──> Jenkins(调度) ──> 构建(JDK25 + Gradle) ──> 镜像(Docker) ──> 部署(K8s/Docker)
```

| 角色 | 职责 |
|------|------|
| Master | 调度任务、管理凭据、展示 UI |
| Agent | 实际执行构建任务（可多节点） |
| 共享库（Shared Library） | 沉淀可复用流水线代码 |

> 【推荐】生产环境使用 Master + 多 Agent 架构，避免单点；构建任务在 Agent 上执行。

---

## 3. 安装（Docker 推荐）

```bash
docker run -d \
  --name jenkins \
  -p 8080:8080 -p 50000:50000 \
  -v jenkins-home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  jenkins/jenkins:lts

# 查看初始管理员密码
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

> 安装完成后访问 http://localhost:8080，按向导安装建议插件（Git、Pipeline、Docker Pipeline）。

---

## 4. 凭据管理

1. 进入 `Manage Jenkins → Credentials → Global`
2. 添加以下凭据：

| 凭据类型 | 用途 |
|---------|------|
| Username with password | Git 账号、镜像仓库账号 |
| Secret text | Token、Secret |
| SSH key | Git 免密拉取 |
| Docker Host | Docker 远程构建 |

> 【强制】禁止将密码/Tok写入 Jenkinsfile 或代码仓库，一律通过凭据引用。

---

## 5. Pipeline as Code

### 5.1 Jenkinsfile（Gradle + Docker 构建）

```groovy
pipeline {
    agent any

    environment {
        IMAGE_REGISTRY = 'registry.pangpang.com'
        IMAGE_NAME = "${IMAGE_REGISTRY}/order-service"
        IMAGE_TAG = "${env.BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                sh './gradlew clean build -x test'
            }
        }

        stage('Test') {
            steps {
                sh './gradlew test'
            }
        }

        stage('Build Image') {
            steps {
                script {
                    docker.build("${IMAGE_NAME}:${IMAGE_TAG}")
                }
            }
        }

        stage('Push Image') {
            steps {
                script {
                    docker.withRegistry("https://${IMAGE_REGISTRY}", 'registry-credentials') {
                        docker.image("${IMAGE_NAME}:${IMAGE_TAG}").push()
                    }
                }
            }
        }
    }

    post {
        success { echo '构建成功' }
        failure { echo '构建失败' }
        always  { cleanWs() }
    }
}
```

> 【强制】流水线必须包含构建、测试、镜像推送三阶段；构建产物由 `IMAGE_TAG = BUILD_NUMBER` 唯一标识。

### 5.2 与 K8s 集成

```groovy
stage('Deploy to K8s') {
    steps {
        sh """
            kubectl set image deployment/order-service \
                order-service=${IMAGE_NAME}:${IMAGE_TAG} -n production
        """
    }
}
```

> 【推荐】K8s 部署使用滚动更新 + 镜像 tag 回滚，避免停机发布。

---

## 6. 触发器配置

| 触发方式 | 适用场景 |
|---------|---------|
| Poll SCM | 定期检查代码变化 |
| Webhook（推荐） | 提交/合并即触发，实时 |
| 定时构建 | 每日定时任务 |
| 手动触发 | 人工发布 |

```groovy
// Jenkinsfile 中定义 Webhook 触发（GitHub）
triggers {
    githubPush()
}
```

---

## 7. 常见问题排查

| 问题现象 | 原因 | 解决方案 |
|---------|------|---------|
| 拉取代码失败 | 凭据错误/权限不足 | 核对凭据与仓库权限 |
| 构建内存不足 | Agent JVM 参数 | 调整 `JAVA_OPTS` 与 Agent 资源 |
| Docker 构建失败 | docker.sock 未挂载 | 挂载 `/var/run/docker.sock` |
| 镜像推送 401 | 仓库凭据错误 | 检查 `docker.withRegistry` 凭据 ID |
| 流水线报 `not found` | 插件缺失 | 安装对应 Pipeline 插件 |

---

## 8. 落地检查清单

| 序号 | 检查项 | 检查方式 | 责任人 |
|------|--------|---------|--------|
| 1 | Jenkins 可正常访问 | 打开管理台 | 运维 |
| 2 | 凭据均已通过凭据管理配置 | 检查 Credentials | 运维 |
| 3 | Jenkinsfile 纳入版本管理 | 仓库核对 | 后端开发 |
| 4 | Webhook 触发生效 | 提交验证触发 | 后端开发 |
| 5 | 镜像推送与部署验证 | 查看构建日志 | 运维 |

---

**文档结束**

*本文档由 pangpang-doc 维护*
