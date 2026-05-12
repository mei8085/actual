# Actual Budget 同步事件发布精校报告 (R4)

**聚焦**：`app.events.emit` 触发条件、`tables` 数组为空的原因、监听侧判定影响

---

## 一、app.events.emit 触发条件精校

### 1.1 代码确认

**文件**: `packages/loot-core/src/server/sync/index.ts:434-442`

```typescript
// 第 434 行: _syncListeners 始终触发
_syncListeners.forEach(func => func(oldData, newData));

// 第 436-442 行: app.events.emit 无条件触发
const tables = getTablesFromMessages(messages.filter(msg => !msg.old));
app.events.emit('sync', {
  type: 'applied',
  tables,
  data: newData,
  prevData: oldData,
});
```

### 1.2 关键结论

| 问题 | 答案 |
|------|------|
| 全 old 批次时 `app.events.emit` 是否触发？ | **是，无条件触发** |
| 触发条件是什么？ | 无额外条件，`applyMessages` 执行到第 437 行就会触发 |
| `type` 是什么？ | 固定为 `'applied'` |

### 1.3 与 `type: 'success'` 的区别

代码中还有其他地方会发出 `sync` 事件：

**1. API 层触发** (`packages/loot-core/src/server/api.ts:73-77`)：
```typescript
connection.send('sync-event', {
  type: 'success',
  tables: rows.map(row => row.dataset),
});
```

**2. 调度器触发** (`packages/loot-core/src/server/schedules/app.ts:588-592`)：
```typescript
connection.send('sync-event', {
  type: 'success',
  tables: ['transactions'],
  syncDisabled: false,
});
```

**两种事件类型对比**：

| 特性 | `type: 'applied'` | `type: 'success'` |
|------|-------------------|-------------------|
| 触发位置 | `applyMessages` 内部（第 437 行） | API 层 / 调度器外部触发 |
| 触发时机 | 消息应用完成后 | 同步操作成功后 |
| `tables` 来源 | `messages.filter(msg => !msg.old)` | `rows.map(row => row.dataset)`（数据库查询） |
| 事件名称 | `app.events.emit('sync', ...)` | `connection.send('sync-event', ...)` |
| 转发方式 | `main-app.ts:10-12` 转发为 `sync-event` | 直接发送 `sync-event` |

### 1.4 事件转发链路

**服务器端** (`packages/loot-core/src/server/main-app.ts:10-12`)：
```typescript
app.events.on('sync', event => {
  connection.send('sync-event', event);
});
```

**客户端** (`packages/desktop-client/src/sync-events.ts:39-46`)：
```typescript
const unlistenSuccess = listen('sync-event', event => {
  // ...
  if (event.type === 'success' || event.type === 'applied') {
    // 两者被同等处理
    // ...
  }
});
```

---

## 二、tables 数组为空的原因

### 2.1 生成逻辑

**文件**: `packages/loot-core/src/server/sync/index.ts:436,569-579`

```typescript
// 第 436 行: 过滤非 old 消息后提取表名
const tables = getTablesFromMessages(messages.filter(msg => !msg.old));

// 第 569-579 行: getTablesFromMessages 实现
function getTablesFromMessages(messages: Message[]): string[] {
  return messages.reduce((acc, message) => {
    const dataset =
      message.dataset === 'schedules_next_date' ? 'schedules' : message.dataset;
    
    if (!acc.includes(dataset)) {
      acc.push(dataset);
    }
    return acc;
  }, []);
}
```

### 2.2 为空场景

| 场景 | `messages.filter(msg => !msg.old)` | `tables` |
|------|------------------------------------|----------|
| **全 old 批次** | `[]`（所有消息都被标记为 old） | `[]` |
| **空批次** | `[]`（没有消息） | `[]` |
| **混合批次（有 new 消息）** | `[m1, m2, ...]`（非 old 消息） | `['transactions', 'accounts', ...]` |

### 2.3 全 old 批次的产生条件

回顾 R3 的异常路径场景：

```
场景: B 已同步 M_B (TS_B > TS_A)，但 Merkle 不一致

时序:
1. B fullSync, since = TS_B
2. Server 返回 newMessages = [], Merkle = H_server
3. B 本地 Merkle = H_b
4. merkle.diff: H_b != H_server
   → diffTime = T_diff (回拉到较早时间点)

5. 递归调用 _fullSync(since = T_diff)
   - T_diff < TS_B

6. Server 查询: WHERE timestamp > T_diff
   → 返回 [M_A, M_B]

7. B 收到 [M_A, M_B]
   → receiveMessages()

8. compareMessages(M_A):
   - 查询: timestamp >= TS_A
   - 本地有 M_B (TS_B > TS_A)
   → 标记 old: true

9. compareMessages(M_B):
   - 查询: timestamp >= TS_B
   - 本地有 M_B (相同时间戳)
   → 重复消息，丢弃

10. messages = [M_A (old: true)]
    → messages.filter(msg => !msg.old) = []
    → tables = []
```

**结论**：全 old 批次产生于：
- 接收的消息都是本地已有更新的消息（被标记为 old）
- 或者接收的消息是重复消息（被丢弃）

---

## 三、监听侧判定分析

### 3.1 监听代码

**文件**: `packages/desktop-client/src/sync-events.ts:46-92`

```typescript
if (event.type === 'success' || event.type === 'applied') {
  // ... 清除 sync repair 标记
  
  const tables = event.tables;
  
  // 检查 1: prefs 表变更
  if (tables.includes('prefs')) {
    void store.dispatch(loadPrefs());
  }
  
  // 检查 2: categories 相关表变更
  if (
    tables.includes('categories') ||
    tables.includes('category_groups') ||
    tables.includes('category_mapping')
  ) {
    void queryClient.invalidateQueries({
      queryKey: categoryQueries.lists(),
    });
  }
  
  // 检查 3: accounts/payees 相关表变更
  if (
    tables.includes('accounts') ||
    tables.includes('payees') ||
    tables.includes('payee_mapping')
  ) {
    void queryClient.invalidateQueries({
      queryKey: payeeQueries.lists(),
    });
  }
  
  // 检查 4: accounts 表变更
  if (tables.includes('accounts')) {
    void queryClient.invalidateQueries({
      queryKey: accountQueries.lists(),
    });
  }
}
```

### 3.2 判定逻辑分析

**`Array.includes()` 的行为**：

```javascript
// 空数组的 includes 检查
[].includes('prefs')           // → false
[].includes('categories')      // → false
[].includes('accounts')        // → false

// 非空数组的 includes 检查
['transactions'].includes('prefs')  // → false
['prefs', 'accounts'].includes('prefs')  // → true
```

### 3.3 全 old 批次的影响

```
场景: tables = []（全 old 批次）

监听侧判定:
1. tables.includes('prefs') → false
   → 不执行 loadPrefs()

2. tables.includes('categories') || ... → false
   → 不失效 categoryQueries.lists()

3. tables.includes('accounts') || ... → false
   → 不失效 payeeQueries.lists()

4. tables.includes('accounts') → false
   → 不失效 accountQueries.lists()

结论: 无任何缓存失效，UI 无感知
```

### 3.4 不同场景的监听侧行为

| 场景 | `tables` | 监听侧行为 |
|------|----------|-----------|
| 正常同步（new 消息） | `['transactions', 'accounts']` | 失效相关缓存，UI 刷新 |
| 全 old 批次 | `[]` | **不失效任何缓存，UI 无感知** |
| 混合批次 | `['categories']` | 失效 categoryQueries，UI 刷新 |
| `type: 'success'` 事件 | `['transactions']`（来自 API 查询） | 正常处理 |

### 3.5 隐式防护机制

**`_syncListeners` 与 `app.events.emit` 的区别**：

```typescript
// 第 434 行: _syncListeners 接收 oldData 和 newData
_syncListeners.forEach(func => func(oldData, newData));

// transaction-rules.ts 示例:
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
```

**分析**：
- `_syncListeners` 比较 `oldData` 和 `newData`
- 全 old 批次时，`oldData === newData`（因为 old 消息不修改业务表）
- 监听器自动"忽略"这些变化，因为没有实际数据变化
- 这是一种**隐式防护**机制

**`app.events.emit` 的 `type: 'applied'` 事件**：
- 依赖 `tables` 数组来判定
- 全 old 批次时，`tables = []`
- 所有 `tables.includes()` 检查都返回 `false`
- **不会触发任何缓存失效**
- 这也是一种**隐式防护**机制

---

## 四、完整事件发布链路

### 4.1 链路图

```
本地修改 / 接收远端消息
    ↓
applyMessages()
    ↓
    ├─ compareMessages() → 标记 old 消息
    ├─ 数据库事务
    │   ├─ !msg.old → apply() 修改业务表
    │   ├─ 所有消息 → 写入 messages_crdt
    │   └─ 所有消息 → 更新 Merkle
    ├─ fetchData() → newData
    │   (从业务表查询，old 消息不影响)
    ├─ _syncListeners.forEach(func => func(oldData, newData))
    │   (全 old 时 oldData === newData)
    ├─ tables = getTablesFromMessages(messages.filter(msg => !msg.old))
    │   (全 old 时 tables = [])
    └─ app.events.emit('sync', {
         type: 'applied',
         tables,
         data: newData,
         prevData: oldData
       })
         (无条件触发)
         ↓
main-app.ts: app.events.on('sync', event => {
  connection.send('sync-event', event);
})
         ↓
desktop-client: listen('sync-event', event => {
  if (event.type === 'success' || event.type === 'applied') {
    const tables = event.tables;
    
    if (tables.includes('prefs')) → false (全 old 时)
    if (tables.includes('categories')) → false (全 old 时)
    if (tables.includes('accounts')) → false (全 old 时)
    
    → 无任何缓存失效
  }
})
```

### 4.2 全 old 批次的完整流程

```
前提:
- B 已同步 M_B (TS_B > TS_A)
- 但 Merkle 不一致，触发递归同步

时序:
1. 递归调用 _fullSync(since = T_diff)
   - T_diff < TS_B

2. Server 查询: WHERE timestamp > T_diff
   → 返回 [M_A, M_B]

3. B 收到 [M_A, M_B]
   → receiveMessages()

4. compareMessages:
   - M_A: 本地有 M_B (TS_B > TS_A) → old: true
   - M_B: 本地有 M_B (相同时间戳) → 丢弃
   
   → messages = [M_A (old: true)]

5. applyMessages([M_A (old: true)]):
   
   // 阶段 1: 准备
   idsPerTable = { transactions: [tx1] }  // old 消息也计入
   oldData = fetchData() 
     → { transactions: { tx1: { amount: 300 } } }
     (从业务表查询，M_B 已应用)
   
   // 阶段 2: 事务
   db.transaction(() => {
     for (const msg of messages) {
       // M_A.old = true → 跳过 apply()
       if (!msg.old) {  // false, 跳过
         apply(msg, ...);
       }
       
       // 所有消息都执行
       INSERT INTO messages_crdt (M_A)
       merkle.insert(M_A)
     }
     
     merkle.prune()
     UPDATE messages_clock
   });
   
   // 阶段 3: 副作用
   newData = fetchData()
     → { transactions: { tx1: { amount: 300 } } }
     (与 oldData 相同！)
   
   // 触发监听器
   _syncListeners.forEach(func => func(oldData, newData))
     → transaction-rules: oldData === newData → 无新增交易
   
   // 生成 tables
   tables = getTablesFromMessages(messages.filter(msg => !msg.old))
     → messages.filter(...) = []
     → tables = []
   
   // 发布事件（无条件触发）
   app.events.emit('sync', {
     type: 'applied',
     tables: [],
     data: { transactions: { tx1: { amount: 300 } } },
     prevData: { transactions: { tx1: { amount: 300 } } },
   })

6. 监听侧 (sync-events.ts):
   tables = []
   
   tables.includes('prefs') → false
   tables.includes('categories') → false
   tables.includes('accounts') → false
   
   → 无任何缓存失效
   → UI 无感知
```

---

## 五、关键代码索引（R4 重点）

| 文件路径 | 行号 | R4 重点 |
|---------|------|---------|
| `packages/loot-core/src/server/sync/index.ts` | 434 | `_syncListeners` 触发 |
| `packages/loot-core/src/server/sync/index.ts` | 436 | `tables` 生成逻辑（过滤 `!msg.old`） |
| `packages/loot-core/src/server/sync/index.ts` | 437-442 | `app.events.emit` 无条件触发 |
| `packages/loot-core/src/server/sync/index.ts` | 569-579 | `getTablesFromMessages` 实现 |
| `packages/loot-core/src/server/main-app.ts` | 10-12 | `sync` 事件转发为 `sync-event` |
| `packages/loot-core/src/server/api.ts` | 73-77 | `type: 'success'` 事件（API 层） |
| `packages/loot-core/src/server/schedules/app.ts` | 588-592 | `type: 'success'` 事件（调度器） |
| `packages/loot-core/src/server/transactions/transaction-rules.ts` | 214 | `addSyncListener` 示例（比较 oldData/newData） |
| `packages/desktop-client/src/sync-events.ts` | 46 | `success` 和 `applied` 同等处理 |
| `packages/desktop-client/src/sync-events.ts` | 61-92 | `tables.includes()` 判定逻辑 |

---

## 六、R3 vs R4 修正对照表

| 条目 | R3 (原) | R4 (精校) |
|------|---------|----------|
| 全 old 批次时 `app.events.emit` 是否触发 | 未详细分析 | **是，无条件触发**（第 437 行） |
| `tables` 为空的原因 | 未深入 | `messages.filter(msg => !msg.old)` 全空时 `tables = []` |
| 监听侧如何判定 | 未深入 | 使用 `tables.includes()`，空数组返回 `false`，不触发任何缓存失效 |
| `_syncListeners` vs `app.events.emit` | 未对比 | `_syncListeners` 比较 `oldData/newData`，`app.events.emit` 依赖 `tables` 数组 |
| 隐式防护机制 | 未深入 | 两套机制：① `oldData === newData` ② `tables = []` |
| `type: 'applied'` vs `type: 'success'` | 未区分 | `applied` 来自 `applyMessages` 内部，`success` 来自 API/调度器，监听侧同等处理 |
| 全 old 批次对 UI 的影响 | 未分析 | **无感知**：`tables = []` 不触发缓存失效，`oldData === newData` 监听器无动作 |
