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

## 三、repair 操作分析

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

### 8.3 边界清晰性

- **CRDT层**: 只关注分布式一致性和哈希计算
- **Sync层**: 只关注消息同步和状态管理
- **业务层**: 只关注业务逻辑，通过 `batchMessages` 与 Sync 层交互
