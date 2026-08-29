# JDK 安装配置

## 1. 文档说明

| 属性 | 说明 |
|------|------|
| 文档版本 | V1.0 |
| 生效日期 | 2026-08-15 |
| 适用范围 | 公司所有后端研发团队（JDK 25 + Temurin） |
| 作者 | 架构组 |

> **用途**：从零安装 JDK 25（Temurin 发行版）、配置 `JAVA_HOME`、实现多版本切换，并完成验证。本项目统一使用 JDK 25，禁止使用旧版本（Java 8/11/17）开发新代码。

---

## 2. 版本约定

| 项 | 约定 |
|----|------|
| **版本** | JDK 25（LTS） |
| **发行版** | Eclipse Temurin（Adoptium） |
| **构建工具要求** | Gradle 8.x 或 Maven 3.9.x（见 [Maven-Gradle-配置.md](./Maven-Gradle-配置.md)） |
| **框架支持** | Spring Boot 4.x、Spring Cloud 2024.x |

> 选择 Temurin 的原因：开源、免费、社区活跃、支持全平台，且与 Spring Boot 4.x 兼容性验证充分。不建议使用 Oracle JDK（商业许可限制）或龙井等衍生版。

---

## 3. 安装 JDK 25

### 3.1 下载

1. 打开 Adoptium 官网：https://adoptium.net/
2. 选择 **Version: 25**、**Operating System** 按本机选择、**Architecture** 选择 `x64`（Intel/AMD）或 `aarch64`（Apple Silicon/ARM 服务器）
3. 下载 **JDK**（非 JRE）安装包或压缩包

> 安装包方式：`.msi`（Windows）/ `.pkg`（macOS）；压缩包方式：`.tar.gz`（Linux/macOS）/ `.zip`（Windows）。推荐压缩包方式，便于多版本管理与卸载。

### 3.2 Linux 安装（tar.gz）

```bash
# 1. 解压到统一目录（建议 /opt/java）
sudo mkdir -p /opt/java
sudo tar -zxvf temurin-25-jdk_x64_linux.tar.gz -C /opt/java
sudo mv /opt/java/jdk-25* /opt/java/jdk-25

# 2. 配置环境变量（写入 /etc/profile.d/jdk.sh 或 ~/.bashrc）
echo 'export JAVA_HOME=/opt/java/jdk-25' | sudo tee /etc/profile.d/jdk.sh
echo 'export PATH=$JAVA_HOME/bin:$PATH' | sudo tee -a /etc/profile.d/jdk.sh

# 3. 使配置立即生效
source /etc/profile.d/jdk.sh
```

### 3.3 macOS 安装

方式一：`brew` 安装（推荐）

```bash
brew tap homebrew/cask-versions
brew install --cask temurin@25

# 查看安装路径
/usr/libexec/java_home -V
```

方式二：`.pkg` 安装后直接可用，`java -version` 验证即可。

### 3.4 Windows 安装（msi）

1. 双击 `.msi` 安装包，按向导默认安装
2. 安装完成后，配置系统环境变量：
   - 新增系统变量 `JAVA_HOME` = `C:\Program Files\Eclipse Adoptium\jdk-25.0.x`
   - 在 `Path` 中新增 `%JAVA_HOME%\bin`

```powershell
# PowerShell 验证
java -version
echo $env:JAVA_HOME
```

---

## 4. 验证安装

```bash
# 查看版本（应显示 openjdk version "25.0.x"）
java -version

# 查看编译器版本
javac -version

# 查看 JAVA_HOME
echo $JAVA_HOME

# 编译运行一个最小程序
cat > Hello.java <<'EOF'
public class Hello {
    public static void main(String[] args) {
        System.out.println("Hello JDK 25");
    }
}
EOF
javac Hello.java && java Hello
```

> 【强制】以上 5 条命令全部通过才视为安装成功；`java -version` 显示的不是 `25` 时，检查 `JAVA_HOME` 与 `Path` 是否被旧版本抢占。

---

## 5. 多版本切换（sdkman / jenv）

> 项目可能同时存在老系统（Java 8/17）与本项目（Java 25），推荐使用版本管理工具。

### 5.1 sdkman（Linux/macOS，推荐）

```bash
# 安装 sdkman
curl -s "https://get.sdkman.io" | bash
source "$HOME/.sdkman/bin/sdkman-init.sh"

# 安装 JDK 25
sdk install java 25-tem

# 切换默认版本
sdk default java 25-tem

# 项目级切换（在项目目录执行）
sdk use java 25-tem
```

### 5.2 Windows：使用环境变量切换

维护多个 `JAVA_HOME` 指向不同版本，切换时手动修改 `JAVA_HOME` 并重开终端即可（或将切换写为 `.bat` 脚本）。

> 【推荐】所有后端项目在根目录提供 `.sdkmanrc` 或 `.jdk` 版本文件，明确记录所需 JDK 版本，新成员 clone 后一条命令切换到位。

---

## 6. 常见问题排查

| 问题现象 | 原因 | 解决方案 |
|---------|------|---------|
| `java -version` 显示旧版本 | 系统自带或旧 JDK 抢先 | 将 `JAVA_HOME/bin` 置于 `PATH` 最前；检查 `/usr/lib/jvm` |
| `JAVA_HOME` 未生效 | 环境变量未加载 | `source` 配置或重开终端；Windows 重开 PowerShell |
| 编译报错 `UnsupportedClassVersionError` | 编译/运行 JDK 版本不一致 | 统一使用 JDK 25 编译与运行 |
| `-source 8 已过时` 警告 | 配置了旧编译版本 | 使用 `-source 25` 或移除兼容配置 |

---

## 7. 落地检查清单

| 序号 | 检查项 | 检查方式 | 责任人 |
|------|--------|---------|--------|
| 1 | `java -version` 显示 25.x | 命令验证 | 开发人员 |
| 2 | `JAVA_HOME` 指向 JDK 25 目录 | `echo $JAVA_HOME` | 开发人员 |
| 3 | `javac -version` 与运行版本一致 | 命令验证 | 开发人员 |
| 4 | 示例程序可编译运行 | `javac` + `java` | 开发人员 |
| 5 | 多版本工具（如 sdkman）配置完成 | `sdk current` | 开发人员 |

---

**文档结束**

*本文档由 pangpang-doc 维护*
