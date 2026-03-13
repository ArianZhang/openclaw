# OpenClaw 项目依赖分析

生成日期：2026-03-10

---

## 一、项目结构概览

```
openclaw/
├── package.json              # 主项目（核心网关 + CLI）
├── packages/                 # 内部包
│   ├── clawdbot/             # 兼容性包（转发到 openclaw）
│   └── moltbot/              # 兼容性包（转发到 openclaw）
├── extensions/               # 插件/扩展（33 个）
│   ├── channels/             # 消息渠道插件
│   ├── providers/            # AI 提供商插件
│   ├── skills/               # 技能插件
│   └── tools/                # 工具插件
├── ui/                       # Web UI 前端
└── vendor/                   # 第三方代码（a2ui）
```

---

## 二、主项目依赖 (package.json)

### 2.1 核心依赖 (Dependencies)

| 包名                              | 版本                      | 用途                      |
| --------------------------------- | ------------------------- | ------------------------- |
| `@agentclientprotocol/sdk`        | 0.14.1                    | Agent Client Protocol SDK |
| `@aws-sdk/client-bedrock`         | ^3.1000.0                 | AWS Bedrock AI 服务客户端 |
| `@buape/carbon`                   | 0.0.0-beta-20260216184201 | 代码分析/死代码检测       |
| `@clack/prompts`                  | ^1.0.1                    | 命令行交互式提示          |
| `@discordjs/voice`                | ^0.19.0                   | Discord 语音功能          |
| `@grammyjs/runner`                | ^2.0.3                    | Telegram Bot Grammy 框架  |
| `@grammyjs/transformer-throttler` | ^1.2.1                    | Telegram 速率限制         |
| `@homebridge/ciao`                | ^1.3.5                    | HomeKit/Zeroconf 服务发现 |
| `@line/bot-sdk`                   | ^10.6.0                   | LINE Bot SDK              |
| `@lydell/node-pty`                | 1.2.0-beta.3              | 伪终端 (PTY) 支持         |
| `@mariozechner/pi-agent-core`     | 0.55.3                    | Pi Agent 核心框架         |
| `@mariozechner/pi-ai`             | 0.55.3                    | Pi AI 集成                |
| `@mariozechner/pi-coding-agent`   | 0.55.3                    | Pi 编程助手               |
| `@mariozechner/pi-tui`            | 0.55.3                    | Pi 终端界面               |
| `@mozilla/readability`            | ^0.6.0                    | 网页内容提取              |
| `@sinclair/typebox`               | 0.34.48                   | JSON Schema 类型定义      |
| `@slack/bolt`                     | ^4.6.0                    | Slack Bolt 框架           |
| `@slack/web-api`                  | ^7.14.1                   | Slack Web API             |
| `@snazzah/davey`                  | ^0.1.9                    | Discord 语音队列          |
| `@whiskeysockets/baileys`         | 7.0.0-rc.9                | WhatsApp Web API          |
| `ajv`                             | ^8.18.0                   | JSON Schema 验证器        |
| `chalk`                           | ^5.6.2                    | 终端彩色输出              |
| `chokidar`                        | ^5.0.0                    | 文件监听                  |
| `cli-highlight`                   | ^2.1.11                   | CLI 语法高亮              |
| `commander`                       | ^14.0.3                   | 命令行参数解析            |
| `croner`                          | ^10.0.1                   | Cron 定时任务             |
| `discord-api-types`               | ^0.38.40                  | Discord API 类型          |
| `dotenv`                          | ^17.3.1                   | 环境变量加载              |
| `express`                         | ^5.2.1                    | Web 服务器框架            |
| `file-type`                       | ^21.3.0                   | 文件类型检测              |
| `gaxios`                          | 7.1.3                     | Google HTTP 客户端        |
| `grammy`                          | ^1.41.0                   | Telegram Bot 框架         |
| `https-proxy-agent`               | ^7.0.6                    | HTTPS 代理支持            |
| `ipaddr.js`                       | ^2.3.0                    | IP 地址处理               |
| `jiti`                            | ^2.6.1                    | TypeScript/ESM 运行时加载 |
| `json5`                           | ^2.2.3                    | JSON5 解析                |
| `jszip`                           | ^3.10.1                   | ZIP 文件处理              |
| `linkedom`                        | ^0.18.12                  | 轻量级 DOM 解析           |
| `long`                            | ^5.3.2                    | 大整数支持                |
| `markdown-it`                     | ^14.1.1                   | Markdown 解析             |
| `node-edge-tts`                   | ^1.2.10                   | Azure TTS 语音合成        |
| `opusscript`                      | ^0.1.1                    | Opus 音频编码             |
| `osc-progress`                    | ^0.3.0                    | 进度条组件                |
| `pdfjs-dist`                      | ^5.5.207                  | PDF.js 渲染               |
| `playwright-core`                 | 1.58.2                    | 浏览器自动化              |
| `qrcode-terminal`                 | ^0.12.0                   | 终端二维码生成            |
| `sharp`                           | ^0.34.5                   | 图片处理                  |
| `sqlite-vec`                      | 0.1.7-alpha.2             | SQLite 向量扩展           |
| `strip-ansi`                      | ^7.2.0                    | ANSI 转义码移除           |
| `tar`                             | 7.5.9                     | TAR 归档                  |
| `tslog`                           | ^4.10.2                   | TypeScript 日志           |
| `undici`                          | ^7.22.0                   | HTTP 客户端               |
| `ws`                              | ^8.19.0                   | WebSocket 支持            |
| `yaml`                            | ^2.8.2                    | YAML 解析                 |
| `zod`                             | ^4.3.6                    | TypeScript 验证库         |

### 2.2 开发依赖 (DevDependencies)

| 包名                         | 版本                 | 用途                |
| ---------------------------- | -------------------- | ------------------- |
| `@grammyjs/types`            | ^3.25.0              | Grammy 类型定义     |
| `@lit-labs/signals`          | ^0.2.0               | Lit 响应式信号      |
| `@lit/context`               | ^1.1.6               | Lit 上下文注入      |
| `@types/express`             | ^5.0.6               | Express 类型        |
| `@types/markdown-it`         | ^14.1.2              | Markdown-it 类型    |
| `@types/node`                | ^25.3.3              | Node.js 类型        |
| `@types/qrcode-terminal`     | ^0.12.2              | 二维码类型          |
| `@types/ws`                  | ^8.18.1              | WebSocket 类型      |
| `@typescript/native-preview` | 7.0.0-dev.20260301.1 | TypeScript 原生预览 |
| `@vitest/coverage-v8`        | ^4.0.18              | Vitest 覆盖率       |
| `lit`                        | ^3.3.2               | Web 组件框架        |
| `oxfmt`                      | 0.35.0               | Ox 代码格式化       |
| `oxlint`                     | ^1.50.0              | Ox 代码检查         |
| `oxlint-tsgolint`            | ^0.15.0              | TypeScript 检查     |
| `signal-utils`               | 0.21.1               | 信号工具库          |
| `tsdown`                     | 0.21.0-beta.2        | TypeScript 打包器   |
| `tsx`                        | ^4.21.0              | TypeScript 执行器   |
| `typescript`                 | ^5.9.3               | TypeScript 编译器   |
| `vitest`                     | ^4.0.18              | 测试框架            |

### 2.3 对等依赖 (PeerDependencies)

| 包名              | 版本    | 用途                   |
| ----------------- | ------- | ---------------------- |
| `@napi-rs/canvas` | ^0.1.89 | Node.js Canvas (可选)  |
| `node-llama-cpp`  | 3.16.2  | 本地 LLaMA 推理 (可选) |

### 2.4 可选依赖 (OptionalDependencies)

| 包名              | 版本    | 用途              |
| ----------------- | ------- | ----------------- |
| `@discordjs/opus` | ^0.10.0 | Discord Opus 音频 |

---

## 三、扩展插件依赖

### 3.1 消息渠道插件 (Channel Extensions)

| 插件                | 依赖包                                                                                |
| ------------------- | ------------------------------------------------------------------------------------- |
| **Discord**         | (无额外依赖)                                                                          |
| **Telegram**        | (无额外依赖)                                                                          |
| **Slack**           | (无额外依赖)                                                                          |
| **WhatsApp**        | (无额外依赖)                                                                          |
| **Signal**          | (无额外依赖)                                                                          |
| **Matrix**          | `@matrix-org/matrix-sdk-crypto-nodejs`, `@vector-im/matrix-bot-sdk`, `music-metadata` |
| **LINE**            | `@line/bot-sdk` (已在主项目)                                                          |
| **Twitch**          | `@twurple/api`, `@twurple/auth`, `@twurple/chat`                                      |
| **IRC**             | (无额外依赖)                                                                          |
| **Feishu/Lark**     | `@larksuiteoapi/node-sdk`                                                             |
| **Google Chat**     | `google-auth-library`                                                                 |
| **Microsoft Teams** | `@microsoft/agents-hosting`, `express`                                                |
| **Zalo**            | `undici`                                                                              |
| **Nextcloud Talk**  | (无额外依赖)                                                                          |
| **Synology Chat**   | (无额外依赖)                                                                          |
| **Nostr**           | `nostr-tools`                                                                         |
| **Tlon/Urbit**      | `@tloncorp/api`, `@tloncorp/tlon-skill`, `@urbit/aura`, `@urbit/http-api`             |
| **BlueBubbles**     | (无额外依赖)                                                                          |
| **iMessage**        | (无额外依赖)                                                                          |

### 3.2 技能/工具插件 (Skill/Tool Extensions)

| 插件                 | 依赖包                                        |
| -------------------- | --------------------------------------------- |
| **Voice Call**       | `@sinclair/typebox`, `commander`, `ws`, `zod` |
| **Memory Core**      | (peer: `openclaw`)                            |
| **Memory LanceDB**   | `@lancedb/lancedb`, `openai`                  |
| **LLM Task**         | `@sinclair/typebox`, `ajv`                    |
| **Lobster**          | `@sinclair/typebox`                           |
| **Diffs**            | `@pierre/diffs`, `playwright-core`            |
| **OpenProse**        | (无额外依赖)                                  |
| **ACPx**             | `acpx`                                        |
| **Diagnostics OTel** | `@opentelemetry/*` (全套 OTel 库)             |
| **Copilot Proxy**    | (无额外依赖)                                  |
| **Gemini Auth**      | (无额外依赖)                                  |
| **Minimax Auth**     | (无额外依赖)                                  |

---

## 四、UI 项目依赖 (ui/package.json)

### 4.1 运行时依赖

| 包名                | 版本    | 用途           |
| ------------------- | ------- | -------------- |
| `@lit-labs/signals` | ^0.2.0  | Lit 响应式信号 |
| `@lit/context`      | ^1.1.6  | Lit 上下文     |
| `@noble/ed25519`    | 3.0.0   | Ed25519 加密   |
| `dompurify`         | ^3.3.1  | XSS 防护       |
| `lit`               | ^3.3.2  | Web 组件框架   |
| `marked`            | ^17.0.3 | Markdown 解析  |
| `signal-polyfill`   | ^0.2.2  | 信号 API 填充  |
| `signal-utils`      | ^0.21.1 | 信号工具       |
| `vite`              | 7.3.1   | 构建工具       |

### 4.2 开发依赖

| 包名                         | 版本    | 用途         |
| ---------------------------- | ------- | ------------ |
| `@vitest/browser-playwright` | 4.0.18  | 浏览器测试   |
| `playwright`                 | ^1.58.2 | 浏览器自动化 |
| `vitest`                     | 4.0.18  | 测试框架     |

---

## 五、内部包 (packages/)

### 5.1 clawdbot

- **用途**: 兼容性包，转发到 openclaw
- **依赖**: `openclaw: workspace:*`

### 5.2 moltbot

- **用途**: 兼容性包，转发到 openclaw
- **依赖**: `openclaw: workspace:*`

---

## 六、依赖分类汇总

### 6.1 按功能分类

#### AI/LLM 相关

- `@mariozechner/pi-*` 系列 (Pi Agent 框架)
- `@aws-sdk/client-bedrock` (AWS Bedrock)
- `openai` (OpenAI API)
- `node-llama-cpp` (本地推理)

#### 消息渠道 SDK

- `@slack/bolt`, `@slack/web-api` (Slack)
- `grammy`, `@grammyjs/*` (Telegram)
- `@whiskeysockets/baileys` (WhatsApp)
- `@line/bot-sdk` (LINE)
- `@matrix-org/*`, `@vector-im/matrix-bot-sdk` (Matrix)
- `@twurple/*` (Twitch)
- `@larksuiteoapi/node-sdk` (飞书)
- `google-auth-library` (Google)
- `@microsoft/agents-hosting` (Teams)
- `nostr-tools` (Nostr)
- `@tloncorp/*`, `@urbit/*` (Urbit)

#### 网络/HTTP

- `express` (Web 服务器)
- `undici` (HTTP 客户端)
- `gaxios` (Google HTTP)
- `https-proxy-agent` (代理)
- `ws` (WebSocket)

#### 数据处理

- `@sinclair/typebox` (类型定义)
- `zod` (验证)
- `ajv` (JSON Schema)
- `yaml` (YAML)
- `json5` (JSON5)
- `jszip` (ZIP)
- `tar` (TAR)

#### 多媒体

- `sharp` (图片处理)
- `@discordjs/voice`, `@discordjs/opus` (音频)
- `opusscript` (Opus 编码)
- `node-edge-tts` (TTS)
- `pdfjs-dist` (PDF)
- `file-type` (文件检测)

#### 数据库/存储

- `sqlite-vec` (向量 SQLite)
- `@lancedb/lancedb` (LanceDB)

#### 开发工具

- `typescript` (编译器)
- `vitest` (测试)
- `tsdown` (打包)
- `tsx` (执行器)
- `oxlint`, `oxfmt` (检查/格式化)
- `@vitest/coverage-v8` (覆盖率)

#### 监控/可观测性

- `@opentelemetry/*` 全套 (OTel)
- `tslog` (日志)

#### 终端/CLI

- `commander` (参数解析)
- `@clack/prompts` (交互)
- `chalk` (颜色)
- `cli-highlight` (高亮)
- `osc-progress` (进度条)
- `qrcode-terminal` (二维码)
- `@lydell/node-pty` (PTY)

---

## 七、pnpm 配置

### 7.1 覆盖配置 (overrides)

用于安全修复或版本统一：

- `hono`: 4.11.10
- `fast-xml-parser`: 5.3.6
- `request`: `@cypress/request@3.0.10` (安全替代)
- `request-promise`: `@cypress/request-promise@5.0.0`
- `form-data`: 2.5.4
- `minimatch`: 10.2.4
- `qs`: 6.14.2
- `tar`: 7.5.9
- `tough-cookie`: 4.1.3

### 7.2 仅构建依赖 (onlyBuiltDependencies)

需要原生编译的包：

- `@lydell/node-pty` (PTY)
- `@matrix-org/matrix-sdk-crypto-nodejs` (Matrix 加密)
- `@napi-rs/canvas` (Canvas)
- `@whiskeysockets/baileys` (WhatsApp)
- `authenticate-pam` (PAM 认证)
- `esbuild` (打包器)
- `koffi` (FFI)
- `node-llama-cpp` (LLaMA)
- `protobufjs` (Protobuf)
- `sharp` (图片处理)

---

## 八、依赖统计

| 类别                        | 数量 |
| --------------------------- | ---- |
| 主项目 dependencies         | 57   |
| 主项目 devDependencies      | 20   |
| 主项目 peerDependencies     | 2    |
| 主项目 optionalDependencies | 1    |
| 扩展插件 (总计)             | 33   |
| - 消息渠道插件              | ~18  |
| - 技能/工具插件             | ~12  |
| - 认证插件                  | 3    |
| UI 项目依赖                 | 9    |
| UI 开发依赖                 | 3    |
| 内部包                      | 2    |

---

## 九、架构特点

1. **插件化架构**: 核心 + 扩展模式，支持按需安装
2. **多渠道支持**: 覆盖主流消息平台 (Slack/Telegram/WhatsApp/Discord 等)
3. **AI 集成**: 支持多模型提供商 (Anthropic/OpenAI/AWS Bedrock/本地 LLaMA)
4. **TypeScript 优先**: 全栈 TS，严格类型检查
5. **模块化设计**: workspace 多包管理，依赖隔离
6. **安全优先**: 依赖覆盖修复已知漏洞
