# Maven / Gradle 配置

## 1. 文档说明

| 属性 | 说明 |
|------|------|
| 文档版本 | V1.0 |
| 生效日期 | 2026-08-15 |
| 适用范围 | 公司所有后端研发团队（Gradle 8 为主，Maven 3.9 备选） |
| 作者 | 架构组 |

> **用途**：本项目构建工具以 **Gradle 8.x** 为主、Maven 3.9.x 为备选。本文档覆盖安装、镜像仓库、JDK 25 编译配置、多环境 profile，以及从 Maven 迁移到 Gradle 的要点。

---

## 2. 工具选型

| 项 | 决定 |
|----|------|
| **主构建工具** | Gradle 8.x |
| **备选** | Maven 3.9.x（存量项目/特定场景） |
| **编译 JDK** | JDK 25（见 [JDK-安装配置.md](./JDK-安装配置.md)） |
| **应用框架** | Spring Boot 4.x、Spring Cloud 2024.x |

> 选择 Gradle 的原因：构建速度快（增量编译/缓存）、脚本灵活、对 JDK 25 与 Spring Boot 4 插件支持及时。Maven 仅保留用于无法迁移的存量项目。

---

## 3. Gradle 安装

### 3.1 安装 Gradle 8

```bash
# Linux/macOS（推荐使用 sdkman 管理）
sdk install gradle 8.10.2
sdk default gradle 8.10.2

# Windows：下载 zip 解压，配置 GRADLE_HOME 与 PATH
# 验证
gradle -v
```

> 【推荐】项目统一使用 **Gradle Wrapper**（`gradle wrapper` 生成的 `gradlew`），并提交 `gradle/wrapper/` 目录，保证团队成员构建版本完全一致。

### 3.2 初始化项目并生成 Wrapper

```bash
# 已有项目（build.gradle 存在时）
gradle wrapper --gradle-version 8.10.2

# 之后统一使用
./gradlew build
```

---

## 4. 镜像仓库配置

> 【强制】为提升国内拉取速度，所有项目必须配置阿里云镜像仓库，禁止直接依赖公网慢速源。

### 4.1 Gradle 全局镜像（~/.gradle/init.gradle）

```gradle
allprojects {
    repositories {
        maven { url 'https://maven.aliyun.com/repository/public' }
        maven { url 'https://maven.aliyun.com/repository/gradle-plugin' }
        mavenCentral()
    }
}
```

### 4.2 Maven 全局镜像（~/.m2/settings.xml）

```xml
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0">
  <mirrors>
    <mirror>
      <id>aliyun</id>
      <mirrorOf>central</mirrorOf>
      <url>https://maven.aliyun.com/repository/public</url>
    </mirror>
  </mirrors>
</settings>
```

---

## 5. JDK 25 编译配置（Gradle）

> 【强制】项目必须显式声明 `java.toolchain` 为 25，避免本机默认 JDK 差异导致编译失败。

### 5.1 build.gradle（Groovy DSL）

```gradle
plugins {
    id 'java'
    id 'org.springframework.boot' version '4.0.0'
    id 'io.spring.dependency-management' version '1.1.6'
}

group = 'com.pangpang'
version = '1.0.0'

java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(25)
    }
}

repositories {
    maven { url 'https://maven.aliyun.com/repository/public' }
    mavenCentral()
}

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}

tasks.named('test') {
    useJUnitPlatform()
}
```

### 5.2 多环境 profile

> 【推荐】通过 `application-{env}.yml` + 构建参数区分 dev/test/prod，禁止硬编码环境差异到代码。

```gradle
// build.gradle 中按环境注入参数
def env = project.hasProperty('env') ? project.property('env') : 'dev'

bootRun {
    args = ["--spring.profiles.active=${env}"]
}

// 打包命令示例
./gradlew bootJar -Penv=prod
```

### 5.3 Spring Boot 4 打包

```bash
# 生成可执行 jar
./gradlew bootJar

# 本地运行
./gradlew bootRun --args='--spring.profiles.active=dev'
```

---

## 6. Maven 配置（备选）

> Maven 仅用于存量项目，新建项目一律使用 Gradle。

### 6.1 pom.xml 关键配置

```xml
<properties>
    <java.version>25</java.version>
    <spring-boot.version>4.0.0</spring-boot.version>
    <spring-cloud.version>2024.0.0</spring-cloud.version>
</properties>

<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
        </plugin>
    </plugins>
</build>
```

### 6.2 常用命令

```bash
mvn clean package -DskipTests -Pprod   # 生产打包
mvn spring-boot:run -Dspring-boot.run.profiles=dev
```

---

## 7. 从 Maven 迁移到 Gradle 要点

| Maven 概念 | Gradle 对应 | 说明 |
|-----------|------------|------|
| `pom.xml` | `build.gradle` | 声明式改为脚本式 |
| `<dependencies>` | `dependencies {}` | 语法变更 |
| `-Pprod` | `-Pprod` | profile 概念一致 |
| `mvn clean package` | `./gradlew clean build` | 命令差异 |
| `settings.xml` | `init.gradle` | 镜像配置位置 |

> 【推荐】迁移时使用 `gradle init` 生成脚手架，并重点核对多模块结构（`settings.gradle` 的 `include` 声明）。

---

## 8. 落地检查清单

| 序号 | 检查项 | 检查方式 | 责任人 |
|------|--------|---------|--------|
| 1 | `gradle -v` 显示 8.x | 命令验证 | 后端开发 |
| 2 | Wrapper 已提交至仓库 | 检查 `gradlew` 存在 | 后端开发 |
| 3 | 镜像仓库为阿里云 | 构建日志确认 | 后端开发 |
| 4 | `./gradlew build` 在 JDK 25 下通过 | 本地构建 | 后端开发 |
| 5 | 多环境 profile 打包正常 | `./gradlew bootJar -Penv=prod` | 后端开发 |

---

**文档结束**

*本文档由 pangpang-doc 维护*
