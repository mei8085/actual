# Actual Budget 多端账本本地数据汇合机制研究报告

## 概述

Actual Budget 采用 **Local-First** 架构实现多端数据同步，核心思想是：
- **本地优先**：所有操作首先在本地执行，无需等待网络响应
- **消息驱动**：数据变更以消息形式捕获和传播
- **CRDT 思想**：使用混合逻辑时钟 (HULC) + Merkle 树实现最终一致性
- **无中心化裁决**：每个客户端独立进行合并裁决，服务器仅作为消息中继

---

## 一、改动捕获机制

### 1.1 数据变更入口

所有数据变更通过 `packages/loot-core/src/server/db/index.ts` 中的三个核心函数捕获：

| 函数 | 操作类型 | 消息特征 |
|------|---------|---------|
| `insert(table, row)` | 插入新记录 | 每个字段生成一条消息 |
| `update(table, params)` | 更新记录 | 仅变更字段生成消息 |
| `delete_(table, id)` | 删除记录 | 设置 `tombstone = 1` |

### 1.2 Message 数据结构

```typescript
type Message = {
  dataset: string;      // 表名，如 'transactions', 'accounts'
  row: string;          // 记录 ID
  column: string;       // 字段名
  value: string | number | null;  // 新值
  timestamp: Timestamp; // HULC 时间戳
};
```

### 1.3 消息生成示例

**插入操作** (`db/index.ts:238-256`)：
```typescript
export async function insert(table, row) {
  const fields = Object.keys(row).filter(k => k !== 'id');
  await sendMessages(
    fields.map(k => ({
      dataset: table,
      row: row.id,
      column: k,
      value: row[k],
      timestamp: Timestamp.send(),  // 生成时间戳
    })),
  );
}
```

**删除操作** (`db/index.ts:258-268`)：
```typescript
export async function delete_(table, id) {
  await sendMessages([{
    dataset: table,
    row: id,
    column: 'tombstone',
    value: 1,
    timestamp: Timestamp.send(),
  }]);
}
```

### 1.4 批处理机制

支持 `batchMessages` 函数将多个操作合并为一个同步批次 (`sync/index.ts:499-524`)：

```typescript
export async function batchMessages(func: () => Promise<void>): Promise<void> {
  IS_BATCHING = true;
  let batched: Message[] = [];
  try {
    await func();  // 期间所有 sendMessages 调用都缓存到 _BATCHED
  } finally {
    IS_BATCHING = false;
    batched = _BATCHED;
    _BATCHED = [];
  }
  if (batched.length > 0) {
    await _sendMessages(batched);
  }
}
```

---

## 二、消息分发机制

### 2.1 本地消息应用流程 (`_sendMessages`)

**文件**: `packages/loot-core/src/server/sync/index.ts:488-497`

```
本地修改 → sendMessages() → applyMessages() 
                              ↓
                    1. 比较消息 (compareMessages)
                    2. 按时间戳排序
                    3. 记录撤销历史 (undo.appendMessages)
                    4. 数据库事务应用
                    5. 更新 Merkle 树
                    6. 触发同步监听器
                              ↓
                    scheduleFullSync() → 延迟 1 秒后上传到服务器
```

### 2.2 全量同步 (`fullSync`)

**文件**: `packages/loot-core/src/server/sync/index.ts:597-844`

#### 2.2.1 同步流程

```
客户端 A                              同步服务器
   │                                     │
   │  1. 获取本地时钟时间戳                │
   │  2. 查询 messages_crdt 获取增量消息   │
   │  3. Protocol Buffer 编码 (可选加密)  │
   │────────────────────────────────────>│
   │                                     │
   │                                     │  1. 存储消息到 group SQLite
   │                                     │  2. 更新服务器端 Merkle 树
   │                                     │  3. 查询其他客户端的新消息
   │<────────────────────────────────────│
   │                                     │
   │  1. 解码响应                        │
   │  2. 比较 Merkle 哈希                 │
   │  3. 应用新消息 (receiveMessages)    │
   │  4. 如不一致，递归同步               │
```

#### 2.2.2 消息编码 (`encoder.ts`)

支持加密传输：
- 未加密：直接序列化 `Message` Protobuf
- 加密：使用 AES-256-GCM，包含 IV 和 AuthTag

```typescript
// 编码流程
requestPb = new SyncProtoBuf.SyncRequest();
for (const msg of messages) {
  envelopePb = new SyncProtoBuf.MessageEnvelope();
  envelopePb.setTimestamp(msg.timestamp.toString());
  
  messagePb = new SyncProtoBuf.Message();
  messagePb.setDataset(msg.dataset);
  messagePb.setRow(msg.row);
  messagePb.setColumn(msg.column);
  messagePb.setValue(msg.value);
  
  if (encryptKeyId) {
    // 加密内容
    encrypted = await encryption.encrypt(binaryMsg, encryptKeyId);
    envelopePb.setIsencrypted(true);
  } else {
    envelopePb.setContent(binaryMsg);
  }
  requestPb.addMessages(envelopePb);
}
```

### 2.3 服务器端处理 (`sync-simple.js`)

**文件**: `packages/sync-server/src/sync-simple.js:70-93`

服务器是**无状态消息中继**：
- 每个 `groupId` 对应一个独立的 SQLite 数据库文件
- 存储原始二进制消息 (`messages_binary` 表)
- 维护 Merkle 树 (`messages_merkles` 表)
- 不做任何业务逻辑或冲突裁决

```javascript
export function sync(messages, since, groupId) {
  const db = getGroupDb(groupId);
  
  // 1. 查询自 since 以来的新消息（返回给客户端）
  const newMessages = db.all(
    `SELECT * FROM messages_binary WHERE timestamp > ? ORDER BY timestamp`,
    [since]
  );
  
  // 2. 存储客户端上传的消息，更新 Merkle 树
  const trie = addMessages(db, messages);
  
  return { trie, newMessages };
}
```

---

## 三、合并裁决机制

### 3.1 核心思想：Last-Write-Wins (LWW)

Actual Budget 采用 **最后写入获胜** 策略，通过 **HULC (Hybrid Unique Logical Clock)** 时间戳实现全局有序。

### 3.2 HULC 时间戳 (`timestamp.ts`)

**文件**: `packages/crdt/src/crdt/timestamp.ts`

#### 3.2.1 时间戳结构

```
格式: ISO时间-计数器-节点ID
示例: 2015-04-24T22:23:42.123Z-1000-0123456789ABCDEF
      └────────────────┘ └──┘ └────────────────┘
           毫秒时间       16位     16字符
                        计数器    节点ID
```

三组件含义：
1. **millis**: 物理时间（毫秒），保证大致的时间顺序
2. **counter**: 16 位十六进制计数器，解决同一毫秒内的冲突
3. **node**: 16 字符唯一节点 ID，保证全局唯一性

#### 3.2.2 时间戳生成 (`Timestamp.send()`)

```typescript
static send(): Timestamp {
  const phys = Date.now();           // 物理时间
  const lOld = clock.timestamp.millis();
  const cOld = clock.timestamp.counter();
  
  // 逻辑时钟不后退
  const lNew = Math.max(lOld, phys);
  // 同一逻辑时间内计数器递增
  const cNew = lOld === lNew ? cOld + 1 : 0;
  
  return new Timestamp(lNew, cNew, clock.timestamp.node());
}
```

#### 3.2.3 时间戳接收 (`Timestamp.recv()`)

接收远程消息时更新本地时钟，确保因果序：

```typescript
static recv(msg: Timestamp): Timestamp {
  const phys = Date.now();
  const lMsg = msg.millis();
  const cMsg = msg.counter();
  
  const lOld = clock.timestamp.millis();
  const cOld = clock.timestamp.counter();
  
  // 取三者最大值：本地逻辑时间、物理时间、消息时间
  const lNew = Math.max(Math.max(lOld, phys), lMsg);
  
  // 计数器策略：
  // - 若本地和消息时间相同：取计数器最大值 + 1
  // - 若仅本地时间更大：本地计数器 + 1
  // - 若仅消息时间更大：消息计数器 + 1
  // - 否则：重置为 0
  const cNew = 
    lNew === lOld && lNew === lMsg ? Math.max(cOld, cMsg) + 1 :
    lNew === lOld ? cOld + 1 :
    lNew === lMsg ? cMsg + 1 : 0;
  
  return new Timestamp(lNew, cNew, clock.timestamp.node());
}
```

### 3.3 消息比较裁决 (`compareMessages`)

**文件**: `packages/loot-core/src/server/sync/index.ts:197-223`

这是合并裁决的**核心函数**：

```typescript
async function compareMessages(messages: Message[]): Promise<Message[]> {
  const newMessages = [];
  
  for (const message of messages) {
    const { dataset, row, column, timestamp } = message;
    const timestampStr = timestamp.toString();
    
    // 查询：同一数据单元 (dataset+row+column) 是否有更新的消息？
    const res = db.runQuery(
      'SELECT timestamp FROM messages_crdt ' +
      'WHERE dataset = ? AND row = ? AND column = ? AND timestamp >= ?',
      [dataset, row, column, timestampStr]
    );
    
    if (res.length === 0) {
      // 无更新消息 → 这是新消息，需要应用
      newMessages.push(message);
    } else if (res[0].timestamp !== timestampStr) {
      // 有更新消息 → 标记为 old，只更新 Merkle 树
      newMessages.push({ ...message, old: true });
    }
    // 时间戳完全相同 → 重复消息，丢弃
  }
  
  return newMessages;
}
```

#### 3.3.1 裁决规则详解

| 情况 | 查询结果 | 处理方式 |
|------|---------|---------|
| **全新消息** | 无匹配记录 | 正常应用到数据库 |
| **旧消息** | 存在更大时间戳 | 标记 `old: true`，仅更新 Merkle 树 |
| **重复消息** | 时间戳完全相同 | 静默丢弃 |

### 3.4 消息应用 (`applyMessages`)

**文件**: `packages/loot-core/src/server/sync/index.ts:261-445`

#### 3.4.1 完整流程

```typescript
export const applyMessages = sequential(async (messages: Message[]) => {
  // 1. 导入模式：快速路径，跳过 CRDT 比较
  if (checkSyncingMode('import')) {
    applyMessagesForImport(messages);
    return;
  }
  
  // 2. 比较消息，标记 old
  if (checkSyncingMode('enabled')) {
    messages = await compareMessages(messages);
  }
  
  // 3. 按时间戳排序（确保确定性顺序）
  messages.sort((m1, m2) => 
    m1.timestamp.toString().localeCompare(m2.timestamp.toString())
  );
  
  // 4. 获取旧数据（用于撤销和监听器）
  const oldData = await fetchData();
  undo.appendMessages(messages, oldData);
  
  // 5. 数据库事务（原子性保证）
  db.transaction(() => {
    for (const msg of messages) {
      const { dataset, row, column, timestamp, value } = msg;
      
      if (!msg.old) {
        // 非 old 消息：应用到业务表
        // INSERT 或 UPDATE 取决于记录是否存在
        apply(msg, getIn(oldData, [dataset, row]));
      }
      
      if (checkSyncingMode('enabled')) {
        // 所有消息（包括 old）：
        // 1. 存入 messages_crdt 表
        db.runQuery(
          'INSERT INTO messages_crdt (timestamp, dataset, row, column, value) VALUES (?, ?, ?, ?, ?)',
          [timestamp.toString(), dataset, row, column, serializeValue(value)]
        );
        // 2. 更新 Merkle 树
        currentMerkle = merkle.insert(currentMerkle, timestamp);
      }
    }
    
    // 保存时钟状态
    db.runQuery(
      'INSERT OR REPLACE INTO messages_clock (id, clock) VALUES (1, ?)',
      [serializeClock({ ...clock, merkle: currentMerkle })]
    );
  });
  
  // 6. 触发副作用
  triggerBudgetChanges(oldData, newData);  // 预算重算
  sheet.get().triggerDatabaseChanges(...);  // 电子表格更新
  _syncListeners.forEach(func => func(oldData, newData));  // UI 刷新
});
```

#### 3.4.2 关键设计要点

1. **`sequential` 装饰器**：确保消息队列串行处理，避免并发冲突
2. **`old` 标记消息**：不修改业务数据，但必须更新 CRDT 表和 Merkle 树，以保证：
   - 后续同步能正确识别消息已接收
   - Merkle 哈希与服务器一致
3. **原子事务**：业务表更新 + CRDT 表更新 + 时钟保存在同一事务中

### 3.5 数据单元冲突粒度

冲突检测的最小单元是 **`(dataset, row, column)`**，即：

- 表级别：不同表的修改互不影响
- 行级别：同一表不同行的修改互不影响  
- **列级别**：同一行的不同字段可独立修改

**示例**：客户端 A 修改交易的 `amount`，客户端 B 同时修改同一交易的 `notes`，两者都会生效，无冲突。

---

## 四、Merkle 树一致性校验

### 4.1 Merkle 树结构 (`merkle.ts`)

**文件**: `packages/crdt/src/crdt/merkle.ts`

#### 4.1.1 三进制基数树

```
TrieNode = {
  '0'?: TrieNode,    // 子节点 0
  '1'?: TrieNode,    // 子节点 1
  '2'?: TrieNode,    // 子节点 2
  hash?: number      // 节点哈希 (XOR 所有子节点哈希)
}
```

#### 4.1.2 时间戳到路径的映射

将时间戳的**分钟级时间**转换为三进制作为树路径：

```typescript
function insert(trie: TrieNode, timestamp: Timestamp) {
  const hash = timestamp.hash();  // murmurhash
  // 分钟级时间 → 三进制字符串
  const key = Math.floor(timestamp.millis() / 1000 / 60).toString(3);
  
  trie = { ...trie, hash: trie.hash ^ hash };
  return insertKey(trie, key, hash);
}
```

路径长度约为 16 位三进制数（足够覆盖到 2242 年）。

### 4.2 差异检测 (`merkle.diff`)

```typescript
export function diff(trie1: TrieNode, trie2: TrieNode): number | null {
  if (trie1.hash === trie2.hash) {
    return null;  // 完全一致
  }
  
  // 遍历树，找到第一个哈希不同的节点
  // 返回该节点对应的时间戳（分钟级）
  // 客户端需从该时间点重新同步
}
```

### 4.3 剪枝策略 (`merkle.prune`)

只保留最近 2 个时间窗口的子节点，防止树无限增长：

```typescript
export function prune(trie: TrieNode, n = 2): TrieNode {
  const keys = getKeys(trie).sort();
  const next = { hash: trie.hash };
  // 只保留最新的 n 个子节点
  for (const k of keys.slice(-n)) {
    next[k] = prune(trie[k], n);
  }
  return next;
}
```

**影响**：剪枝是**有损**的，可能需要多轮同步才能完全一致（见 `_fullSync` 中的递归逻辑）。

---

## 五、端到端同步示例

### 场景：双客户端修改同一交易金额

#### 初始状态

| 客户端 A | 客户端 B | 服务器 |
|---------|---------|--------|
| clock: T1 | clock: T1 | Merkle: H1 |
| 交易 amount: 100 | 交易 amount: 100 | - |

#### 时序

```
时间线: ─────────────────────────────────────────────────────>

客户端 A:           [修改 amount=200, 生成 TS=TA]
                         ↓
                   applyMessages() 本地应用
                         ↓
                   scheduleFullSync() [延迟 1s]

客户端 B:                    [修改 amount=300, 生成 TS=TB]
                                   ↓
                             applyMessages() 本地应用
                                   ↓
                             scheduleFullSync() [延迟 1s]
                                   ↓
                             上传到服务器: {TS=TB, value=300}
                                   ↓
                             服务器 Merkle 更新为 H2

客户端 A 同步开始:
  ├─ 上传: {TS=TA, value=200}
  ├─ 服务器返回: {TS=TB, value=300} + Merkle=H2
  ├─ 比较 Merkle: 本地 H1 ≠ H2
  └─ 应用消息 TS=TB:
      ├─ compareMessages(TS=TB)
      │   └─ 查询 messages_crdt: 存在 TA < TB
      │   └─ 不标记 old
      └─ apply(): 更新 amount=300 (B 的值获胜)

客户端 B 后续同步:
  ├─ 上传: {TS=TB, value=300} (已存在，被忽略)
  ├─ 服务器返回: {TS=TA, value=200} + Merkle=H2
  └─ 应用消息 TS=TA:
      ├─ compareMessages(TS=TA)
      │   └─ 查询 messages_crdt: 存在 TB > TA
      │   └─ 标记 old: true
      └─ 仅更新 CRDT 表和 Merkle，不修改业务数据
```

#### 最终结果

| 客户端 | amount | 原因 |
|-------|--------|------|
| 客户端 A | 300 | 接收并应用了 B 的更新 |
| 客户端 B | 300 | 本地修改，A 的更新被判定为 old |
| 服务器 | - | 中继所有消息 |

**关键**：`TB > TA`（字符串比较），所以 B 的修改获胜，与修改发生的实际物理时间无关。

---

## 六、协作关系总结

### 6.1 三大组件职责

| 组件 | 模块 | 核心职责 |
|------|------|---------|
| **改动捕获** | `db/index.ts` | 将 CRUD 操作转换为带时间戳的 Message |
| **消息分发** | `sync/index.ts` + `sync-server` | 本地应用 + 服务器中继 + Merkle 校验 |
| **合并裁决** | `compareMessages` + `Timestamp` | LWW 策略，时间戳决定胜负 |

### 6.2 数据流向图

```
┌─────────────────────────────────────────────────────────────┐
│                      客户端 A                                │
│                                                             │
│  用户操作                                                    │
│     ↓                                                       │
│  insert/update/delete_                                       │
│     ↓                                                       │
│  Message[] (带 Timestamp.send())                             │
│     ↓                                                       │
│  sendMessages ──→ applyMessages                              │
│                      ↓                                      │
│                 ┌─────────────┐                             │
│                 │ compare-    │  ←─── 合并裁决              │
│                 │ Messages    │                             │
│                 └─────────────┘                             │
│                      ↓                                      │
│                 本地数据库事务                               │
│                      ↓                                      │
│                 messages_crdt (CRDT日志)                    │
│                 业务表 (实际数据)                            │
│                 Merkle树 (哈希校验)                         │
│                      ↓                                      │
│                 scheduleFullSync() ────────┐                │
└─────────────────────────────────────────────┼────────────────┘
                                              │
                                              ▼ HTTP/Protobuf
┌─────────────────────────────────────────────────────────────┐
│                      同步服务器                              │
│                                                             │
│  /sync 端点                                                  │
│     ↓                                                       │
│  simpleSync.sync()                                          │
│     ↓                                                       │
│  messages_binary (原始消息存储)                              │
│  messages_merkles (Merkle树)                                │
│     ↓                                                       │
│  返回: newMessages[] + trie                                 │
└─────────────────────────────────────────────────────────────┘
                                              │
                                              │
┌─────────────────────────────────────────────┼────────────────┐
│                      客户端 B                │                │
│                                              ▼                │
│  fullSync() 接收消息                                          │
│     ↓                                                       │
│  receiveMessages() → Timestamp.recv() 更新本地时钟           │
│     ↓                                                       │
│  applyMessages                                              │
│     ↓                                                       │
│  compareMessages 裁决新旧                                    │
│     ↓                                                       │
│  应用到本地数据库                                            │
└─────────────────────────────────────────────────────────────┘
```

### 6.3 关键协作点

1. **时间戳作为唯一真相源**：
   - 生成时保证单调递增 (`Timestamp.send`)
   - 接收时推进本地时钟 (`Timestamp.recv`)
   - 裁决时字符串比较决定胜负

2. **Merkle 树作为一致性证明**：
   - 客户端和服务器独立维护
   - 差异检测触发增量同步
   - 剪枝策略平衡性能与完整性

3. **CRDT 表作为完整日志**：
   - `messages_crdt` 存储所有历史消息
   - 支持离线后的增量同步
   - 支持从任意时间点重建状态

---

## 七、关键文件索引

| 文件路径 | 功能描述 |
|---------|---------|
| `packages/crdt/src/crdt/timestamp.ts` | HULC 时间戳实现 |
| `packages/crdt/src/crdt/merkle.ts` | Merkle 树实现 |
| `packages/loot-core/src/server/db/index.ts` | 数据操作 → 消息生成 |
| `packages/loot-core/src/server/sync/index.ts` | 同步核心逻辑 |
| `packages/loot-core/src/server/sync/encoder.ts` | Protobuf 编解码 + 加密 |
| `packages/sync-server/src/sync-simple.js` | 服务器端同步逻辑 |
| `packages/sync-server/src/app-sync.ts` | 同步 HTTP 端点 |
| `packages/loot-core/src/server/undo.ts` | 撤销/重做（基于消息历史） |
