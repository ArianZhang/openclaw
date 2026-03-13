# Clients、Channels 与 Gateway 架构关系详细分析

**版本**: 2026-03-04
**分析目标**: 深入理解客户端层、渠道适配层与 Gateway 之间的架构关系

---

## 一、架构关系总览

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         客户端层 (Clients)                                   │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐    │
│  │   macOS App  │  │    iOS App   │  │  Android App │  │   WebChat    │    │
│  │  (SwiftUI)   │  │  (SwiftUI)   │  │   (Kotlin)   │  │  (Lit HTML)  │    │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘    │
│         │                 │                 │                 │            │
│         └─────────────────┴────────┬────────┴─────────────────┘            │
│                                    │                                       │
│                           GatewayBrowserClient                             │
│                           GatewayClient (Node)                             │
└────────────────────────────────────┼────────────────────────────────────────┘
                                     │
                     WebSocket (ws://127.0.0.1:18789)
                     协议版本：PROTOCOL_VERSION = 3
                                     │
┌────────────────────────────────────┼────────────────────────────────────────┐
│                          Gateway (控制平面)                                 │
│                                    │                                       │
│  ┌─────────────────────────────────▼───────────────────────────────────┐   │
│  │                      WebSocket 服务器                                │   │
│  │  ┌─────────────────────────────────────────────────────────────────┐│   │
│  │  │                    认证层 (auth.ts)                              ││   │
│  │  │  - Token 认证 / Password 认证 / Tailscale 认证 / Trusted Proxy  ││   │
│  │  │  - Device Pairing (设备配对)                                     ││   │
│  │  │  - Rate Limiting (速率限制)                                      ││   │
│  │  └─────────────────────────────────────────────────────────────────┘│   │
│  │  ┌─────────────────────────────────────────────────────────────────┐│   │
│  │  │               协议处理 (protocol/index.ts)                       ││   │
│  │  │  - Request Frame: {type:"req", id, method, params}              ││   │
│  │  │  - Response Frame: {type:"res", id, ok, payload/error}          ││   │
│  │  │  - Event Frame: {type:"event", event, payload, seq}             ││   │
│  │  └─────────────────────────────────────────────────────────────────┘│   │
│  │  ┌─────────────────────────────────────────────────────────────────┐│   │
│  │  │             服务器方法 (server-methods/)                         ││   │
│  │  │  - agent.ts: agent 请求处理                                      ││   │
│  │  │  - chat.ts: chat 消息处理                                        ││   │
│  │  │  - channels.ts: 渠道状态查询                                     ││   │
│  │  │  - config.ts: 配置管理                                           ││   │
│  │  │  - cron.ts: Cron 任务管理                                        ││   │
│  │  │  - devices.ts: 设备配对管理                                      ││   │
│  │  │  - sessions.ts: 会话管理                                         ││   │
│  │  └─────────────────────────────────────────────────────────────────┘│   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                       │
│         ┌──────────────────────────┼──────────────────────────┐            │
│         │                          │                          │            │
│         ▼                          ▼                          ▼            │
│  ┌─────────────┐           ┌─────────────┐           ┌─────────────┐       │
│  │ Agent 系统  │           │  Routing    │           │   Events    │       │
│  │ (agents/)   │           │ (routing/)  │           │  (events)   │       │
│  └─────────────┘           └─────────────┘           └─────────────┘       │
│                                    │                                       │
└────────────────────────────────────┼────────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                        渠道适配层 (Channels)                                │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │                    渠道注册表 (registry.ts)                            │ │
│  │  CHAT_CHANNEL_ORDER: [telegram, whatsapp, discord, irc, ...]          │ │
│  │  getChatChannelMeta(): 获取渠道元数据                                  │ │
│  │  normalizeChannelId(): 渠道 ID 标准化                                   │ │
│  └───────────────────────────────────────────────────────────────────────┘ │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │                    渠道 Dock (dock.ts)                                 │ │
│  │  DOCKS: Record<ChannelId, ChannelDock>                                │ │
│  │  - capabilities: 能力声明                                             │ │
│  │  - config: 配置解析 (allowFrom/defaultTo)                             │ │
│  │  - groups: 群组策略 (mention/tool policy)                             │ │
│  │  - threading: 线程支持                                                │ │
│  │  - mentions: @提及处理                                                 │ │
│  └───────────────────────────────────────────────────────────────────────┘ │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │                  渠道插件接口 (plugins/types.ts)                       │ │
│  │  - ChannelGatewayAdapter: 网关适配                                    │ │
│  │  - ChannelMessagingAdapter: 消息适配                                  │ │
│  │  - ChannelOutboundAdapter: 出站适配                                   │ │
│  │  - ChannelPairingAdapter: 配对适配                                    │ │
│  │  - ChannelStatusAdapter: 状态适配                                     │ │
│  │  - ChannelConfigAdapter: 配置适配                                     │ │
│  └───────────────────────────────────────────────────────────────────────┘ │
│                                    │                                       │
│         ┌─────────┬─────────┬─────────┬─────────┬─────────┬─────────┐      │
│         ▼         ▼         ▼         ▼         ▼         ▼         ▼      │
│    ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐      │
│    │Telegram│ │WhatsApp│ │ Discord│ │  IRC   │ │ Google │ │  Slack │ ...  │
│    │grammY  │ │Baileys │ │discord │ │   IRC  │ │  Chat  │ │  Bolt  │      │
│    └────────┘ └────────┘ └────────┘ └────────┘ └────────┘ └────────┘      │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 二、客户端层 (Clients) 详细分析

### 2.1 客户端类型

| 客户端      | 实现位置        | 技术栈                 | 连接方式               |
| ----------- | --------------- | ---------------------- | ---------------------- |
| macOS App   | `apps/macos/`   | Swift/SwiftUI          | GatewayClient (Swift)  |
| iOS App     | `apps/ios/`     | Swift/SwiftUI          | GatewayClient (Swift)  |
| Android App | `apps/android/` | Kotlin/Jetpack Compose | GatewayClient (Kotlin) |
| WebChat UI  | `ui/src/ui/`    | Lit Web Components     | GatewayBrowserClient   |
| CLI         | `src/cli/`      | TypeScript             | GatewayClient (Node)   |

### 2.2 客户端连接实现

#### 2.2.1 Web 客户端 (`ui/src/ui/gateway.ts`)

**核心类**: `GatewayBrowserClient`

**关键属性**:

```typescript
class GatewayBrowserClient {
  private ws: WebSocket | null = null;
  private pending = new Map<string, Pending>();
  private lastSeq: number | null = null; // 事件序号跟踪
  private connectNonce: string | null = null; // 挑战 nonce
  private backoffMs = 800; // 退避时间
}
```

**连接流程**:

```typescript
// 1. 创建 WebSocket 连接
this.ws = new WebSocket(this.opts.url);

// 2. 接收 connect.challenge 事件
if (evt.event === "connect.challenge") {
  this.connectNonce = payload.nonce;
  this.sendConnect();
}

// 3. 使用设备私钥签名 nonce
const payload = buildDeviceAuthPayload({...});
const signature = await signDevicePayload(deviceIdentity.privateKey, payload);

// 4. 发送 connect 请求
this.request<GatewayHelloOk>("connect", params);

// 5. 接收 hello-ok 响应 + 初始快照
this.opts.onHello?.(hello);
```

**事件处理**:

```typescript
handleMessage(raw: string) {
  // 事件帧
  if (frame.type === "event") {
    // 检测事件丢失
    if (seq > this.lastSeq + 1) {
      this.opts.onGap?.({ expected: this.lastSeq + 1, received: seq });
    }
    this.lastSeq = seq;
    this.opts.onEvent?.(evt);
  }

  // 响应帧
  if (frame.type === "res") {
    pending.resolve(res.payload);
    // 或
    pending.reject(new GatewayRequestError({...}));
  }
}
```

#### 2.2.2 Node 客户端 (`src/gateway/client.ts`)

**核心类**: `GatewayClient`

**关键属性**:

```typescript
class GatewayClient {
  private ws: WebSocket | null = null;
  private pending = new Map<string, Pending>();
  private backoffMs = 1000;
  private lastSeq: number | null = null;

  // 设备身份
  private opts: {
    deviceIdentity: DeviceIdentity; // 设备密钥对
    role: "operator" | "node";
    scopes: string[];
  };

  // Tick 监控 (检测连接静默)
  private lastTick: number | null = null;
  private tickIntervalMs = 30_000;
  private tickTimer: NodeJS.Timeout | null = null;
}
```

**设备认证流程**:

```typescript
sendConnect() {
  const nonce = this.connectNonce;
  const role = this.opts.role ?? "operator";
  const scopes = this.opts.scopes ?? ["operator.admin"];

  // 构建设备认证 payload (v3 格式)
  const payload = buildDeviceAuthPayloadV3({
    deviceId: deviceIdentity.deviceId,
    clientId: GATEWAY_CLIENT_NAMES.GATEWAY_CLIENT,
    clientMode: GATEWAY_CLIENT_MODES.BACKEND,
    role,
    scopes,
    signedAtMs: Date.now(),
    token: authToken ?? null,
    nonce,
    platform,
    deviceFamily,
  });

  // 使用设备私钥签名
  const signature = signDevicePayload(deviceIdentity.privateKeyPem, payload);

  // 发送 connect 请求
  this.request<HelloOk>("connect", {
    minProtocol: PROTOCOL_VERSION,
    maxProtocol: PROTOCOL_VERSION,
    client: { id, version, platform, mode },
    auth: { token, deviceToken, password },
    device: { id, publicKey, signature, signedAt, nonce },
    role,
    scopes,
  });
}
```

**TLS 指纹验证** (用于 WSS 连接):

```typescript
validateTlsFingerprint(): Error | null {
  const socket = this.ws._socket;
  const cert = socket.getPeerCertificate();
  const fingerprint = normalizeFingerprint(cert.fingerprint256);

  if (fingerprint !== expected) {
    return new Error("gateway tls fingerprint mismatch");
  }
  return null;
}
```

**Tick 监控** (检测连接静默):

```typescript
startTickWatch() {
  this.tickTimer = setInterval(() => {
    const gap = Date.now() - this.lastTick;
    if (gap > this.tickIntervalMs * 2) {
      this.ws?.close(4000, "tick timeout");
    }
  }, interval);
}
```

### 2.3 客户端事件订阅

| 事件类型                  | 触发条件            | 客户端处理                              |
| ------------------------- | ------------------- | --------------------------------------- |
| `tick`                    | 定期心跳 (默认 30s) | 更新 `lastTick`，检测连接静默           |
| `agent`                   | Agent 流式输出      | `handleAgentEvent()` - 更新 UI 流式状态 |
| `chat`                    | 聊天消息发送/接收   | `handleChatEvent()` - 更新聊天历史      |
| `presence`                | 在线状态变化        | 更新 presenceEntries 列表               |
| `health`                  | 健康状态变化        | 更新 debugHealth                        |
| `cron`                    | Cron 任务触发       | 重新加载 Cron 列表                      |
| `device.pair.requested`   | 新设备配对请求      | 重新加载设备列表                        |
| `exec.approval.requested` | 执行批准请求        | 添加到批准队列                          |

### 2.4 协议版本

**当前版本**: `PROTOCOL_VERSION = 3`

**帧格式**:

```typescript
// 请求帧
interface RequestFrame {
  type: "req";
  id: string; // UUID
  method: string;
  params: unknown;
}

// 响应帧
interface ResponseFrame {
  type: "res";
  id: string;
  ok: boolean;
  payload?: unknown;
  error?: { code: string; message: string; details?: unknown };
}

// 事件帧
interface EventFrame {
  type: "event";
  event: string;
  payload?: unknown;
  seq?: number; // 事件序号
  stateVersion?: number; // 状态版本 (presence/health)
}
```

---

## 三、Gateway 详细分析

### 3.1 Gateway 核心组件

#### 3.1.1 认证层 (`src/gateway/auth.ts`)

**认证模式**:

```typescript
type ResolvedGatewayAuthMode =
  | "none" // 无认证
  | "token" // Token 认证
  | "password" // 密码认证
  | "trusted-proxy" // 可信代理
  | "tailscale"; // Tailscale 认证
```

**认证流程**:

```typescript
async function authorizeGatewayConnect(params): Promise<GatewayAuthResult> {
  // 1. trusted-proxy 模式
  if (auth.mode === "trusted-proxy") {
    return authorizeTrustedProxy(params);
  }

  // 2. 速率限制检查
  if (limiter) {
    const rlCheck = limiter.check(ip, rateLimitScope);
    if (!rlCheck.allowed) {
      return { ok: false, reason: "rate_limited", retryAfterMs: ... };
    }
  }

  // 3. Tailscale Header 认证 (仅 WS Control UI)
  if (allowTailscaleHeaderAuth && auth.allowTailscale && !localDirect) {
    const tailscaleCheck = await resolveVerifiedTailscaleUser({...});
    if (tailscaleCheck.ok) {
      return { ok: true, method: "tailscale", user: tailscaleCheck.user.login };
    }
  }

  // 4. Token 认证
  if (auth.mode === "token") {
    if (!safeEqualSecret(connectAuth.token, auth.token)) {
      return { ok: false, reason: "token_mismatch" };
    }
    return { ok: true, method: "token" };
  }

  // 5. 密码认证
  if (auth.mode === "password") {
    if (!safeEqualSecret(connectAuth.password, auth.password)) {
      return { ok: false, reason: "password_mismatch" };
    }
    return { ok: true, method: "password" };
  }
}
```

**速率限制**:

- 按 IP 地址跟踪失败次数
- 作用域：`AUTH_RATE_LIMIT_SCOPE_SHARED_SECRET`
- 成功认证后重置计数器

#### 3.1.2 协议定义 (`src/gateway/protocol/index.ts`)

**验证函数** (使用 AJV):

```typescript
export const validateRequestFrame = ajv.compile<RequestFrame>(RequestFrameSchema);
export const validateResponseFrame = ajv.compile<ResponseFrame>(ResponseFrameSchema);
export const validateEventFrame = ajv.compile<EventFrame>(EventFrameSchema);
export const validateConnectParams = ajv.compile<ConnectParams>(ConnectParamsSchema);
```

**TypeBox Schema 示例**:

```typescript
const RequestFrameSchema = Type.Object({
  type: Type.Literal("req"),
  id: Type.String(),
  method: Type.String(),
  params: Type.Optional(Type.Unknown()),
});
```

#### 3.1.3 服务器方法架构 (`src/gateway/server-methods/`)

**请求处理器类型**:

```typescript
type GatewayRequestHandler = (params: {
  req: RequestFrame;
  params: unknown;
  respond: RespondFn;
  context: GatewayRequestContext;
  client: GatewayClientInfo;
  isWebchatConnect: boolean;
}) => void | Promise<void>;

type RespondFn = (
  ok: boolean,
  payload?: unknown,
  error?: ErrorShape,
  opts?: { cached?: boolean; runId?: string; error?: unknown },
) => void;
```

### 3.2 Agent 请求处理流程 (`src/gateway/server-methods/agent.ts`)

**核心流程**:

```typescript
agent: async ({ params, respond, context, client }) => {
  // 1. 验证参数
  if (!validateAgentParams(params)) {
    respond(false, undefined, errorShape(...));
    return;
  }

  // 2. 解析会话键
  const sessionKey = request.sessionKey || resolveMainSessionKey(cfg);
  const agentId = resolveAgentIdFromSessionKey(sessionKey);

  // 3. 加载/创建会话
  const { cfg, storePath, entry } = loadSessionEntry(sessionKey);

  // 4. 去重检查
  const cached = context.dedupe.get(`agent:${idem}`);
  if (cached) {
    respond(cached.ok, cached.payload, cached.error, { cached: true });
    return;
  }

  // 5. 注册 Agent 运行上下文
  const runId = idem;
  registerAgentRunContext(runId, { sessionKey, agentId });

  // 6. 发送初始响应 (accepted)
  respond(true, { runId, status: "accepted" });

  // 7. 执行 Agent 命令
  const result = await agentCommandFromIngress({...});

  // 8. 发送最终响应
  respond(true, { runId, status: "ok", summary: "completed", result });
}
```

**会话键解析**:

```typescript
// 会话键格式：agent:<agentId>:<rest>
// - agent:<agentId>:main                    - 主会话
// - agent:<agentId>:main:channel:<channel>  - 渠道会话
// - agent:<agentId>:<channel>:direct:<peer> - DM 会话

function resolveAgentIdFromSessionKey(sessionKey: string): string {
  const parsed = parseAgentSessionKey(sessionKey);
  return normalizeAgentId(parsed?.agentId ?? DEFAULT_AGENT_ID);
}
```

### 3.3 Chat 请求处理流程 (`src/gateway/server-methods/chat.ts`)

**chat.send 处理**:

```typescript
"chat.send": async ({ params, respond, context, client }) => {
  // 1. 验证参数
  if (!validateChatSendParams(params)) {
    respond(false, undefined, errorShape(...));
    return;
  }

  // 2. 消息清理
  const sanitized = sanitizeChatSendMessageInput(p.message);

  // 3. 检查停止命令 (/stop)
  if (isChatStopCommandText(inboundMessage)) {
    abortChatRunsForSessionKey(sessionKey);
    respond(true, { ok: true, aborted: true });
    return;
  }

  // 4. 加载会话
  const { cfg, entry, canonicalKey: sessionKey } = loadSessionEntry(rawSessionKey);

  // 5. 检查发送策略
  const sendPolicy = resolveSendPolicy({...});
  if (sendPolicy === "deny") {
    respond(false, undefined, errorShape("send blocked by session policy"));
    return;
  }

  // 6. 创建 AbortController
  const abortController = new AbortController();
  context.chatAbortControllers.set(clientRunId, {...});

  // 7. 发送 ACK
  respond(true, { runId: clientRunId, status: "started" });

  // 8. 分发消息到 dispatchInboundMessage
  await dispatchInboundMessage({ ctx, cfg, dispatcher, replyOptions });
}
```

**chat.history 处理**:

```typescript
"chat.history": async ({ params, respond, context }) => {
  // 1. 读取会话消息
  const rawMessages = readSessionMessages(sessionId, storePath);

  // 2. 清理历史消息
  const sanitized = sanitizeChatHistoryMessages(rawMessages);
  // - 移除 thinkingSignature
  // - 截断过长文本 (12000 字符)
  // - 移除图片数据 (保留 bytes 计数)
  // - 移除 details/usage/cost 字段

  // 3. 替换过大消息
  const replaced = replaceOversizedChatHistoryMessages({...});

  // 4. 按字节预算限制
  const capped = capArrayByJsonBytes(messages, maxHistoryBytes);
  const bounded = enforceChatHistoryFinalBudget({...});

  respond(true, { sessionKey, sessionId, messages: bounded.messages });
}
```

### 3.4 事件广播机制

**广播函数**:

```typescript
function broadcast(
  event: string,
  payload?: unknown,
  opts?: {
    seq?: number;
    stateVersion?: { presence: number; health: number };
    scope?: string;
    dropIfSlow?: boolean;
  },
) {
  for (const client of clients) {
    const frame: EventFrame = { type: "event", event, payload, seq };
    client.ws.send(JSON.stringify(frame));
  }
}
```

**事件序号跟踪**:

- 每个连接维护独立的 `seq` 计数器
- 客户端检测事件丢失：`if (seq > this.lastSeq + 1)`
- `onGap` 回调通知客户端

---

## 四、渠道适配层 (Channels) 详细分析

### 4.1 渠道架构层次

```
┌─────────────────────────────────────────────────────────────────┐
│                    渠道抽象层 (Dock)                             │
│  dock.ts: ChannelDock 定义统一接口                               │
│  - capabilities: 能力声明                                       │
│  - config: 配置解析                                             │
│  - groups: 群组策略                                             │
│  - threading: 线程支持                                          │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    渠道注册表 (registry.ts)                      │
│  CHAT_CHANNEL_ORDER: 核心渠道顺序                                │
│  CHAT_CHANNEL_META: 渠道元数据                                   │
│  normalizeChannelId(): ID 标准化                                  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    渠道插件接口 (plugins/types.ts)               │
│  40+ 接口方法定义                                                │
│  - ChannelGatewayAdapter                                        │
│  - ChannelMessagingAdapter                                      │
│  - ChannelOutboundAdapter                                       │
│  - ChannelPairingAdapter                                        │
│  - ChannelStatusAdapter                                         │
│  - ChannelConfigAdapter                                         │
│  - ChannelGroupAdapter                                          │
│  - ChannelMentionAdapter                                        │
│  - ChannelThreadingAdapter                                      │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    具体渠道实现                                   │
│  核心渠道: telegram, whatsapp, discord, slack, signal, imessage  │
│  扩展渠道: matrix, msteams, line, irc, googlechat, ...          │
│  插件渠道: extensions/ 下的其他渠道                              │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 渠道 Dock 接口 (`src/channels/dock.ts`)

**ChannelDock 定义**:

```typescript
type ChannelDock = {
  id: ChannelId;

  // 能力声明
  capabilities: {
    chatTypes: ("direct" | "group" | "channel" | "thread")[];
    nativeCommands?: boolean;
    blockStreaming?: boolean;
    polls?: boolean;
    reactions?: boolean;
    media?: boolean;
    threads?: boolean;
  };

  // 命令处理
  commands?: ChannelCommandAdapter;

  // 出站限制
  outbound?: {
    textChunkLimit?: number; // 消息分块限制
  };

  // 流式配置
  streaming?: {
    blockStreamingCoalesceDefaults?: {
      minChars?: number;
      idleMs?: number;
    };
  };

  // 配置解析
  config?: {
    resolveAllowFrom(cfg, accountId): string[];
    formatAllowFrom(allowFrom): string[];
    resolveDefaultTo(cfg, accountId): string | undefined;
  };

  // 群组策略
  groups?: {
    resolveRequireMention(ctx): boolean;
    resolveToolPolicy(ctx): GroupToolPolicy;
  };

  // @提及处理
  mentions?: {
    stripPatterns(ctx): RegExp[];
  };

  // 线程支持
  threading?: {
    resolveReplyToMode(ctx): "off" | "reply" | "thread";
    buildToolContext(ctx): ChannelThreadingToolContext;
  };
};
```

**核心渠道列表**:

```typescript
const CHAT_CHANNEL_ORDER = [
  "telegram",
  "whatsapp",
  "discord",
  "irc",
  "googlechat",
  "slack",
  "signal",
  "imessage",
] as const;
```

### 4.3 渠道配置解析示例

**Telegram 配置解析**:

```typescript
telegram: {
  config: {
    resolveAllowFrom: ({ cfg, accountId }) =>
      stringifyAllowFrom(
        resolveTelegramAccount({ cfg, accountId }).config.allowFrom ?? []
      ),
    formatAllowFrom: ({ allowFrom }) =>
      trimAllowFromEntries(allowFrom)
        .map((entry) => entry.replace(/^(telegram|tg):/i, ""))
        .map((entry) => entry.toLowerCase()),
    resolveDefaultTo: ({ cfg, accountId }) => {
      const val = resolveTelegramAccount({ cfg, accountId }).config.defaultTo;
      return val != null ? String(val) : undefined;
    },
  },
  groups: {
    resolveRequireMention: resolveTelegramGroupRequireMention,
    resolveToolPolicy: resolveTelegramGroupToolPolicy,
  },
  threading: {
    resolveReplyToMode: ({ cfg }) => cfg.channels?.telegram?.replyToMode ?? "off",
    buildToolContext: ({ context, hasRepliedRef }) => ({
      currentChannelId: context.To?.trim() || undefined,
      currentThreadTs: context.MessageThreadId ? String(context.MessageThreadId) : undefined,
      hasRepliedRef,
    }),
  },
}
```

**Discord 配置解析**:

```typescript
discord: {
  config: {
    resolveAllowFrom: ({ cfg, accountId }) => {
      const account = resolveDiscordAccount({ cfg, accountId });
      return (account.config.allowFrom ?? account.config.dm?.allowFrom ?? [])
        .map((entry) => String(entry));
    },
    formatAllowFrom: ({ allowFrom }) => formatDiscordAllowFrom(allowFrom),
    // 移除 <@!123456>、discord:、user:、pk: 前缀
  },
  mentions: {
    stripPatterns: () => ["<@!?\\d+>"],
  },
}
```

### 4.4 渠道插件接口 (`src/channels/plugins/types.ts`)

**核心适配器接口**:

```typescript
// 网关适配器 - 渠道生命周期管理
interface ChannelGatewayAdapter {
  connect(): Promise<void>;
  disconnect(): Promise<void>;
  getAccountStatus(cfg, accountId): Promise<ChannelAccountSnapshot>;
}

// 消息适配器 - 入站消息处理
interface ChannelMessagingAdapter {
  onMessage(callback: (msg: ChannelMessage) => void): void;
}

// 出站适配器 - 发送消息
interface ChannelOutboundAdapter {
  sendText(target, text): Promise<void>;
  sendMedia(target, media): Promise<void>;
  sendPoll?(target, poll): Promise<void>;
}

// 配对适配器 - 设备配对
interface ChannelPairingAdapter {
  startPairing(cfg, accountId): Promise<LoginWithQrResult>;
  approvePairing(cfg, accountId, code): Promise<void>;
  rejectPairing(cfg, accountId, code): Promise<void>;
}

// 状态适配器 - 健康检查
interface ChannelStatusAdapter {
  getStatusSummary(cfg, accountId): Promise<ChannelStatusSummary>;
  probeAccount?(params): Promise<ProbeResult>;
}

// 配置适配器 - 配置解析
interface ChannelConfigAdapter<Account> {
  listAccountIds(cfg): string[];
  defaultAccountId?(cfg): string;
  resolveAccount(cfg, accountId): Account;
  isEnabled(account, cfg): boolean;
  isConfigured?(account, cfg): Promise<boolean>;
  resolveAllowFrom(cfg, accountId): Array<string | number>;
  formatAllowFrom(allowFrom): string[];
  resolveDefaultTo(cfg, accountId): string | undefined;
}

// 群组适配器 - 群组访问控制
interface ChannelGroupAdapter {
  resolveRequireMention(ctx): boolean;
  resolveToolPolicy(ctx): GroupToolPolicy;
  resolveGroupIntroHint?(ctx): string;
}

// 线程适配器 - 消息线程支持
interface ChannelThreadingAdapter {
  resolveReplyToMode(ctx): "off" | "reply" | "thread";
  buildToolContext(ctx): ChannelThreadingToolContext;
  allowExplicitReplyTagsWhenOff?: boolean;
}
```

### 4.5 渠道消息处理流程

#### 4.5.1 入站消息 (渠道 → Gateway → 客户端)

```typescript
// 1. 渠道收到消息 (webhook/polling)
channel.on("message", async (rawMessage) => {
  // 2. 规范化为统一格式
  const normalized = normalizeChannelMessage(rawMessage);

  // 3. 访问控制检查
  const allowFrom = channel.config.resolveAllowFrom({ cfg, accountId });
  if (!isAllowed(senderId, allowFrom)) {
    return;
  }

  // 4. 检查群组@提及要求
  if (chatType === "group") {
    const requireMention = channel.groups.resolveRequireMention(ctx);
    if (requireMention && !isMentioned(botId, message)) {
      return;
    }
  }

  // 5. 生成会话键
  const sessionKey = resolveSessionKey(channel, account, peer);

  // 6. 创建消息上下文
  const ctx: MsgContext = {
    Body: normalized.text,
    Provider: channel.id,
    AccountId: accountId,
    From: senderId,
    To: recipientId,
    ChatType: chatType,
    MessageThreadId: threadId,
  };

  // 7. 分发消息
  await dispatchInboundMessage({ ctx, cfg, dispatcher });
});
```

#### 4.5.2 出站消息 (客户端 → Gateway → 渠道)

```typescript
// 1. 客户端发送 chat.send 请求
// {method:"chat.send", params:{sessionKey, message, attachments}}

// 2. Gateway 处理请求
const { cfg, entry } = loadSessionEntry(sessionKey);
const channel = entry?.channel;
const target = entry?.lastTo;

// 3. 格式化消息
const prefixOptions = createReplyPrefixOptions({ cfg, agentId, channel });
const dispatcher = createReplyDispatcher({ ...prefixOptions, deliver: ... });

// 4. 调用渠道发送
await channel.sendText(target, text);

// 5. 记录发送状态
updateSessionStore(sessionKey, sentMessage);

// 6. 发送 chat 事件给客户端
broadcast("chat", { event: "message-sent", sessionKey, message: sentMessage });
```

---

## 五、路由与会话管理

### 5.1 会话键 (SessionKey) 结构

**会话键格式**:

```typescript
// 标准格式：agent:<agentId>:<rest>

// 主会话
agent:<agentId>:main

// 渠道会话
agent:<agentId>:main:channel:<channelId>

// DM 会话
agent:<agentId>:<channel>:direct:<peerId>

// 群组会话
agent:<agentId>:<channel>:group:<groupId>

// 线程会话
agent:<agentId>:<channel>:thread:<threadId>
```

**会话键解析**:

```typescript
function parseAgentSessionKey(sessionKey: string): ParsedAgentSessionKey | null {
  if (!sessionKey.startsWith("agent:")) {
    return null;
  }
  const parts = sessionKey.split(":");
  if (parts.length < 3) {
    return null;
  }
  const agentId = parts[1];
  const rest = parts.slice(2).join(":");
  return { agentId, rest };
}

function classifySessionKeyShape(sessionKey: string): SessionKeyShape {
  if (!sessionKey?.trim()) {
    return "missing";
  }
  if (parseAgentSessionKey(sessionKey)) {
    return "agent";
  }
  return sessionKey.toLowerCase().startsWith("agent:") ? "malformed_agent" : "legacy_or_alias";
}
```

### 5.2 账户 ID 标准化

```typescript
const DEFAULT_ACCOUNT_ID = "default";

function normalizeAccountId(accountId: string | null | undefined): string {
  const trimmed = (accountId ?? "").trim();
  return trimmed ? trimmed.toLowerCase() : DEFAULT_ACCOUNT_ID;
}
```

### 5.3 会话存储

**存储位置**: `~/.openclaw/sessions/<agent>/sessions.json`

**会话条目结构**:

```typescript
type SessionEntry = {
  sessionId: string;
  updatedAt: number;
  channel?: string;
  chatType?: ChatType;
  lastTo?: string;
  lastAccountId?: string;
  lastThreadId?: string;
  deliveryContext?: {
    channel: string;
    to: string;
    accountId: string;
    threadId: string;
  };
  sendPolicy?: "allow" | "deny";
  skillsSnapshot?: string;
  modelOverride?: string;
  providerOverride?: string;
  thinkingLevel?: "low" | "medium" | "high";
  label?: string;
  spawnedBy?: string;
  spawnDepth?: number;
  groupId?: string;
  groupChannel?: string;
  space?: string;
};
```

---

## 六、入站消息分发流程

### 6.1 分发器创建 (`src/auto-reply/reply/reply-dispatcher.ts`)

```typescript
function createReplyDispatcher(options: ReplyDispatcherOptions): ReplyDispatcher {
  const reservations = new Set<string>();
  const pendingWrites = new Map<string, string>();
  let idleTimeout: NodeJS.Timeout | null = null;

  return {
    reserve(channelId: string) {
      reservations.add(channelId);
    },
    release(channelId: string) {
      reservations.delete(channelId);
    },
    write(channelId: string, text: string) {
      pendingWrites.set(channelId, text);
      resetIdleTimeout();
    },
    async deliver(payload, info) {
      await options.deliver(payload, info);
    },
    markComplete() {
      reservations.clear();
    },
    async waitForIdle() {
      return new Promise((resolve) => {
        if (pendingWrites.size === 0) {
          resolve();
          return;
        }
        idleTimeout = setTimeout(resolve, IDLE_TIMEOUT_MS);
      });
    },
  };
}
```

### 6.2 入站消息分发 (`src/auto-reply/dispatch.ts`)

```typescript
async function dispatchInboundMessage(params: {
  ctx: MsgContext;
  cfg: OpenClawConfig;
  dispatcher: ReplyDispatcher;
  replyOptions?: GetReplyOptions;
}): Promise<DispatchInboundResult> {
  // 1. 完成上下文
  const finalized = finalizeInboundContext(params.ctx);

  // 2. 使用调度器包装
  return await withReplyDispatcher({
    dispatcher: params.dispatcher,
    run: () =>
      dispatchReplyFromConfig({
        ctx: finalized,
        cfg: params.cfg,
        dispatcher: params.dispatcher,
        replyOptions: params.replyOptions,
      }),
  });
}
```

### 6.3 配置驱动的回复分发 (`src/auto-reply/reply/dispatch-from-config.ts`)

```typescript
async function dispatchReplyFromConfig(params: {
  ctx: FinalizedMsgContext;
  cfg: OpenClawConfig;
  dispatcher: ReplyDispatcher;
  replyOptions?: GetReplyOptions;
}): Promise<DispatchFromConfigResult> {
  // 1. 检查 autoReply 配置
  const autoReplyConfig = params.cfg.autoReply;

  // 2. 调用 agentCommand 生成回复
  const result = await agentCommandFromIngress({
    message: params.ctx.BodyForAgent,
    sessionId: params.ctx.SessionKey,
    thinking: params.replyOptions?.thinking,
  });

  // 3. 通过调度器发送回复
  await params.dispatcher.deliver(
    {
      channel: params.ctx.Provider,
      to: params.ctx.From,
      text: result.text,
    },
    { kind: "final" },
  );

  return { ok: true, dispatched: true };
}
```

---

## 七、完整消息流转图

```
┌──────────┐                                    ┌──────────┐
│  User A  │                                    │  User B  │
│ (Telegram│                                    │  (WebChat│
│   App)   │                                    │    App)  │
└────┬─────┘                                    └────▲─────┘
     │                                               │
     │ 1. 发送消息                                   │
     ▼                                               │
┌─────────────────────────────────────────────────────────────┐
│                      渠道适配层                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  Telegram Adapter (grammY)                          │   │
│  │  - onMessage: 接收消息                               │   │
│  │  - normalize: 规范化                                 │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
     │
     │ 2. dispatchInboundMessage()
     ▼
┌─────────────────────────────────────────────────────────────┐
│                         Gateway                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  server-methods/chat.ts                              │  │
│  │  - dispatchInboundMessage()                          │  │
│  │  - createReplyDispatcher()                           │  │
│  └──────────────────────────────────────────────────────┘  │
│                          │                                  │
│                          ▼                                  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  auto-reply/dispatch.js                              │  │
│  │  - 检查 autoReply 配置                                │  │
│  │  - 调用 agentCommand()                               │  │
│  └──────────────────────────────────────────────────────┘  │
│                          │                                  │
│                          ▼                                  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  commands/agent.js                                   │  │
│  │  - agentCommandFromIngress()                         │  │
│  │  - 构建 Agent 上下文                                  │  │
│  └──────────────────────────────────────────────────────┘  │
│                          │                                  │
│                          ▼                                  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  agents/agent.ts                                     │  │
│  │  - 调用 LLM Provider                                  │  │
│  │  - 处理工具调用                                      │  │
│  │  - 流式响应                                          │  │
│  └──────────────────────────────────────────────────────┘  │
│                          │                                  │
│                          ▼                                  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Gateway Event Broadcast                             │  │
│  │  - emit("agent", {...})  ← 流式事件                  │  │
│  │  - emit("chat", {...})   ← 最终回复                  │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
     │                                              │
     │ 3. agent 事件 (流式)                          │ 4. chat 事件
     ▼                                              ▼
┌──────────┐                              ┌──────────────┐
│  macOS   │                              │   WebChat    │
│   App    │                              │     UI       │
│          │                              │              │
│  handleAgentEvent()                    │ handleChatEvent()
│  - 更新流式状态                         │  - 添加消息到聊天│
│  - 显示工具调用                         │  - 更新会话列表│
└──────────┘                              └──────────────┘
```

---

## 八、关键代码位置索引

### 8.1 客户端连接

| 功能         | 文件位置                         | 关键函数/类                                       |
| ------------ | -------------------------------- | ------------------------------------------------- |
| Web 客户端   | `ui/src/ui/gateway.ts`           | `GatewayBrowserClient`                            |
| Node 客户端  | `src/gateway/client.ts`          | `GatewayClient`                                   |
| 设备身份     | `src/infra/device-identity.ts`   | `loadOrCreateDeviceIdentity`, `signDevicePayload` |
| 设备认证存储 | `src/infra/device-auth-store.ts` | `loadDeviceAuthToken`, `storeDeviceAuthToken`     |
| 设备配对     | `src/infra/device-pairing.ts`    | `approveDevicePairing`, `listDevicePairing`       |
| TLS 指纹     | `src/infra/tls/fingerprint.ts`   | `normalizeFingerprint`                            |

### 8.2 Gateway 处理

| 功能           | 文件位置                                 | 关键函数/类                                  |
| -------------- | ---------------------------------------- | -------------------------------------------- |
| WebSocket 服务 | `src/gateway/index.ts`                   | (服务器入口)                                 |
| 认证           | `src/gateway/auth.ts`                    | `authorizeGatewayConnect`                    |
| 认证速率限制   | `src/gateway/auth-rate-limit.ts`         | `AuthRateLimiter`                            |
| 协议           | `src/gateway/protocol/index.ts`          | `validateRequestFrame`, `validateEventFrame` |
| 协议 Schema    | `src/gateway/protocol/schema.ts`         | TypeBox schemas                              |
| Agent 处理     | `src/gateway/server-methods/agent.ts`    | `agentHandlers`                              |
| Chat 处理      | `src/gateway/server-methods/chat.ts`     | `chatHandlers`                               |
| 渠道处理       | `src/gateway/server-methods/channels.ts` | `channelsHandlers`                           |
| 会话处理       | `src/gateway/server-methods/sessions.ts` | `sessionsHandlers`                           |
| 事件广播       | `src/gateway/server-broadcast.ts`        | `createGatewayBroadcaster`                   |
| WS 日志        | `src/gateway/ws-log.ts`                  | `formatForLog`                               |

### 8.3 渠道系统

| 功能      | 文件位置                                 | 关键函数/类                                   |
| --------- | ---------------------------------------- | --------------------------------------------- |
| 渠道注册  | `src/channels/registry.ts`               | `CHAT_CHANNEL_ORDER`, `getChatChannelMeta`    |
| 渠道 Dock | `src/channels/dock.ts`                   | `DOCKS`, `listChannelDocks`, `getChannelDock` |
| 插件接口  | `src/channels/plugins/types.ts`          | `Channel*Adapter`                             |
| 插件类型  | `src/channels/plugins/types.core.ts`     | 核心类型定义                                  |
| 插件类型  | `src/channels/plugins/types.adapters.ts` | 适配器接口                                    |
| 插件类型  | `src/channels/plugins/types.plugin.ts`   | `ChannelPlugin`                               |
| 群组策略  | `src/channels/plugins/group-mentions.ts` | `resolve*GroupRequireMention`                 |
| 消息清理  | `src/gateway/chat-sanitize.js`           | `stripEnvelopeFromMessage`                    |

### 8.4 路由与会话

| 功能       | 文件位置                            | 关键函数/类                               |
| ---------- | ----------------------------------- | ----------------------------------------- |
| 会话键     | `src/routing/session-key.ts`        | `resolveSessionKey`, `normalizeAccountId` |
| 会话键工具 | `src/sessions/session-key-utils.ts` | `parseAgentSessionKey`                    |
| 账户 ID    | `src/routing/account-id.ts`         | `DEFAULT_ACCOUNT_ID`                      |
| 会话存储   | `src/config/sessions/store.ts`      | `loadSessionStore`, `updateSessionStore`  |
| 会话类型   | `src/config/sessions/types.ts`      | `SessionEntry`                            |
| 会话工具   | `src/gateway/session-utils.ts`      | `loadSessionEntry`, `readSessionMessages` |

### 8.5 入站分发

| 功能       | 文件位置                                       | 关键函数/类                |
| ---------- | ---------------------------------------------- | -------------------------- |
| 消息分发   | `src/auto-reply/dispatch.ts`                   | `dispatchInboundMessage`   |
| 回复调度器 | `src/auto-reply/reply/reply-dispatcher.ts`     | `createReplyDispatcher`    |
| 配置分发   | `src/auto-reply/reply/dispatch-from-config.ts` | `dispatchReplyFromConfig`  |
| 上下文     | `src/auto-reply/templating.js`                 | `finalizeInboundContext`   |
| 回复前缀   | `src/channels/reply-prefix.ts`                 | `createReplyPrefixOptions` |

---

## 九、架构升级注意事项

### 9.1 依赖关系总结

```
Clients (客户端层)
    │
    │ WebSocket 连接
    │ - 认证 (token/device pairing)
    │ - 请求/响应
    │ - 事件订阅
    ▼
Gateway (控制平面)
    │
    │ 内部调用
    │ - server-methods 处理请求
    │ - 事件广播
    │ - 会话管理
    ▼
Channels (渠道适配层)
    │
    │ 渠道 API 调用
    │ - 入站消息
    │ - 出站消息
    │ - 状态查询
    ▼
External APIs (外部 API)
    - Telegram Bot API
    - WhatsApp Web (Baileys)
    - Discord API
    - Slack API
    - etc.
```

### 9.2 重构升级关键点

#### 9.2.1 客户端层升级

**注意事项**:

1. **协议兼容性** - 修改协议需同步更新所有客户端
   - Swift (macOS/iOS)
   - Kotlin (Android)
   - TypeScript (Web/CLI)

2. **认证流程** - 设备配对流程涉及多端
   - `src/infra/device-identity.ts`
   - `src/infra/device-pairing.ts`
   - 移动端对应实现

3. **事件序号** - 事件丢失检测机制
   - `seq` 字段用于检测事件丢失
   - `onGap` 回调处理

#### 9.2.2 Gateway 层升级

**注意事项**:

1. **server-methods 接口** - 所有请求处理器的统一签名

   ```typescript
   type GatewayRequestHandler = (params: {
     req: RequestFrame;
     params: unknown;
     respond: RespondFn;
     context: GatewayRequestContext;
     client: GatewayClientInfo;
     isWebchatConnect: boolean;
   }) => void | Promise<void>;
   ```

2. **TypeBox Schema** - 所有参数验证使用 TypeBox
   - 定义在 `src/gateway/protocol/schema.ts`
   - 修改方法需同步更新 schema

3. **事件广播** - 事件类型定义在 `src/gateway/server-broadcast.ts`
   - 新增事件需更新客户端订阅逻辑

#### 9.2.3 渠道层升级

**注意事项**:

1. **Dock 接口** - 轻量级接口，避免重依赖
   - `src/channels/dock.ts` 保持轻量
   - 重逻辑放在 `src/channels/plugins/<channel>.ts`

2. **插件注册** - 新增渠道需
   - 在 `extensions/<channel>/` 实现插件
   - 注册到插件系统
   - 更新 `CHAT_CHANNEL_ORDER` (如为核心渠道)

3. **配置 Schema** - 每个渠道有自己的 Zod schema
   - `src/config/zod-schema.providers-*.ts`

### 9.3 性能瓶颈分析

| 瓶颈点             | 当前实现       | 优化方向     |
| ------------------ | -------------- | ------------ |
| WebSocket 事件广播 | 逐个客户端发送 | 引入消息队列 |
| 会话存储           | JSON 文件读写  | 迁移到数据库 |
| 渠道轮询           | 独立轮询器     | 统一调度器   |
| 嵌入生成           | 同步批量处理   | 异步任务队列 |
| 媒体处理           | 本地处理       | 专用媒体服务 |

### 9.4 数据一致性

**会话数据**:

- 存储位置：`~/.openclaw/sessions/<agent>/sessions.json`
- 写入方式：原子写入 (先写 temp 再 rename)
- 并发控制：文件锁 (`acquireFileLock`)

**配置数据**:

- 存储位置：`~/.openclaw/config.json5`
- 缓存策略：内存缓存 + 文件变更监听
- 重载机制：`config-reload.ts`

**凭证数据**:

- 存储位置：`~/.openclaw/credentials/`
- 加密：部分凭证加密存储
- 访问：运行时解密到内存

---

## 十、常见问题排查

### 10.1 客户端连接问题

| 问题       | 可能原因       | 排查方法                      |
| ---------- | -------------- | ----------------------------- |
| 连接被拒绝 | Gateway 未启动 | `openclaw gateway` 检查       |
| 认证失败   | Token 不匹配   | 检查 `OPENCLAW_GATEWAY_TOKEN` |
| 设备未配对 | 新设备需要批准 | `openclaw pairing list` 查看  |
| 事件丢失   | 网络不稳定     | 检查 `seq` gap                |
| 连接静默   | tick 超时      | 检查 `tick timeout` 日志      |

### 10.2 渠道消息问题

| 问题         | 可能原因           | 排查方法                   |
| ------------ | ------------------ | -------------------------- |
| 消息未送达   | allowFrom 阻止     | 检查渠道配置               |
| 群聊无响应   | mentionGating 开启 | 检查是否被@                |
| 渠道断开     | 认证过期           | `openclaw channels status` |
| 媒体发送失败 | 大小超限           | 检查渠道媒体限制           |

### 10.3 Agent 执行问题

| 问题         | 可能原因          | 排查方法                   |
| ------------ | ----------------- | -------------------------- |
| Agent 无响应 | Provider 认证失败 | 检查 `models.authProfiles` |
| 工具执行失败 | 权限不足          | 检查 `system.run` 允许列表 |
| 会话历史丢失 | 存储文件损坏      | 检查 `sessions.json`       |

---

## 十一、总结

### 核心架构原则

1. **单一 Gateway 实例** - 每个主机运行一个 Gateway，统一管理
2. **WebSocket 中心** - 所有通信通过 WS 协议
3. **插件化渠道** - 渠道通过插件系统扩展
4. **会话隔离** - 每个 Agent/会话独立存储
5. **类型安全** - TypeBox schema 定义协议

### 三层职责划分

| 层次         | 职责                         | 关键接口                                |
| ------------ | ---------------------------- | --------------------------------------- |
| **Clients**  | 用户交互、UI 渲染、本地存储  | `GatewayClient`, `GatewayBrowserClient` |
| **Gateway**  | 请求路由、事件广播、会话管理 | `server-methods/*`, `protocol/*`        |
| **Channels** | 渠道适配、消息收发、状态管理 | `Channel*Adapter`, `ChannelDock`        |

### 升级重构优先级

1. **高优先级** - 数据库抽象层、API 网关
2. **中优先级** - 可观测性增强、事件队列优化
3. **低优先级** - 微服务拆分 (需充分测试)

---

**文档生成时间**: 2026-03-04
**最后更新**: 2026-03-04 (深入代码分析)
**参考文档**:

- https://docs.openclaw.ai/gateway
- https://docs.openclaw.ai/channels
- https://docs.openclaw.ai/concepts/agent
- https://docs.openclaw.ai/gateway/protocol
