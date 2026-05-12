# Actual Budget 多端同步时序深度复盘报告 (R2)

**核心修正点**：服务器先查增量后入库的时序设计、`old` 标记的真正作用、递归 fullSync 的防回环机制

---

## 一、服务器同步时序：先查增量，再入库

### 1.1 关键代码确认

**文件**: `packages/sync-server/src/sync-simple.js:70-93`

```javascript
export function sync(messages, since, groupId) {
  const db = getGroupDb(groupId);
  
  // 步骤1：先查询增量消息（使用 since 参数过滤）
  const newMessages = db.all(
    `SELECT * FROM messages_binary
         WHERE timestamp > ?
         ORDER BY timestamp`,
    [since],  // ← 关键：timestamp > since，严格大于
  );
  
  // 步骤2：再入库客户端上传的消息
  const trie = addMessages(db, messages);
  
  db.close();
  
  // 返回：步骤1查询到的旧消息，不包含本次上传的消息
  return {
    trie,
    newMessages: newMessages.map(msg => { /* ... */ }),
  };
}
```

**测试服务器也遵循同样的顺序** (`mockSyncServer.ts:38-74`)：
```typescript
handlers['/sync/sync'] = async (data) => {
  // 1. 先查
  const newMessages = currentMessages.filter(msg => msg.timestamp > since);
  
  // 2. 再插
  messages.forEach(msg => {
    if (!currentMessages.find(m => m.timestamp === msg.getTimestamp())) {
      currentMessages.push({ ... });
      currentClock.merkle = merkle.insert(...);
    }
  });
  
  // 3. 返回步骤1的查询结果
  responsePb.setMerkle(...);
  newMessages.forEach(msg => responsePb.addMessages(envelopePb));
  return responsePb.serializeBinary();
};
```

### 1.2 时序设计的意图

| 设计点 | 说明 |
|--------|------|
| **`timestamp > since`** | 严格大于，不包含边界 |
| **先查后插** | 客户端上传的消息 **不会** 出现在本次响应的 `newMessages` 中 |
| **`INSERT OR IGNORE`** | 服务器端去重，同一时间戳不会重复存储 |

### 1.3 回包路径图解

```
客户端 A                              同步服务器
   │                                     │
   │  since = TS_last_synced             │
   │  messages = [M1, M2, M3]            │
   │────────────────────────────────────>│
   │                                     │
   │                                     │  ① 查询: SELECT * WHERE timestamp > since
   │                                     │     ↓
   │                                     │     返回: [M_other1, M_other2] (其他客户端的消息)
   │                                     │
   │                                     │  ② 入库: INSERT OR IGNORE M1, M2, M3
   │                                     │     ↓
   │                                     │     更新 Merkle trie
   │                                     │
   │<────────────────────────────────────│
   │                                     │
   │  newMessages = [M_other1, M_other2] │
   │  merkle = 更新后的 trie             │
   │                                     │
   │  ⚠️ 注意: M1, M2, M3 不在回包中      │
```

---

## 二、消息回环防止的三层机制

### 2.1 第一层：服务器端时序隔离

**原理**：先查后插 + `timestamp > since`

**场景**：客户端上传自己的消息，服务器不会回传

```
时间线:
T0: 客户端 A lastSyncedTimestamp = S0
T1: 客户端 A 生成 M1 (TS = TS1 > S0)
T2: 客户端 A 调用 fullSync(since=S0, messages=[M1])
    
    服务器:
    ① 查: WHERE timestamp > S0 → 此时 M1 尚未入库 → 返回 []
    ② 插: 存入 M1
    
T3: 客户端 A 收到 newMessages = []，不包含 M1

T4: 下次同步 since = currentTime (大于 TS1)
    → M1 也不会被查询到 (timestamp > since 不成立)
```

**结论**：服务器时序设计天然防止了 **即时回环**。

---

### 2.2 第二层：客户端 `old` 标记裁决

**文件**: `packages/loot-core/src/server/sync/index.ts:197-223`

#### 2.2.1 `compareMessages` 核心逻辑

```typescript
async function compareMessages(messages: Message[]): Promise<Message[]> {
  const newMessages = [];
  
  for (const message of messages) {
    const { dataset, row, column, timestamp } = message;
    const timestampStr = timestamp.toString();
    
    // 查询条件：同一数据单元，是否有时间戳 >= 本消息的记录？
    const res = db.runQuery(
      'SELECT timestamp FROM messages_crdt ' +
      'WHERE dataset = ? AND row = ? AND column = ? AND timestamp >= ?',
      [dataset, row, column, timestampStr]  // ← >=，包含等于
    );
    
    if (res.length === 0) {
      // 情况1：本地完全没有该数据单元的更新 → 新消息，需要应用
      newMessages.push(message);
    } else if (res[0].timestamp !== timestampStr) {
      // 情况2：有更新的消息存在 → 标记为 old
      newMessages.push({ ...message, old: true });
    }
    // 情况3：时间戳完全相等 (res[0].timestamp === timestampStr)
    //        → 静默丢弃，不加入 newMessages
  }
  
  return newMessages;
}
```

#### 2.2.2 三种裁决结果

| 情况 | 查询结果 | 处理 | `old` 标记 |
|------|---------|------|-----------|
| **全新** | 无匹配 | 应用到业务表 + 写入 CRDT | `undefined` (false) |
| **过时** | 存在更大时间戳 | 只写入 CRDT + 更新 Merkle | `true` |
| **重复** | 时间戳完全相等 | 静默丢弃 | - |

#### 2.2.3 `old` 消息在 `applyMessages` 中的处理

**文件**: `packages/loot-core/src/server/sync/index.ts:338-385`

```typescript
db.transaction(() => {
  for (const msg of messages) {
    const { dataset, row, column, timestamp, value } = msg;
    
    if (!msg.old) {
      // 非 old 消息：修改业务数据表
      // INSERT 或 UPDATE 实际数据
      apply(msg, getIn(oldData, [dataset, row]));
    }
    
    if (checkSyncingMode('enabled')) {
      // 所有消息（包括 old）：
      // 1. 必须写入 messages_crdt 表
      db.runQuery(
        'INSERT INTO messages_crdt (timestamp, dataset, row, column, value) VALUES (?, ?, ?, ?, ?)',
        [timestamp.toString(), dataset, row, column, serializeValue(value)],
      );
      // 2. 必须更新 Merkle 树
      currentMerkle = merkle.insert(currentMerkle, timestamp);
    }
  }
  
  // 保存时钟状态
  db.runQuery('INSERT OR REPLACE INTO messages_clock ...');
});
```

#### 2.2.4 `old` 标记的真正作用

`old: true` 的消息：
- ✅ **写入** `messages_crdt` 表（CRDT 日志完整性）
- ✅ **更新** Merkle 树（一致性校验）
- ❌ **不修改** 业务数据表（Last-Write-Wins 语义）
- ❌ **不触发** 同步监听器和 UI 更新

**设计意图**：
1. 防止旧数据覆盖新数据（LWW 保证）
2. 确保 CRDT 日志完整，后续同步可正确识别
3. 确保 Merkle 哈希与服务器一致

---

### 2.3 第三层：递归 `fullSync` 的 `diffTime` 推进

**文件**: `packages/loot-core/src/server/sync/index.ts:673-844`

#### 2.3.1 单次同步流程

```typescript
async function _fullSync(sinceTimestamp, count, prevDiffTime): Promise<Message[]> {
  // 1. 确定本次同步起点
  const since = sinceTimestamp || lastSyncedTimestamp || default5minAgo;
  
  // 2. 查询本地增量消息（要上传的）
  const messages = getMessagesSince(since);  // SELECT WHERE timestamp > since
  
  // 3. 发送到服务器，获取响应
  const res = await postBinary(...);  // 服务器返回：newMessages + merkle
  
  // 4. 应用服务器返回的消息
  let receivedMessages = [];
  if (res.messages.length > 0) {
    receivedMessages = await receiveMessages(res.messages);
  }
  
  // 5. 比较 Merkle 哈希，检测是否还有差异
  const diffTime = merkle.diff(res.merkle, getClock().merkle);
  
  if (diffTime !== null) {
    // 还有差异：递归同步，从 diffTime 开始
    receivedMessages = receivedMessages.concat(
      await _fullSync(
        new Timestamp(diffTime, 0, '0').toString(),  // 新起点
        localTimeChanged ? 0 : count + 1,             // 重试计数
        diffTime,                                     // 用于检测死循环
      ),
    );
  } else {
    // 完全一致：保存 lastSyncedTimestamp
    await prefs.savePrefs({
      lastSyncedTimestamp: getClock().timestamp.toString(),
    });
  }
  
  return receivedMessages;
}
```

#### 2.3.2 `diffTime` 推进机制

**Merkle 差异检测** (`merkle.ts:78-139`)：
```typescript
export function diff(trie1: TrieNode, trie2: TrieNode): number | null {
  if (trie1.hash === trie2.hash) {
    return null;  // 完全一致
  }
  
  // 遍历三进制树，找到第一个哈希不同的节点
  // 返回该节点对应的分钟级时间戳
  // 客户端需要从该时间点重新同步
}
```

**递归同步图解**：
```
第1轮 fullSync(since=T0):
  上传: M1, M2 (T0 < TS < T_current)
  下载: M_server1, M_server2
  Merkle diff: 发现 T1 处有差异
  → 递归调用 _fullSync(since=T1, count=1)

第2轮 fullSync(since=T1):
  上传: M3, M4 (T1 < TS < T_current)
  下载: M_server3
  Merkle diff: 发现 T2 处有差异
  → 递归调用 _fullSync(since=T2, count=2)

第3轮 fullSync(since=T2):
  上传: M5
  下载: []
  Merkle diff: null (完全一致)
  → 更新 lastSyncedTimestamp = T_current
  → 退出
```

#### 2.3.3 循环防护机制

**文件**: `packages/loot-core/src/server/sync/index.ts:769-817`

```typescript
if (
  (count >= 10 && diffTime === prevDiffTime) ||  // 连续10次卡在同一时间点
  count >= 100                                  // 硬上限100次
) {
  // 尝试重建 Merkle 哈希
  const rebuiltMerkle = rebuildMerkleHash();
  
  if (rebuiltMerkle.trie.hash === res.merkle.hash) {
    // 重建后一致：说明是本地 Merkle 状态损坏
    logger.log('Merkle hash in db:', deserializeClock(clocks[0].clock).merkle.hash);
  }
  
  throw new SyncError('out-of-sync');  // 抛出异常，终止
}
```

**防护策略**：
| 条件 | 触发 | 处理 |
|------|------|------|
| `count >= 10 && diffTime === prevDiffTime` | 连续10次无法推进 | 尝试重建 Merkle，仍失败则报错 |
| `count >= 100` | 递归超过100次 | 直接报错 |
| `localTimeChanged` | 同步期间本地有新修改 | 重置 `count=0`（正常用户操作不判错） |

---

## 三、完整同步时序（修正版）

### 3.1 双客户端冲突同步场景

**初始状态**：
- 客户端 A、B 都与服务器同步，`lastSyncedTimestamp = T0`
- 交易 `tx1.amount = 100`

#### 时序步骤

```
时间线: ─────────────────────────────────────────────────────────────>

T1: 客户端 A 修改 amount = 200
    → 生成 M_A: {dataset:transactions, row:tx1, column:amount, value:200, TS=TS_A}
    → 本地 applyMessages: 写入业务表 + messages_crdt
    → scheduleFullSync() [1秒后触发]

T2: 客户端 B 修改 amount = 300
    → 生成 M_B: {dataset:transactions, row:tx1, column:amount, value:300, TS=TS_B}
    → 本地 applyMessages: 写入业务表 + messages_crdt
    → scheduleFullSync()

    假设: TS_B > TS_A (B 的时间戳更大，字符串比较)

─────────────────────────────────────────────────────────────────────

T3: 客户端 B fullSync 触发
    ┌─────────────────────────────────────────────────────────────┐
    │ 客户端 B                                                   │
    │  since = T0                                                │
    │  messages = getMessagesSince(T0) = [M_B]                   │
    │───────────────────────────────────────────────────────────>│
    │                     同步服务器                             │
    │                     ① 查: WHERE timestamp > T0            │
    │                        → 返回: [] (空)                    │
    │                     ② 插: INSERT OR IGNORE M_B            │
    │                        → Merkle 更新为 H1                 │
    │<───────────────────────────────────────────────────────────│
    │  newMessages = []                                          │
    │  merkle = H1                                                │
    │                                                             │
    │  compare Merkle: 本地 H0 == H1?                             │
    │    本地应用 M_B 后已更新 Merkle → H0 == H1                  │
    │  diffTime = null → 完全一致                                 │
    │  更新 lastSyncedTimestamp = TS_B                           │
    └─────────────────────────────────────────────────────────────┘

T4: 客户端 A fullSync 触发
    ┌─────────────────────────────────────────────────────────────┐
    │ 客户端 A                                                   │
    │  since = T0                                                │
    │  messages = getMessagesSince(T0) = [M_A]                   │
    │───────────────────────────────────────────────────────────>│
    │                     同步服务器                             │
    │                     ① 查: WHERE timestamp > T0            │
    │                        → 返回: [M_B] (来自 B)             │
    │                     ② 插: INSERT OR IGNORE M_A            │
    │                        → Merkle 更新为 H2                 │
    │<───────────────────────────────────────────────────────────│
    │  newMessages = [M_B]                                       │
    │  merkle = H2                                                │
    │                                                             │
    │  receiveMessages([M_B]):                                   │
    │    Timestamp.recv(TS_B) → 推进本地时钟                     │
    │    applyMessages([M_B]):                                   │
    │      compareMessages(M_B):                                │
    │        查 messages_crdt:                                  │
    │          dataset=transactions, row=tx1, column=amount,   │
    │          timestamp >= TS_B                                │
    │        本地只有 M_A (TS_A < TS_B) → res.length = 0       │
    │        → 不标记 old → 正常应用                            │
    │                                                             │
    │      应用 M_B: amount = 300 (覆盖 A 的修改)                │
    │      写入 M_B 到 messages_crdt                             │
    │      更新 Merkle → 本地变为 H2                            │
    │                                                             │
    │  compare Merkle: H2 == H2 → diffTime = null                │
    │  更新 lastSyncedTimestamp = max(TS_A, TS_B)               │
    └─────────────────────────────────────────────────────────────┘

T5: 客户端 B 后续 fullSync
    ┌─────────────────────────────────────────────────────────────┐
    │ 客户端 B                                                   │
    │  since = TS_B                                              │
    │  messages = getMessagesSince(TS_B) = [] (无新消息)         │
    │───────────────────────────────────────────────────────────>│
    │                     同步服务器                             │
    │                     ① 查: WHERE timestamp > TS_B          │
    │                        → 返回: [M_A] (TS_A < TS_B? 不!)   │
    │                        → 实际上: TS_A < TS_B → 返回 []    │
    │                     ② 插: 无消息                           │
    │<───────────────────────────────────────────────────────────│
    │  newMessages = []                                          │
    │                                                             │
    │  注意: M_A 的时间戳 TS_A < TS_B                            │
    │  所以 WHERE timestamp > TS_B 不包含 M_A                    │
    │  M_A 不会被下载到 B                                        │
    └─────────────────────────────────────────────────────────────┘
```

#### 关键观察

1. **服务器先查后插**：
   - B 首次同步时，服务器先查（空），再插 M_B
   - A 首次同步时，服务器先查（返回 M_B），再插 M_A

2. **A 接收 M_B 时的裁决**：
   - `compareMessages(M_B)` 查询 `timestamp >= TS_B`
   - 本地只有 M_A (TS_A < TS_B) → 无匹配 → 不标记 old
   - M_B 正常应用，覆盖 A 的本地修改

3. **B 不会收到 M_A**：
   - B 的 `since = TS_B`
   - M_A 的 TS_A < TS_B → `WHERE timestamp > TS_B` 不匹配
   - 即使通过 Merkle diff 递归同步，也不会拿到 M_A
   - 因为 LWW 语义下，M_A 是过时消息，B 不需要它

---

## 四、Merkle 树剪枝与多轮同步

### 4.1 剪枝策略

**文件**: `packages/crdt/src/crdt/merkle.ts:141-164`

```typescript
export function prune(trie: TrieNode, n = 2): TrieNode {
  if (!trie.hash) return trie;
  
  const keys = getKeys(trie).sort();
  const next = { hash: trie.hash };
  
  // 只保留最新的 n 个子节点（默认 n=2）
  for (const k of keys.slice(-n)) {
    next[k] = prune(trie[k], n);
  }
  
  return next;
}
```

### 4.2 剪枝的影响

```
Merkle 树结构（三进制，分钟级时间戳为路径）：

剪枝前:
root(hash=H)
  ├── 0 (旧消息窗口)
  │     ├── 0
  │     └── 1
  ├── 1 (近期窗口)
  │     ├── 0
  │     └── 2
  └── 2 (最新窗口)
        ├── 1
        └── 2

剪枝后 (n=2):
root(hash=H)
  ├── 1 (近期窗口)  ← 保留
  │     └── ...
  └── 2 (最新窗口)  ← 保留
        └── ...
  
  窗口 0 被丢弃，但 root hash 不变（只丢子节点引用）
```

### 4.3 剪枝导致的多轮同步

**问题**：剪枝后，`merkle.diff` 可能无法精确定位最早的差异点。

**解决方案**：递归同步 + `diffTime` 逐步推进

```
第1轮:
  diffTime = T100 (剪枝后只能定位到最近的差异窗口)
  _fullSync(since=T100)
  
第2轮:
  同步 T100 后的消息
  diffTime = T50 (可能发现更早的差异)
  _fullSync(since=T50)

第3轮:
  同步 T50 后的消息
  diffTime = null (最终一致)
```

**循环防护**：
- 正常情况：每次 `diffTime` 向前推进
- 异常情况：连续10次 `diffTime` 不变 → 报错
- 最坏情况：100次递归后强制终止

---

## 五、防止消息回环的完整机制总结

### 5.1 三层防护体系

```
                    ┌─────────────────────────────────────┐
                    │        第一层：服务器时序           │
                    │    先查增量 (timestamp > since)    │
                    │    再入库                          │
                    │    → 本次上传的消息不回包          │
                    └───────────────┬─────────────────────┘
                                    │
                                    ▼
                    ┌─────────────────────────────────────┐
                    │      第二层：客户端 old 标记        │
                    │    compareMessages 查询本地 CRDT   │
                    │    有更新消息 → old: true          │
                    │    → 只更新 Merkle/CRDT            │
                    │    → 不修改业务数据                │
                    └───────────────┬─────────────────────┘
                                    │
                                    ▼
                    ┌─────────────────────────────────────┐
                    │    第三层：递归 fullSync 推进       │
                    │    diffTime 逐轮前移               │
                    │    count 计数器防止死循环          │
                    │    lastSyncedTimestamp 避免重复    │
                    └─────────────────────────────────────┘
```

### 5.2 不同场景的防护效果

| 场景 | 防护层 | 效果 |
|------|--------|------|
| 客户端上传自己的消息 | 服务器时序 | 不回包，无感知 |
| 从服务器收到其他客户端的旧消息 | `old` 标记 | 不修改业务数据 |
| Merkle 不一致需要多轮同步 | 递归 + `diffTime` | 逐步推进，最终一致 |
| 网络分区后恢复 | `lastSyncedTimestamp` | 从断点继续 |
| 数据损坏或时钟漂移 | count 限制 + 重建 Merkle | 检测异常并报错 |

---

## 六、关键代码索引（R2 重点）

| 文件路径 | 修正点 |
|---------|--------|
| `packages/sync-server/src/sync-simple.js:72-77` | 先查后插时序 (`newMessages` 查询在 `addMessages` 之前) |
| `packages/sync-server/src/sync-simple.js:30` | `INSERT OR IGNORE` 服务器端去重 |
| `packages/loot-core/src/server/sync/index.ts:205-210` | `compareMessages` 的 `timestamp >= ?` 查询条件 |
| `packages/loot-core/src/server/sync/index.ts:217-218` | `old: true` 标记逻辑 |
| `packages/loot-core/src/server/sync/index.ts:344-345` | `old` 消息跳过业务表更新 |
| `packages/loot-core/src/server/sync/index.ts:357-365` | `old` 消息仍更新 CRDT 表和 Merkle |
| `packages/loot-core/src/server/sync/index.ts:704` | `getMessagesSince(since)` 客户端增量查询 |
| `packages/loot-core/src/server/sync/index.ts:752-830` | `merkle.diff` + 递归 `_fullSync` |
| `packages/loot-core/src/server/sync/index.ts:769-817` | 循环防护和异常处理 |
| `packages/crdt/src/crdt/merkle.ts:141-164` | `prune` 剪枝策略 |

---

## 七、R1 报告的主要修正

| 条目 | R1 (原报告) | R2 (修正后) |
|------|------------|-------------|
| 服务器时序 | 未明确说明先查后插 | **先查增量，再入库**，回包不含本次上传消息 |
| `old` 标记作用 | 笼统说明"只更新 Merkle" | **详细说明**：不修改业务表，但必须更新 CRDT 表和 Merkle，确保日志完整性 |
| 递归同步 | 简单提及 | **详细分析**：`diffTime` 推进机制、剪枝导致多轮同步、count 防死循环 |
| 回环防止 | 未系统分析 | **三层防护体系**：服务器时序、`old` 标记、递归推进 |
| `since` 查询条件 | 未强调 | **严格大于** (`timestamp > since`)，是防回环的关键设计 |
