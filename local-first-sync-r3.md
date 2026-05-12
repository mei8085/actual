# Actual Budget 同步时序深度复盘报告 (R3)

**聚焦**：old 消息副作用链路、diffTime 回拉场景、旧消息重发时序推导

---

## 一、applyMessages 副作用链路完整分析

### 1.1 完整执行流程（含 old 消息）

**文件**: `packages/loot-core/src/server/sync/index.ts:261-445`

```typescript
export const applyMessages = sequential(async (messages: Message[]) => {
  
  // 阶段 1: 裁决过滤
  // ============================================
  if (checkSyncingMode('enabled')) {
    messages = await compareMessages(messages);
    // 此时 messages 可能包含 old: true 的消息
  }
  
  messages.sort(...);  // 按时间戳排序，确保确定性顺序
  
  // 阶段 2: 构建变更范围
  // ============================================
  const idsPerTable: Record<string, string[]> = {};
  messages.forEach(msg => {
    if (msg.dataset === 'prefs') return;
    if (idsPerTable[msg.dataset] == null) {
      idsPerTable[msg.dataset] = [];
    }
    idsPerTable[msg.dataset].push(msg.row);
  });
  // ⚠️ 注意：old 消息也会被计入 idsPerTable！
  // 但 fetchData 从业务表查询，old 消息不修改业务表
  
  // 阶段 3: 抓取旧数据（用于撤销和对比）
  // ============================================
  async function fetchData(): Promise<DataMap> {
    const data = new Map();
    for (const table of Object.keys(idsPerTable)) {
      const rows = await fetchAll(table, idsPerTable[table]);
      // 从业务表查询，不是从 messages_crdt 查询
      for (const row of rows) {
        setIn(data, [table, row.id], row);
      }
    }
    return data;
  }
  
  const oldData = await fetchData();
  
  // 阶段 4: 记录撤销历史
  // ============================================
  undo.appendMessages(messages, oldData);
  // ⚠️ old 消息也会被加入 undo 历史！
  // 撤销时需要特殊处理 old 消息
  
  // 阶段 5: 准备时钟状态
  // ============================================
  let clock, currentMerkle;
  if (checkSyncingMode('enabled')) {
    clock = getClock();
    currentMerkle = clock.merkle;
  }
  
  // 阶段 6: 电子表格缓存屏障
  // ============================================
  if (sheet.get()) {
    sheet.get().startCacheBarrier();
  }
  
  // 阶段 7: 数据库事务（核心）
  // ============================================
  db.transaction(() => {
    const added = new Set();
    
    for (const msg of messages) {
      const { dataset, row, column, timestamp, value } = msg;
      
      // 🔴 old 消息跳过这一部分
      if (!msg.old) {
        // 修改业务数据表
        apply(msg, getIn(oldData, [dataset, row]) || added.has(dataset + row));
        
        if (dataset === 'prefs') {
          prefsToSet[row] = value;
        } else {
          added.add(dataset + row);
        }
      }
      
      // 🟢 old 消息仍然执行这一部分
      if (checkSyncingMode('enabled')) {
        // 1. 必须写入 messages_crdt 表
        db.runQuery(
          'INSERT INTO messages_crdt (timestamp, dataset, row, column, value) VALUES (?, ?, ?, ?, ?)',
          [timestamp.toString(), dataset, row, column, serializeValue(value)],
        );
        
        // 2. 必须更新 Merkle 树
        currentMerkle = merkle.insert(currentMerkle, timestamp);
      }
      
      // 🟡 old 消息也会触发
      if (dataset === 'preferences' && row === 'budgetType') {
        void setBudgetType(value);
      }
    }
    
    // Merkle 剪枝
    if (checkSyncingMode('enabled')) {
      currentMerkle = merkle.prune(currentMerkle);
      db.runQuery(
        'INSERT OR REPLACE INTO messages_clock (id, clock) VALUES (1, ?)',
        [serializeClock({ ...clock, merkle: currentMerkle })],
      );
    }
  });
  
  // 阶段 8: 更新内存时钟
  // ============================================
  if (checkSyncingMode('enabled')) {
    clock.merkle = currentMerkle;
  }
  
  // 阶段 9: 保存偏好设置（仅非 old 消息）
  // ============================================
  if (Object.keys(prefsToSet).length > 0) {
    // prefsToSet 只在 !msg.old 时填充
    void prefs.savePrefs(prefsToSet, { avoidSync: true });
    connection.send('prefs-updated');
  }
  
  // 阶段 10: 抓取新数据（用于对比和监听器）
  // ============================================
  const newData = await fetchData();
  // ⚠️ 关键：如果只有 old 消息，newData === oldData
  // 因为 old 消息不修改业务表
  
  // 阶段 11: 电子表格更新
  // ============================================
  if (sheet.get()) {
    sheet.startTransaction();
    
    // 🟢 所有消息都会触发，包括 old
    // 但 triggerBudgetChanges 比较 oldData vs newData
    // 如果 oldData === newData，预算不会有实际变化
    triggerBudgetChanges(oldData, newData);
    
    // 🟢 所有消息都会触发
    // 同样，如果 oldData === newData，可能无实际变化
    sheet.get().triggerDatabaseChanges(oldData, newData);
    
    sheet.endTransaction();
    
    // 交易相关的全局聚合单元格重算
    if (idsPerTable.transactions?.length) {
      const globalAggregateCells = [
        'accounts-balance',
        'onbudget-accounts-balance',
        'offbudget-accounts-balance',
        'closed-accounts-balance',
      ];
      for (const cellName of globalAggregateCells) {
        const fullName = resolveName('__global', cellName);
        if (s.hasCell(fullName)) {
          s.recompute(fullName);
        }
      }
    }
    
    sheet.get().endCacheBarrier();
  }
  
  // 阶段 12: 同步监听器（所有消息）
  // ============================================
  _syncListeners.forEach(func => func(oldData, newData));
  // 🟢 所有消息都会触发，包括 old
  // ⚠️ 但如果只有 old 消息，oldData === newData
  // 监听器需要自行判断是否有实际变化
  
  // 阶段 13: 应用事件（仅非 old 消息）
  // ============================================
  const tables = getTablesFromMessages(messages.filter(msg => !msg.old));
  // 🔴 只过滤非 old 消息
  
  app.events.emit('sync', {
    type: 'applied',
    tables,  // 可能为空数组（如果全是 old 消息）
    data: newData,
    prevData: oldData,
  });
  
  return messages;
});
```

### 1.2 old 消息副作用总结

| 阶段 | 操作 | old 消息是否执行 | 影响 |
|------|------|-----------------|------|
| compareMessages | 标记 old | - | 过滤重复消息 |
| idsPerTable 构建 | 收集变更行 | ✅ 是 | old 消息的行也被收集 |
| fetchData (oldData) | 从业务表查询 | - | 不受 old 消息影响 |
| undo.appendMessages | 记录撤销历史 | ✅ 是 | old 消息也在撤销栈中 |
| apply() | 修改业务表 | ❌ 否 | 不影响实际数据 |
| INSERT messages_crdt | 写入 CRDT 日志 | ✅ 是 | 日志完整性 |
| merkle.insert | 更新 Merkle 树 | ✅ 是 | 哈希一致性 |
| setBudgetType | 特殊偏好处理 | ✅ 是 | 可能触发副作用 |
| prefs.savePrefs | 保存偏好 | ❌ 否 | 仅非 old 消息 |
| fetchData (newData) | 从业务表查询 | - | 与 oldData 相同（仅 old 时） |
| triggerBudgetChanges | 预算重算 | ✅ 是 | 但 oldData === newData 时无变化 |
| triggerDatabaseChanges | 电子表格更新 | ✅ 是 | 同上 |
| globalAggregateCells | 聚合单元格重算 | ✅ 是 | 可能触发重算（取决于 idsPerTable） |
| _syncListeners | 同步监听器 | ✅ 是 | 但 oldData === newData |
| app.events.emit('sync', applied) | 应用事件 | ❌ 否 | tables 可能为空 |

### 1.3 监听器实际行为分析

**`_syncListeners` 触发时的数据对比**：

```
场景 A: 只有 new 消息
  oldData = { transactions: { tx1: { amount: 100 } } }
  newData = { transactions: { tx1: { amount: 200 } } }
  → oldData !== newData → 监听器感知到变化

场景 B: 只有 old 消息
  oldData = { transactions: { tx1: { amount: 300 } } }
  newData = { transactions: { tx1: { amount: 300 } } }
  → oldData === newData → 监听器感知不到实际变化

场景 C: new + old 混合
  oldData = { transactions: { tx1: { amount: 100 }, tx2: { amount: 50 } } }
  newData = { transactions: { tx1: { amount: 200 }, tx2: { amount: 50 } } }
  → oldData !== newData → 监听器感知到变化（来自 new 消息）
```

**transaction-rules.ts 中的监听器示例**：

```typescript
// transaction-rules.ts:201-214
function onApplySync(oldData: DataMap, newData: DataMap) {
  const transactions = getIn(newData, ['transactions']) || new Map();
  const addedIds: string[] = [];
  
  transactions.forEach((row, id) => {
    const oldRow = getIn(oldData, ['transactions', id]);
    if (!oldRow) {
      addedIds.push(id);  // 新增交易
    }
  });
  
  // 只处理新增的交易
  for (const id of addedIds) {
    const transaction = transactions.get(id);
    // 应用交易规则...
  }
}

unlistenSync = addSyncListener(onApplySync);
```

**分析**：
- 监听器比较 `oldData` 和 `newData`
- 如果只有 old 消息，`oldData === newData`，`addedIds` 为空
- 监听器自动"忽略"old 消息，因为没有实际数据变化
- 这是一种**隐式防护**机制

---

## 二、diffTime 回拉场景深度分析

### 2.1 递归同步中的 since 回拉机制

**文件**: `packages/loot-core/src/server/sync/index.ts:752-830`

```typescript
const diffTime = merkle.diff(res.merkle, getClock().merkle);

if (diffTime !== null) {
  // 循环防护检查
  if ((count >= 10 && diffTime === prevDiffTime) || count >= 100) {
    // 尝试重建 Merkle
    const rebuiltMerkle = rebuildMerkleHash();
    
    if (rebuiltMerkle.trie.hash === res.merkle.hash) {
      // 重建后一致，说明本地 Merkle 状态损坏
    }
    
    throw new SyncError('out-of-sync');
  }
  
  // 🚨 关键：回拉 since
  receivedMessages = receivedMessages.concat(
    await _fullSync(
      new Timestamp(diffTime, 0, '0').toString(),  // since 被回拉到 diffTime
      localTimeChanged ? 0 : count + 1,
      diffTime,
    ),
  );
}
```

### 2.2 Merkle 剪枝导致的回拉场景

**Merkle 树结构** (`merkle.ts:141-164`)：

```typescript
export function prune(trie: TrieNode, n = 2): TrieNode {
  if (!trie.hash) return trie;
  
  const keys = getKeys(trie).sort();  // 三进制路径键
  const next: TrieNode = { hash: trie.hash };
  
  // 只保留最新的 n 个子节点（默认 n=2）
  for (const k of keys.slice(-n)) {
    next[k] = prune(trie[k], n);
  }
  
  return next;
}
```

**剪枝图解**：

```
剪枝前（分钟级时间戳路径，三进制）：
root(hash=H)
  ├── '0' (窗口 0: T-10 分钟 ~ T-5 分钟)
  │     ├── '0'
  │     └── '2'
  ├── '1' (窗口 1: T-5 分钟 ~ T 分钟)
  │     ├── '1'
  │     └── '2'
  └── '2' (窗口 2: T 分钟 ~ T+5 分钟)
        ├── '0'
        └── '1'

剪枝后 (n=2):
root(hash=H)  ← 根哈希不变（只丢子节点引用）
  ├── '1' (窗口 1)  ← 保留
  │     └── ...
  └── '2' (窗口 2)  ← 保留
        └── ...

  窗口 0 被丢弃，但 root hash 仍包含其贡献！
```

### 2.3 merkle.diff 的回拉逻辑

**文件**: `packages/crdt/src/crdt/merkle.ts:78-139`

```typescript
export function diff(trie1: TrieNode, trie2: TrieNode): number | null {
  if (trie1.hash === trie2.hash) {
    return null;  // 完全一致
  }
  
  let node1 = trie1;
  let node2 = trie2;
  let k = '';  // 累积的路径键
  
  while (true) {
    const keyset = new Set([...getKeys(node1), ...getKeys(node2)]);
    const keys = [...keyset.values()].sort();
    
    let diffkey: null | '0' | '1' | '2' = null;
    
    for (let i = 0; i < keys.length; i++) {
      const key = keys[i];
      const next1 = node1[key];
      const next2 = node2[key];
      
      // 🚨 剪枝导致的回拉触发点
      if (!next1 || !next2) {
        // 一方有这个子节点，另一方没有（可能被剪枝）
        // 无法继续向下遍历，返回当前累积的时间戳
        break;
      }
      
      if (next1.hash !== next2.hash) {
        // 找到哈希不同的子节点，继续向下
        diffkey = key;
        break;
      }
    }
    
    if (!diffkey) {
      // 无法继续向下，将路径转换为分钟级时间戳返回
      return keyToTimestamp(k);
    }
    
    // 继续向下遍历
    k += diffkey;
    node1 = node1[diffkey] || emptyTrie();
    node2 = node2[diffkey] || emptyTrie();
  }
}
```

### 2.4 回拉场景示例

**场景**：
- 客户端 B 已同步 M_B（`TS_B > TS_A`）
- 但由于 Merkle 剪枝，B 的 Merkle 树缺少某些旧节点引用
- 服务器的 Merkle 树更完整（或剪枝策略不同）
- `merkle.diff` 发现不一致，返回较早的 `diffTime`

```
时间线:

T0: 三客户端初始同步
    A, B, Server: lastSyncedTimestamp = T0
    Merkle 一致

T1: A 修改 amount = 200 → 生成 M_A (TS_A)
T2: B 修改 amount = 300 → 生成 M_B (TS_B)
    假设: TS_B > TS_A

T3: B fullSync 成功
    ┌─────────────────────────────────────┐
    │ B: since = T0                      │
    │ 上传: [M_B]                        │
    │                                     │
    │ Server:                            │
    │   ① 查: WHERE timestamp > T0 → []  │
    │   ② 插: M_B                        │
    │   ③ Merkle 更新为 H_server1        │
    │                                     │
    │ B: 收到 newMessages = []           │
    │    Merkle diff: 本地 H_B1 == H_server1 │
    │    diffTime = null                 │
    │    lastSyncedTimestamp = TS_B      │
    └─────────────────────────────────────┘

T4: A fullSync 成功
    ┌─────────────────────────────────────┐
    │ A: since = T0                      │
    │ 上传: [M_A]                        │
    │                                     │
    │ Server:                            │
    │   ① 查: WHERE timestamp > T0       │
    │      → [M_B] (来自 B)              │
    │   ② 插: M_A                        │
    │   ③ Merkle 更新为 H_server2        │
    │       (包含 M_A + M_B)             │
    │                                     │
    │ A: 收到 newMessages = [M_B]        │
    │    receiveMessages([M_B]):         │
    │      compareMessages(M_B):         │
    │        查询 timestamp >= TS_B      │
    │        本地只有 M_A (TS_A < TS_B)  │
    │        → 不标记 old                │
    │      应用 M_B: amount = 300        │
    │      Merkle 更新为 H_A2            │
    │                                     │
    │    Merkle diff: H_A2 == H_server2  │
    │    diffTime = null                 │
    │    lastSyncedTimestamp = TS_B      │
    └─────────────────────────────────────┘

T5: 时间推移，Merkle 剪枝发生
    Server 剪枝: 保留最近 2 个窗口
    B 剪枝: 保留最近 2 个窗口
    但剪枝时机不同 → Merkle 结构可能不同
    （虽然 root hash 应该相同...）

T6: B 再次 fullSync（正常路径）
    ┌─────────────────────────────────────┐
    │ B: since = TS_B (从 lastSyncedTimestamp) │
    │                                     │
    │ Server:                            │
    │   ① 查: WHERE timestamp > TS_B     │
    │      → [] (M_A 的 TS_A < TS_B)     │
    │   ② 插: [] (B 无新消息)            │
    │   ③ Merkle = H_server2             │
    │                                     │
    │ B: newMessages = []                │
    │    Merkle diff: H_B2 == H_server2? │
    │    → 应该一致，diffTime = null      │
    │    lastSyncedTimestamp 不变        │
    │                                     │
    │ ✅ 正常情况：B 不会收到 M_A         │
    └─────────────────────────────────────┘

T6': B 再次 fullSync（异常路径 - Merkle 不一致）
    ┌─────────────────────────────────────┐
    │ B: since = TS_B                    │
    │                                     │
    │ Server: 返回 Merkle = H_server2    │
    │                                     │
    │ B: Merkle diff: H_B2 != H_server2  │
    │    diffTime = T_diff (回拉到较早时间) │
    │                                     │
    │    递归调用 _fullSync:              │
    │      since = T_diff ( < TS_B )     │
    │                                     │
    │ Server:                            │
    │   查: WHERE timestamp > T_diff     │
    │      → [M_A, M_B] (两者都 > T_diff)│
    │                                     │
    │ B: 收到 newMessages = [M_A, M_B]   │
    │    receiveMessages:                │
    │      compareMessages(M_A):         │
    │        查询 timestamp >= TS_A      │
    │        本地有 M_B (TS_B > TS_A)    │
    │        → 标记 old: true!           │
    │                                     │
    │      compareMessages(M_B):         │
    │        查询 timestamp >= TS_B      │
    │        本地有 M_B (相同时间戳)      │
    │        → 重复消息，丢弃!            │
    │                                     │
    │    应用结果:                        │
    │      M_A: old → 只更新 CRDT/Merkle │
    │      M_B: 重复 → 丢弃              │
    │                                     │
    │    Merkle diff: 检查是否一致        │
    │    → 可能需要多轮递归               │
    │                                     │
    │ ✅ 异常情况：B 收到 M_A，但被正确处理 │
    └─────────────────────────────────────┘
```

### 2.5 R2 结论修正

| 问题 | R2 结论 | R3 修正 |
|------|---------|---------|
| B 是否会收到 M_A | 不会（因为 `since = TS_B`） | **可能会**（当 Merkle diff 回拉 `since` 时） |
| 为何可能收到 | - | Merkle 剪枝、时钟漂移、网络分区恢复等 |
| 后果 | - | `compareMessages` 标记 `old: true`，不修改业务数据 |
| 防护机制 | 服务器时序 | **`old` 标记裁决**（第二道防线） |

---

## 三、旧消息重发的完整时序推导

### 3.1 正常路径：不会收到旧消息

```
前提:
- B 已成功同步 M_B
- lastSyncedTimestamp = TS_B

时序:
1. B 调用 fullSync()
2. since = lastSyncedTimestamp = TS_B
3. 上传: getMessagesSince(TS_B) → [] (B 无新消息)
4. Server 查询: WHERE timestamp > TS_B
   - M_A: TS_A < TS_B → 不匹配
   - M_B: TS_B > TS_B? 否（严格大于）→ 不匹配
   → 返回 []
5. B 收到 newMessages = []
6. Merkle diff: null（假设一致）
7. 结束

✅ 结果: B 不会收到 M_A
```

### 3.2 异常路径：可能收到旧消息

```
前提:
- B 已成功同步 M_B
- lastSyncedTimestamp = TS_B
- 但 Merkle 不一致（剪枝、损坏等）

时序:
1. B 调用 fullSync()
2. since = TS_B
3. Server 返回 newMessages = [] + Merkle = H_server
4. B 本地 Merkle = H_b
5. Merkle diff: H_b != H_server
   → diffTime = T_diff (回拉到较早时间点)

6. 递归调用 _fullSync(since = T_diff, count = 1)
   - T_diff < TS_B（回拉）
   
7. Server 查询: WHERE timestamp > T_diff
   - M_A: TS_A > T_diff? 可能是
   - M_B: TS_B > T_diff? 可能是
   → 返回 [M_A, M_B]（取决于 T_diff）

8. B 收到 [M_A, M_B]
   → 进入 receiveMessages()

9. compareMessages(M_A):
   - 查询: messages_crdt 中
     dataset=transactions, row=tx1, column=amount, timestamp >= TS_A
   - 本地有 M_B (TS_B > TS_A)
   - res.length = 1, res[0].timestamp = TS_B
   - TS_B !== TS_A
   → 标记 old: true

10. compareMessages(M_B):
    - 查询: timestamp >= TS_B
    - 本地有 M_B (TS_B === TS_B)
    - res.length = 1, res[0].timestamp = TS_B
    - TS_B === TS_B
    → 重复消息，丢弃（不加入 newMessages）

11. applyMessages([M_A (old: true)]):
    - M_A.old = true
    - 不调用 apply() → 不修改业务表
    - 写入 messages_crdt
    - 更新 Merkle

12. 再次 Merkle diff:
    - 如果一致 → 结束
    - 如果不一致 → 继续递归

✅ 结果: B 收到 M_A，但被 old 标记正确处理
```

### 3.3 防护层次总结

```
第一层：服务器时序（正常路径）
  ┌─────────────────────────────────────────┐
  │ lastSyncedTimestamp = TS_B              │
  │ Server 查询: timestamp > TS_B           │
  │ M_A (TS_A < TS_B) 不匹配                │
  │ → B 不会收到 M_A                        │
  └──────────────────────┬──────────────────┘
                         │ Merkle 不一致时失效
                         ▼
第二层：客户端 old 标记裁决（异常路径）
  ┌─────────────────────────────────────────┐
  │ 收到 M_A                                │
  │ compareMessages(M_A):                   │
  │   查询: timestamp >= TS_A               │
  │   本地有 M_B (TS_B > TS_A)              │
  │   → 标记 old: true                      │
  │                                         │
  │ applyMessages:                          │
  │   !msg.old? → 否，跳过 apply()          │
  │   → 不修改业务表                        │
  │   → 只更新 CRDT/Merkle                  │
  └──────────────────────┬──────────────────┘
                         │ 极端情况？
                         ▼
第三层：循环防护（最终防线）
  ┌─────────────────────────────────────────┐
  │ count >= 10 && diffTime === prevDiffTime │
  │ 或 count >= 100                         │
  │ → 重建 Merkle，仍失败则报错             │
  │ → 抛出 SyncError('out-of-sync')         │
  └─────────────────────────────────────────┘
```

---

## 四、关键代码索引（R3 重点）

| 文件路径 | 行号 | R3 重点 |
|---------|------|---------|
| `packages/loot-core/src/server/sync/index.ts` | 261-445 | `applyMessages` 完整流程 |
| `packages/loot-core/src/server/sync/index.ts` | 284-294 | `idsPerTable` 构建（含 old 消息） |
| `packages/loot-core/src/server/sync/index.ts` | 296-309 | `fetchData` 从业务表查询 |
| `packages/loot-core/src/server/sync/index.ts` | 314 | `undo.appendMessages`（含 old 消息） |
| `packages/loot-core/src/server/sync/index.ts` | 338-385 | 事务内：`!msg.old` 跳过 `apply()` |
| `packages/loot-core/src/server/sync/index.ts` | 357-365 | old 消息仍更新 CRDT/Merkle |
| `packages/loot-core/src/server/sync/index.ts` | 399 | 第二次 `fetchData` → `newData` |
| `packages/loot-core/src/server/sync/index.ts` | 402-431 | 电子表格更新（含 old 消息） |
| `packages/loot-core/src/server/sync/index.ts` | 434 | `_syncListeners`（含 old 消息） |
| `packages/loot-core/src/server/sync/index.ts` | 436-442 | `app.events.emit`（仅非 old 消息） |
| `packages/loot-core/src/server/sync/index.ts` | 752-830 | `diffTime` 回拉 + 递归调用 |
| `packages/loot-core/src/server/sync/index.ts` | 769-817 | 循环防护 |
| `packages/loot-core/src/server/sync/index.ts` | 821 | `since = new Timestamp(diffTime, 0, '0')` |
| `packages/crdt/src/crdt/merkle.ts` | 78-139 | `merkle.diff` 回拉逻辑 |
| `packages/crdt/src/crdt/merkle.ts` | 141-164 | `prune` 剪枝策略 |
| `packages/loot-core/src/server/transactions/transaction-rules.ts` | 214 | `addSyncListener` 示例 |

---

## 五、R2 vs R3 修正对照表

| 条目 | R2 (原) | R3 (修正) |
|------|---------|----------|
| B 是否会收到 M_A | 不会（since = TS_B） | **可能会**（Merkle diff 回拉时） |
| 后果 | - | 被 `old` 标记裁决正确处理 |
| 防护层次 | 两层 | **三层**：服务器时序 + old 裁决 + 循环防护 |
| `_syncListeners` 触发 | 未详细分析 | 所有消息（含 old）都触发，但 `oldData === newData` |
| `app.events.emit('sync')` | 未详细分析 | 仅非 old 消息触发 |
| `fetchData` 查询位置 | 未强调 | 从**业务表**查询，不是从 `messages_crdt` |
| `idsPerTable` 包含范围 | 未分析 | old 消息的行也被收集 |
| undo 历史 | 未分析 | old 消息也会被加入撤销栈 |
| 电子表格更新 | 未分析 | 所有消息都触发，但实际变化取决于 `oldData !== newData` |
| `diffTime` 回拉原因 | 未深入 | Merkle 剪枝导致的路径丢失 |
| 回拉防护 | 未深入 | 循环防护（10 次停滞/100 次上限）+ Merkle 重建 |
