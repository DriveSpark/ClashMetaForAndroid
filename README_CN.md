# ClashMetaForAndroid 构建问题修复与打包指南

这份文档总结了解决 ClashMetaForAndroid 构建问题的全过程，旨在帮助学生理解 Android 项目构建中可能遇到的环境、依赖和网络问题及其解决方案。

## 1. 问题背景

在尝试构建项目时，我们遇到了三个核心障碍：

1.  **Go 环境错误**：Gradle 构建任务 `:core:externalGolangBuildMetaReleaseArmeabiV7a` 失败，提示无法启动 `go` 命令。
2.  **资源文件缺失**：编译时报错 `Android resource linking failed`，提示 `application_name_meta` 等字符串资源找不到。
3.  **网络连接拒绝**：Gradle 下载依赖时报错 `Connection refused: 127.0.0.1:7897`，导致 `lint-gradle` 等插件无法下载。

## 2. 详细修复步骤

### 2.1 环境配置 (local.properties)

**问题原因**：Android Studio 或命令行构建时，Gradle 无法从系统环境变量中自动获取 Go 语言的安装路径。

**解决方案**：
在项目根目录的 `local.properties` 文件中，手动指定 Go 二进制文件的位置。

**操作**：
打开 `local.properties`，添加以下行（请根据实际 Go 安装路径修改）：

```properties
# macOS/Linux 常见路径
go.dir=/usr/local/go/bin
# Windows 常见路径可能是 C\:\\Program Files\\Go\\bin
```

### 2.2 构建脚本修正 (core/build.gradle.kts)

**问题原因**：即便配置了 `go.dir`，`GolangBuildTask`（自定义构建任务）在执行时可能仍未将 Go 路径加入到 PATH 环境变量中。

**解决方案**：
修改 `core/build.gradle.kts`，添加逻辑将 `go.dir` 动态注入到任务的执行环境中。

**代码改动**：
在 `core/build.gradle.kts` 中引入必要的类，并在任务配置中添加路径处理逻辑：

```kotlin
// 1. 引入必要包
import java.util.Properties
import java.io.File

// ... (中间代码省略)

// 2. 在 afterEvaluate 块中添加 PATH 配置逻辑
afterEvaluate {
    // 读取 local.properties
    val localProperties = Properties()
    val localPropertiesFile = rootProject.file("local.properties")
    if (localPropertiesFile.exists()) {
        localProperties.load(localPropertiesFile.inputStream())
    }

    // 获取 go.dir
    val goDir = localProperties.getProperty("go.dir")
        ?: listOf("/usr/local/go/bin", "/opt/homebrew/bin").firstOrNull { file("$it/go").exists() }

    // 为每个 Go 构建任务配置环境变量
    tasks.withType(GolangBuildTask::class.java).forEach {
        it.inputs.dir(golangSource)

        if (it is Exec && goDir != null) {
            val currentPath = System.getenv("PATH") ?: ""
            // 将 Go 路径添加到 PATH 最前面
            it.environment("PATH", "$goDir${File.pathSeparator}$currentPath")
            println("Configuring PATH for ${it.name}: $goDir")
        }
    }
}
```

### 2.3 依赖完整性 (Git Submodule)

**问题原因**：项目依赖 `Clash.Meta` 的核心代码，这些代码位于 Git 子模块 `core/src/foss/golang` 中。如果只 clone 了主项目而未初始化子模块，该目录为空，导致编译失败和资源缺失报错。

**解决方案**：
初始化并更新 Git 子模块。

**命令**：

```bash
git submodule update --init --recursive
```

### 2.4 网络环境处理

**问题原因**：Gradle 配置或系统环境变量中设置了代理（如 `127.0.0.1:7897`），但代理软件未运行，导致连接被拒绝。

**解决方案**：

- **方法一（推荐）**：启动代理软件（如 Clash），确保监听 7897 端口。
- **方法二**：在终端清除代理变量后再构建。
  ```bash
  unset http_proxy https_proxy all_proxy
  ```
- **方法三**：检查 `~/.gradle/gradle.properties` 是否有全局代理配置并清理。

## 3. 打包与产物

修复上述问题后，执行打包命令：

```bash
# 编译 Release 包
./gradlew :app:assembleMetaRelease
```

**最终产物路径**：
`app/build/outputs/apk/meta/release/`

**文件说明**：

- `cmfa-*-meta-universal-release.apk`: 通用版，体积较大但兼容性最好。
- `cmfa-*-meta-arm64-v8a-release.apk`: 针对主流现代手机的优化版。

## 4. 注意事项 (常见坑点)

1.  **阅读文档**：拿到开源项目第一件事是看 `README.md`，通常会有 "Build" 或 "Compilation" 章节说明依赖。
2.  **子模块检查**：看到文件夹为空或莫名其妙的 "File not found" / "Resource missing"，首先检查是否有 Git Submodule (`.gitmodules` 文件存在是关键线索)。
3.  **环境路径**：Android Studio 的环境变量与终端 Shell 可能不一致。在 `local.properties` 中显式配置路径是最稳妥的方法。
4.  **Gradle 代理**：Gradle 的网络连接非常敏感。报错 `Connection refused` 99% 是因为配置了代理但没开，或者代理连不上。
5.  **Clean 构建**：修改了构建脚本 (`build.gradle.kts`) 后，建议执行 `./gradlew clean` 清理缓存，避免旧的配置残留干扰。
