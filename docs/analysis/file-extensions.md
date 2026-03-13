# OpenClaw 项目文件后缀分析

生成日期：2026-03-10

## 统计概览

| 类别                | 主要后缀                          | 文件数量（不含 node_modules） |
| ------------------- | --------------------------------- | ----------------------------- |
| TypeScript 核心代码 | `.ts`, `.mts`, `.cts`             | ~7,240+                       |
| JavaScript 构建产物 | `.js`, `.mjs`, `.cjs`             | ~1,000+                       |
| Swift 原生代码      | `.swift`                          | 567                           |
| 文档                | `.md`, `.mdx`                     | 817                           |
| Kotlin Android 代码 | `.kt`, `.kts`                     | 117                           |
| 配置文件            | `.json`, `.yml`, `.yaml`, `.toml` | 175+                          |
| Shell 脚本          | `.sh`                             | 58                            |
| 图片资源            | `.png`, `.jpg`, `.svg`            | 100+                          |

---

## 核心代码文件

| 后缀     | 说明                 | 用途                                           |
| -------- | -------------------- | ---------------------------------------------- |
| `.ts`    | TypeScript           | 项目主要编程语言，带类型系统的 JavaScript 超集 |
| `.mts`   | ES Module TypeScript | 使用 ES6 模块系统的 TS 文件                    |
| `.cts`   | CommonJS TypeScript  | 使用 CommonJS 模块系统的 TS 文件               |
| `.js`    | JavaScript           | 标准脚本文件，部分为构建输出                   |
| `.mjs`   | ES Module JavaScript | 使用 ES6 模块系统的 JS 文件                    |
| `.cjs`   | CommonJS JavaScript  | 使用 CommonJS 模块系统的 JS 文件               |
| `.swift` | Swift                | iOS/macOS 原生应用开发语言                     |
| `.kt`    | Kotlin               | Android 应用开发语言                           |
| `.kts`   | Kotlin Script        | Kotlin 脚本文件                                |
| `.go`    | Go                   | Google 开发的系统编程语言                      |
| `.py`    | Python               | 脚本和工具文件                                 |

---

## 构建产物和资源

| 后缀           | 说明                  | 用途                             |
| -------------- | --------------------- | -------------------------------- |
| `.map`         | Source Map            | 源代码映射，用于调试构建后的代码 |
| `.tsbuildinfo` | TypeScript Build Info | TypeScript 增量编译缓存信息      |
| `.plist`       | Property List         | Apple 平台的 XML 格式配置文件    |

---

## 文档文件

| 后缀     | 说明           | 用途                                       |
| -------- | -------------- | ------------------------------------------ |
| `.md`    | Markdown       | 文档文件（README、指南、规范等）           |
| `.mdx`   | Markdown + JSX | 支持 React 组件的 Markdown，用于交互式文档 |
| `.prose` | ProseMirror    | 富文本编辑器内容格式                       |
| `.txt`   | Plain Text     | 纯文本文件                                 |

---

## 配置文件

| 后缀                        | 说明                     | 用途                                       |
| --------------------------- | ------------------------ | ------------------------------------------ |
| `.json`                     | JSON                     | 配置文件、包定义 (package.json)、数据交换  |
| `.jsonc`                    | JSON with Comments       | 支持注释的 JSON 配置                       |
| `.jsonl`                    | JSON Lines               | 每行一个 JSON 对象的数据文件               |
| `.yml`                      | YAML                     | CI/CD、Docker Compose 等配置文件           |
| `.yaml`                     | YAML                     | 同上，不同扩展名                           |
| `.toml`                     | TOML                     | 配置文件格式（如 Rust Cargo、Python 配置） |
| `.xml`                      | XML                      | 配置文件、Android 布局、manifest 等        |
| `.properties`               | Java Properties          | Java 配置文件（Android 项目）              |
| `.npmrc`                    | NPM Config               | npm 包管理器配置                           |
| `.editorconfig`             | EditorConfig             | 代码编辑器统一配置                         |
| `.eslintrc`                 | ESLint Config            | JavaScript/TypeScript 代码检查配置         |
| `.nycrc`                    | NYC Config               | 代码覆盖率工具配置                         |
| `.gitignore`                | Git Ignore               | 指定 Git 不跟踪的文件模式                  |
| `.openapi-generator-ignore` | OpenAPI Generator Ignore | OpenAPI 代码生成忽略规则                   |

---

## 样式和界面

| 后缀    | 说明                   | 用途             |
| ------- | ---------------------- | ---------------- |
| `.css`  | Cascading Style Sheets | 网页样式文件     |
| `.scss` | Sass CSS               | CSS 预处理器文件 |

---

## 图片和媒体资源

| 后缀             | 说明            | 用途                             |
| ---------------- | --------------- | -------------------------------- |
| `.png`           | PNG             | 无损压缩图片，用于图标和界面元素 |
| `.jpg` / `.jpeg` | JPEG            | 有损压缩照片格式                 |
| `.svg`           | SVG             | 矢量图形，用于图标和可缩放图形   |
| `.ttf`           | TrueType Font   | 字体文件                         |
| `.pfb`           | PostScript Font | Adobe PostScript 字体文件        |
| `.bcmap`         | ByteCode Map    | 字体字节码映射文件               |
| `.ico`           | Icon            | Windows/网站图标文件             |

---

## 脚本和自动化

| 后缀      | 说明         | 用途                              |
| --------- | ------------ | --------------------------------- |
| `.sh`     | Shell Script | Bash/Zsh 脚本文件                 |
| `.coffee` | CoffeeScript | CoffeeScript 脚本（部分依赖使用） |
| `.lua`    | Lua          | Lua 脚本（部分依赖使用）          |

---

## iOS/macOS 开发特有

| 后缀           | 说明                | 用途                        |
| -------------- | ------------------- | --------------------------- |
| `.xcconfig`    | Xcode Configuration | Xcode 构建设置文件          |
| `.xcfilelist`  | Xcode File List     | Xcode 构建输入/输出文件列表 |
| `.swiftformat` | SwiftFormat Config  | Swift 代码格式化配置        |
| `.xcworkspace` | Xcode Workspace     | Xcode 工作区配置            |
| `.xcodeproj`   | Xcode Project       | Xcode 项目文件（目录）      |
| `.storyboard`  | iOS Storyboard      | iOS 界面 storyboard 文件    |
| `.xcassets`    | Xcode Assets        | Xcode 资源目录              |

---

## Android 开发特有

| 后缀                      | 说明         | 用途                 |
| ------------------------- | ------------ | -------------------- |
| `.gradle` / `.gradle.kts` | Gradle Build | Android 构建配置文件 |
| `.pro`                    | ProGuard     | Android 代码混淆配置 |

---

## 数据和示例

| 后缀       | 说明    | 用途               |
| ---------- | ------- | ------------------ |
| `.sample`  | Sample  | 示例配置或数据文件 |
| `.example` | Example | 配置模板示例       |

---

## 压缩和归档

| 后缀   | 说明     | 用途            |
| ------ | -------- | --------------- |
| `.tar` | TAR      | 打包归档文件    |
| `.tgz` | Gzip TAR | 压缩的 TAR 归档 |
| `.zip` | ZIP      | 压缩归档文件    |

---

## 其他特殊文件

| 后缀            | 说明                   | 用途                             |
| --------------- | ---------------------- | -------------------------------- |
| `.node`         | Node.js Native Module  | 编译后的 C/C++ Node.js 原生模块  |
| `.resolved`     | Swift Package Resolved | Swift Package Manager 依赖锁文件 |
| `.hash`         | Hash File              | 文件完整性校验哈希值             |
| `.service`      | systemd Service        | Linux systemd 服务单元文件       |
| `.timer`        | systemd Timer          | Linux systemd 定时器单元文件     |
| `.sum`          | Go Sum                 | Go module 依赖校验和文件         |
| `.go.sum`       | Go Sum                 | Go module 校验和文件             |
| `.shellcheckrc` | ShellCheck Config      | Shell 脚本静态分析配置           |
| `.in`           | Input Template         | 输入模板文件（如 configure.in）  |
| `.sample`       | Sample                 | 示例文件                         |

---

## 项目架构总结

OpenClaw 是一个**全平台消息聚合应用**，技术栈包括：

### 后端/核心层

- **TypeScript** (7,240+ 文件) - 核心业务逻辑、CLI 工具、网关服务
- **Node.js** - 运行时环境

### 桌面端

- **Swift** (567 文件) - macOS 原生应用（菜单栏应用）

### 移动端

- **Swift** - iOS 原生应用
- **Kotlin** (113 文件) - Android 原生应用

### 其他语言

- **Go** - 部分系统组件
- **Python** - 工具和脚本

### 文档体系

- **Markdown** (811 文件) - 用户文档、开发文档、API 文档
- **MDX** (6 文件) - 交互式文档

### 配置管理

- **JSON/YAML/TOML** - 各类配置文件
- **Shell 脚本** (58 文件) - 构建、部署、维护脚本
