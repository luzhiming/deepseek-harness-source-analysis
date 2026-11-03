# 会话快照：listSessions 与 load

> **管什么**：读会话 = **拿一份快照**。这四个读取入口各自返回什么快照、怎么拿、以及快照的边界在哪。
> **真源**：`packages/session-query/session-query/src/`（`index.ts` / `corpus.ts` / `cold-read.ts` / `tracing.ts`），`@deepseek-ai/dsh-session-query@0.2.1-alpha.2`，HEAD `e5e4168b90`。
> **配套**：[00-Session 的秘密](00-Session%20的秘密.md) 已经讲清了底层——store 是 `Map<SessionId, SessionEntry>`、`prepare`/`enter`/`announce` 三步、seq 就是下标、快照不可变。本篇**不重复那些**。

---

## 一、四个入口，两个岔口

**这四个入口的共同点是：返回的都是快照。** 两个返回类型直接就叫 `Snapshot`——`SessionSurfaceSnapshot`、`SessionLogSnapshot`；DSH 源码里也管它叫 "cut"（`SessionObservation` 的注释：*"One exact immutable Session **cut**"*）。

| 入口 | 读 header | 读事件日志 | 折叠表层 | 回放校验 | 返回 |
|------|:---:|:---:|:---:|:---:|---|
| `listSessions()` | ✅ | ❌ | ❌ | ❌ | `SessionRecord[]` |
| `listEvents(id)` | ✅ | ✅ | ✅ | ❌ | `SessionEventRecord[]` |
| `readSurface(id)` | ✅ | ✅ | ✅ | ❌ | `SessionSurfaceSnapshot` |
| `readSession(id)` | ✅ | ✅ | ❌ | ✅ | `SessionLogSnapshot` |

**岔口一：读不读事件日志**

`listSessions` 是唯一的例外——它只要 header，不碰日志。剩下三个都要加载完整逻辑日志。

**岔口二：加载之后做哪种校验**

- `listEvents` / `readSurface` → 一次**表层折叠**（`analyzeEventLog`），顺带得出 `surface` 三态
- `readSession` → 一次**回放校验**（`Session.create`）

---

## 二、`listSessions()`：只列 header

> **`listSessions()` 返回跨内存与磁盘两个来源的完整会话清单,内存的覆盖磁盘的;每条记录用 `live`(活着吗)、`formatStatus`(日志能直接读吗)、`persisted`(关掉后会消失吗)三个标志标明它的状态。**

```
ctx.sessionQuery.listSessions()      index.ts:174
  └─ _corpus.listSessions(signal)    corpus.ts:62
```

四步：

| # | 动作 | 结果 |
|---|------|------|
| ① | `persistence.list()` | 逐条记为 `{ live: false, persisted: true }` |
| ② | `ctx.sessions.list()` | 逐条覆盖为 `{ live: true, persisted: 是否也落过盘 }` |
| ③ | `assertSessionHeadersCompatible(...)` | 冲突 → `SESSION_QUERY_SOURCE_CONFLICT` |
| ④ | `sort(compareSessions)` | `createdAt` 降序，同值按 id 字典序 |

**一个 Map 合并两个来源**，所以 `live`/`persisted` 是正交的两个标志：

| 在内存 | 在磁盘 | 结果 |
|--------|--------|------|
| ✅ | ✅ | `live: true, persisted: true` |
| ❌ | ✅ | `live: false, persisted: true` |
| ✅ | ❌ | `live: true, persisted: false` |

**核心源码**（`corpus.ts:62`）：

```ts
async listSessions(signal?: AbortSignal): Promise<SessionRecord[]> {
  signal?.throwIfAborted()
  const persistence = this._persistence
  const persisted = persistence === undefined ? [] : await listPersisted(persistence, signal)
  signal?.throwIfAborted()
  const records = new Map<SessionId, SessionRecord>()
  for (const { header, formatStatus } of persisted) {
    records.set(header.id, {
      header: structuredClone(header),
      live: false,
      persisted: true,
      ...(formatStatus === undefined ? {} : { formatStatus }),
    });
  }
  for (const session of this._ctx.sessions.list()) {
    const durable = records.get(session.id)
    if (durable !== undefined) assertSessionHeadersCompatible(session.header, durable.header)
    records.set(session.id, {
      header: structuredClone(session.header),
      live: true,
      persisted: durable !== undefined,
      formatStatus: 'current',
    });
  }
  return [...records.values()].sort(compareSessions)
}
```

> 排序是确定的（`corpus.ts:313`）：`createdAt` 降序，同值按 id 字典序——**同毫秒创建的会话也有确定顺序**。

### 三个标志各自由谁决定

| 标志 | 判据 | 为什么                                                                                  |
|------|------|--------------------------------------------------------------------------------------|
| `live` | id 在 `ctx.sessions` 里吗 | **只有内存知道**——磁盘侧无从得知某个会话当前是否在跑，所以先填 `false`，遍历内存时纠正成 `true` ，只要被内存覆盖的就是live，表示当前的会话状态 |
| `persisted` | id 在 `persistence.list()` 里吗 | 要**同时**反映两个来源，所以是 `durable !== undefined`,要表达的是：当前这个会话是否持久化过！                        |
| `formatStatus` | 磁盘侧给什么就是什么；内存侧一律 `'current'` | **正在跑的会话必然是当前格式**；只有磁盘副本才可能需要迁移                                                      |

### 三种状态组合

| 在内存 | 在磁盘 | `live` | `persisted` | `formatStatus` | 什么情况 |
|:---:|:---:|:---:|:---:|---|---|
| ✅ | ✅ | `true` | `true` | `current` | 正在跑，且已落盘 |
| ❌ | ✅ | `false` | `true` | `current` 或 `migration-required` | 历史会话（可能老格式） |
| ✅ | ❌ | `true` | `false` | `current` | 刚创建，还没落盘 |

> 覆盖前那行 `assertSessionHeadersCompatible` 比对 6 个身份字段（`id`/`createdAt`/`cwd`/`parentSession`/`isSeeded`/`delegationDepth`）——不一致就抛 `SESSION_QUERY_SOURCE_CONFLICT`。**不比对**的 3 个（`version`/`origin`/`agentPreset`）是新版才有的字段，属于新旧差异而非矛盾。

---

## 三、`_corpus.load()`：三个入口共用的加载点

`listEvents` / `readSurface` / `readSession` 都调它。

**核心源码**（`corpus.ts:98`）：

```ts
async load(sessionId: SessionId, signal?: AbortSignal): Promise<LogicalSession> {
  signal?.throwIfAborted()
  const live = this._ctx.sessions.get(sessionId)
  if (live !== undefined) {
    const snapshot = snapshotLive(live)
    signal?.throwIfAborted()
    return snapshot
  }
  const persistence = this._persistence
  if (persistence === undefined) throw notFound(sessionId)
  const listed = (await listPersisted(persistence, signal)).find(snapshot => snapshot.header.id === sessionId)
  signal?.throwIfAborted()
  if (listed === undefined) throw notFound(sessionId)
  const loaded = await inspectPersisted(persistence, sessionId, signal)
  signal?.throwIfAborted()
  const attached = this._ctx.sessions.get(sessionId)
  if (attached !== undefined) {
    const snapshot = snapshotLive(attached)
    signal?.throwIfAborted()
    return snapshot
  }
  assertSessionHeadersCompatible(loaded.header, listed.header)
  const snapshot = {
    header: structuredClone(loaded.header),
    inheritedEventCount: loaded.inheritedEventCount,
    events: loaded.events.map(event => structuredClone(event)),
  }
  signal?.throwIfAborted()
  return snapshot
}
```

### 这条链路的四个动作

`_corpus.load(sessionId, signal)` 依次做四件事：

| # | 动作 | 结果 |
|---|------|------|
| 1 | 先从内存查 `ctx.sessions.get(sessionId)` | 查到就返回 |
| 2 | 内存没有 → 从磁盘查 `persistence.list()` | 磁盘也没有 → `SESSION_QUERY_SESSION_NOT_FOUND` |
| 3 | 磁盘有 → 读出完整日志 | 返回一份**克隆**（**不加到内存**） |
| 4 | 读磁盘**之前**再查一次内存 | 这期间转 live 了 → 改用内存那份 |

**第 3 步不写内存。** `load()` 全程只**读** `ctx.sessions`，从不 `set` / `enter` / `create`。返回的 `LogicalSession` 里，header 和 events 全是 `structuredClone` 出来的副本——**磁盘上的会话依旧躺在磁盘上**，直到有人真的去恢复它。

### 第 4 步为什么要复查

因为**读磁盘是慢的**。在这段时间里，内存的状态可能变了：

```
① 查内存：没有 → 决定读磁盘
   ↓
   ┌──── 读磁盘（几百毫秒）────┐
   │  这期间，会话被 attach 了   │
   └──────────────────────────┘
   ↓
④ 再查内存：有了 → 改用内存那份
```

**内存那份更新鲜。** 不复查的话，就可能把**已经过时**的数据当成最新数据交出去——这类不一致很难排查：数据看起来合法、格式正确，只是少了几条事件。

这条链路上的竞态窗口不止一处。`observation.ts` 的冷读路径在 `stat` 之后、读磁盘之后各查一次——**每次慢操作之后都补一次检查**。

### 但它保障不了"最新"

这条链路（以及任何快照机制）**保障不了返回的 session 一定是最新的**——无论来自磁盘还是内存。因为：

- 读完磁盘之后，磁盘可能又变了
- 拿到快照之后，内存里的会话可能又追加了事件

**返回的就是快照——那一刻的状态。**

> **这是一个有趣的设计原则：**
>
> **慢操作之后复查，能消除"操作期间"的变化；但消除不了"操作之后"的变化。**
>
> **后者只能靠快照语义来界定——返回的数据就是那一刻的，之后发生什么都与你无关。**

那么"想基于它做操作"怎么办？答案是**不要基于快照做**——`observeSession()` 提供的租约机制才是干这个的，见 `03-observeSession 与修订号.md`（待补）。

