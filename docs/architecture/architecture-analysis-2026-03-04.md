# OpenClaw 项目架构详细分析报告

**版本**: 2026-03-04
**生成时间**: 2026-03-04
**作者**: Claude (qwen3.5-plus)
**目的**: 为后续架构升级和重构提供完整的基准参考

---

## 一、项目概述

**OpenClaw** 是一个个人 AI 助手平台，采用**本地优先 (local-first)** 架构。用户可以通过已有的通讯渠道（WhatsApp、Telegram、Slack、Discord 等）与 AI 交互，同时支持 macOS/iOS/Android 多端应用。

**核心理念**: Gateway 作为控制平面 (control plane)，统一管理所有通讯渠道、会话、工具和事件。

**产品定位**:

- 个人单用户助手
- 本地运行，隐私友好
- 多渠道统一入口
- 可扩展的插件系统
- AI 驱动的自动化

**官方资源**:

- 网站: https://openclaw.ai
- 文档: https://docs.openclaw.ai
- GitHub: https://github.com/openclaw/openclaw
- Discord: https://discord.gg/clawd

---

## 二、整体架构层次

```
┌─────────────────────────────────────────────────────────────────┐
│                    客户端层 (Clients)                            │
│   macOS App │ iOS App │ Android App │ CLI │ WebChat UI         │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼ WebSocket (127.0.0.1:18789)
┌─────────────────────────────────────────────────────────────────┐
│                    Gateway (控制平面)                            │
│  ┌─────────────┬─────────────┬─────────────┬─────────────────┐ │
│  │  会话管理   │  渠道管理   │  工具系统   │   配置系统      │ │
│  └─────────────┴─────────────┴─────────────┴─────────────────┘ │
│  ┌─────────────┬─────────────┬─────────────┬─────────────────┐ │
│  │  记忆系统   │  调度系统   │  认证系统   │   事件系统      │ │
│  └─────────────┴─────────────┴─────────────┴─────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    渠道适配层 (Channels)                         │
│ WhatsApp│Telegram│Slack│Discord│Signal│iMessage│Google Chat... │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    AI Provider 层                                │
│  Anthropic │ OpenAI │ Google │ 本地模型 │ 其他 Provider        │
└─────────────────────────────────────────────────────────────────┘
```

### 2.1 架构核心原则

1. **单一 Gateway 实例** - 每个主机运行一个 Gateway 进程，控制所有渠道会话
2. **WebSocket 中心** - 所有客户端通过 WS 协议与 Gateway 通信
3. **插件化扩展** - 渠道/工具/Provider 均可通过插件扩展
4. **本地存储优先** - 会话/配置/记忆数据存储在本地
5. **类型安全** - TypeBox schema 定义协议，TypeScript 严格模式

---

## 三、完整目录结构

### 3.1 根目录结构

```
openclaw/
├── src/                          # 核心源代码
├── extensions/                   # 插件/扩展目录
├── apps/                         # 客户端应用
│   ├── macos/                    # macOS 应用
│   ├── ios/                      # iOS 应用
│   ├── android/                  # Android 应用
│   └── shared/                   # 共享代码
├── ui/                           # Web UI
│   ├── src/
│   │   └── ui/                   # UI 组件
│   └── public/
├── docs/                         # 文档
├── test/                         # 测试资源
├── dist/                         # 构建输出
├── scripts/                      # 构建/部署脚本
├── git-hooks/                    # Git 钩子
├── patches/                      # npm 补丁
├── .agents/                      # Agent 配置
└── package.json                  # 项目配置
```

### 3.2 源代码目录 (`src/`) 详细结构

| 目录           | 文件数   | 职责               | 核心文件                                                       |
| -------------- | -------- | ------------------ | -------------------------------------------------------------- |
| `gateway/`     | ~217     | WebSocket 网关核心 | `client.ts`, `auth.ts`, `boot.ts`, `server-methods/`           |
| `channels/`    | ~56 dirs | 通讯渠道适配器     | `registry.ts`, `dock.ts`, `typing.ts`, `session.ts`            |
| `agents/`      | ~486     | AI Agent 运行时    | `agent.ts`, `agent-scope.ts`, `acp-spawn.ts`, `auth-profiles/` |
| `commands/`    | ~268     | CLI 命令实现       | `agent.ts`, `send.ts`, `auth/`, `gateway/`, `config/`          |
| `config/`      | ~196     | 配置系统           | `config.ts`, `types.ts`, `validation.ts`, `zod-schema.*`       |
| `infra/`       | ~288     | 基础设施           | `ports.ts`, `fs-safe.ts`, `http-body.ts`, `device-pairing.ts`  |
| `cli/`         | ~161     | CLI 框架           | `program/`, `deps.ts`, `progress.ts`, `prompt.ts`              |
| `routing/`     | 12       | 消息路由           | `resolve-route.ts`, `session-key.ts`, `account-id.ts`          |
| `memory/`      | ~91      | 记忆系统           | `manager.ts`, `qmd-manager.ts`, `embeddings.ts`, `mmr.ts`      |
| `media/`       | ~40      | 媒体处理           | `mime.ts`, `store.ts`, `capture/`, `transcode/`                |
| `auto-reply/`  | ~65      | 自动回复           | `reply.ts`, `templating.ts`, `skill-commands.ts`               |
| `providers/`   | 13 dirs  | AI Provider        | `anthropic/`, `openai/`, `google/`, `azure/`                   |
| `hooks/`       | ~38      | 钩子系统           | `types.ts`, `registry.ts`, `executor.ts`                       |
| `cron/`        | ~69      | 定时任务           | `scheduler.ts`, `types.ts`, `runner.ts`                        |
| `plugins/`     | ~66      | 插件系统           | `registry.ts`, `types.ts`, `http-registry.ts`                  |
| `plugin-sdk/`  | ~56      | 插件 SDK           | `index.ts`, `types.ts`, `webhook-path.ts`                      |
| `acp/`         | 23 dirs  | Agent 协议         | `runtime/`, `types/`, `spawn.ts`                               |
| `discord/`     | ~40      | Discord 专用       | `accounts.ts`, `monitor/`, `message-actions.ts`                |
| `slack/`       | ~40      | Slack 专用         | `accounts.ts`, `message-actions.ts`, `threading.ts`            |
| `telegram/`    | ~40      | Telegram 专用      | `accounts.ts`, `probe.ts`, `outbound-params.ts`                |
| `signal/`      | ~30      | Signal 专用        | `accounts.ts`, `signal-cli/`                                   |
| `whatsapp/`    | ~30      | WhatsApp 专用      | `normalize.ts`, `resolve-outbound-target.ts`                   |
| `imessage/`    | ~30      | iMessage 专用      | `accounts.ts`, `target-parsing-helpers.ts`                     |
| `line/`        | ~30      | LINE 专用          | `accounts.ts`, `flex-templates.ts`, `markdown-to-line.ts`      |
| `web/`         | ~30      | Web/WhatsApp Web   | `accounts.ts`, `media.ts`, `monitor/`                          |
| `terminal/`    | ~20      | 终端 UI            | `table.ts`, `links.ts`, `palette.ts`, `ansi.ts`                |
| `logging/`     | ~15      | 日志系统           | `logger.ts`, `subsystem.ts`, `redact.ts`                       |
| `security/`    | ~10      | 安全检查           | `dm-policy-shared.ts`                                          |
| `process/`     | ~17      | 进程管理           | `exec.ts`, `exec-wrapper-resolution.ts`                        |
| `pairing/`     | 10       | 设备配对           | `pairing-challenge.ts`                                         |
| `browser/`     | ~124     | 浏览器控制         | `controller/`, `actions/`, `snapshots/`                        |
| `canvas-host/` | 8        | Canvas 渲染        | `a2ui/`, `snapshot.ts`                                         |
| `node-host/`   | 17       | 节点服务           |                                                                |

### 3.3 扩展插件目录 (`extensions/`)

| 扩展                      | 文件数 | 功能描述                        |
| ------------------------- | ------ | ------------------------------- |
| `acpx/`                   | ~8     | ACP 扩展协议实现                |
| `bluebubbles/`            | ~8     | BlueBubbles (iMessage) 渠道适配 |
| `copilot-proxy/`          | ~6     | GitHub Copilot 代理             |
| `device-pair/`            | ~4     | 设备配对服务                    |
| `diagnostics-otel/`       | ~7     | OpenTelemetry 诊断              |
| `diffs/`                  | ~11    | 差异比较工具                    |
| `discord/`                | ~6     | Discord 渠道扩展                |
| `feishu/`                 | ~8     | 飞书渠道适配                    |
| `google-gemini-cli-auth/` | ~8     | Google Gemini CLI 认证          |
| `googlechat/`             | ~7     | Google Chat 渠道                |
| `imessage/`               | ~6     | iMessage 渠道扩展               |
| `irc/`                    | ~7     | IRC 协议渠道                    |
| `line/`                   | ~6     | LINE 渠道扩展                   |
| `llm-task/`               | ~8     | LLM 任务管理                    |
| `lobster/`                | ~10    | (内部项目集成)                  |
| `matrix/`                 | ~8     | Matrix 协议渠道                 |
| `mattermost/`             | ~7     | Mattermost 渠道                 |
| `memory-core/`            | ~6     | 记忆核心接口                    |
| `memory-lancedb/`         | ~8     | LanceDB 记忆后端                |
| `minimax-portal-auth/`    | ~7     | MiniMax 认证                    |
| `msteams/`                | ~8     | Microsoft Teams 渠道            |
| `nextcloud-talk/`         | ~7     | Nextcloud Talk 渠道             |
| `nostr/`                  | ~10    | Nostr 去中心化社交              |
| `open-prose/`             | ~7     | OpenProse 集成                  |
| `phone-control/`          | ~5     | 手机控制功能                    |
| `qwen-portal-auth/`       | ~6     | Qwen 认证                       |
| `shared/`                 | ~4     | 共享工具                        |
| `signal/`                 | ~6     | Signal 渠道扩展                 |
| `slack/`                  | ~6     | Slack 渠道扩展                  |
| `synology-chat/`          | ~7     | Synology Chat 渠道              |
| `talk-voice/`             | ~4     | 语音对话功能                    |
| `telegram/`               | ~6     | Telegram 渠道扩展               |
| `test-utils/`             | ~5     | 测试工具                        |
| `thread-ownership/`       | ~5     | 线程所有权管理                  |
| `tlon/`                   | ~8     | Tlon (Urbit) 渠道               |
| `twitch/`                 | ~10    | Twitch 直播渠道                 |
| `voice-call/`             | ~9     | 语音通话功能                    |
| `whatsapp/`               | ~6     | WhatsApp 渠道扩展               |
| `zalo/`                   | ~9     | Zalo 越南社交                   |
| `zalouser/`               | ~9     | Zalo Personal 渠道              |

### 3.4 客户端应用 (`apps/`)

```
apps/
├── macos/
│   ├── Sources/
│   │   ├── OpenClaw/           # 主应用
│   │   ├── OpenClawCore/       # 核心逻辑
│   │   ├── OpenClawKit/        # 共享组件
│   │   ├── OpenClawProtocol/   # 协议模型 (Swift)
│   │   └── OpenClawUI/         # UI 组件
│   ├── Tests/
│   └── Resources/
├── ios/
│   ├── Sources/
│   │   ├── OpenClaw/
│   │   └── OpenClawCore/
│   └── Tests/
├── android/
│   ├── app/
│   │   ├── src/main/java/ai/openclaw/android/
│   │   └── src/main/res/
│   └── benchmark/
└── shared/
    └── OpenClawKit/
```

---

## 四、核心功能模块详解

### 4.1 Gateway 网关系统

**核心文件**:

- `src/gateway/client.ts` - WebSocket 客户端连接管理
- `src/gateway/auth.ts` - 认证与授权
- `src/gateway/boot.ts` - 启动检查流程
- `src/gateway/credentials.ts` - 凭证管理
- `src/gateway/server-methods/` - 服务器方法实现

**职责**:

1. **WebSocket 服务器** - 监听默认端口 18789
2. **客户端连接管理** - 处理 connect/disconnect
3. **设备认证** - token 验证、设备配对
4. **事件分发** - 广播 agent/chat/presence 事件
5. **请求路由** - 转发到对应处理方法
6. **健康检查** - 响应 health 请求

**支持的请求方法**:

```typescript
// 连接相关
connect        - 建立 WS 连接
disconnect     - 断开连接

// 状态查询
health         - 健康检查
status         - 渠道状态
channels       - 渠道列表

// 消息操作
send           - 发送消息
agent          - 运行 Agent 任务
message send   - CLI 消息发送

// 系统控制
shutdown       - 关闭网关
config reload  - 重载配置
presence       - 在线状态
```

**认证流程**:

```
1. 客户端发送 connect 请求
   {method:"connect", params:{deviceIdentity, auth:{token}}}

2. Gateway 验证 token (如配置了 OPENCLAW_GATEWAY_TOKEN)

3. 新设备需要配对批准
   - 生成 pairing code
   - 用户通过 CLI 批准：openclaw pairing approve <channel> <code>

4. 颁发 device token 用于后续连接
```

**依赖关系**:

- 依赖 `src/config/` 读取配置
- 依赖 `src/infra/ports.ts` 检查端口可用性
- 依赖 `src/infra/device-pairing.ts` 处理设备配对
- 依赖 `src/channels/` 获取渠道状态
- 依赖 `src/logging/` 记录日志

### 4.2 渠道系统 (Channels)

**核心文件**:

- `src/channels/registry.ts` - 渠道注册表
- `src/channels/dock.ts` - 渠道状态管理
- `src/channels/typing.ts` - 输入状态指示
- `src/channels/session.ts` - 会话记录
- `src/channels/plugins/types.ts` - 渠道插件接口

**支持的渠道** (19 种):

| 渠道            | 实现目录                     | SDK/协议                   | 认证方式        |
| --------------- | ---------------------------- | -------------------------- | --------------- |
| WhatsApp        | `src/web/`, `src/whatsapp/`  | Baileys (WhatsApp Web)     | QR Code         |
| Telegram        | `src/telegram/`              | grammY                     | Phone/Token     |
| Slack           | `src/slack/`                 | Bolt                       | OAuth/App Token |
| Discord         | `src/discord/`               | discord.js                 | Bot Token       |
| Google Chat     | `src/googlechat/`            | Chat API                   | OAuth 2.0       |
| Signal          | `src/signal/`                | signal-cli                 | Phone Number    |
| iMessage        | `src/imessage/`              | AppleScript/IMessageBridge | macOS Only      |
| BlueBubbles     | `extensions/bluebubbles/`    | BlueBubbles API            | API Key         |
| IRC             | `extensions/irc/`            | irc-framework              | Server Config   |
| Microsoft Teams | `extensions/msteams/`        | Graph API                  | OAuth 2.0       |
| Matrix          | `extensions/matrix/`         | matrix-js-sdk              | Access Token    |
| LINE            | `src/line/`                  | LINE Bot SDK               | Channel Secret  |
| Feishu          | `extensions/feishu/`         | Feishu Open API            | App ID/Secret   |
| Mattermost      | `extensions/mattermost/`     | Mattermost API             | Personal Token  |
| Nextcloud Talk  | `extensions/nextcloud-talk/` | Talk API                   | App Password    |
| Nostr           | `extensions/nostr/`          | NIP-01                     | Private Key     |
| Synology Chat   | `extensions/synology-chat/`  | Chat API                   | Webhook URL     |
| Tlon            | `extensions/tlon/`           | Urbit API                  | Ship Key        |
| Twitch          | `extensions/twitch/`         | Twitch API                 | OAuth Token     |
| Zalo            | `extensions/zalo/`           | Zalo API                   | Cookie/Token    |

**渠道插件接口定义** (`src/plugin-sdk/index.ts`):

```typescript
// 核心适配器接口
interface ChannelGatewayAdapter {
  connect(): Promise<void>;
  disconnect(): Promise<void>;
  getAccountStatus(): Promise<ChannelAccountSnapshot>;
}

interface ChannelMessagingAdapter {
  onMessage(callback: (msg: ChannelMessage) => void): void;
  sendMessage(target: string, content: MessageContent): Promise<void>;
}

interface ChannelOutboundAdapter {
  sendText(target: string, text: string): Promise<void>;
  sendMedia(target: string, media: MediaPayload): Promise<void>;
}

interface ChannelPairingAdapter {
  startPairing(): Promise<ChannelLoginWithQrStartResult>;
  approvePairing(code: string): Promise<void>;
  rejectPairing(code: string): Promise<void>;
}

interface ChannelStatusAdapter {
  getStatusSummary(): Promise<ChannelStatusSummary>;
  probe(): Promise<ProbeResult>;
}

interface ChannelThreadingAdapter {
  supportsThreading(): boolean;
  getThreadingContext(): Promise<ChannelThreadingContext>;
}

// ... 共 40+ 接口方法
```

**渠道配置结构**:

```typescript
interface ChannelConfig {
  enabled: boolean;
  accountId?: string; // 账户标识
  secret?: SecretInput; // 认证凭据
  allowFrom?: string[]; // 允许的用户列表
  dmPolicy?: "open" | "pairing" | "blocked";
  groupPolicy?: GroupPolicy;
  replyPrefix?: boolean;
  mentionGating?: boolean;
  // ... 渠道特定配置
}
```

**消息处理流程**:

```
1. 渠道收到消息 (webhook/polling)
   ↓
2. 渠道适配器解析消息
   ↓
3. 规范化为统一格式 (normalize)
   ↓
4. 检查 allowFrom/mentionGating 策略
   ↓
5. 生成会话键 (session key)
   ↓
6. 记录到会话存储
   ↓
7. 触发 agent 处理 (如配置自动回复)
   ↓
8. 格式化回复 (chunking/mention)
   ↓
9. 通过渠道发送
```

**依赖关系**:

- 依赖 `src/config/` 读取渠道配置
- 依赖 `src/routing/` 解析目标地址
- 依赖 `src/media/` 处理媒体附件
- 依赖 `src/infra/` 处理网络和文件
- 依赖 `src/plugins/` 加载插件渠道

### 4.3 Agent 系统

**核心文件**:

- `src/agents/agent.ts` - Agent 核心逻辑
- `src/agents/agent-scope.ts` - Agent 作用域管理
- `src/agents/acp-spawn.ts` - ACP 派发生成
- `src/commands/agent.ts` - CLI agent 命令
- `src/acp/runtime/` - ACP 运行时

**职责**:

1. **接收用户消息** - 从渠道或 CLI 获取输入
2. **构建上下文** - 会话历史、工具定义、系统提示
3. **调用 LLM Provider** - 发送请求并处理流式响应
4. **工具执行** - 解析并执行工具调用 (browser/canvas/nodes)
5. **响应格式化** - 适配渠道输出格式
6. **会话状态管理** - 更新会话历史、使用量统计

**Agent 运行时配置**:

```typescript
interface AgentConfig {
  defaults: {
    model: string; // 默认模型
    thinking: "low" | "high"; // 思考深度
    deliver: boolean; // 是否发送回复
    tools: string[]; // 启用的工具
  };
  sessions: {
    mode: "main" | "per-peer" | "per-group";
    queueMode: "parallel" | "sequential";
  };
}
```

**ACP (Agent Client Protocol)**:

```typescript
// ACP 运行时接口
interface AcpRuntime {
  prompt(input: AcpRuntimeTurnInput): AsyncIterable<AcpRuntimeEvent>;
  ensure(input: AcpRuntimeEnsureInput): Promise<void>;
  getStatus(): Promise<AcpRuntimeStatus>;
}

// ACP 事件类型
type AcpRuntimeEvent =
  | { type: "content"; delta: string }
  | { type: "tool-call"; name: string; input: unknown }
  | { type: "tool-result"; result: unknown }
  | { type: "complete"; summary: string };
```

**工具调用流程**:

```
1. LLM 返回 tool_call
   ↓
2. 解析工具名称和参数
   ↓
3. 查找工具实现 (内置/插件)
   ↓
4. 执行工具 (可能有权限检查)
   ↓
5. 返回工具结果给 LLM
   ↓
6. LLM 生成最终回复
```

**内置工具列表**:

| 工具               | 文件位置               | 功能              |
| ------------------ | ---------------------- | ----------------- |
| `browser.open`     | `src/browser/`         | 打开网页/执行操作 |
| `browser.snapshot` | `src/browser/`         | 获取页面截图      |
| `canvas.eval`      | `src/canvas-host/`     | 执行 Canvas 代码  |
| `canvas.reset`     | `src/canvas-host/`     | 重置 Canvas       |
| `camera.snap`      | `src/nodes/`           | 拍摄照片          |
| `camera.record`    | `src/nodes/`           | 录制视频          |
| `screen.record`    | `src/nodes/`           | 屏幕录制          |
| `location.get`     | `src/nodes/`           | 获取位置          |
| `notify`           | `src/nodes/`           | 发送通知          |
| `system.run`       | `src/process/`         | 执行命令          |
| `session.list`     | `src/agents/sessions/` | 列出会话          |
| `session.delete`   | `src/agents/sessions/` | 删除会话          |
| `memory.search`    | `src/memory/`          | 语义搜索          |

**依赖关系**:

- 依赖 `src/providers/` 调用 LLM
- 依赖 `src/tools/` 执行工具
- 依赖 `src/memory/` 获取上下文
- 依赖 `src/config/` 读取配置
- 依赖 `src/routing/` 解析会话
- 依赖 `src/gateway/` 发送响应

### 4.4 配置系统

**核心文件**:

- `src/config/config.ts` - 配置加载/保存
- `src/config/types.ts` - 类型定义
- `src/config/validation.ts` - 配置验证
- `src/config/zod-schema.*.ts` - Zod Schema 定义
- `src/config/sessions/` - 会话存储

**配置文件位置**:

- 全局配置: `~/.openclaw/config.json5`
- 渠道配置: `~/.openclaw/channels/<channel>/`
- 会话存储: `~/.openclaw/sessions/<agent>/sessions.json`
- 凭证存储: `~/.openclaw/credentials/`

**配置结构** (简化):

```typescript
interface OpenClawConfig {
  // 全局设置
  gateway: {
    port: number;
    bind: string;
    token?: string;
  };

  // 模型配置
  models: {
    default: string;
    providers: {
      anthropic?: { apiKey?: SecretInput };
      openai?: { apiKey?: SecretInput };
      google?: { credentials?: SecretInput };
    };
    authProfiles: AuthProfile[];
  };

  // 渠道配置
  channels: {
    telegram?: TelegramConfig;
    whatsapp?: WhatsAppConfig;
    slack?: SlackConfig;
    discord?: DiscordConfig;
    // ...
  };

  // Agent 配置
  agents: {
    defaults: {
      model: string;
      tools: string[];
    };
    items?: AgentConfig[];
  };

  // 记忆配置
  memory: {
    enabled: boolean;
    backend: "sqlite" | "lancedb";
    embeddings: { provider: string; model: string };
  };

  // Cron 任务
  cron: {
    enabled: boolean;
    jobs: CronJobConfig[];
  };

  // Hooks
  hooks: {
    items: HookConfig[];
  };

  // 插件
  plugins: {
    items: PluginConfig[];
  };
}
```

**配置加载流程**:

```
1. 读取配置文件 (JSON5)
   ↓
2. 迁移旧版本配置 (如需要)
   ↓
3. Zod schema 验证
   ↓
4. 解析 SecretInput (环境变量/文件引用)
   ↓
5. 合并运行时覆盖
   ↓
6. 缓存到内存
```

**配置验证 Schema** (示例):

```typescript
const TelegramConfigSchema = z.object({
  enabled: z.boolean().default(true),
  accountId: z.string().optional(),
  secret: SecretInputSchema.optional(),
  allowFrom: z.array(z.string()).default([]),
  dmPolicy: z.enum(["open", "pairing", "blocked"]).default("pairing"),
  groupPolicy: GroupPolicySchema.optional(),
});
```

**依赖关系**:

- 依赖 `src/infra/fs-safe.ts` 安全文件读写
- 依赖 `src/config/sessions/` 管理会话存储
- 依赖 `zod` 进行运行时验证
- 依赖 `src/config/zod-schema.*` 定义 schema

### 4.5 记忆系统 (Memory)

**核心文件**:

- `src/memory/manager.ts` - 记忆管理器
- `src/memory/qmd-manager.ts` - QMD (Query Metadata) 管理
- `src/memory/embeddings.ts` - 嵌入生成
- `src/memory/mmr.ts` - MMR (Maximal Marginal Relevance)
- `src/memory/search-manager.ts` - 搜索管理

**功能**:

1. **文档索引** - 监听文件变化并建立索引
2. **向量嵌入** - 调用 Provider 生成嵌入向量
3. **语义搜索** - 基于向量相似度搜索
4. **MMR 重排序** - 平衡相关性和多样性
5. **QMD 管理** - 查询元数据缓存

**支持的嵌入 Provider**:

| Provider | 文件                    | 模型                         |
| -------- | ----------------------- | ---------------------------- |
| OpenAI   | `embeddings-openai.ts`  | text-embedding-3-small/large |
| Gemini   | `embeddings-gemini.ts`  | text-embedding-004           |
| Voyage   | `embeddings-voyage.ts`  | voyage-3/voyage-code-3       |
| Mistral  | `embeddings-mistral.ts` | mistral-embed                |
| Ollama   | `embeddings-ollama.ts`  | 本地模型                     |

**存储后端**:

- **SQLite + sqlite-vec** (默认) - 轻量级本地存储
- **LanceDB** (`extensions/memory-lancedb/`) - 向量数据库

**记忆管理流程**:

```
1. 文件监听 (chokidar)
   ↓
2. 检测文件变化
   ↓
3. 读取文件内容
   ↓
4. 分块 (chunking)
   ↓
5. 生成嵌入向量 (batch)
   ↓
6. 存储到向量数据库
   ↓
7. 搜索时计算相似度
   ↓
8. MMR 重排序
   ↓
9. 返回结果
```

**搜索 API**:

```typescript
interface MemoryManager {
  // 语义搜索
  search(query: string, options: SearchOptions): Promise<SearchResult[]>;

  // 添加文档
  addDocument(path: string, content: string): Promise<void>;

  // 删除文档
  removeDocument(path: string): Promise<void>;

  // 重新索引
  reindex(): Promise<void>;
}
```

**依赖关系**:

- 依赖 `src/providers/` 生成嵌入
- 依赖 `sqlite-vec` 或 `lancedb` 存储
- 依赖 `chokidar` 监听文件
- 依赖 `src/media/` 处理 PDF/图片

### 4.6 工具系统 (Tools)

**核心文件**:

- `src/agents/tools/` - 工具定义
- `src/browser/` - 浏览器控制
- `src/canvas-host/` - Canvas 渲染
- `src/nodes/` - 节点命令

**工具注册机制**:

```typescript
interface AnyAgentTool {
  name: string;
  description: string;
  inputSchema: TypeBox.TSchema;
  execute(context: ToolContext, input: unknown): Promise<ToolResult>;
}

// 工具注册表
const toolRegistry = new Map<string, AnyAgentTool>();

function registerTool(tool: AnyAgentTool): void;
function getTool(name: string): AnyAgentTool | undefined;
```

**浏览器工具详解** (`src/browser/`):

```typescript
// 浏览器控制器
interface BrowserController {
  // 导航
  open(url: string): Promise<void>;
  navigate(url: string): Promise<void>;
  back(): Promise<void>;
  forward(): Promise<void>;

  // 交互
  click(selector: string): Promise<void>;
  type(selector: string, text: string): Promise<void>;
  scroll(direction: "up" | "down"): Promise<void>;

  // 截图
  snapshot(): Promise<Buffer>;

  // 内容提取
  getContent(): Promise<string>;
  getLinks(): Promise<string[]>;
}

// 浏览器配置
interface BrowserConfig {
  profile?: string; // 用户配置文件
  headless: boolean;
  viewport: { width: number; height: number };
  proxy?: string;
}
```

**Canvas 工具详解** (`src/canvas-host/`):

```typescript
// Canvas API
interface CanvasHost {
  // 初始化
  reset(): Promise<void>;

  // 代码执行
  eval(code: string): Promise<CanvasResult>;

  // 状态
  snapshot(): Promise<CanvasSnapshot>;

  // A2UI (Agent-to-UI)
  a2ui: {
    bundle(): Promise<void>;
    serve(): void;
  };
}
```

**节点命令详解**:

| 命令            | 平台              | 功能             |
| --------------- | ----------------- | ---------------- |
| `camera.snap`   | macOS/iOS/Android | 拍摄照片         |
| `camera.record` | macOS/iOS/Android | 录制视频         |
| `screen.record` | macOS/iOS/Android | 屏幕录制         |
| `location.get`  | iOS/Android       | 获取位置         |
| `notify`        | macOS/iOS/Android | 发送通知         |
| `system.run`    | macOS/Linux/WSL   | 执行命令         |
| `android.*`     | Android           | Android 设备控制 |

**依赖关系**:

- 依赖 `src/process/` 执行系统命令
- 依赖 `src/media/` 处理媒体输出
- 依赖 `playwright-core` 浏览器自动化
- 依赖 `src/gateway/` 与节点通信

### 4.7 插件系统

**核心文件**:

- `src/plugins/registry.ts` - 插件注册表
- `src/plugins/types.ts` - 插件类型定义
- `src/plugins/http-registry.ts` - HTTP 路由注册
- `src/plugin-sdk/index.ts` - 插件 SDK 导出

**插件类型**:

1. **渠道插件** - 添加新的通讯渠道
2. **工具插件** - 添加新的 Agent 工具
3. **Provider 插件** - 添加新的 AI 模型
4. **服务插件** - 添加 HTTP 服务/Webhook

**插件接口**:

```typescript
interface OpenClawPluginService {
  // 插件元数据
  name: string;
  version: string;
  description: string;

  // 生命周期
  activate(ctx: OpenClawPluginServiceContext): Promise<void>;
  deactivate(): Promise<void>;

  // 配置
  getConfigSchema(): TypeBox.TSchema;
}

interface OpenClawPluginApi {
  // 注册 HTTP 路由
  registerHttpRoute(path: string, handler: GatewayRequestHandler): void;

  // 注册工具
  registerTool(tool: AnyAgentTool): void;

  // 注册渠道
  registerChannel(channelId: string, adapter: ChannelGatewayAdapter): void;

  // 日志
  logger: PluginLogger;

  // 配置访问
  config: OpenClawConfig;
}
```

**插件 SDK 导出** (`src/plugin-sdk/index.ts`):

```typescript
// 类型导出
export type {
  ChannelId,
  ChannelConfigAdapter,
  ChannelMessagingAdapter,
  ChannelOutboundAdapter,
  ChannelPairingAdapter,
  ChannelStatusAdapter,
  // ... 40+ 类型
} from "./types.js";

// 工具函数
export { normalizePluginHttpPath } from "./http-path.js";
export { registerPluginHttpRoute } from "./http-registry.js";
export { emptyPluginConfigSchema } from "./config-schema.js";

// 辅助函数
export {
  buildChannelConfigSchema,
  promptChannelAccessConfig,
  resolveAllowlistMatchSimple,
  // ...
} from "./helpers.js";
```

**插件加载流程**:

```
1. 扫描 extensions/ 目录
   ↓
2. 读取 package.json 检查 plugin 字段
   ↓
3. 安装依赖 (npm install --omit=dev)
   ↓
4. 通过 jiti 加载插件模块
   ↓
5. 调用 activate() 初始化
   ↓
6. 注册工具/渠道/路由
```

**依赖关系**:

- 依赖 `jiti` 动态加载 TypeScript
- 依赖 `src/gateway/` 注册 HTTP 路由
- 依赖 `src/channels/` 注册渠道
- 依赖 `src/agents/tools/` 注册工具

### 4.8 CLI 系统

**核心文件**:

- `src/cli/program.ts` - CLI 程序入口
- `src/cli/program/build-program.ts` - 命令树构建
- `src/cli/deps.ts` - 依赖注入
- `src/cli/prompt.ts` - 交互式提示
- `src/cli/progress.ts` - 进度条/Spinner

**命令结构**:

```
openclaw
├── gateway           # 启动网关
├── agent             # 运行 Agent
│   └── --message     # 消息内容
│   └── --thinking    # 思考深度
│   └── --session     # 会话 ID
├── send              # 发送消息
│   └── --to          # 目标地址
│   └── --message     # 消息内容
├── channels          # 渠道管理
│   ├── list          # 列出渠道
│   ├── status        # 渠道状态
│   ├── setup         # 设置渠道
│   └── probe         # 探测渠道
├── config            # 配置管理
│   ├── get           # 获取配置
│   ├── set           # 设置配置
│   └── edit          # 编辑配置
├── auth              # 认证管理
│   ├── login         # 登录
│   └── logout        # 登出
├── memory            # 记忆管理
│   ├── sync          # 同步文档
│   ├── search        # 搜索
│   └── status        # 状态
├── pairing           # 设备配对
│   ├── list          # 列出待批准
│   ├── approve       # 批准
│   └── reject        # 拒绝
├── onboard           # 引导向导
├── doctor            # 诊断检查
└── update            # 更新
    ├── check         # 检查更新
    └── run           # 执行更新
```

**CLI 依赖注入**:

```typescript
interface CliDeps {
  config: OpenClawConfig;
  runtime: RuntimeEnv;
  // 其他依赖
}

function createDefaultDeps(): CliDeps;
```

**依赖关系**:

- 依赖 `commander` 命令解析
- 依赖 `@clack/prompts` 交互式提示
- 依赖 `osc-progress` 进度显示
- 依赖 `src/config/` 读取配置
- 依赖 `src/gateway/` 执行命令

### 4.9 调度系统 (Cron)

**核心文件**:

- `src/cron/scheduler.ts` - 调度器
- `src/cron/runner.ts` - 执行器
- `src/cron/types.ts` - 类型定义

**功能**:

1. **定时任务** - 基于 cron 表达式调度
2. **Agent 触发** - 定时运行 Agent 任务
3. **Webhook 触发** - 定时调用 Webhook
4. **活跃时间** - 只在指定时间段运行

**Cron 配置**:

```typescript
interface CronJobConfig {
  id: string;
  name: string;
  schedule: string; // cron 表达式
  enabled: boolean;
  action: {
    type: "agent" | "webhook";
    message?: string; // agent 类型
    url?: string; // webhook 类型
  };
  activeHours?: {
    start: string; // "09:00"
    end: string; // "18:00"
    timezone?: string; // "Asia/Shanghai"
  };
}
```

**依赖关系**:

- 依赖 `croner` cron 表达式解析
- 依赖 `src/commands/agent.ts` 执行 agent 任务
- 依赖 `src/infra/fetch.ts` 发送 webhook

### 4.10 认证系统

**核心文件**:

- `src/gateway/auth.ts` - Gateway 认证
- `src/agents/auth-profiles/` - Provider 认证
- `src/pairing/` - 设备配对
- `src/infra/device-pairing.ts` - 设备配对存储

**认证类型**:

1. **Gateway 认证** - WebSocket 连接认证
2. **Provider 认证** - AI 模型 API 认证
3. **渠道认证** - 渠道 API 认证
4. **设备配对** - 新设备连接批准

**Provider 认证配置**:

```typescript
interface AuthProfile {
  id: string;
  provider: "anthropic" | "openai" | "google" | ...;
  authType: "api-key" | "oauth";
  credentials: {
    apiKey?: SecretInput;
    oauthToken?: SecretInput;
    // ...
  };
  models: string[];
  priority: number;           // 故障转移优先级
}
```

**设备配对流程**:

```
1. 新设备发起 connect
   ↓
2. Gateway 生成 pairing code
   ↓
3. 发送到已批准设备/渠道
   ↓
4. 用户批准：openclaw pairing approve <channel> <code>
   ↓
5. 存储设备信息到 pairing store
   ↓
6. 颁发 device token
   ↓
7. 设备使用 token 进行后续连接
```

**依赖关系**:

- 依赖 `src/infra/device-pairing.ts` 存储配对信息
- 依赖 `src/config/` 读取认证配置
- 依赖 `crypto` 生成 token
- 依赖 `src/channels/` 发送配对码

### 4.11 路由系统 (Routing)

**核心文件**:

- `src/routing/resolve-route.ts` - 路由解析
- `src/routing/session-key.ts` - 会话键生成
- `src/routing/account-id.ts` - 账户解析

**职责**:

1. **会话键生成** - 基于渠道/账户/对话生成唯一键
2. **路由决策** - 确定消息发往哪个 Agent
3. **账户解析** - 从配置解析默认账户

**会话键格式**:

```typescript
// 会话键结构
type SessionKey = string;

// 生成规则
// - main: 主会话 (直接消息)
// - group:<channel>:<groupId>: 群聊会话
// - agent:<agentId>:<channel>:<peerId>: Agent 专属会话
```

**路由解析流程**:

```
1. 接收渠道消息
   ↓
2. 提取渠道 ID、账户 ID、对话 ID
   ↓
3. 检查 Agent 绑定配置
   ↓
4. 生成会话键
   ↓
5. 加载/创建会话
   ↓
6. 路由到对应 Agent
```

**依赖关系**:

- 依赖 `src/config/` 读取 Agent 绑定
- 依赖 `src/channels/` 获取渠道信息
- 依赖 `src/agents/` 执行 Agent 任务

### 4.12 媒体系统 (Media)

**核心文件**:

- `src/media/mime.ts` - MIME 类型检测
- `src/media/store.ts` - 媒体存储
- `src/media/capture/` - 媒体捕获
- `src/media/transcode/` - 媒体转码

**功能**:

1. **媒体检测** - 自动检测文件类型
2. **大小限制** - 渠道特定的大小限制
3. **转码** - 图片/音频/视频格式转换
4. **临时存储** - 临时文件管理
5. **媒体理解** - 图片描述/音频转录

**媒体处理流程**:

```
1. 接收媒体文件
   ↓
2. 检测 MIME 类型
   ↓
3. 检查大小限制
   ↓
4. 转码 (如需要)
   ↓
5. 存储到临时目录
   ↓
6. 发送到渠道/LLM
   ↓
7. 清理临时文件
```

**依赖关系**:

- 依赖 `file-type` MIME 检测
- 依赖 `sharp` 图片处理
- 依赖 `@discordjs/opus` 音频处理
- 依赖 `pdfjs-dist` PDF 处理
- 依赖 `src/infra/fs-safe.ts` 安全文件操作

### 4.13 自动回复系统 (Auto-reply)

**核心文件**:

- `src/auto-reply/reply.ts` - 回复生成
- `src/auto-reply/templating.ts` - 模板引擎
- `src/auto-reply/skill-commands.ts` - 技能命令
- `src/auto-reply/history.ts` - 历史管理

**功能**:

1. **模板渲染** - 基于配置的模板生成回复
2. **技能命令** - 预定义命令处理
3. **历史管理** - 会话历史跟踪
4. **分块输出** - 适配渠道消息长度限制

**回复配置**:

```typescript
interface ReplyConfig {
  enabled: boolean;
  templates: {
    greeting?: string;
    fallback?: string;
    // ...
  };
  skills: {
    enabled: string[];
    commands: SkillCommand[];
  };
  history: {
    limit: number;
    mode: "per-peer" | "global";
  };
}
```

**依赖关系**:

- 依赖 `src/agents/` 执行 Agent 任务
- 依赖 `src/channels/` 发送回复
- 依赖 `src/config/` 读取配置

### 4.14 Hooks 系统

**核心文件**:

- `src/hooks/types.ts` - Hook 类型
- `src/hooks/registry.ts` - Hook 注册表
- `src/hooks/executor.ts` - Hook 执行器

**支持的 Hook 事件**:

```typescript
type HookEvent =
  | "before-agent-run"
  | "after-agent-run"
  | "before-message-send"
  | "after-message-send"
  | "before-tool-execution"
  | "after-tool-execution"
  | "channel-connected"
  | "channel-disconnected";
```

**Hook 配置**:

```typescript
interface HookConfig {
  id: string;
  event: HookEvent;
  script: string; // 脚本路径
  timeout?: number; // 超时 (ms)
  continueOnError?: boolean; // 错误时继续
}
```

**依赖关系**:

- 依赖 `src/process/` 执行脚本
- 依赖 `src/config/` 读取配置

---

## 五、数据流详解

### 5.1 入站消息流程 (Inbound Flow)

```
┌─────────────────────────────────────────────────────────────────┐
│                    入站消息完整流程                              │
└─────────────────────────────────────────────────────────────────┘

1. 渠道接收消息
   │
   ▼
   ┌──────────────────────────────────────┐
   │ Channel Adapter (渠道适配器)          │
   │ - webhook/polling 接收               │
   │ - 原始消息解析                        │
   └──────────────────────────────────────┘
   │
   ▼
2. 消息规范化
   ┌──────────────────────────────────────┐
   │ normalize()                          │
   │ - 提取文本/媒体/附件                  │
   │ - 统一消息格式                        │
   │ - 处理特殊格式 (mention/reply)        │
   └──────────────────────────────────────┘
   │
   ▼
3. 安全/访问检查
   ┌──────────────────────────────────────┐
   │ Access Control                       │
   │ - allowFrom 检查                     │
   │ - mentionGating 检查 (群聊)           │
   │ - dmPolicy 检查 (私聊)                │
   └──────────────────────────────────────┘
   │
   ▼
4. 会话键生成
   ┌──────────────────────────────────────┐
   │ Session Key Resolution               │
   │ - 基于 channel/account/peer           │
   │ - 确定 Agent 路由                     │
   └──────────────────────────────────────┘
   │
   ▼
5. 会话状态更新
   ┌──────────────────────────────────────┐
   │ Session Store                        │
   │ - 加载/创建会话                       │
   │ - 添加消息到历史                      │
   │ - 更新使用时间戳                      │
   └──────────────────────────────────────┘
   │
   ▼
6. Agent 触发 (如配置)
   ┌──────────────────────────────────────┐
   │ Agent Processing                     │
   │ - 构建上下文 (历史 + 工具)            │
   │ - 调用 LLM Provider                   │
   │ - 处理工具调用                        │
   └──────────────────────────────────────┘
   │
   ▼
7. Hook 执行 (如配置)
   ┌──────────────────────────────────────┐
   │ Hook Executor                        │
   │ - before-agent-run                   │
   │ - after-agent-run                    │
   └──────────────────────────────────────┘
```

### 5.2 出站消息流程 (Outbound Flow)

```
┌─────────────────────────────────────────────────────────────────┐
│                    出站消息完整流程                              │
└─────────────────────────────────────────────────────────────────┘

1. Agent 生成回复
   │
   ▼
   ┌──────────────────────────────────────┐
   │ Agent Response                       │
   │ - LLM 生成文本                        │
   │ - 可能包含工具调用结果                │
   └──────────────────────────────────────┘
   │
   ▼
2. 回复格式化
   ┌──────────────────────────────────────┐
   │ Format Response                      │
   │ - 添加回复前缀 (如配置)               │
   │ - 处理@提及                          │
   │ - 分割长消息 (chunking)               │
   └──────────────────────────────────────┘
   │
   ▼
3. 媒体处理 (如有)
   ┌──────────────────────────────────────┐
   │ Media Processing                     │
   │ - 加载媒体 URL                        │
   │ - 转码/压缩                          │
   │ - 上传到渠道 (如需要)                 │
   └──────────────────────────────────────┘
   │
   ▼
4. 目标解析
   ┌──────────────────────────────────────┐
   │ Target Resolution                    │
   │ - 解析目标地址                        │
   │ - 验证渠道支持                        │
   └──────────────────────────────────────┘
   │
   ▼
5. 渠道发送
   ┌──────────────────────────────────────┐
   │ Channel Send                         │
   │ - 调用渠道 API                        │
   │ - 处理失败重试                        │
   │ - 记录发送状态                        │
   └──────────────────────────────────────┘
   │
   ▼
6. 状态更新
   ┌──────────────────────────────────────┐
   │ Update Session                       │
   │ - 添加回复到历史                      │
   │ - 更新使用统计                        │
   └──────────────────────────────────────┘
```

### 5.3 Gateway 事件流

```
┌─────────────────────────────────────────────────────────────────┐
│                    Gateway 事件类型与流向                        │
└─────────────────────────────────────────────────────────────────┘

事件类型                触发条件                    订阅客户端
─────────────────────────────────────────────────────────────────
agent                  Agent 开始/流式输出/完成      所有控制客户端
chat                   新消息到达                   会话相关客户端
presence               用户在线状态变化              所有客户端
health                 健康检查响应                 监控客户端
heartbeat              心跳保活                     所有客户端
cron                   Cron 任务触发                管理客户端
shutdown               网关关闭                     所有客户端
channel-status         渠道状态变化                 所有客户端
config-reload          配置重载                     所有客户端
```

---

## 六、技术栈详细清单

### 6.1 后端依赖 (Backend Dependencies)

**核心框架**:
| 包名 | 版本 | 用途 |
|------|------|------|
| `typescript` | ^5.9.3 | 语言 |
| `tsdown` | 0.21.0-beta.2 | 构建工具 |
| `tsx` | ^4.21.0 | TypeScript 执行 |
| `vitest` | ^4.0.18 | 测试框架 |
| `oxlint` | ^1.50.0 | 代码检查 |
| `oxfmt` | 0.35.0 | 代码格式化 |

**运行时依赖**:
| 包名 | 版本 | 用途 |
|------|------|------|
| `@sinclair/typebox` | 0.34.48 | Schema 定义 |
| `zod` | ^4.3.6 | 运行时验证 |
| `express` | ^5.2.1 | HTTP 服务器 |
| `ws` | ^8.19.0 | WebSocket |
| `commander` | ^14.0.3 | CLI 框架 |
| `@clack/prompts` | ^1.0.1 | 交互式提示 |
| `chalk` | ^5.6.2 | 终端颜色 |
| `dotenv` | ^17.3.1 | 环境变量 |
| `yaml` | ^2.8.2 | YAML 解析 |
| `json5` | ^2.2.3 | JSON5 解析 |

**渠道 SDK**:
| 包名 | 版本 | 渠道 |
|------|------|------|
| `grammy` | ^1.41.0 | Telegram |
| `@slack/bolt` | ^4.6.0 | Slack |
| `@discordjs/voice` | ^0.19.0 | Discord 语音 |
| `@whiskeysockets/baileys` | 7.0.0-rc.9 | WhatsApp |
| `@line/bot-sdk` | ^10.6.0 | LINE |

**AI/ML**:
| 包名 | 版本 | 用途 |
|------|------|------|
| `@aws-sdk/client-bedrock` | ^3.1000.0 | AWS Bedrock |
| `@mariozechner/pi-agent-core` | 0.55.3 | Pi Agent |
| `playwright-core` | 1.58.2 | 浏览器自动化 |

**媒体处理**:
| 包名 | 版本 | 用途 |
|------|------|------|
| `sharp` | ^0.34.5 | 图片处理 |
| `@discordjs/opus` | ^0.10.0 | 音频编码 |
| `opusscript` | ^0.1.1 | Opus 编码 |
| `pdfjs-dist` | ^5.5.207 | PDF 处理 |

**存储**:
| 包名 | 版本 | 用途 |
|------|------|------|
| `sqlite-vec` | 0.1.7-alpha.2 | 向量存储 |
| `jszip` | ^3.10.1 | ZIP 处理 |

**网络**:
| 包名 | 版本 | 用途 |
|------|------|------|
| `undici` | ^7.22.0 | HTTP 客户端 |
| `https-proxy-agent` | ^7.0.6 | 代理支持 |
| `ipaddr.js` | ^2.3.0 | IP 地址处理 |

**工具**:
| 包名 | 版本 | 用途 |
|------|------|------|
| `croner` | ^10.0.1 | Cron 调度 |
| `chokidar` | ^5.0.0 | 文件监听 |
| `tar` | 7.5.9 | tar 处理 |
| `qrcode-terminal` | ^0.12.0 | QR 码生成 |
| `markdown-it` | ^14.1.1 | Markdown 解析 |

### 6.2 前端依赖 (UI Dependencies)

| 包名                | 版本   | 用途           |
| ------------------- | ------ | -------------- |
| `lit`               | ^3.3.2 | Web Components |
| `@lit-labs/signals` | ^0.2.0 | 状态管理       |
| `@lit/context`      | ^1.1.6 | 上下文传递     |
| `signal-utils`      | 0.21.1 | 信号工具       |

### 6.3 移动端依赖

**macOS/iOS**:

- Swift 5.9+
- SwiftUI
- Observation 框架
- Sparkle (自动更新)

**Android**:

- Kotlin
- Jetpack Compose
- Gradle
- AndroidX

---

## 七、测试体系详解

### 7.1 测试配置文件

| 配置文件                      | 用途         | 命令                   |
| ----------------------------- | ------------ | ---------------------- |
| `vitest.unit.config.ts`       | 单元测试     | `pnpm test:fast`       |
| `vitest.gateway.config.ts`    | Gateway 测试 | `pnpm test:gateway`    |
| `vitest.e2e.config.ts`        | E2E 测试     | `pnpm test:e2e`        |
| `vitest.live.config.ts`       | Live 测试    | `pnpm test:live`       |
| `vitest.channels.config.ts`   | 渠道测试     | `pnpm test:channels`   |
| `vitest.extensions.config.ts` | 扩展测试     | `pnpm test:extensions` |

### 7.2 测试类型

**单元测试**:

- 核心逻辑测试
- 工具函数测试
- 配置解析测试
- 覆盖率要求：70%+

**Gateway 测试**:

- WebSocket 连接测试
- 认证流程测试
- 事件分发测试
- 请求处理测试

**E2E 测试**:

- Docker 容器化测试
- 完整流程测试
- 多渠道集成测试

**Live 测试**:

- 真实 API 测试
- 需要配置凭据
- 环境变量：`OPENCLAW_LIVE_TEST=1`

**Docker 测试**:
| 测试 | 脚本 | 用途 |
|------|------|------|
| live-models | `scripts/test-live-models-docker.sh` | 模型 API 测试 |
| live-gateway | `scripts/test-live-gateway-docker.sh` | Gateway 测试 |
| onboard | `scripts/e2e/onboard-docker.sh` | 引导流程测试 |
| gateway-network | `scripts/e2e/gateway-network-docker.sh` | 网络配置测试 |
| qr | `scripts/e2e/qr-import-docker.sh` | QR 登录测试 |
| doctor-switch | `scripts/e2e/doctor-install-switch-docker.sh` | Doctor 测试 |
| plugins | `scripts/e2e/plugins-docker.sh` | 插件测试 |

### 7.3 测试覆盖率

**覆盖范围要求**:

- 行覆盖率：70%+
- 分支覆盖率：70%+
- 函数覆盖率：70%+
- 语句覆盖率：70%+

**覆盖报告**:

```bash
pnpm test:coverage
```

---

## 八、安全与权限

### 8.1 安全默认值

**DM 访问控制**:

- 默认 `dmPolicy="pairing"` - 未知用户需配对
- `dmPolicy="open"` - 公开访问 (需显式配置)
- `dmPolicy="blocked"` - 完全阻止

**群组访问控制**:

- `mentionGating` - 需@提及才响应
- `groupPolicy` - 群组工具策略

**执行安全**:

- `system.run` 命令白名单
- 危险命令检测
- 执行前批准机制

### 8.2 认证机制

**Gateway 认证**:

- Token 认证 (`OPENCLAW_GATEWAY_TOKEN`)
- 设备配对认证
- 签名挑战响应

**Provider 认证**:

- API Key
- OAuth 2.0
- 凭证轮换

### 8.3 数据安全

**本地存储**:

- 配置文件：`~/.openclaw/config.json5`
- 会话存储：`~/.openclaw/sessions/`
- 凭证存储：`~/.openclaw/credentials/`

**敏感信息处理**:

- 日志脱敏
- 环境变量引用
- 文件引用 (`file://` 路径)

---

## 九、架构升级注意事项

### 9.1 核心依赖关系图

```
┌─────────────────────────────────────────────────────────────────┐
│                    模块依赖关系概览                              │
└─────────────────────────────────────────────────────────────────┘

                    ┌──────────────┐
                    │   CLI (cli/) │
                    └──────┬───────┘
                           │
                    ┌──────▼───────┐
                    │   Config     │
                    │  (config/)   │
                    └──────┬───────┘
                           │
         ┌─────────────────┼─────────────────┐
         │                 │                 │
┌────────▼────────┐ ┌──────▼───────┐ ┌──────▼───────┐
│    Gateway      │ │   Routing    │ │   Plugins    │
│   (gateway/)    │ │  (routing/)  │ │  (plugins/)  │
└────────┬────────┘ └──────────────┘ └──────────────┘
         │
    ┌────┴────┐
    │         │
┌───▼───┐ ┌──▼──────┐
│Channels│ │ Agents  │
│       │ │(agents/)│
└───────┘ └────┬────┘
               │
         ┌─────┴─────┐
         │           │
    ┌────▼────┐ ┌───▼────┐
    │Providers│ │ Tools  │
    └─────────┘ └────────┘
```

### 9.2 重构升级注意事项

#### 9.2.1 高耦合模块

**需要特别注意的高耦合模块**:

1. **`src/gateway/`** - 与几乎所有模块有依赖
   - 重构时需保持 API 向后兼容
   - 建议先抽取接口定义
   - 测试覆盖率需达到 90%+

2. **`src/config/`** - 配置系统
   - 类型定义分散在 35+ 文件
   - 迁移需保持 Zod schema 兼容
   - 注意 `SecretInput` 处理逻辑

3. **`src/channels/`** - 渠道系统
   - 56 个目录，40+ 接口方法
   - 插件接口定义在 `src/plugin-sdk/`
   - 每个渠道有独立测试

4. **`src/agents/`** - Agent 运行时
   - 486 个文件，核心逻辑复杂
   - ACP 协议定义在 `src/acp/`
   - 与 Provider 深度耦合

#### 9.2.2 关键接口边界

**不应随意修改的核心接口**:

```typescript
// src/plugin-sdk/index.ts 导出的公共 API
// 任何修改都会影响插件兼容性

export type {
  ChannelGatewayAdapter,
  ChannelMessagingAdapter,
  ChannelOutboundAdapter,
  // ... 40+ 类型
};

// src/config/types.ts 导出的配置类型
export type {
  OpenClawConfig,
  ChannelConfig,
  AgentConfig,
  // ...
};

// src/gateway/ 的 WS 协议
// 修改需同步更新 Swift/Android 客户端
```

#### 9.2.3 数据迁移注意事项

**会话存储迁移**:

```typescript
// 会话存储位置
// ~/.openclaw/sessions/<agent>/sessions.json

// 迁移时需考虑:
// 1. 备份原文件
// 2. 版本检测
// 3. 增量迁移
// 4. 回滚机制
```

**配置迁移**:

```typescript
// 配置文件迁移脚本位置
// src/config/legacy-migrate.ts

// 迁移步骤:
// 1. 检测配置版本
// 2. 应用迁移逻辑
// 3. 验证迁移结果
// 4. 更新版本号
```

#### 9.2.4 测试策略

**重构时的测试优先级**:

1. **单元测试** - 核心逻辑不变
2. **集成测试** - 接口兼容性
3. **E2E 测试** - 完整流程
4. **Live 测试** - 真实 API

**测试命令**:

```bash
# 快速测试
pnpm test:fast

# Gateway 测试
pnpm test:gateway

# 完整测试套件
pnpm test:all
```

#### 9.2.5 性能考虑

**已知性能瓶颈**:

1. **会话加载** - 大会话文件加载慢
2. **嵌入生成** - 批量嵌入耗时
3. **媒体处理** - 大文件转码慢
4. **渠道轮询** - 多渠道并发请求

**优化方向**:

- 引入缓存层 (Redis)
- 异步任务队列 (Bull/RabbitMQ)
- 数据库迁移 (PostgreSQL)
- CDN 媒体存储

### 9.3 架构升级建议

#### 9.3.1 微服务拆分

**可拆分的独立服务**:

1. **Gateway 服务** - WebSocket 控制平面
2. **Channel 服务** - 渠道适配器集群
3. **Agent 服务** - Agent 运行时
4. **Memory 服务** - 向量搜索服务
5. **Media 服务** - 媒体处理服务

**拆分注意事项**:

- 定义清晰的 gRPC/REST API
- 引入服务发现机制
- 实现分布式追踪
- 考虑数据一致性

#### 9.3.2 数据库抽象

**当前状态**: SQLite (本地文件)

**抽象层设计**:

```typescript
interface DatabaseAdapter {
  // 会话存储
  getSession(key: string): Promise<SessionEntry>;
  saveSession(key: string, entry: SessionEntry): Promise<void>;

  // 向量存储
  searchVectors(query: number[], limit: number): Promise<VectorResult[]>;
  insertVectors(vectors: VectorEntry[]): Promise<void>;

  // 配置存储
  getConfig(): Promise<OpenClawConfig>;
  saveConfig(config: OpenClawConfig): Promise<void>;
}

// 实现
class SQLiteAdapter implements DatabaseAdapter { ... }
class PostgreSQLAdapter implements DatabaseAdapter { ... }
class LanceDBAdapter implements DatabaseAdapter { ... }
```

#### 9.3.3 API 网关

**当前状态**: WebSocket only

**建议新增**:

- REST API (OpenAPI/Swagger)
- GraphQL API (可选)
- WebSocket (保持)

**API 设计原则**:

- 版本化 (`/api/v1/`)
- 认证统一 (JWT/OAuth)
- 速率限制
- 文档自动生成

#### 9.3.4 可观测性

**当前状态**: 基础日志

**建议增强**:

```typescript
// OpenTelemetry 集成
import { trace, metrics, logs } from "@opentelemetry/api";

// 追踪
const tracer = trace.getTracer("openclaw");
const span = tracer.startSpan("agent.run");
// ...
span.end();

// 指标
const meter = metrics.getMeter("openclaw");
const counter = meter.createCounter("messages.processed");

// 日志
const logger = logs.getLogger("openclaw");
logger.emit({ body: "Message processed" });
```

**监控指标**:

- 请求延迟 (p50/p95/p99)
- 错误率
- 渠道连接状态
- Agent 执行时间
- 内存/CPU 使用

#### 9.3.5 容器化部署

**当前状态**: Docker 测试支持

**建议增强**:

- Docker Compose 配置
- Kubernetes Helm Chart
- 健康检查端点
- 配置热重载
- 优雅关闭

**Docker Compose 示例**:

```yaml
version: "3.8"
services:
  gateway:
    image: openclaw/gateway:latest
    ports:
      - "18789:18789"
    volumes:
      - ./config:/app/config
      - ./sessions:/app/sessions
    environment:
      - OPENCLAW_GATEWAY_TOKEN=${GATEWAY_TOKEN}
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:18789/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  redis:
    image: redis:7
    ports:
      - "6379:6379"

  postgres:
    image: postgres:15
    environment:
      - POSTGRES_PASSWORD=${DB_PASSWORD}
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
```

---

## 十、版本信息

### 10.1 当前版本

- **CLI 版本**: 2026.3.3 (来自 `package.json`)
- **发布渠道**: stable/beta/dev
- **Node 要求**: >=22.12.0
- **包管理器**: pnpm@10.23.0

### 10.2 版本历史关键节点

| 版本      | 日期       | 关键变更 |
| --------- | ---------- | -------- |
| 2026.3.3  | 2026-03-03 | 最新版本 |
| 2026.2.17 | 2026-02-17 | 插件发布 |
| ...       | ...        | ...      |

完整历史见 `CHANGELOG.md`

---

## 十一、关键文件索引

### 11.1 入口文件

| 文件            | 用途       |
| --------------- | ---------- |
| `src/index.ts`  | 主入口     |
| `src/entry.ts`  | CLI 入口   |
| `openclaw.mjs`  | 二进制入口 |
| `dist/index.js` | 构建后入口 |

### 11.2 配置 Schema

| 文件                                          | 用途            |
| --------------------------------------------- | --------------- |
| `src/config/zod-schema.core.ts`               | 核心配置 Schema |
| `src/config/zod-schema.providers-core.ts`     | Provider Schema |
| `src/config/zod-schema.providers-whatsapp.ts` | WhatsApp Schema |
| `src/config/zod-schema.agent-runtime.ts`      | Agent Schema    |

### 11.3 协议定义

| 文件                                                      | 用途         |
| --------------------------------------------------------- | ------------ |
| `src/gateway/protocol.ts`                                 | Gateway 协议 |
| `dist/protocol.schema.json`                               | JSON Schema  |
| `apps/macos/Sources/OpenClawProtocol/GatewayModels.swift` | Swift 模型   |

### 11.4 类型定义

| 文件                                  | 用途             |
| ------------------------------------- | ---------------- |
| `src/config/types.ts`                 | 配置类型导出     |
| `src/plugin-sdk/index.ts`             | 插件 SDK 类型    |
| `src/gateway/server-methods/types.ts` | Gateway 方法类型 |

---

## 十二、总结

本报告详细分析了 OpenClaw 项目的完整架构，包括:

1. **整体架构层次** - 客户端/Gateway/渠道/Provider 四层
2. **完整目录结构** - 80+ 源代码目录，40+ 插件
3. **核心功能模块** - 14 个主要系统详解
4. **数据流分析** - 入站/出站/事件流
5. **技术栈清单** - 50+ 核心依赖
6. **测试体系** - 6 种测试类型
7. **安全机制** - 认证/授权/数据保护
8. **升级建议** - 微服务/数据库/API/可观测性

**架构升级优先级建议**:

1. **高优先级** - 数据库抽象层、API 网关
2. **中优先级** - 可观测性增强、容器化部署
3. **低优先级** - 微服务拆分 (需充分测试)

---

**文档生成时间**: 2026-03-04
**下次更新建议**: 重大架构变更后

https://docs.openclaw.ai/gateway
https://docs.openclaw.ai/channels
https://docs.openclaw.ai/concepts/agent
https://docs.openclaw.ai/install/updating
https://docs.openclaw.ai/gateway/security
