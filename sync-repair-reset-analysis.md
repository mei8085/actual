# Repair 与 Reset 操作的层级协作机制分析

## 一、操作概述

当预算数据出现同步异常时，Actual Budget 提供两种恢复一致性的操作：

| 操作类型 | 触发场景 | 恢复策略 | 数据影响范围 |
|---------|---------|---------|-------------|
| **repair** | Merkle 哈希不一致 | 重建本地 Merkle 树 | 仅 CRDT 元数据 |
| **reset** | 严重同步冲突、密钥变更 | 清空本地同步状态并重新同步 | CRDT 元数据 + 已删除业务数据 |

---

## 二、架构分层与职责边界

### 2.1 三层架构概览

```
┌─────────────────────────────────────────────────────────────────┐
│                      应用层 (Application)                       │
│  ┌──────────────────┐    ┌──────────────────┐                  │
│  │   Budget Service │    │ Transaction Service│                 │
│  │  (预算业务逻辑)   │    │   (交易业务逻辑)   │                  │
│  └────────┬─────────┘    └────────┬─────────┘                  │
└───────────┼────────────────────────┼────────────────────────────┘
            │                        │
            ▼                        ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Sync 层 (Sync Layer)                       │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  applyMessages / receiveMessages / sendMessages          │    │
│  │  batchMessages / fullSync / scheduleFullSync             │    │
│  └─────────────────────────────────────────────────────────┘    │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                     CRDT 层 (CRDT Layer)                        │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐      │
│  │   Merkle     │    │  Timestamp   │    │    Clock     │      │
│  │    Tree      │    │    Manager   │    │   Manager    │      │
│  └──────────────┘    └──────────────┘    └──────────────┘      │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 各层职责边界

| 层级 | 职责描述 | 核心数据结构 |
|-----|---------|-------------|
| **CRDT层** | 维护分布式一致性、时间戳管理、Merkle树哈希计算 | `Clock`、`Timestamp`、`TrieNode` |
| **Sync层** | 消息序列化/反序列化、消息应用、全量同步调度 | `Message`、`DataMap` |
| **业务服务层** | 预算业务逻辑、交易业务逻辑、业务规则执行 | `TransactionEntity`、`CategoryEntity` |

---

## 三、跨层协作完整流程

### 3.1 同步入口到消息落库的完整顺序

当同步消息从服务器接收后，经历以下完整流程：

```
┌──────────────────────────────────────────────────────────────────────────┐
│                        同步入口                                        │
│  receiveMessages(messages)                                              │
│         │                                                               │
│         ▼                                                               │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │ 阶段1: 时间戳验证                                               │   │
│  │  Timestamp.recv(msg.timestamp)                                  │   │
│  │  → 检测 clock drift 异常                                        │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│         │                                                               │
│         ▼                                                               │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │ 阶段2: 消息比较过滤                                             │   │
│  │  compareMessages(messages)                                      │   │
│  │  → 查询 messages_crdt 判断消息新旧                               │   │
│  │  → 标记 old 消息（无需应用但需记录到 Merkle）                      │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│         │                                                               │
│         ▼                                                               │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │ 阶段3: 消息排序                                                 │   │
│  │  按 timestamp 升序排序                                          │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│         │                                                               │
│         ▼                                                               │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │ 阶段4: 预取旧数据                                               │   │
│  │  fetchData() → DataMap (oldData)                                │   │
│  │  → 为 undo 和变更检测准备                                        │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│         │                                                               │
│         ▼                                                               │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │ 阶段5: 数据库事务应用（关键）                                    │   │
│  │  db.transaction(() => {                                         │   │
│  │    ├─ apply(msg) → INSERT/UPDATE 业务表                        │   │
│  │    ├─ INSERT INTO messages_crdt                               │   │
│  │    └─ merkle.insert() → 更新内存 Merkle 树                     │   │
│  │  })                                                             │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│         │                                                               │
│         ▼                                                               │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │ 阶段6: 更新内存状态                                             │   │
│  │  clock.merkle = currentMerkle                                   │   │
│  │  INSERT OR REPLACE INTO messages_clock                          │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│         │                                                               │
│         ▼                                                               │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │ 阶段7: 获取新数据                                               │   │
│  │  fetchData() → DataMap (newData)                               │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│         │                                                               │
│         ▼                                                               │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │ 阶段8: 预算变更触发                                             │   │
│  │  sheet.startTransaction()                                       │   │
│  │  triggerBudgetChanges(oldData, newData)                         │   │
│  │  → handleTransactionChange() → recompute(sum-amount-{catId})   │   │
│  │  → handleBudgetChange() → set(budget-{catId}, carryover, goal) │   │
│  │  → handleAccountChange() → recompute 相关分类汇总               │   │
│  │  → handleCategoryChange() → 分类变更处理                        │   │
│  │  sheet.endTransaction()                                         │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│         │                                                               │
│         ▼                                                               │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │ 阶段9: 交易汇总刷新                                             │   │
│  │  若涉及 transactions:                                           │   │
│  │    ├─ recompute(accounts-balance)                              │   │
│  │    ├─ recompute(onbudget-accounts-balance)                     │   │
│  │    ├─ recompute(offbudget-accounts-balance)                    │   │
│  │    └─ recompute(closed-accounts-balance)                       │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│         │                                                               │
│         ▼                                                               │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │ 阶段10: 通知监听者                                              │   │
│  │  _syncListeners.forEach(func => func(oldData, newData))         │   │
│  │  app.events.emit('sync', { type: 'applied', tables, data })    │   │
│  └──────────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────────┘
```

### 3.2 关键代码路径

**同步入口函数 `receiveMessages`：**
```typescript
export function receiveMessages(messages: Message[]): Promise<Message[]> {
try {
    messages.forEach(msg => {
    Timestamp.recv(msg.timestamp);  // CRDT层：时间戳验证
    });
} catch (e) {
    if (e instanceof Timestamp.ClockDriftError) {
    throw new SyncError('clock-drift');  // 抛出时钟漂移异常
    }
    throw e;
}
return runMutator(() => applyMessages(messages));  // Sync层：应用消息
}
```

**消息应用核心流程 `applyMessages`：**
```typescript
export const applyMessages = sequential(async (messages: Message[]) => {
// 阶段1: 消息比较过滤
messages = await compareMessages(messages);

// 阶段2: 获取旧数据用于变更检测
const oldData = await fetchData();

// 阶段3: 数据库事务应用
db.transaction(() => {
    for (const msg of messages) {
    if (!msg.old) {
        apply(msg);  // 应用到业务表
    }
    db.runQuery('INSERT INTO messages_crdt ...');  // 记录CRDT消息
    currentMerkle = merkle.insert(currentMerkle, timestamp);  // 更新Merkle
    }
});

// 阶段4: 触发预算变更
const newData = await fetchData();
triggerBudgetChanges(oldData, newData);

// 阶段5: 刷新全局汇总
if (idsPerTable.transactions?.length) {
    // recompute global aggregate cells...
}
});
```

### 3.3 预算变更触发的详细逻辑

`triggerBudgetChanges` 根据变更的数据类型触发不同处理：

| 数据表 | 处理函数 | 触发条件 | 刷新内容 |
|-------|---------|---------|---------|
| `transactions` | `handleTransactionChange` | date/acct/amount/category/tombstone变化 | `sum-amount-{catId}` |
| `zero_budgets`/`reflect_budgets` | `handleBudgetChange` | 预算金额变化 | `budget-{catId}`, `carryover`, `goal` |
| `category_mapping` | `handleCategoryMappingChange` | 分类映射变化 | `sum-amount-{transferId}` |
| `categories` | `handleCategoryChange` | 分类增删改 | 分类相关计算单元 |
| `category_groups` | `handleCategoryGroupChange` | 分类组变化 | 分组相关计算单元 |
| `accounts` | `handleAccountChange` | offbudget状态变化 | 关联分类的 `sum-amount` |

---

## 四、异常处理路径详解

### 4.1 Clock Drift（时钟漂移）异常

**触发条件**：客户端时间戳与服务器时间戳差异过大。

**异常上抛路径**：

```
CRDT层
    │
    ├─ Timestamp.recv(msg.timestamp)
    │       │
    │       └─→ 检测到时钟漂移
    │               │
    │               ▼
    │       throw Timestamp.ClockDriftError()
    │               │
    │               ▼
Sync层
    │
    ├─ receiveMessages() 捕获异常
    │       │
    │       └─→ throw new SyncError('clock-drift')
    │               │
    │               ▼
    │   errorHandler() 处理
    │       │
    │       └─→ app.events.emit('sync', { type: 'error', subtype: 'clock-drift' })
    │               │
    │               ▼
应用层
    │
    └─ 监听 sync 事件的 UI 组件显示时钟漂移错误提示
```

**代码实现**：
```typescript
// packages/loot-core/src/server/sync/index.ts
export function receiveMessages(messages: Message[]): Promise<Message[]> {
try {
    messages.forEach(msg => {
    Timestamp.recv(msg.timestamp);
    });
} catch (e) {
    if (e instanceof Timestamp.ClockDriftError) {
    throw new SyncError('clock-drift');
    }
    throw e;
}
return runMutator(() => applyMessages(messages));
}
```

### 4.2 哈希长期不一致（Out of Sync）异常

**触发条件**：Merkle 树差异循环超过 10 次且 diffTime 相同，或超过 100 次循环。

**异常上抛路径**：

```
Sync层
    │
    ├─ _fullSync() 执行同步循环
    │       │
    │       ├─ merkle.diff(res.merkle, getClock().merkle)
    │       │       │
    │       │       └─→ diffTime !== null (存在差异)
    │       │               │
    │       │               ▼
    │       │   循环计数检查:
    │       │   if ((count >= 10 && diffTime === prevDiffTime) || count >= 100)
    │       │           │
    │       │           ▼
    │       │   rebuildMerkleHash() 尝试重建
    │       │           │
    │       │           ▼
    │       │   throw new SyncError('out-of-sync')
    │       │           │
    │       │           ▼
    │       └─ errorHandler() 处理
    │               │
    │               └─→ app.events.emit('sync', { type: 'error', subtype: 'out-of-sync' })
    │                       │
    │                       ▼
应用层
    │
    └─ 监听 sync 事件，提示用户执行 repair 或 reset
```

**代码实现**：
```typescript
// packages/loot-core/src/server/sync/index.ts
async function _fullSync(sinceTimestamp, count, prevDiffTime) {
// ... 同步逻辑 ...
const diffTime = merkle.diff(res.merkle, getClock().merkle);

if (diffTime !== null) {
    if ((count >= 10 && diffTime === prevDiffTime) || count >= 100) {
    const rebuiltMerkle = rebuildMerkleHash();
    
    if (rebuiltMerkle.trie.hash === res.merkle.hash) {
        // 重建后匹配，说明是内存时钟问题
        logger.log('Merkle hash in db:', hash);
    }
    
    throw new SyncError('out-of-sync');
    }
    
    // 继续循环同步
    return _fullSync(new Timestamp(diffTime, 0, '0').toString(), ...);
}
}
```

### 4.3 上传失败（Upload Failure）异常

**触发条件**：reset 操作最后阶段上传文件到云端失败。

**异常上抛路径**：

```
Sync层
    │
    ├─ resetSync(keyState?)
    │       │
    │       ├─ 阶段1: cloudStorage.checkKey()
    │       │       │
    │       │       └─→ 失败 → return { error: { reason: 'file-has-new-key' } }
    │       │               │
    │       │               ▼
    │       ├─ 阶段2: cloudStorage.resetSyncState()
    │       │       │
    │       │       └─→ 失败 → return { error }
    │       │               │
    │       │               ▼
    │       ├─ 阶段3: 数据库清理 (runMutator)
    │       │               │
    │       │               ▼
    │       └─ 阶段4: cloudStorage.upload()
    │               │
    │               └─→ 失败
    │                       │
    │                       ▼
    │               try { await upload() }
    │               catch (e) {
    │                   if (e.reason) return { error: e };
    │                   captureException(e);
    │                   return { error: { reason: 'upload-failure' } };
    │               } finally {
    │                   connection.send('prefs-updated');  // 确保通知
    │               }
    │                       │
    │                       ▼
应用层
    │
    └─ 接收 resetSync 返回值，显示上传失败错误
```

**代码实现**：
```typescript
// packages/loot-core/src/server/sync/reset.ts
export async function resetSync(keyState?) {
// 密钥检查
if (!keyState) {
    const { valid, error } = await cloudStorage.checkKey();
    if (error) return { error };
    if (!valid) return { error: { reason: 'file-has-new-key' } };
}

// 重置云状态
const { error } = await cloudStorage.resetSyncState(keyState);
if (error) return { error };

// 数据库清理
await runMutator(async () => {
    db.execQuery(`DELETE FROM messages_crdt; ...`);
    await db.loadClock();
});

// 上传到云端
try {
    await cloudStorage.upload();
} catch (e) {
    if (e.reason) {
    return { error: e };
    }
    captureException(e);
    return { error: { reason: 'upload-failure' } };
} finally {
    connection.send('prefs-updated');
}

return {};
}
```

### 4.4 异常处理对比表

| 异常类型 | 触发层 | 上抛路径 | 通知方式 | 用户操作 |
|---------|-------|---------|---------|---------|
| **Clock Drift** | CRDT层 | `Timestamp.recv` → `SyncError('clock-drift')` | `emit('sync', { type: 'error', subtype: 'clock-drift' })` | 检查系统时间 |
| **Out of Sync** | Sync层 | `_fullSync` 循环检测 → `SyncError('out-of-sync')` | `emit('sync', { type: 'error', subtype: 'out-of-sync' })` | 执行 repair/reset |
| **Upload Failure** | 云存储层 | `cloudStorage.upload()` → 返回错误对象 | `connection.send('prefs-updated')` + 返回 error | 重试上传 |

### 4.5 reset 失败路径的返回契约与通知闭环

#### 4.5.1 resetSync 调用链路

```
客户端层 (desktop-client)
        │
        ├─ store.dispatch(resetSync())
        │       │
        │       ▼
        │   createAppAsyncThunk: `${sliceName}/resetSync`
        │       │
        │       ▼
        │   await send('sync-reset')
        │       │
        │       ▼
Sync层 (loot-core)
        │
        ├─ app.method('sync-reset', resetSync)
        │       │
        │       ▼
        │   export async function resetSync(keyState?)
        │       │
        │       ├─ cloudStorage.checkKey() → { valid, error }
        │       │       │
        │       │       └─ error → return { error }
        │       │
        │       ├─ cloudStorage.resetSyncState() → { error }
        │       │       │
        │       │       └─ error → return { error }
        │       │
        │       ├─ runMutator(() => { db.execQuery(...) })
        │       │
        │       └─ cloudStorage.upload()
        │               │
        │               └─ error → return { error: { reason: 'upload-failure' } }
        │
        ▼
返回 { error?: { reason: string; meta?: unknown } }
        │
        ▼
客户端层消费
        │
        ├─ const { error } = await send('sync-reset');
        │       │
        │       └─ error?.reason === 'encrypt-failure' && error.meta?.isMissingKey
        │               │
        │               └─→ pushModal('fix-encryption-key')
        │       │
        │       └─ error?.reason === 'file-has-new-key'
        │               │
        │               └─→ pushModal('fix-encryption-key')
        │       │
        │       └─ error?.reason === 'encrypt-failure'
        │               │
        │               └─→ pushModal('create-encryption-key')
        │       │
        │       └─ 其他 error → alert(getUploadError(error))
        │
        └─ 无 error → dispatch(sync())
```

#### 4.5.2 resetSync 返回契约

| 返回字段 | 类型 | 含义 | 触发场景 |
|---------|------|------|---------|
| `error` | `undefined \| { reason: string; meta?: unknown }` | 错误信息 | 任意阶段失败 |
| `error.reason` | `string` | 错误原因 | 标识具体失败类型 |
| `error.meta` | `unknown` | 错误元数据 | 附加错误详情 |

#### 4.5.3 error.reason 消费位置

```typescript
// 客户端层消费 (appSlice.ts)
const { error } = await send('sync-reset');

if (error) {
alert(getUploadError(error));

if (
    (error.reason === 'encrypt-failure' && error.meta?.isMissingKey) ||
    error.reason === 'file-has-new-key'
) {
    dispatch(pushModal({ modal: { name: 'fix-encryption-key', ... } }));
} else if (error.reason === 'encrypt-failure') {
    dispatch(pushModal({ modal: { name: 'create-encryption-key', ... } }));
}
} else {
await dispatch(sync());
}
```

#### 4.5.4 sync 事件链路对比（Clock Drift vs Out of Sync）

**Clock Drift 事件链路：**

```
CRDT层
    │
    ├─ Timestamp.recv(msg.timestamp)
    │       │
    │       └─→ ClockDriftError
    │               │
    │               ▼
Sync层
    │
    ├─ receiveMessages() 捕获
    │       │
    │       └─→ throw SyncError('clock-drift')
    │               │
    │               ▼
    ├─ errorHandler()
    │       │
    │       └─→ app.events.emit('sync', { type: 'error', subtype: 'clock-drift', meta })
    │               │
    │               ▼
平台层
    │
    ├─ connection.send('sync-event', { type: 'error', subtype: 'clock-drift' })
    │               │
    │               ▼
客户端层
    │
    ├─ listen('sync-event', event => { ... })
    │       │
    │       └─→ event.subtype === 'clock-drift'
    │               │
    │               └─→ addNotification({ title: 'Time sync issue', ... })
```

**Out of Sync 事件链路：**

```
Sync层
    │
    ├─ _fullSync()
    │       │
    │       └─→ 循环检测失败 (count >= 10 || count >= 100)
    │               │
    │               ▼
    │       throw SyncError('out-of-sync')
    │               │
    │               ▼
    ├─ errorHandler()
    │       │
    │       └─→ app.events.emit('sync', { type: 'error', subtype: 'out-of-sync', meta })
    │               │
    │               ▼
平台层
    │
    ├─ connection.send('sync-event', { type: 'error', subtype: 'out-of-sync' })
    │               │
    │               ▼
客户端层
    │
    ├─ listen('sync-event', event => { ... })
    │       │
    │       └─→ event.subtype === 'out-of-sync'
    │               │
    │               └─→ 根据 attemptedSyncRepair 状态显示不同通知
    │                       │
    │                       ├─ 首次: 显示 Repair 按钮
    │                       └─ 重试后仍失败: 显示 Reset sync 按钮
```

#### 4.5.5 预算层与交易层信号接收对比

| 异常类型 | 预算层 (Budget Service) | 交易层 (Transaction Service) | 说明 |
|---------|----------------------|---------------------------|------|
| **Clock Drift** | ❌ 无信号 | ❌ 无信号 | 发生在消息接收前，业务层未参与 |
| **Out of Sync** | ❌ 无直接信号 | ❌ 无直接信号 | 同步循环检测失败，未触发业务变更 |
| **Upload Failure** | ❌ 无信号 | ❌ 无信号 | 发生在数据库操作后，业务数据已清理 |
| **sync-success/applied** | ✅ 收到 `tables.includes('categories')` 信号 | ✅ 收到 `tables.includes('transactions')` 信号 | 正常同步完成时触发 |
| **sync-error (invalid-schema)** | ⚠️ 间接影响 | ⚠️ 间接影响 | 数据库 schema 不兼容，需更新应用 |

#### 4.5.6 各层信号汇总表

| 层级 | Clock Drift | Out of Sync | Upload Failure | Normal Sync |
|-----|------------|-------------|----------------|-------------|
| **CRDT层** | ✅ 触发异常 | ❌ 不涉及 | ❌ 不涉及 | ✅ 时间戳验证通过 |
| **Sync层** | ✅ 捕获并转发 | ✅ 检测并抛出 | ✅ 捕获并返回 | ✅ 消息应用成功 |
| **数据库层** | ❌ 未执行 | ❌ 未执行 | ✅ 已清理完成 | ✅ 数据已更新 |
| **预算服务层** | ❌ 无信号 | ❌ 无信号 | ❌ 无信号 | ✅ `triggerBudgetChanges()` 调用 |
| **交易服务层** | ❌ 无信号 | ❌ 无信号 | ❌ 无信号 | ✅ `batchMessages()` 调用 |
| **客户端UI层** | ✅ 显示通知 | ✅ 显示修复/重置按钮 | ✅ 显示错误提示 | ✅ 更新数据缓存 |

---

## 五、repair 操作分析

### 3.1 操作定义与触发条件

**`repairSync()`** - 当检测到 Merkle 哈希不一致但数据本身完整时触发。

```typescript
// packages/loot-core/src/server/sync/repair.ts
export async function repairSync(): Promise<void> {
const rebuilt = rebuildMerkleHash();  // 重建 Merkle 树
const clock = getClock();             // 获取当前时钟
clock.merkle = rebuilt.trie;          // 更新内存中的 Merkle 树
db.runQuery(/* 持久化到 messages_clock */); // 持久化到数据库
}
```

### 3.2 执行流程

```
触发 repairSync
        │
        ▼
┌────────────────────────────────┐
│  rebuildMerkleHash()           │
│  ├─ SELECT timestamp FROM     │
│  │   messages_crdt            │  ← 读取所有CRDT消息时间戳
│  └─ merkle.insert() × N       │  ← 逐个插入重建Merkle树
└────────────────────────────────┘
        │
        ▼
┌────────────────────────────────┐
│  更新内存 Clock.merkle         │  ← 同步内存状态
└────────────────────────────────┘
        │
        ▼
┌────────────────────────────────┐
│  INSERT OR REPLACE INTO       │  ← 持久化到数据库
│  messages_clock               │
└────────────────────────────────┘
```

### 3.3 层级协作关系

| 层级 | 参与方式 | 职责 |
|-----|---------|------|
| **CRDT层** | `getClock()` 获取时钟，`merkle.insert()` 重建树 | Merkle树计算 |
| **Sync层** | 调用 `repairSync()` | 协调修复流程 |
| **业务层** | **不参与** | 数据无变更 |

### 3.4 数据流转

```
messages_crdt (DB)
        │ SELECT timestamp
        ▼
merkle.insert() → TrieNode (内存)
        │
        ▼
messages_clock (DB) ← INSERT OR REPLACE
```

---

## 四、reset 操作分析

### 4.1 操作定义与触发条件

**`resetSync(keyState?)`** - 当出现严重同步冲突或密钥变更时触发。

### 4.2 执行流程

```
触发 resetSync(keyState?)
        │
        ▼
┌────────────────────────────────┐
│  阶段1: 密钥检查               │
│  └─ cloudStorage.checkKey()   │  ← 验证密钥有效性
└────────────────────────────────┘
        │
        ▼
┌────────────────────────────────┐
│  阶段2: 重置云同步状态          │
│  └─ cloudStorage.resetSyncState() │
└────────────────────────────────┘
        │
        ▼
┌────────────────────────────────┐
│  阶段3: 清理本地数据            │
│  ├─ DELETE FROM messages_crdt  │  ← 清空CRDT消息
│  ├─ DELETE FROM messages_clock │  ← 清空时钟状态
│  ├─ DELETE FROM transactions   │  ← 清理已删除交易
│  ├─ DELETE FROM accounts      │  ← 清理已删除账户
│  ├─ DELETE FROM payees        │  ← 清理已删除收款人
│  ├─ DELETE FROM categories    │  ← 清理已删除分类
│  ├─ ANALYZE                   │
│  └─ VACUUM                    │  ← 回收数据库空间
└────────────────────────────────┘
        │
        ▼
┌────────────────────────────────┐
│  阶段4: 重置偏好设置            │
│  └─ savePrefs({ groupId: null, │
│       lastSyncedTimestamp: null })
└────────────────────────────────┘
        │
        ▼
┌────────────────────────────────┐
│  阶段5: 更新密钥(可选)          │
│  └─ 更新 encrypt-keys 存储     │
└────────────────────────────────┘
        │
        ▼
┌────────────────────────────────┐
│  阶段6: 上传文件到云端          │
│  └─ cloudStorage.upload()     │  ← 成为新的权威版本
└────────────────────────────────┘
```

### 4.3 层级协作关系

| 层级 | 参与方式 | 职责 |
|-----|---------|------|
| **CRDT层** | `db.loadClock()` 重新加载时钟 | 重置时钟状态 |
| **Sync层** | 调用 `resetSync()`，触发 `connection.send('prefs-updated')` | 协调重置流程 |
| **业务层** | 通过数据库清理间接影响 | 清理已删除业务数据 |

### 4.4 数据流转

```
┌─────────────────────────────────────────────────────────────────┐
│                        数据清理路径                              │
├─────────────────────────────────────────────────────────────────┤
│  messages_crdt ── DELETE ──→ (清空CRDT消息历史)                  │
│  messages_clock ── DELETE ──→ (清空时钟状态)                     │
│  transactions (tombstone=1) ── DELETE ──→ (清理已删除交易)        │
│  accounts (tombstone=1) ── DELETE ──→ (清理已删除账户)            │
│  payees (tombstone=1) ── DELETE ──→ (清理已删除收款人)           │
│  categories (tombstone=1) ── DELETE ──→ (清理已删除分类)          │
│  category_groups (tombstone=1) ── DELETE ──→ (清理已删除分类组)    │
│  schedules (tombstone=1) ── DELETE ──→ (清理已删除计划)           │
│  rules (tombstone=1) ── DELETE ──→ (清理已删除规则)               │
└─────────────────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────────────┐
│                        状态重置                                  │
├─────────────────────────────────────────────────────────────────┤
│  prefs: groupId=null, lastSyncedTimestamp=null, lastUploaded=null │
│  encrypt-keys: 更新密钥(如果提供)                                 │
└─────────────────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────────────┐
│                        上传到云端                                │
├─────────────────────────────────────────────────────────────────┤
│  cloudStorage.upload() ──→ 成为权威版本供其他客户端同步            │
└─────────────────────────────────────────────────────────────────┘
```

---

## 五、异常处理路径

### 5.1 repair 操作的异常处理

| 异常场景 | 处理方式 | 恢复策略 |
|---------|---------|---------|
| 数据库查询失败 | 抛出原始异常 | 上层捕获处理 |
| Merkle树构建失败 | 抛出异常 | 需人工介入检查数据完整性 |

### 5.2 reset 操作的异常处理

| 阶段 | 异常场景 | 处理方式 | 返回错误 |
|-----|---------|---------|---------|
| 密钥检查 | 密钥无效 | 返回错误对象 | `{ reason: 'file-has-new-key' }` |
| 云状态重置 | 重置失败 | 返回错误对象 | `{ error }` |
| 文件上传 | 上传失败 | 捕获异常 | `{ reason: 'upload-failure' }` |

```typescript
// packages/loot-core/src/server/sync/reset.ts
try {
await cloudStorage.upload();
} catch (e) {
if (e.reason) {
    return { error: e };
}
captureException(e);
return { error: { reason: 'upload-failure' } };
} finally {
connection.send('prefs-updated'); // 确保通知客户端
}
```

---

## 六、关键设计要点

### 6.1 事务性保证

`resetSync` 使用 `runMutator` 包裹数据库操作，确保原子性：

```typescript
await runMutator(async () => {
db.execQuery(`
    DELETE FROM messages_crdt;
    DELETE FROM messages_clock;
    DELETE FROM transactions WHERE tombstone = 1;
    // ... 其他删除操作
    ANALYZE;
    VACUUM;
`);
await db.loadClock();  // 重新加载时钟
});
```

### 6.2 墓碑机制 (Tombstone)

业务数据删除采用软删除策略，`resetSync` 仅清理标记为 `tombstone = 1` 的数据：

```sql
DELETE FROM transactions WHERE tombstone = 1;
DELETE FROM accounts WHERE tombstone = 1;
-- ...
```

### 6.3 密钥管理

支持密钥变更场景，当 `keyState` 传入时：

1. 更新本地密钥存储 (`encrypt-keys`)
2. 保存密钥 ID 到偏好设置

---

## 七、操作对比总结

| 维度 | repair | reset |
|-----|--------|-------|
| **数据影响** | 仅 CRDT 元数据 | CRDT 元数据 + 已删除业务数据 |
| **触发场景** | Merkle 哈希不一致 | 严重冲突、密钥变更 |
| **云端交互** | 无 | 上传新权威版本 |
| **恢复方式** | 重建哈希 | 清空状态 + 重新同步 |
| **数据丢失风险** | 无 | 已删除数据永久删除 |
| **执行代价** | 低 (本地操作) | 高 (云端上传) |

---

## 八、协作机制总结

### 8.1 层间交互模式

```
应用层触发
        │
        ▼
Sync层: repairSync() / resetSync()
        │
        ├─→ CRDT层: getClock(), merkle.insert()
        │
        ├─→ 数据库层: DELETE/INSERT操作
        │
        └─→ 云存储层: checkKey(), resetSyncState(), upload()
```

### 8.2 一致性保障机制

1. **原子性**: `runMutator` 确保数据库操作的原子性
2. **幂等性**: 操作设计保证可重复执行
3. **最终一致性**: 重置后通过上传成为权威版本，其他客户端拉取同步

### 9.3 边界清晰性

- **CRDT层**: 只关注分布式一致性和哈希计算
- **Sync层**: 只关注消息同步和状态管理
- **业务层**: 只关注业务逻辑，通过 `batchMessages` 与 Sync 层交互
