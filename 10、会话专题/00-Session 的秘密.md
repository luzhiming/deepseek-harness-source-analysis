# Session 的秘密

> 源码：`deepseek_harness/deepseek-harness`（0.2.1-alpha.2，HEAD `e5e4168b90`），路径均相对仓库根目录。

---

## 0、一句话

DSH 的会话**没有"消息列表"**。它只有**一条只追加的事件日志**,和一个从这条日志**现场折叠**出来的"模型可见表层"。

所谓"会话"= 一条不可变的事件流 + 一份可重放的折叠规则。
所谓"消息"= 折叠之后落在表层上的那些节点。

---

## 1、四个读取入口

`ctx.sessionQuery` 的四个入口,是**读多少数据的四级梯度**:

| 入口 | 读多少 | 返回什么 | 代价 |
|------|--------|----------|------|
| `listSessions()` | **只读 header** | 会话清单 + `live`/`persisted` 标志 | 最轻 |
| `listEvents(id)` | 读日志,只留元数据 | 每条事件的 `seq`/`type`/`time`/`surface` | 中 |
| `readSurface(id)` | 读日志 + 折叠 | 当前模型上下文 + `capturedThroughSeq` | 重 |
| `readSession(id)` | **读完整原始日志** | 全部事件,经回放校验 | 最重 |

**核心源码** —— 四个入口都走同一个语料库解析（`session-query/src/index.ts`）：

```ts
// 最轻：只列 header，一条事件都不读
listSessions(signal?: AbortSignal): Promise<SessionRecord[]> {
  return this._corpus.listSessions(signal)
}

// 读完整原始日志——经 Session.create 回放校验
async readSession(sessionId: SessionId): Promise<SessionLogSnapshot> {
  const loaded = await this._corpus.load(sessionId)
  Session.create(sessionId, loaded.events, loaded.header,
    loaded.inheritedEventCount, currentSessionMessageProjections)
  return { session: structuredClone(loaded.header), events: loaded.events.map(snapshotSessionEvent), ... }
}

// 轻量逐事件记录——每条带 surface 归属
async listEvents(sessionId: SessionId): Promise<SessionEventRecord[]> {
  const loaded = await this._corpus.load(sessionId)
  return tracing.eventRecords(sessionId, loaded.events)
}

// 完整当前模型表层——现场折叠
async readSurface(sessionId: SessionId): Promise<SessionSurfaceSnapshot> {
  const loaded = await this._corpus.load(sessionId)
  return {
    session: structuredClone(loaded.header),
    inheritedEventCount: loaded.inheritedEventCount,
    capturedThroughSeq: loaded.events.at(-1)?.seq ?? null,
    events: tracing.currentSurfaceEvents(sessionId, loaded.events),
  }
}
```

三点结论：

1. **`listSessions` 是唯一不碰日志的入口**——它只调 `_corpus.listSessions()`（列举 header），其余三个都要 `_corpus.load()` **加载并校验完整逻辑日志**。这就是代价梯度的来源。
2. **后三个入口共用一次折叠**：`listEvents`（`tracing.eventRecords`）、`readSurface`（`tracing.currentSurfaceEvents`）、`traceEvent` 都走 `analyzeEventLog` → `foldSurface`——**同一份规范折叠**，所以搜索/追踪与模型实际看到的历史一致。
3. **`readSession` 走的是 `Session.create`**——它构造一个 detached 的 Session 对象只为**回放校验**，不会让会话变成 live。

`live`/`persisted` 是**查询时刻的观测结论**（`src/corpus.ts:62-87`）：`live` = id 在 `ctx.sessions` 里，`persisted` = id 在持久化后端里——**两者独立，可同时为 true**。header 冲突时直接失败（`SESSION_QUERY_SOURCE_CONFLICT`），不合并、不挑一个。

> **`listSessions` 是唯一轻量的入口**——`readSession`/`readSurface`/`filterEvents` 都要加载并校验完整逻辑日志（`session-query` README「已知限制」）。大量会话场景下先 `listSessions` 再定点读，比无脑遍历便宜得多。

---

## 2、Session 怎么存

### 2.1 三层结构:从 `listSessions()` 挖到最底

**调用链路**：

```
ctx.sessionQuery.listSessions()                        ← session-query/src/index.ts:174
  └─ corpus.listSessions()                              ← session-query/src/corpus.ts:62
       ├─ persistence.list()   → 记为 { live: false, persisted: true }
       └─ ctx.sessions.list()  → 记为 { live: true,  persisted: 是否也落过盘 }
            └─ [...this.store.values()].map(e => e.session)
                 ▼
SessionStore.store = new Map<SessionId, SessionEntry>()    core/session/src/index.ts:1065
  └─ SessionEntry { id, session, carrier, emitCtx,          core/session/src/index.ts:421
                    announced, announcing, appending,
                    detachRequested, detach }
       └─ Session
            ├─ private log: SessionEvent[] = []             core/session/src/index.ts:526  ★
            ├─ private readonly surfaceManager: SurfaceManager
            ├─ readonly header: SessionHeader
            ├─ readonly inheritedEventCount: SessionLogOffset
            └─ private eventsSnapshot: readonly SessionEvent[] | undefined
```

**核心源码** —— `SessionStore` 的全部状态就是这一个 Map（`core/session/src/index.ts:1065`）：

```ts
private store = new Map<SessionId, SessionEntry>()
```

`SessionEntry` 装的不是会话，是**会话 + 生命周期状态**（`index.ts:421`）：

```ts
interface SessionEntry {
  readonly id: SessionId
  readonly session: Session
  readonly carrier: Scoped<Session>
  readonly emitCtx: Context
  announced: boolean
  announcing: boolean
  appending: boolean
  detachRequested: boolean
  detach(): void
}
```

四个布尔量管的是**发布时序**——创建事件、append 事件、拆卸三者的先后；`detach` 是拆卸能力。

`Session` 本体（`index.ts:526`）：

```ts
export class Session {
  private log: SessionEvent[] = []
  private readonly surfaceManager: SurfaceManager
  readonly header: SessionHeader
  readonly inheritedEventCount: SessionLogOffset
  readonly firstLiveSeq: SessionLogOffset
  readonly firstLifecycleSeq: SessionLogOffset
}
```

三点结论：

1. **"会话"在内存里是三层**：`Map<SessionId, SessionEntry>` → `Session` → `SessionEvent[]`。**会话对象里没有任何消息**——只有一条事件数组、一个 header、和一个表层折叠器。
2. **`live` 的物理来源**：key 在这个 Map 里 = live。`listSessions()` 的 `live` 标志由此而来（`corpus.ts:78`）。
3. **持久化是另一个来源**：`corpus.ts:67` 先把 `persistence.list()` 的结果记为 `{ live: false, persisted: true }`，再用 `ctx.sessions.list()` 覆盖成 `{ live: true, persisted: durable !== undefined }`。**两者独立，可同时为 true**。

> 另有 `attachments = new WeakMap<Session, SessionEntry>()`（`index.ts:433`）做反向索引——从 Session 找回它的 entry。用 WeakMap 是为了不阻止 Session 被回收。

### 2.2 写入路径:create → prepare → enter

`create()` 是编排（`index.ts:1126`）：

```ts
create(id?, options?): Session {
  const session = this.prepare(id, options)
  this.ctx.effect(function* (this: SessionStore) {
    yield this.enter(session)
    this.announce(session)
  }.bind(this), 'sessions.create()')
  return session
}
```

| 动作 | 做什么 |
|------|--------|
| `prepare` | 校验 id/cwd，构造 Session——**不入 store** |
| `enter` | `store.set(id, entry)` + `attachments.set(...)`，返回 detach 能力 |
| `announce` | 发 `session/created` |

**为什么拆三步**（源码注释 `index.ts:1144`）：让作用域卸载时会话和 agent 按**确定顺序**一起拆。一步做完的话，卸载时两者是竞态的兄弟 effect，拆卸钩子可能先于 agent 收尾事件触发，那些事件就丢了。

**核心源码** —— `prepare` 不入 store（`index.ts:1158`）：

```ts
prepare(id?: SessionId, options?: PrepareSessionOptions): Session {
  let sessionId: SessionId
  if (id === undefined) {
    // 未指定 id 时自动生成 `session-<n>`。counter 单调递增，所以同一进程内
    // 生成的 id 天然不重复；`while (this.store.has(sessionId))` 是第二道保险——
    // 若调用方显式创建的会话恰好占了这个名字（比如手动用了 `session-1`），
    // 就继续递增到下一个空闲名字，不会因撞名而失败。
    // 注意这个循环只查当前 store：已被 detach 掉的会话不在 Map 里，
    // 所以同一个 `session-<n>` 在会话被移除后**可以**被再次使用。
    do sessionId = brandString<SessionId>(`session-${++this.counter}`)
    while (this.store.has(sessionId))
  } else {
    sessionId = brandString<SessionId>(id)
  }
  if (this.store.has(sessionId)) throw new Error(`session "${sessionId}" already exists`)
  ...
  return Session.fromRestore(...)   // 构造了，但没进 store
}
```

关于SessionId我们重点说明一下：

1. **`counter` counter 的作用域是 SessionStore 实例——每个实例一份,从 0 开始**。通常一个 ctx 只挂一个 store,所以约等于"每个 ctx 一份";但手动 new 多个 store 时,它们的 counter 互不感知,自动生成的 id 可能重名。
2. **`do...while` 的顺序是先赋值再判断**，所以第一次循环体就产出了 `session-1`（counter 从 0 预增到 1），而不是 `session-0`。

> 显式指定 id 的路径更严格：`if (this.store.has(sessionId)) throw` —— **直接抛错，不绕**。所以自动生成的路径"撞了就换一个"，手动指定的路径"撞了就报错"，两种语义刻意不同。
>
> 这个差别是可观察的：自动生成到 `session-2` 时，如果 `session-2` 已被显式占用，它会**跳到 `session-3`**；而显式再创建 `session-2` 则直接抛 `session "session-2" already exists`。

`enter` 才真正写入（`index.ts:1231`）：

```ts
const entry: SessionEntry = {
  id, session, carrier, emitCtx: this.ctx,
  announced: false, announcing: false, appending: false, detachRequested: false,
  detach: () => { this.detachEntered(entry) },
}
// kv 存储，key就是SessoinId,value 就是 SessionEntry；这种结构的意义就是我们可以O(1)时间复杂度获取到我们的SessionEntry
this.store.set(id, entry)
attachments.set(session, entry)
```

### 2.3 事件长什么样

**信封**（`types.ts:501`）：

```ts
export type SessionEvent<T extends SessionEventType = SessionEventType> = {
  type: T
  seq: SessionSeq
  time: number                                  // Unix epoch 毫秒
  data: SessionEventMap[T]
  surfaceOp: ...                               // 怎么进入表层
  sourceEventSeqs?: readonly SessionSeq[]      // 引用了哪些更早的事件
  ignorable?: boolean
}
```

**先看 `type` 字段的取值域。** `SessionEventType` 就是 `SessionEventMap` 的键（`types.ts:438`）：

```ts
/** The appendable event-type keys of {@link SessionEventMap}, plugin-merged extensions included. */
export type SessionEventType = keyof SessionEventMap
```

`SessionEventMap` 是**可合并扩展**的接口——`core/session` 声明了 14 个基础成员，插件通过 declaration merging 追加自己的：

```ts
// core/session/src/types.ts:287
export interface SessionEventMap {
  'turn/start': { turn: number }
  'turn/end': { turn: number; reason: TurnEndReason }
  'step/start': { turn: number; step: number }
  'step/end': { turn: number; step: number }
  'user/message': UserMessage
  'developer/message': { turn: number; step: number; message: DeveloperMessage; headerSeq?: SessionSeq }
  'system/message': { turn: number; step: number; message: SystemMessage }
  'assistant/message': { turn: number; step: number; stream: AssistantStreamRecord[]; message: AssistantMessage }
  'assistant/attempt': { turn: number; step: number; stream: AssistantStreamRecord[] }
  'tool/call': { turn: number; step: number; callId: ToolCallId; name: string; arguments: string }
  'tool/result': { turn: number; step: number; callId: ToolCallId; message: ToolResultMessage; error?: { name: string; code: string } }
  'request/header': { ... }
  'request/context': RequestContext
  'session/end-seed': { inherited?: true }
}
```

加上插件合并的，**全仓库共 63 个事件类型**（`KNOWN_SESSION_EVENT_TYPES`，`core/session/src/known-event-types.ts`）。按用途分五类：

| 类别 | 数量 | 类型 |
|------|------|------|
| **轮次/步骤结构** | 4 | `turn/start` `turn/end` `step/start` `step/end` |
| **模型可见消息**（能进表层） | 5 | `system/message` `developer/message` `user/message` `assistant/message` `tool/result` |
| **工具与请求** | 4 | `tool/call` `request/header` `request/context` `assistant/attempt` |
| **fork 边界** | 1 | `session/end-seed` |
| **插件事件** | 49 | `compaction/*`(4) `team/*`(4) `tool-workflow/*`(4) `approval/*`(3) `feedback/*`(3) `subagent/*`(3) `command/*`(2) `hook/*`(2) `image/*`(2) `job/*`(2) `llm/*`(2) `session/*`(2) `tool/*`(2) 以及 `todo/write`、`goal/change`、`plan/mode`、`model/selection`、`permission/*`、`sandbox/mode`、`schedule/change`、`session-log-deepseek/*`、`web/*`、`working-directory/change`、`workspace/changes`、`agent/inbox/spliced`、`agent-preset/selected`、`deliverables/presented` 等 |

其中只有 5 种是 **"表层事件"**（`SurfaceEventType`，`types.ts:447`）——只有它们**能携带 `surfaceOp`**，也就是只有它们能进入模型可见表层：

```ts
export type SurfaceEventType =
  | 'system/message'
  | 'developer/message'
  | 'user/message'
  | 'assistant/message'
  | 'tool/result'
```

> `KNOWN_SESSION_EVENT_TYPES` 是**本 build 认识的词表**——持久化读取路径遇到不在这个集合里的类型，除非事件带 `ignorable` 标记，否则拒绝解释（那多半是更新的 harness 写的日志，静默跳过会重建出错误的会话）。下游插件的事件天然在这个列表之外。


### 2.4 header:日志之外的那部分

```ts
export interface SessionHeader {
  readonly version: typeof SESSION_FORMAT_VERSION   // 当前 = 4
  readonly id: SessionId
  readonly createdAt: number
  readonly cwd?: string
  readonly parentSession?: SessionId                // 从哪个会话 fork 而来
  readonly isSeeded: boolean
  readonly origin?: 'subagent'
  readonly delegationDepth?: number
  readonly agentPreset?: string
}
```

源码注释点明（`index.ts:540`）：

> Kept out of the event log — it is a storage concern, not replayable conversation state.

**header 是存储元数据，不是对话状态。** 不可变、不进日志、不参与重放。

### 2.5 seq 就是数组下标

```ts
/** The next event's sequence number — always the log length (the `seq = log.length` contiguity contract). */
get seq(): SessionLogOffset {
  return SessionLogOffset(this.log.length)
}
```

**seq 就是数组下标**，不需要额外计数器维护。

`index.ts:728` 的快照：

```ts
snapshotEvents(fromSeq = 0, toSeqExclusive = this.seq): readonly SessionEvent[] {
  if (fromSeq === 0 && toSeqExclusive === this.log.length) {
    this.eventsSnapshot ??= Object.freeze([...this.log])   // 全量快照缓存
    return this.eventsSnapshot
  }
  return Object.freeze(this.log.slice(fromSeq, toSeqExclusive))
}
```

**旧快照在追加后纹丝不动**——`Object.freeze([...this.log])` 展开成新数组再冻结，**不返回日志本身的引用**。这是「只追加日志」能成立的前提：调用方拿到的任何快照，都不会因为后续 append 而改变。

---

**相关阅读**：`02-四个读取入口的完整调用链.md`（待补）—— 这四个入口从 API 到内存的完整分支，每个读多少、校验多严。
