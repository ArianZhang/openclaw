# Channel、Agent、Chat 三者关系详细分析

**版本**: 2026-03-04
**分析目标**: 深入理解 Channel、Agent、Chat 三者的交互关系、数据流转和潜在问题点

---

## 一、核心概念定义

### 1.1 Channel (渠道)

**定义**: Channel 是 OpenClaw 与外部消息平台（Telegram、WhatsApp、Discord 等）的适配层，负责消息的入站接收和出站发送。

**核心职责**:

- 入站消息接收和规范化
- 出站消息发送
- 渠道特定能力适配（群组策略、@提及、线程支持等）
- 渠道状态管理和健康检查

**关键文件**:

- `src/channels/registry.ts` - 渠道注册表
- `src/channels/dock.ts` - 渠道 Dock（轻量级能力声明）
- `src/channels/plugins/types.ts` - 渠道插件接口定义
- `src/channels/plugins/types.adapters.ts` - 适配器接口
- `src/infra/outbound/deliver.ts` - 出站消息交付

### 1.2 Agent (智能体)

**定义**: Agent 是基于 LLM 的智能处理单元，负责理解用户输入、调用工具、生成回复。

**核心职责**:

- 接收用户消息并生成智能回复
- 调用外部工具（bash、API、文件操作等）
- 流式输出响应内容
- 管理会话状态和上下文

**关键文件**:

- `src/commands/agent.ts` - Agent 命令入口
- `src/agents/agent.ts` - Agent 核心逻辑
- `src/auto-reply/reply/agent-runner.ts` - Agent 运行器
- `src/infra/agent-events.ts` - Agent 事件发射

### 1.3 Chat (聊天)

**定义**: Chat 是用户与 Agent 之间的会话抽象，包括消息历史、会话状态和交付机制。

**核心职责**:

- 会话状态管理（会话键、历史记录）
- 消息交付调度（tool/block/final 三种类型）
- 输入/输出消息的规范化
- 与 Gateway 的事件同步

**关键文件**:

- `src/gateway/server-methods/chat.ts` - Chat RPC 处理
- `src/auto-reply/reply/reply-dispatcher.ts` - 回复调度器
- `src/auto-reply/dispatch.ts` - 入站消息分发
- `src/config/sessions/` - 会话存储

---

## 二、三层架构关系图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           外部消息平台                                       │
│   Telegram    WhatsApp    Discord    Slack    Signal    iMessage    ...     │
└───────┬───────────┬───────────┬───────────┬───────────┬───────────┬─────────┘
        │           │           │           │           │           │
        │  onMessage()         │           │           │           │
        ▼           ▼           │           │           │           │
┌───────────────────────────────┼─────────────────────────────────────────────┐
│                         Channel 层 (渠道适配层)                               │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │  入站处理 (Inbound)                                                      ││
│  │  - 消息接收 (grammY/Baileys/discord.js/...)                             ││
│  │  - 消息规范化 (normalizeChannelMessage)                                  ││
│  │  - 访问控制检查 (allowFrom/mentionGating)                                ││
│  │  - 生成 MsgContext                                                       ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │  出站处理 (Outbound)                                                     ││
│  │  - loadChannelOutboundAdapter()                                          ││
│  │  - deliverOutboundPayloads()                                             ││
│  │  - sendText/sendMedia/sendPoll                                           ││
│  └─────────────────────────────────────────────────────────────────────────┘│
└───────────────────────────────┬─────────────────────────────────────────────┘
                                │
                                │ dispatchInboundMessage()
                                │ MsgContext: { Body, Provider, From, To, ... }
                                ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         Auto-Reply 层 (自动回复调度)                          │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │  dispatch.ts                                                             ││
│  │  - finalizeInboundContext()                                              ││
│  │  - dispatchReplyFromConfig()                                             ││
│  │  - withReplyDispatcher()                                                 ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │  reply-dispatcher.ts (ReplyDispatcher)                                   ││
│  │  - sendToolResult(payload)  →  enqueue("tool", payload)                 ││
│  │  - sendBlockReply(payload)  →  enqueue("block", payload)                ││
│  │  - sendFinalReply(payload)  →  enqueue("final", payload)                ││
│  │  - deliver: options.deliver(normalized, { kind })                        ││
│  └─────────────────────────────────────────────────────────────────────────┘│
└───────────────────────────────┬─────────────────────────────────────────────┘
                                │
                                │ getReplyFromConfig()
                                ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          Agent 层 (智能处理层)                                │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │  get-reply.ts                                                            ││
│  │  - initSessionState() - 初始化会话状态                                   ││
│  │  - resolveReplyDirectives() - 解析指令 (/think, /verbose, ...)           ││
│  │  - handleInlineActions() - 处理内联动作                                  ││
│  │  - runPreparedReply() - 执行准备好的回复                                 ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │  agent-runner.ts / agent-runner-execution.ts                             ││
│  │  - runAgentTurnWithFallback() - 运行 Agent 轮次 (带 fallback)             ││
│  │  - buildReplyPayloads() - 构建回复负载                                   ││
│  │  - createBlockReplyPipeline() - 创建流式回复管道                         ││
│  │  - emitAgentEvent() - 发射 Agent 事件 (流式输出)                          ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │  commands/agent.ts                                                       ││
│  │  - agentCommandFromIngress() - Agent 命令入口                             ││
│  │  - runAgentAttempt() - 运行 Agent 尝试                                    ││
│  │  - runWithModelFallback() - 模型 fallback                                 ││
│  │  - runCliAgent() / runEmbeddedPiAgent()                                  ││
│  └─────────────────────────────────────────────────────────────────────────┘│
└───────────────────────────────┬─────────────────────────────────────────────┘
                                │
                                │ ReplyPayload[]: { text, mediaUrl, mediaUrls }
                                │ onBlockReply(payload) / onToolResult(payload)
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                      Chat 交付层 (消息交付)                                   │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │  dispatch-from-config.ts                                                 ││
│  │  - onBlockReply: dispatcher.sendBlockReply(ttsPayload)                   ││
│  │  - onToolResult: dispatcher.sendToolResult(deliveryPayload)              ││
│  │  - 最终回复：dispatcher.sendFinalReply(ttsReply)                         ││
│  │  - routeReply(): 路由到起始渠道                                          ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │  reply-dispatcher.ts (enqueue 函数)                                      ││
│  │  - 序列化发送链: sendChain = sendChain.then(async () => {...})          ││
│  │  - 标准化负载：normalizeReplyPayload(payload)                            ││
│  │  - 错误处理：options.onError(err, { kind })                              ││
│  │  - 空闲检测：pending === 0 → options.onIdle()                            ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │  route-reply.ts                                                          ││
│  │  - deliverOutboundPayloads()                                             ││
│  │  - 镜像到会话记录：mirror: { sessionKey, agentId, text, mediaUrls }      ││
│  └─────────────────────────────────────────────────────────────────────────┘│
└───────────────────────────────┬─────────────────────────────────────────────┘
                                │
                                │ loadChannelOutboundAdapter()
                                │ createPluginHandler()
                                ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                      Channel 出站层 (最终发送)                                │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │  infra/outbound/deliver.ts                                               ││
│  │  - createChannelHandler() - 创建渠道处理器                               ││
│  │  - createPluginHandler() - 创建插件处理器                                ││
│  │  - chunkMarkdownTextWithMode() - 文本分块                                ││
│  │  - 消息钩子：toPluginMessageSentEvent()                                  ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │  渠道特定发送器                                                           ││
│  │  - telegram/send.ts: sendMessageTelegram()                              ││
│  │  - discord/send.ts: sendMessageDiscord()                                ││
│  │  - slack/send.ts: sendMessageSlack()                                    ││
│  │  - signal/send.ts: sendMessageSignal()                                  ││
│  │  - imessage/send.ts: sendMessageIMessage()                              ││
│  │  - web/outbound.ts: sendMessageWhatsApp()                               ││
│  └─────────────────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           外部消息平台                                       │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 三、入站消息流转详解

### 3.1 完整入站流程

```
1. Channel 接收消息
   │
   │ Telegram: bot.on("message", handler)
   │ Discord: client.on("messageCreate", handler)
   │ WhatsApp: Baileys socket.on("messages.upsert", handler)
   │
   ▼
2. 渠道适配器规范化消息
   │
   │  telegram/monitor.ts:
   │    const ctx: MsgContext = {
   │      Body: text,
   │      BodyForAgent: stampedMessage,
   │      Provider: "telegram",
   │      From: chatId,
   │      To: botId,
   │      AccountId: accountId,
   │      ChatType: "direct" | "group",
   │      MessageThreadId: threadId,
   │      SessionKey: resolvedSessionKey,
   │      ...
   │    }
   │
   ▼
3. 访问控制检查
   │
   │  - resolveChannelDock(channelId)
   │  - dock.config.resolveAllowFrom({ cfg, accountId })
   │  - 检查 senderId 是否在 allowFrom 列表中
   │  - 群聊检查：dock.groups.resolveRequireMention(ctx)
   │
   ▼
4. 生成会话键 (SessionKey)
   │
   │  buildPeerSessionKey({
   │    agentId,
   │    channel: "telegram",
   │    peerId: chatId,
   │    peerKind: "direct" | "group",
   │    accountId
   │  })
   │  → "agent:main:telegram:direct:123456789"
   │
   ▼
5. 调用 dispatchInboundMessage()
   │
   │  src/auto-reply/dispatch.ts:35
   │  export async function dispatchInboundMessage(params: {
   │    ctx: MsgContext | FinalizedMsgContext;
   │    cfg: OpenClawConfig;
   │    dispatcher: ReplyDispatcher;
   │    replyOptions?: GetReplyOptions;
   │  })
   │
   ▼
6. 完成上下文 (finalizeInboundContext)
   │
   │  - 标准化所有字段
   │  - 处理 GroupResolution
   │  - 确定 SessionKey
   │
   ▼
7. 创建 ReplyDispatcher
   │
   │  createReplyDispatcher({
   │    deliver: async (payload, info) => {
   │      // 实际发送逻辑
   │      await routeReply({ payload, channel, to, cfg });
   │    }
   │  })
   │
   ▼
8. 调度回复 (dispatchReplyFromConfig)
   │
   │  - 触发 Hook: message_received
   │  - 检查 fast abort (停止命令)
   │  - 检查 sendPolicy
   │  - 调用 getReplyFromConfig()
   │
   ▼
9. getReplyFromConfig() → Agent 处理
   │
   │  - initSessionState()
   │  - resolveReplyDirectives()
   │  - handleInlineActions()
   │  - runPreparedReply()
   │
   ▼
10. Agent 生成回复
    │
    │  - runAgentTurnWithFallback()
    │  - emitAgentEvent() 发射流式事件
    │  - onBlockReply(payload) → dispatcher.sendBlockReply()
    │  - onToolResult(payload) → dispatcher.sendToolResult()
    │
    ▼
11. 调度器交付消息
    │
    │  enqueue(kind, payload):
    │    sendChain = sendChain.then(async () => {
    │      const normalized = normalizeReplyPayload(payload, {...});
    │      await options.deliver(normalized, { kind });
    │    });
    │
    ▼
12. routeReply() 路由到渠道
    │
    │   - normalizeReplyPayload(payload)
    │   - deliverOutboundPayloads({
    │       cfg, channel, to, payloads,
    │       mirror: { sessionKey, text, mediaUrls }
    │     })
    │
    ▼
13. Channel 出站发送
    │
    │   loadChannelOutboundAdapter(channel)
    │   → outbound.sendText({ cfg, to, text, accountId })
    │
    ▼
14. 消息钩子
    │
    │   fireAndForgetHook(
    │     hookRunner.runMessageSent(...),
    │     "message_sent hook failed"
    │   );
```

### 3.2 MsgContext 结构

```typescript
type MsgContext = {
  // 消息内容
  Body: string; // 原始消息体
  BodyForAgent: string; // 给 Agent 的消息体（可能包含时间戳）
  BodyForCommands: string; // 给命令处理器的消息体
  RawBody: string; // 原始未处理消息
  CommandBody: string; // 命令体

  // 会话标识
  SessionKey: string; // 会话键
  MessageSid: string; // 消息 ID
  MessageSidFull?: string;
  MessageSidFirst?: string;
  MessageSidLast?: string;

  // 渠道信息
  Provider: string; // 渠道标识 (telegram, discord, ...)
  Surface: string; // 表面渠道 (可能与 Provider 不同)
  OriginatingChannel?: string; // 起始渠道
  OriginatingTo?: string; // 起始目标

  // 参与者信息
  From: string; // 发送者 ID
  To: string; // 接收者 ID
  AccountId: string; // 渠道账户 ID
  ChatType: "direct" | "group"; // 聊天类型
  GroupChannel?: string; // 群组渠道
  GroupSubject?: string; // 群组主题

  // 线程信息
  MessageThreadId?: string; // 消息线程 ID
  ReplyToId?: string; // 回复目标 ID

  // 媒体信息
  MediaType?: string;
  MediaTypes?: string[];
  MediaUrls?: string[];

  // 命令信息
  CommandSource: "native" | "text";
  CommandAuthorized: boolean;
  CommandTargetSessionKey?: string;

  // 时间戳
  Timestamp?: number;
};
```

---

## 四、Agent 响应生成详解

### 4.1 Agent 响应生成流程

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  runPreparedReply()                                                          │
│                                                                               │
│  1. 准备 Agent 运行参数                                                         │
│     - provider, model (可能带 fallback)                                        │
│     - thinkLevel, verboseLevel, reasoningLevel                                │
│     - sessionCtx (系统提示、历史消息)                                           │
│                                                                               │
│  2. 创建 Block Reply Pipeline (如果启用流式)                                   │
│     createBlockReplyPipeline({                                               │
│       onBlockReply,                                                          │
│       timeoutMs: 15000,                                                       │
│       coalescing: { minChars: 1500, idleMs: 1000 }                           │
│     })                                                                        │
│                                                                               │
│  3. 运行 Agent 轮次                                                              │
│     runAgentTurnWithFallback({                                               │
│       provider, model,                                                       │
│       onBlockReply: (payload) => {                                           │
│         blockReplyPipeline.process(payload);                                 │
│       },                                                                     │
│       onToolResult: (payload) => {                                           │
│         opts.onToolResult?.(payload);                                        │
│       }                                                                      │
│     })                                                                        │
│                                                                               │
│  4. Agent 内部事件发射 (agent-runner-execution.ts)                              │
│     emitAgentEvent({                                                         │
│       runId,                                                                 │
│       stream: "assistant" | "tool" | "lifecycle" | "error",                  │
│       data: { ... }                                                          │
│     });                                                                       │
│                                                                               │
│  5. 构建最终回复 Payload                                                       │
│     buildReplyPayloads({                                                     │
│       assistantMessage,                                                      │
│       toolOutputs,                                                           │
│       usage                                                                  │
│     })                                                                        │
│                                                                               │
│  6. 刷新 Block Pipeline                                                       │
│     blockReplyPipeline?.flush();                                             │
│                                                                               │
│  7. 返回 ReplyPayload[]                                                        │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4.2 Agent 事件类型

```typescript
// src/infra/agent-events.ts

type AgentEventStream =
  | "lifecycle" // Agent 生命周期事件 (start, model-selected, ...)
  | "tool" // 工具调用事件 (tool-input, tool-output, ...)
  | "assistant" // 助手响应事件 (content-delta, reasoning, ...)
  | "error" // 错误事件
  | (string & {}); // 扩展类型

type AgentEventPayload = {
  runId: string;
  seq: number; // 每运行递增的序列号
  stream: AgentEventStream;
  ts: number; // 时间戳
  data: Record<string, unknown>;
  sessionKey?: string;
};

// 事件发射
export function emitAgentEvent(event: Omit<AgentEventPayload, "seq" | "ts">) {
  const nextSeq = (seqByRun.get(event.runId) ?? 0) + 1;
  seqByRun.set(event.runId, nextSeq);

  const enriched: AgentEventPayload = {
    ...event,
    seq: nextSeq,
    ts: Date.now(),
  };

  // 通知所有监听器 (包括 Gateway 广播)
  for (const listener of listeners) {
    try {
      listener(enriched);
    } catch {
      /* ignore */
    }
  }
}
```

### 4.3 Gateway 事件广播

```typescript
// src/gateway/server-chat.ts

// Agent 事件广播给 WebSocket 客户端
const broadcast = createGatewayBroadcaster({ clients });

// 监听 Agent 事件并广播
onAgentEvent((evt) => {
  const context = getAgentRunContext(evt.runId);

  // 广播到所有订阅的客户端
  broadcast("agent", {
    runId: evt.runId,
    seq: evt.seq,
    stream: evt.stream,
    data: evt.data,
    sessionKey: evt.sessionKey,
  });
});

// Chat 事件广播 (消息发送/接收)
broadcast("chat", {
  runId,
  sessionKey,
  seq,
  state: "final" | "error",
  message: {...},
});
```

---

## 五、Chat 交付层详解

### 5.1 ReplyDispatcher 工作原理

```typescript
// src/auto-reply/reply/reply-dispatcher.ts

export function createReplyDispatcher(options: ReplyDispatcherOptions): ReplyDispatcher {
  let sendChain: Promise<void> = Promise.resolve();  // 序列化发送链
  let pending = 1;  //  reservations (初始为 1，防止过早空闲)
  let completeCalled = false;
  let sentFirstBlock = false;  // 用于人类延迟

  const queuedCounts: Record<ReplyDispatchKind, number> = {
    tool: 0,
    block: 0,
    final: 0,
  };

  const enqueue = (kind: ReplyDispatchKind, payload: ReplyPayload) => {
    // 1. 标准化负载 (添加 responsePrefix, 处理 heartbeat 等)
    const normalized = normalizeReplyPayloadInternal(payload, {...});
    if (!normalized) {
      return false;  // 空负载被跳过
    }

    queuedCounts[kind] += 1;
    pending += 1;

    // 2. 确定是否需要人类延迟 (仅 block 回复，第一个除外)
    const shouldDelay = kind === "block" && sentFirstBlock;
    if (kind === "block") {
      sentFirstBlock = true;
    }

    // 3. 序列化到发送链
    sendChain = sendChain
      .then(async () => {
        if (shouldDelay) {
          const delayMs = getHumanDelay(options.humanDelay);  // 800-2500ms 随机
          await sleep(delayMs);
        }
        await options.deliver(normalized, { kind });
      })
      .catch((err) => {
        options.onError?.(err, { kind });
      })
      .finally(() => {
        pending -= 1;
        if (pending === 0) {
          unregister();  // 从全局追踪中移除
          options.onIdle?.();  // 通知空闲
        }
      });

    return true;
  };

  return {
    sendToolResult: (payload) => enqueue("tool", payload),
    sendBlockReply: (payload) => enqueue("block", payload),
    sendFinalReply: (payload) => enqueue("final", payload),
    waitForIdle: () => sendChain,
    getQueuedCounts: () => ({ ...queuedCounts }),
    markComplete: () => { /* 清除 reservation */ },
  };
}
```

### 5.2 三种交付类型对比

| 类型    | 触发时机         | 用途          | 是否阻塞链 | 人类延迟      |
| ------- | ---------------- | ------------- | ---------- | ------------- |
| `tool`  | Agent 调用工具后 | 工具结果/输出 | 是         | 否            |
| `block` | 流式响应块       | 流式文本分块  | 是         | 是 (第二个起) |
| `final` | Agent 完成       | 最终完整回复  | 是         | 否            |

### 5.3 Block Reply Pipeline

```typescript
// src/auto-reply/reply/block-reply-pipeline.ts

export function createBlockReplyPipeline(params: {
  onBlockReply: (payload: ReplyPayload, context?: { abortSignal?: AbortSignal }) => void;
  timeoutMs: number;
  coalescing?: {
    minChars?: number;
    idleMs?: number;
  };
}) {
  let buffer = "";
  let flushTimer: NodeJS.Timeout | null = null;
  const { minChars = 1500, idleMs = 1000 } = coalescing ?? {};

  const flush = () => {
    if (buffer.length < minChars) return;

    // 按段落/句子/换行分割
    const chunks = chunkByParagraph(buffer, {
      maxChars: coalescing?.maxChars,
      breakPreference: "paragraph",
    });

    if (chunks.length > 0) {
      buffer = chunks.pop() ?? ""; // 保留最后一个不完整块
      for (const chunk of chunks) {
        params.onBlockReply({ text: chunk });
      }
    }

    resetTimer();
  };

  const resetTimer = () => {
    if (flushTimer) clearTimeout(flushTimer);
    flushTimer = setTimeout(() => {
      flush();
      if (buffer.trim()) {
        params.onBlockReply({ text: buffer });
        buffer = "";
      }
    }, idleMs);
  };

  return {
    process: (payload: ReplyPayload) => {
      if (payload.text) {
        buffer += payload.text;
        resetTimer();

        // 如果积累足够，立即刷新
        if (buffer.length >= minChars * 2) {
          flush();
        }
      }
    },
    flush: () => {
      if (flushTimer) clearTimeout(flushTimer);
      if (buffer.trim()) {
        params.onBlockReply({ text: buffer });
        buffer = "";
      }
    },
  };
}
```

---

## 六、出站消息交付详解

### 6.1 routeReply() 流程

```typescript
// src/auto-reply/reply/route-reply.ts

export async function routeReply(params: RouteReplyParams): Promise<RouteReplyResult> {
  const { payload, channel, to, accountId, threadId, cfg } = params;

  // 1. 抑制 reasoning payload (某些渠道不支持)
  if (shouldSuppressReasoningPayload(payload)) {
    return { ok: true };
  }

  // 2. 解析 Agent ID (多 Agent 场景)
  const resolvedAgentId = params.sessionKey
    ? resolveSessionAgentId({ sessionKey: params.sessionKey, config: cfg })
    : undefined;

  // 3. 标准化负载 (应用 responsePrefix 等)
  const normalized = normalizeReplyPayload(payload, {
    responsePrefix: resolveEffectiveMessagesConfig(cfg, resolvedAgentId, {...}).responsePrefix,
  });

  if (!normalized) {
    return { ok: true };  // 空负载
  }

  // 4. 检查是否为空回复
  if (!normalized.text.trim() && (normalized.mediaUrls?.length ?? 0) === 0) {
    return { ok: true };
  }

  // 5. 检查渠道是否可路由
  if (channel === INTERNAL_MESSAGE_CHANNEL) {
    return { ok: false, error: "Webchat routing not supported" };
  }

  const channelId = normalizeChannelId(channel);
  if (!channelId) {
    return { ok: false, error: `Unknown channel: ${channel}` };
  }

  // 6. 交付出站负载
  const { deliverOutboundPayloads } = await import("../../infra/outbound/deliver.js");

  const results = await deliverOutboundPayloads({
    cfg,
    channel: channelId,
    to,
    accountId,
    payloads: [normalized],
    replyToId,
    threadId,
    session: buildOutboundSessionContext({...}),
    mirror: params.mirror !== false ? {
      sessionKey: params.sessionKey,
      agentId: resolvedAgentId,
      text: normalized.text,
      mediaUrls: normalized.mediaUrls,
    } : undefined,
  });

  return { ok: true, messageId: results.at(-1)?.messageId };
}
```

### 6.2 deliverOutboundPayloads() 详解

```typescript
// src/infra/outbound/deliver.ts

export async function deliverOutboundPayloads(params: {
  cfg: OpenClawConfig;
  channel: OutboundChannel;
  to: string;
  accountId?: string;
  payloads: NormalizedOutboundPayload[];
  replyToId?: string | null;
  threadId?: string | number | null;
  session?: OutboundSessionContext;
  mirror?: { sessionKey: string; agentId: string; text: string; mediaUrls: string[] };
  abortSignal?: AbortSignal;
}): Promise<OutboundDeliveryResult[]> {

  // 1. 创建渠道处理器
  const handler = await createChannelHandler({
    cfg, channel, to, accountId, replyToId, threadId,
  });

  const results: OutboundDeliveryResult[] = [];

  // 2. 序列化发送每个 payload
  for (const payload of params.payloads) {
    throwIfAborted(params.abortSignal);

    // 3. 如果有 sendPayload，直接使用
    if (handler.sendPayload) {
      const result = await handler.sendPayload(payload, { replyToId, threadId });
      results.push(result);
      continue;
    }

    // 4. 处理媒体 + 文本
    const hasMedia = payload.mediaUrl || (payload.mediaUrls?.length ?? 0) > 0;

    if (hasMedia) {
      // 发送媒体（带标题）
      const result = await handler.sendMedia(
        payload.text ?? "",
        payload.mediaUrl || payload.mediaUrls![0],
        { replyToId, threadId }
      );
      results.push(result);

      // 如果有额外媒体，继续发送
      for (const extraUrl of (payload.mediaUrls?.slice(1) ?? [])) {
        const extraResult = await handler.sendMedia("", extraUrl, { replyToId, threadId });
        results.push(extraResult);
      }
    } else if (payload.text) {
      // 5. 纯文本发送（可能需要分块）
      const limit = handler.textChunkLimit ??
                    resolveTextChunkLimit(channel, cfg) ??
                    4096;

      const chunker = handler.chunker ??
                     (channel === "telegram" ? chunkByParagraph : null);

      if (chunker) {
        const chunks = chunker(payload.text, limit);
        for (const chunk of chunks) {
          const result = await handler.sendText(chunk, { replyToId, threadId });
          results.push(result);
        }
      } else {
        const result = await handler.sendText(payload.text, { replyToId, threadId });
        results.push(result);
      }
    }
  }

  // 6. 镜像到会话记录（如果启用）
  if (params.mirror) {
    await appendAssistantMessageToSessionTranscript({
      storePath: resolveStorePath(cfg.session?.store, { agentId: params.mirror.agentId }),
      sessionKey: params.mirror.sessionKey,
      text: params.mirror.text,
      mediaUrls: params.mirror.mediaUrls,
    });
  }

  // 7. 触发 message_sent 钩子
  const hookRunner = getGlobalHookRunner();
  if (hookRunner?.hasHooks("message_sent")) {
    fireAndForgetHook(
      hookRunner.runMessageSent(
        toPluginMessageSentEvent({...}),
        toPluginMessageContext({...})
      ),
      "message_sent hook failed"
    );
  }

  return results;
}
```

### 6.3 ChannelOutboundAdapter 接口

```typescript
// src/channels/plugins/types.adapters.ts

export type ChannelOutboundAdapter = {
  deliveryMode: "direct" | "gateway" | "hybrid";

  // 文本分块配置
  chunker?: ((text: string, limit: number) => string[]) | null;
  chunkerMode?: "text" | "markdown";
  textChunkLimit?: number;

  // 投票限制
  pollMaxOptions?: number;

  // 目标解析
  resolveTarget?: (params: {
    cfg?: OpenClawConfig;
    to?: string;
    allowFrom?: string[];
    accountId?: string | null;
    mode?: ChannelOutboundTargetMode;
  }) => { ok: true; to: string } | { ok: false; error: Error };

  // 发送方法
  sendPayload?: (ctx: ChannelOutboundPayloadContext) => Promise<OutboundDeliveryResult>;
  sendText?: (ctx: ChannelOutboundContext) => Promise<OutboundDeliveryResult>;
  sendMedia?: (ctx: ChannelOutboundContext) => Promise<OutboundDeliveryResult>;
  sendPoll?: (ctx: ChannelPollContext) => Promise<ChannelPollResult>;
};

export type ChannelOutboundContext = {
  cfg: OpenClawConfig;
  to: string;
  text: string;
  mediaUrl?: string;
  mediaLocalRoots?: readonly string[];
  gifPlayback?: boolean;
  replyToId?: string | null;
  threadId?: string | number | null;
  accountId?: string | null;
  identity?: OutboundIdentity;
  deps?: OutboundSendDeps;
  silent?: boolean;
};
```

---

## 七、会话状态管理

### 7.1 SessionKey 结构

```typescript
// src/routing/session-key.ts

// 格式：agent:<agentId>:<rest>

// 主会话
"agent:main:main";

// 渠道特定会话
"agent:main:telegram:direct:123456789";
"agent:main:discord:group:987654321";
"agent:main:slack:thread:threadId";

// 线程会话
"agent:main:telegram:direct:123456789:thread:replyToId";

// 解析函数
function parseAgentSessionKey(sessionKey: string): ParsedAgentSessionKey | null {
  if (!sessionKey.startsWith("agent:")) return null;
  const parts = sessionKey.split(":");
  if (parts.length < 3) return null;
  const agentId = parts[1];
  const rest = parts.slice(2).join(":");
  return { agentId, rest };
}

// 从会话键解析 Agent ID
function resolveAgentIdFromSessionKey(sessionKey: string): string {
  const parsed = parseAgentSessionKey(sessionKey);
  return normalizeAgentId(parsed?.agentId ?? DEFAULT_AGENT_ID);
}
```

### 7.2 SessionEntry 结构

```typescript
// src/config/sessions/types.ts

type SessionEntry = {
  sessionId: string;
  updatedAt: number;

  // 渠道信息
  channel?: string;
  chatType?: "direct" | "group" | "channel" | "thread";
  lastTo?: string;
  lastAccountId?: string;
  lastThreadId?: string;

  // 交付上下文
  deliveryContext?: {
    channel: string;
    to: string;
    accountId: string;
    threadId: string;
  };

  // 发送策略
  sendPolicy?: "allow" | "deny";

  // 模型配置
  modelOverride?: string;
  providerOverride?: string;

  // 思考/详细级别
  thinkingLevel?: "low" | "medium" | "high";
  verboseLevel?: "off" | "minimal" | "normal" | "full";
  reasoningLevel?: "low" | "medium" | "high";

  // 技能快照
  skillsSnapshot?: string;

  // 生成信息
  label?: string;
  spawnedBy?: string;
  spawnDepth?: number;

  // 群组信息
  groupId?: string;
  groupChannel?: string;
  space?: string;

  // TTS 配置
  ttsAuto?: "off" | "on" | "auto";

  // CLI 会话 ID
  cliSessionIds?: Record<string, string>;
  claudeCliSessionId?: string;
};
```

### 7.3 会话更新流程

```typescript
// Agent 运行后更新会话

// 1. 更新内存中的会话存储
activeSessionEntry.updatedAt = Date.now();
activeSessionEntry.channel = resolvedChannel;
activeSessionEntry.lastTo = resolvedTo;
activeSessionEntry.lastAccountId = resolvedAccountId;

// 2. 持久化到文件系统
await updateSessionStoreEntry({
  storePath,
  sessionKey,
  update: async () => ({
    updatedAt: Date.now(),
    channel: resolvedChannel,
    lastTo: resolvedTo,
    lastAccountId: resolvedAccountId,
    modelOverride: selectedModel?.model,
    providerOverride: selectedModel?.provider,
  }),
});

// 3. 记录运行使用情况
await persistRunSessionUsage({
  sessionKey,
  storePath,
  usage: runUsage,
  compactionCount: runCompactionCount,
});
```

---

## 八、关键点总结

### 8.1 数据流关键点

| 阶段         | 关键函数                      | 输入                     | 输出                       |
| ------------ | ----------------------------- | ------------------------ | -------------------------- |
| Channel 入站 | `onMessage()`                 | 原始消息                 | `MsgContext`               |
| 访问控制     | `resolveAllowFrom()`          | `MsgContext`, `cfg`      | boolean                    |
| 分发入口     | `dispatchInboundMessage()`    | `MsgContext`, `cfg`      | `DispatchInboundResult`    |
| 回复生成     | `getReplyFromConfig()`        | `MsgContext`, `opts`     | `ReplyPayload[]`           |
| Agent 执行   | `runAgentTurnWithFallback()`  | provider, model, body    | stream events              |
| 事件发射     | `emitAgentEvent()`            | `AgentEventPayload`      | (broadcast)                |
| 交付调度     | `dispatcher.sendBlockReply()` | `ReplyPayload`           | queued                     |
| 出站交付     | `deliverOutboundPayloads()`   | payloads, channel        | `OutboundDeliveryResult[]` |
| 渠道发送     | `outbound.sendText()`         | `ChannelOutboundContext` | `OutboundDeliveryResult`   |

### 8.2 三种回复类型的交付差异

```
Tool Result:
  Agent → onToolResult() → dispatcher.sendToolResult()
    → enqueue("tool") → normalizeReplyPayload()
    → deliver() → routeReply() → deliverOutboundPayloads()
    → outbound.sendText()/sendMedia()

Block Reply:
  Agent → onBlockReply() → dispatcher.sendBlockReply()
    → enqueue("block") → [可能的人类延迟 800-2500ms]
    → normalizeReplyPayload() → deliver() → ...

Final Reply:
  Agent 完成 → replies 数组 → dispatcher.sendFinalReply()
    → enqueue("final") → normalizeReplyPayload() → deliver() → ...
```

### 8.3 流式 vs 非流式

**流式模式** (blockStreamingEnabled = true):

```
Agent 流式输出 → Block Reply Pipeline (累积/分块)
  → 多个 sendBlockReply() → 渠道逐块发送
  → 最终 sendFinalReply() 可能为空 (已被 block 消费)
```

**非流式模式** (blockStreamingEnabled = false):

```
Agent 完整响应 → buildReplyPayloads()
  → 单个 sendFinalReply() → 渠道一次性发送
```

---

## 九、潜在问题点分析

### 9.1 性能瓶颈

| 问题点              | 位置                                      | 影响              | 优化方向              |
| ------------------- | ----------------------------------------- | ----------------- | --------------------- |
| 序列化发送          | `reply-dispatcher.ts:enqueue()`           | 阻塞后续回复      | 考虑并行交付 (同渠道) |
| Block Pipeline 超时 | `block-reply-pipeline.ts:timeoutMs=15000` | 长文本延迟        | 动态调整超时时间      |
| 人类延迟累积        | 多个 block 各延迟 800-2500ms              | 长回复显著延迟    | 可配置总延迟上限      |
| 会话存储锁竞争      | `updateSessionStore()`                    | 高并发时阻塞      | 批量写入/异步队列     |
| Hook 同步阻塞       | `fireAndForgetHook()`                     | Hook 失败影响交付 | 已用 fire-and-forget  |

### 9.2 数据一致性问题

| 问题                       | 场景                                  | 可能后果     | 缓解措施                         |
| -------------------------- | ------------------------------------- | ------------ | -------------------------------- |
| 交付成功但镜像失败         | `mirror: true` 时 transcript 写入失败 | 会话记录丢失 | 已 try-catch 记录日志            |
| 部分交付成功               | 多 payload 循环中第 N 个失败          | 消息不完整   | 序列化保证顺序                   |
| sessionKey 解析错误        | malformed session key                 | 会话隔离失效 | `classifySessionKeyShape()` 检查 |
| Channel 切换导致状态不同步 | `OriginatingChannel` vs `Surface`     | 回复路由错误 | 明确优先级：Provider > Surface   |

### 9.3 渠道特异性问题

| 渠道     | 限制                         | 处理                                      |
| -------- | ---------------------------- | ----------------------------------------- |
| Telegram | 文本 4096 字符限制           | `chunkByParagraph`, `TELEGRAM_TEXT_LIMIT` |
| Discord  | 2000 字符限制，Markdown 差异 | `chunkerMode: "markdown"`, limit 2000     |
| Slack    | Thread 回复需 thread_ts      | `threadId` 参数传递                       |
| WhatsApp | 媒体大小限制                 | `resolveChannelMediaMaxBytes()`           |
| Signal   | 富文本格式差异               | `markdownToSignalTextChunks()`            |

### 9.4 竞态条件

```
场景 1: 快速连续消息
  Message A → dispatch → getReplyFromConfig → Agent Run A
  Message B (100ms later) → dispatch → getReplyFromConfig → Agent Run B

  问题：Run B 可能读取 Run A 未持久化的会话状态

  现状：通过 per-session-key queue 限制 (queue.js)
  风险：queue debounce 配置不当仍可能竞态

场景 2: Block Reply 与 Final Reply 顺序
  Block 1 → enqueue → pending++
  Block 2 → enqueue → pending++
  Final → enqueue → pending++

  问题：如果 Block 1 失败，Block 2 和 Final 仍会执行

  现状：.catch(onError) 但不中断链
  风险：渠道可能收到乱序/不完整消息

场景 3: Agent 事件广播与交付不同步
  emitAgentEvent("assistant", { text: "..." }) → broadcast to clients
  onBlockReply({ text: "..." }) → send to channel

  问题：客户端可能先收到事件，渠道后收到消息
  现状：独立路径，无同步保证
  风险：UI 显示与实际消息不同步
```

### 9.5 错误恢复

| 错误类型     | 当前处理                        | 改进建议                 |
| ------------ | ------------------------------- | ------------------------ |
| 渠道发送失败 | 记录日志，继续下一 payload      | 重试机制 (指数退避)      |
| Agent 超时   | `runWithModelFallback` 切换模型 | 保留部分响应而非完全失败 |
| 会话存储损坏 | catch 并忽略                    | 告警 + 自动备份恢复      |
| Hook 失败    | fire-and-forget + 日志          | 可选的 hook 失败阻断     |
| Mirror 失败  | 记录日志                        | 异步重试队列             |

---

## 十、优化建议

### 10.1 架构优化

```
建议 1: 交付队列优先级
  当前：所有 payload 按 FIFO 顺序发送
  建议：tool > final > block (确保工具结果优先显示)

建议 2: 渠道交付并行化
  当前：同一 dispatcher 内严格串行
  建议：不同渠道的 payload 可并行交付

建议 3: 会话状态写缓冲
  当前：每次 Agent run 后同步写入
  建议：批量写入 (如 100ms 窗口) + 定期 flush
```

### 10.2 可观测性增强

```
建议 1: 交付追踪 ID
  当前：payload 无唯一标识
  建议：每个 payload 生成 traceId，贯穿 dispatch → deliver → channel

建议 2: 渠道交付指标
  当前：仅基础日志
  建议：per-channel 成功率、延迟直方图、错误分类

建议 3: Agent 响应分解
  当前：笼统的 "Agent 响应时间"
  建议：分解为 directive 解析/模型调用/工具执行/交付时间
```

### 10.3 配置优化

```
建议 1: 渠道级 deliveryMode 配置
  当前：硬编码在 adapter 中
  建议：配置文件中可覆盖 (如 telegram.deliveryMode = "gateway")

建议 2: Block Streaming 动态配置
  当前：静态阈值 (minChars=1500, idleMs=1000)
  建议：per-channel 配置 + 自适应调整

建议 3: 人类延迟精细化
  当前：全局 random(800-2500ms)
  建议：per-message-type (工具结果无需延迟，闲聊需延迟)
```

---

## 十一、调试指南

### 11.1 入站消息追踪

```bash
# 启用详细日志
export OPENCLAW_LOG_VERBOSE=1

# 追踪特定会话
export OPENCLAW_DEBUG_SESSION="agent:main:telegram:direct:123456"

# 查看 Hook 事件
export OPENCLAW_HOOKS_DEBUG=1
```

### 11.2 交付问题诊断

```typescript
// 在关键位置添加日志
// dispatch-from-config.ts
logVerbose(`dispatch-from-config: received payload: ${JSON.stringify(payload)}`);

// reply-dispatcher.ts
logVerbose(`enqueue: kind=${kind} textLength=${payload.text?.length}`);

// route-reply.ts
logVerbose(`route-reply: channel=${channel} to=${to} ok=${result.ok}`);

// outbound/deliver.ts
logVerbose(`deliver: sending ${payload.text?.length} chars to ${channel}`);
```

### 11.3 会话状态检查

```bash
# 查看会话存储
cat ~/.openclaw/sessions/main/sessions.json | jq '.["agent:main:telegram:direct:123456"]'

# 查看会话 transcript
cat ~/.openclaw/sessions/main/transcripts/<sessionId>.jsonl
```

---

**文档生成时间**: 2026-03-04
**参考文档**:

- https://docs.openclaw.ai/gateway
- https://docs.openclaw.ai/channels
- https://docs.openclaw.ai/concepts/agent
- https://docs.openclaw.ai/sessions
