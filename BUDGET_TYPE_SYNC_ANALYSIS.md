# BudgetType 跨设备同步完整链路分析

## 目录

1. [概述](#1-概述)
2. [本地修改到消息生成（Step 1）](#2-本地修改到消息生成step-1)
3. [消息持久化与本地应用（Step 2）](#3-消息持久化与本地应用step-2)
4. [全量同步上传到服务器（Step 3）](#4-全量同步上传到服务器step-3)
5. [其他客户端拉取与应用（Step 4）](#5-其他客户端拉取与应用step-4)
6. [budgetType 特殊处理触发预算重建（Step 5）](#6-budgettype-特殊处理触发预算重建step-5)
7. [导入流程中的特殊处理](#7-导入流程中的特殊处理)
8. [关键数据结构](#8-关键数据结构)

---

## 1. 概述

本报告详细分析 Actual Budget 中 `budgetType`（预算模式：Envelope / Tracking）从本地修改到跨设备同步的完整链路。整个流程涉及 5 个关键阶段：

```
[前端设置页面] → [API层] → [数据库层] → [同步消息生成] → [本地应用]
                                                         ↓
                                                  [调度全量同步]
                                                         ↓
                                                  [服务器端存储]
                                                         ↓
                                                  [其他客户端拉取]
                                                         ↓
                                                  [触发预算模式切换]
```

---

## 2. 本地修改到消息生成（Step 1）

### 2.1 前端触发

**位置**: `packages/desktop-client/src/components/settings/BudgetTypeSettings.tsx:14-28`

```typescript
export function BudgetTypeSettings() {
  // 1. 从 Redux store 读取当前预算类型，默认为 'envelope'
  const [budgetType = 'envelope', setBudgetType] = useSyncedPref('budgetType');

  async function onSwitchType() {
    const newBudgetType = budgetType === 'envelope' ? 'tracking' : 'envelope';
    // 2. 切换预算类型，触发同步偏好保存
    setBudgetType(newBudgetType);
    // 3. 重置预算缓存
    await send('reset-budget-cache');
  }
}
```

### 2.2 useSyncedPref Hook

**位置**: `packages/desktop-client/src/hooks/useSyncedPrefs.ts`

```typescript
export function useSyncedPref<K extends keyof SyncedPrefs>(
  prefName: K,
): [SyncedPrefs[K], SetSyncedPrefAction<K>] {
  const dispatch = useDispatch();
  const setPref = useCallback<SetSyncedPrefAction<K>>(
    value => {
      // 触发 saveSyncedPrefs action，最终调用后端 API
      void dispatch(saveSyncedPrefs({ prefs: { [prefName]: value } }));
    },
    [prefName, dispatch],
  );
  // 从 Redux store 读取值
  const pref = useSelector(state => state.prefs.synced[prefName]);

  return [pref, setPref];
}
```

### 2.3 后端偏好保存 Handler

**位置**: `packages/loot-core/src/server/preferences/app.ts:30-53`

```typescript
// 标记为 mutator，使其在 mutator 上下文中执行
app.method('preferences/save', mutator(undoable(saveSyncedPrefs)));

async function saveSyncedPrefs({
  id,
  value,
}: {
  id: keyof SyncedPrefs;
  value: string | undefined;
}) {
  if (!id) {
    return;
  }

  // 调用 db.update 生成同步消息
  await db.update('preferences', {
    id,
    value,
  });
}
```

### 2.4 数据库层生成同步消息

**位置**: `packages/loot-core/src/server/db/index.ts:205-223`

```typescript
export async function update(table, params) {
  const fields = Object.keys(params).filter(k => k !== 'id');

  if (params.id == null) {
    throw new Error('update: id is required');
  }

  // 关键：为每个变更字段生成一条同步消息
  await sendMessages(
    fields.map(k => {
      return {
        dataset: table,           // 'preferences'
        row: params.id,           // 'budgetType'
        column: k,                // 'value'
        value: params[k],         // 'envelope' 或 'tracking'
        timestamp: Timestamp.send(), // 生成 Lamport 时间戳
      };
    }),
  );
}
```

**生成的消息结构示例**：

```typescript
{
  dataset: 'preferences',
  row: 'budgetType',
  column: 'value',
  value: 'tracking',
  timestamp: Timestamp { 
    millis: 1715865600000,  // Unix 时间戳
    counter: 0,             // 逻辑时钟
    node: 'client-uuid'     // 客户端唯一标识
  }
}
```

---

## 3. 消息持久化与本地应用（Step 2）

### 3.1 sendMessages 入口

**位置**: `packages/loot-core/src/server/sync/index.ts:526-532`

```typescript
export async function sendMessages(messages: Message[]) {
  if (IS_BATCHING) {
    // 如果在批量处理模式下，先暂存消息
    _BATCHED = _BATCHED.concat(messages);
  } else {
    // 立即处理消息
    return _sendMessages(messages);
  }
}
```

### 3.2 _sendMessages 处理流程

**位置**: `packages/loot-core/src/server/sync/index.ts:488-497`

```typescript
async function _sendMessages(messages: Message[]): Promise<void> {
  try {
    // 应用消息到本地数据库
    await applyMessages(messages);
  } catch (e) {
    void errorHandler(e);
    throw e;
  }

  // 关键：调度下一次全量同步
  await scheduleFullSync();
}
```

### 3.3 applyMessages 核心逻辑

**位置**: `packages/loot-core/src/server/sync/index.ts:261-445`

```typescript
export const applyMessages = sequential(async (messages: Message[]) => {
  // 导入模式下使用简化处理
  if (checkSyncingMode('import')) {
    applyMessagesForImport(messages);
    return undefined;
  } else if (checkSyncingMode('enabled')) {
    // 比较消息，过滤掉已应用的旧消息
    const newMessages = await compareMessages(messages);
    
    // 获取变更影响的表
    const tables = Array.from(
      new Set(newMessages.filter(msg => !msg.old).map(msg => msg.dataset)),
    );

    // 通知 undo 系统
    undo.appendMessages(messages, oldData);

    // 读取时钟和 merkle 树
    let clock;
    let currentMerkle;
    if (checkSyncingMode('enabled')) {
      clock = getClock();
      currentMerkle = clock.merkle;
    }

    // ===== 关键：数据库事务 =====
    db.transaction(() => {
      const added = new Set();

      for (const msg of messages) {
        const { dataset, row, column, timestamp, value } = msg;

        if (!msg.old) {
          // 应用消息到数据库（INSERT 或 UPDATE）
          apply(msg, getIn(oldData, [dataset, row]) || added.has(dataset + row));

          if (dataset === 'prefs') {
            prefsToSet[row] = value;
          } else {
            added.add(dataset + row);
          }
        }

        if (checkSyncingMode('enabled')) {
          // ===== 关键：写入 messages_crdt 表 =====
          db.runQuery(
            db.cache(`INSERT INTO messages_crdt (timestamp, dataset, row, column, value)
             VALUES (?, ?, ?, ?, ?)`),
            [
              timestamp.toString(),  // 序列化时间戳
              dataset,               // 'preferences'
              row,                   // 'budgetType'
              column,                // 'value'
              serializeValue(value), // 序列化值（类型前缀）
            ],
          );

          // 更新 merkle 树用于一致性校验
          currentMerkle = merkle.insert(currentMerkle, timestamp);
        }

        // ===== 关键：budgetType 特殊处理 =====
        // 这是触发预算模式切换的关键点！
        if (dataset === 'preferences' && row === 'budgetType') {
          void setBudgetType(value);  // 立即触发预算类型切换
        }
      }

      if (checkSyncingMode('enabled')) {
        currentMerkle = merkle.prune(currentMerkle);

        // 保存时钟到数据库
        db.runQuery(
          db.cache(
            'INSERT OR REPLACE INTO messages_clock (id, clock) VALUES (1, ?)',
          ),
          [serializeClock({ ...clock, merkle: currentMerkle })],
        );
      }
    });

    // 更新内存中的时钟
    if (checkSyncingMode('enabled')) {
      clock.merkle = currentMerkle;
    }

    // 保存元数据偏好
    if (Object.keys(prefsToSet).length > 0) {
      void prefs.savePrefs(prefsToSet, { avoidSync: true });
      connection.send('prefs-updated');
    }

    // 触发预算变更监听
    if (sheet.get()) {
      sheet.startTransaction();
      triggerBudgetChanges(oldData, newData);
      sheet.get().triggerDatabaseChanges(oldData, newData);
      sheet.endTransaction();
    }

    // 触发同步事件
    _syncListeners.forEach(func => func(oldData, newData));
    const tables = getTablesFromMessages(messages.filter(msg => !msg.old));
    app.events.emit('sync', {
      type: 'applied',
      tables,
      data: newData,
      prevData: oldData,
    });

    return messages;
  }
});
```

### 3.4 apply 函数（数据库实际写入）

**位置**: `packages/loot-core/src/server/sync/index.ts:80-108`

```typescript
function apply(msg: Message, prev?: boolean) {
  const { dataset, row, column, value } = msg;

  if (dataset === 'prefs') {
    // prefs 存储在 metadata.json，不在数据库中
    // 不做任何处理，在事务外单独处理
  } else {
    let query;
    try {
      if (prev) {
        // 行已存在，执行 UPDATE
        query = {
          sql: `UPDATE ${dataset} SET ${column} = ? WHERE id = ?`,
          params: [value, row],
        };
      } else {
        // 行不存在，执行 INSERT
        query = {
          sql: `INSERT INTO ${dataset} (id, ${column}) VALUES (?, ?)`,
          params: [row, value],
        };
      }

      db.runQuery(db.cache(query.sql), query.params);
    } catch (error) {
      throw new SyncError('invalid-schema', {
        error: { message: error.message, stack: error.stack },
        query,
      });
    }
  }
}
```

---

## 4. 全量同步上传到服务器（Step 3）

### 4.1 scheduleFullSync 调度

**位置**: `packages/loot-core/src/server/sync/index.ts:549-567`

```typescript
let syncTimeout = null;

export function scheduleFullSync(): Promise<
  { messages: Message[] } | { error: unknown }
> {
  // 清除已存在的定时器，实现防抖
  clearFullSyncTimeout();

  if (checkSyncingMode('enabled') && !checkSyncingMode('offline')) {
    if (process.env.NODE_ENV === 'test') {
      // 测试环境立即执行
      return fullSync().then(res => {
        if (isError(res)) {
          throw res.error;
        }
        return res;
      });
    } else {
      // 延迟执行，默认 1000ms（见第 40 行）
      syncTimeout = setTimeout(fullSync, FULL_SYNC_DELAY);
    }
  }
}
```

### 4.2 fullSync 入口

**位置**: `packages/loot-core/src/server/sync/index.ts:597-671`

```typescript
export const fullSync = once(async function (): Promise<
  | { messages: Message[] }
  | { error: { message: string; reason: string; meta: unknown } }
> {
  app.events.emit('sync', { type: 'start' });
  let messages;

  try {
    // 执行实际同步逻辑
    messages = await _fullSync(null, 0, null);
  } catch (e) {
    // 错误处理...
    return { error: { message: e.message, reason: e.reason, meta: e.meta } };
  }

  const tables = getTablesFromMessages(messages);

  app.events.emit('sync', {
    type: 'success',
    tables,
    syncDisabled: checkSyncingMode('disabled'),
  });
  return { messages };
});
```

### 4.3 _fullSync 核心逻辑

**位置**: `packages/loot-core/src/server/sync/index.ts:673-844`

```typescript
async function _fullSync(
  sinceTimestamp: string,
  count: number,
  prevDiffTime: number,
): Promise<Message[]> {
  // 获取当前预算的元数据
  const {
    id: currentId,
    cloudFileId,
    groupId,
    lastSyncedTimestamp,
  } = prefs.getPrefs() || {};

  clearFullSyncTimeout();

  if (
    checkSyncingMode('disabled') ||
    checkSyncingMode('offline') ||
    !currentId
  ) {
    return [];
  }

  // 快照当前同步时间点
  const currentTime = getClock().timestamp.toString();

  // ===== 关键：获取需要同步的消息 =====
  // sinceTimestamp: 递归同步时传入的更早时间点
  // lastSyncedTimestamp: 上次成功同步的时间
  // 默认: 5 分钟前
  const since =
    sinceTimestamp ||
    lastSyncedTimestamp ||
    new Timestamp(Date.now() - 5 * 60 * 1000, 0, '0').toString();

  // 从 messages_crdt 表查询所有 since 之后的消息
  const messages = getMessagesSince(since);

  // 获取用户 token
  const userToken = await asyncStorage.getItem('user-token');

  // ===== 关键：编码并发送到服务器 =====
  const buffer = await encoder.encode(groupId, cloudFileId, since, messages);

  const resBuffer = await postBinary(
    getServer().SYNC_SERVER + '/sync',
    buffer,
    {
      'X-ACTUAL-TOKEN': userToken,
    },
  );

  // 检查预算文件是否还存在
  if (!prefs.getPrefs() || prefs.getPrefs().groupId !== groupId) {
    return [];
  }

  // 解码服务器响应
  const res = await encoder.decode(resBuffer);

  const localTimeChanged = getClock().timestamp.toString() !== currentTime;

  // ===== 关键：应用服务器返回的新消息 =====
  let receivedMessages: Message[] = [];
  if (res.messages.length > 0) {
    receivedMessages = await receiveMessages(
      res.messages.map(msg => ({
        ...msg,
        value: deserializeValue(msg.value as string),
      })),
    );
  }

  // ===== 关键：Merkle 树一致性校验 =====
  const diffTime = merkle.diff(res.merkle, getClock().merkle);

  if (diffTime !== null) {
    // Merkle 树不一致，需要递归同步更早的消息
    // 防止无限循环：最多 100 次，连续 10 次相同 diffTime 则停止

    if ((count >= 10 && diffTime === prevDiffTime) || count >= 100) {
      // 尝试重建 Merkle 树
      const rebuiltMerkle = rebuildMerkleHash();
      
      if (rebuiltMerkle.trie.hash === res.merkle.hash) {
        // 重建后一致，保存新时钟
        const clocks = await db.all<db.DbClockMessage>(
          'SELECT * FROM messages_clock',
        );
        // ...
      }

      throw new SyncError('out-of-sync');
    }

    // 递归同步更早的消息
    receivedMessages = receivedMessages.concat(
      await _fullSync(
        new Timestamp(diffTime, 0, '0').toString(),
        localTimeChanged ? 0 : count + 1,
        diffTime,
      ),
    );
  } else {
    // 完全同步，更新 lastSyncedTimestamp
    const requiresUpdate =
      getClock().timestamp.toString() !== lastSyncedTimestamp;

    if (requiresUpdate) {
      await prefs.savePrefs({
        lastSyncedTimestamp: getClock().timestamp.toString(),
      });
    }
  }

  return receivedMessages;
}
```

### 4.4 getMessagesSince 查询

**位置**: `packages/loot-core/src/server/sync/index.ts:534-540`

```typescript
export function getMessagesSince(since: string): Message[] {
  return db.runQuery(
    'SELECT timestamp, dataset, row, column, value FROM messages_crdt WHERE timestamp > ?',
    [since],
    true,
  );
}
```

---

## 5. 其他客户端拉取与应用（Step 4）

### 5.1 receiveMessages 入口

**位置**: `packages/loot-core/src/server/sync/index.ts:447-460`

```typescript
export function receiveMessages(messages: Message[]): Promise<Message[]> {
  try {
    // 更新本地 Lamport 时钟
    messages.forEach(msg => {
      Timestamp.recv(msg.timestamp);
    });
  } catch (e) {
    if (e instanceof Timestamp.ClockDriftError) {
      throw new SyncError('clock-drift');
    }
    throw e;
  }

  // 在 mutator 上下文中应用消息
  return runMutator(() => applyMessages(messages));
}
```

### 5.2 Timestamp.recv 时钟更新

**来源**: `@actual-app/crdt` 包

```typescript
// Lamport 逻辑时钟算法伪代码
class Timestamp {
  static recv(remoteTimestamp: Timestamp) {
    // 取本地时钟和远程时钟的最大值 + 1
    const localMillis = this.localClock.millis;
    const remoteMillis = remoteTimestamp.millis;
    
    if (remoteMillis > localMillis) {
      // 远程时钟更晚，更新本地时钟
      this.localClock.millis = remoteMillis;
      this.localClock.counter = remoteTimestamp.counter + 1;
    } else if (remoteMillis === localMillis) {
      // 毫秒相同，更新计数器
      this.localClock.counter = Math.max(
        this.localClock.counter,
        remoteTimestamp.counter
      ) + 1;
    }
    // 否则本地时钟更晚，不需要更新
  }
}
```

---

## 6. budgetType 特殊处理触发预算重建（Step 5）

### 6.1 触发点

**位置**: `packages/loot-core/src/server/sync/index.ts:367-370`

```typescript
// 在 applyMessages 的数据库事务内
if (dataset === 'preferences' && row === 'budgetType') {
  void setBudgetType(value);  // 立即触发预算类型切换
}
```

### 6.2 setBudgetType 函数

**位置**: `packages/loot-core/src/server/budget/base.ts:321-346`

```typescript
export async function setType(type) {
  const meta = sheet.get().meta();
  if (type === meta.budgetType) {
    return;  // 类型未变，直接返回
  }

  // 1. 更新元数据中的预算类型
  meta.budgetType = type;

  // 2. 重置已创建月份集合，触发重新创建
  meta.createdMonths = new Set();

  // 3. 删除所有预算相关的电子表格单元格
  const nodes = sheet.get().getNodes();
  db.transaction(() => {
    for (const name of nodes.keys()) {
      const [sheetName, cellName] = name.split('!');
      if (sheetName.match(/^budget\d+/)) {
        sheet.get().deleteCell(sheetName, cellName);
      }
    }
  });

  // 4. 标记缓存脏状态，重新加载用户预算
  sheet.get().startCacheBarrier();
  void sheet.loadUserBudgets(db);

  // 5. 重新创建所有预算
  const bounds = await createAllBudgets();
  sheet.get().endCacheBarrier();

  return bounds;
}
```

### 6.3 createAllBudgets 创建流程

**位置**: `packages/loot-core/src/server/budget/base.ts:236-283`

```typescript
export async function createBudget(months) {
  const { data: groups }: { data: CategoryGroupEntity[] } = await aqlQuery(
    q('category_groups').select('*'),
  );
  const categories = groups.flatMap(group => group.categories);

  sheet.startTransaction();
  const meta = sheet.get().meta();
  meta.createdMonths = meta.createdMonths || new Set();

  const budgetType = getBudgetType();

  // Envelope 模式特有：创建空白月份
  if (budgetType === 'envelope') {
    envelopeBudget.createBudget(meta, categories, months);
  }

  months.forEach(month => {
    if (!meta.createdMonths.has(month)) {
      const prevMonth = monthUtils.prevMonth(month);
      const { start, end } = monthUtils.bounds(month);
      const sheetName = monthUtils.sheetForMonth(month);
      const prevSheetName = monthUtils.sheetForMonth(prevMonth);

      // 为每个分类创建单元格
      categories.forEach(cat => {
        createCategory(cat, sheetName, prevSheetName, start, end);
      });

      // 创建分类组汇总（根据预算类型使用不同逻辑）
      groups.forEach(group => {
        if (budgetType === 'envelope') {
          envelopeBudget.createCategoryGroup(group, sheetName);
        } else {
          trackingBudget.createCategoryGroup(group, sheetName);
        }
      });

      // 创建月度汇总（根据预算类型使用不同逻辑）
      if (budgetType === 'envelope') {
        envelopeBudget.createSummary(
          groups,
          categories,
          prevSheetName,
          sheetName,
        );
      } else {
        trackingBudget.createSummary(groups, sheetName);
      }

      meta.createdMonths.add(month);
    }
  });

  sheet.get().setMeta(meta);
  sheet.endTransaction();
  await sheet.waitOnSpreadsheet();
}
```

---

## 7. 导入流程中的特殊处理

### 7.1 applyMessagesForImport 简化处理

**位置**: `packages/loot-core/src/server/sync/index.ts:225-250`

```typescript
function applyMessagesForImport(messages: Message[]): void {
  db.transaction(() => {
    for (let i = 0; i < messages.length; i++) {
      const msg = messages[i];
      const { dataset } = msg;

      if (!msg.old) {
        try {
          apply(msg);
        } catch {
          apply(msg, true);
        }

        // ===== 关键：禁止设置元数据偏好 =====
        // 注意：这里只禁止 'prefs'（存储在 metadata.json）
        // 'preferences' 表（包含 budgetType）不受影响，可以正常写入！
        if (dataset === 'prefs') {
          throw new Error('Cannot set prefs while importing');
        }
      }
    }
  });
}
```

### 7.2 prefs vs preferences 区别总结

| 概念 | 存储位置 | 包含 budgetType? | 导入时是否可写 |
|-----|---------|-----------------|---------------|
| **prefs** (MetadataPrefs) | `metadata.json` 文件 | ❌ 不包含 | ❌ 禁止写入 |
| **preferences** | SQLite `preferences` 表 | ✅ 包含 | ✅ 可以写入 |

### 7.3 Actual 格式导入的特殊处理

**位置**: `packages/loot-core/src/server/importers/actual.ts:1-49`

```typescript
export async function importActual(_filepath: string, buffer: Buffer) {
  // 关闭当前预算
  await handlers['close-budget']();

  let id;
  try {
    // ===== 关键：直接复制数据库文件 =====
    // 这会完整复制原有的 preferences 表，包括 budgetType
    ({ id } = await cloudStorage.importBuffer(
      { cloudFileId: null, groupId: null },
      buffer,
    ));
  } catch (e) {
    // 错误处理...
  }

  // 清除缓存数据
  const sqliteDb = await sqlite.openDatabase(
    fs.join(fs.getBudgetDir(id), 'db.sqlite'),
  );
  sqlite.execQuery(
    sqliteDb,
    `
      DELETE FROM kvcache;
      DELETE FROM kvcache_key;
    `,
  );
  sqlite.closeDatabase(sqliteDb);

  // 重新加载预算，此时会从数据库读取 budgetType
  await handlers['load-budget']({ id });
  await handlers['get-budget-bounds']();
  await waitOnSpreadsheet();
  // ...
}
```

---

## 8. 关键数据结构

### 8.1 Message 消息结构

**位置**: `packages/loot-core/src/server/sync/index.ts:252-259`

```typescript
export type Message = {
  dataset: string;      // 表名，如 'preferences'
  row: string;          // 行 ID，如 'budgetType'
  column: string;       // 列名，如 'value'
  value: string | number | null;  // 值
  timestamp: Timestamp; // Lamport 时间戳
  old?: unknown;        // 是否为旧消息（用于冲突解决）
};
```

### 8.2 messages_crdt 表结构

```sql
CREATE TABLE messages_crdt (
  timestamp TEXT PRIMARY KEY,  -- 序列化的 Lamport 时间戳
  dataset TEXT NOT NULL,       -- 表名
  row TEXT NOT NULL,           -- 行 ID
  column TEXT NOT NULL,        -- 列名
  value TEXT NOT NULL          -- 序列化值（带类型前缀）
);
```

### 8.3 值序列化格式

**位置**: `packages/loot-core/src/server/sync/index.ts:157-182`

```typescript
export function serializeValue(value: string | number | null): string {
  if (value === null) {
    return '0:';                  // null 类型
  } else if (typeof value === 'number') {
    return 'N:' + value;          // 数字类型
  } else if (typeof value === 'string') {
    return 'S:' + value;          // 字符串类型
  }
  throw new Error('Unserializable value type');
}

export function deserializeValue(value: string): string | number | null {
  const type = value[0];
  switch (type) {
    case '0': return null;
    case 'N': return parseFloat(value.slice(2));
    case 'S': return value.slice(2);
    default: throw new Error('Invalid type key');
  }
}
```

### 8.4 Timestamp Lamport 时钟

```typescript
// 伪代码，来自 @actual-app/crdt
class Timestamp {
  millis: number;     // Unix 时间戳（毫秒）
  counter: number;    // 逻辑计数器，解决同一毫秒的冲突
  node: string;       // 客户端唯一标识

  static send(): Timestamp {
    // 生成新的发送时间戳
    const now = Date.now();
    if (now > this.lastMillis) {
      this.lastMillis = now;
      this.lastCounter = 0;
    } else {
      this.lastCounter++;
    }
    return new Timestamp(this.lastMillis, this.lastCounter, this.nodeId);
  }

  static recv(remote: Timestamp): void {
    // 接收远程时间戳，更新本地时钟
    // Lamport 算法：max(local, remote) + 1
  }

  toString(): string {
    // 序列化：millis:counter:node
    return `${this.millis}:${this.counter}:${this.node}`;
  }
}
```

---

## 9. 完整链路时序图

```
客户端 A (修改预算类型)                    服务器                    客户端 B (接收同步)
     |                                       |                           |
     | 1. 用户点击切换按钮                    |                           |
     | → useSyncedPref.setBudgetType()       |                           |
     | → dispatch saveSyncedPrefs            |                           |
     | → 调用 API preferences/save           |                           |
     |                                       |                           |
     | 2. db.update('preferences')           |                           |
     | → 生成 Message                         |                           |
     |   { dataset: 'preferences',           |                           |
     |     row: 'budgetType',                |                           |
     |     column: 'value',                  |                           |
     |     value: 'tracking',                |                           |
     |     timestamp: Timestamp.send() }     |                           |
     |                                       |                           |
     | 3. sendMessages()                     |                           |
     | → applyMessages()                     |                           |
     |   → 写入 preferences 表               |                           |
     |   → 写入 messages_crdt 表             |                           |
     |   → 检测到 budgetType 变更            |                           |
     |     → setBudgetType('tracking')       |                           |
     |       → 删除所有预算单元格             |                           |
     |       → 重建预算                      |                           |
     | → scheduleFullSync()                  |                           |
     |   → setTimeout(1000ms)                |                           |
     |                                       |                           |
     | 4. fullSync()                         |                           |
     | → getMessagesSince(lastSync)          |                           |
     | → 编码消息                            |                           |
     | → POST /sync ───────────────────────→ |                           |
     |                                       | → 存储消息                |
     |                                       | → 更新 Merkle 树          |
     |                                       |                           |
     |                                       | ←──────────────────────── |
     |                                       |   定期拉取 (polling)      |
     |                                       |                           |
     |                                       | ←──────────────────────── |
     |                                       |   POST /sync              |
     |                                       | → 返回新消息              |
     |                                       |                           |
     |                                       | ───────────────────────→ |
     |                                       |   编码后的消息            |
     |                                       |                           |
     |                                       |                           | 5. receiveMessages()
     |                                       |                           |   → Timestamp.recv()
     |                                       |                           |   → applyMessages()
     |                                       |                           |     → 写入 preferences 表
     |                                       |                           |     → 写入 messages_crdt 表
     |                                       |                           |     → 检测到 budgetType 变更
     |                                       |                           |       → setBudgetType('tracking')
     |                                       |                           |         → 删除所有预算单元格
     |                                       |                           |         → 重建预算
     |                                       |                           |
     |                                       |                           | ✓ 预算模式跨设备同步完成！
```

---

## 10. 关键设计决策总结

| 决策点 | 实现方式 | 目的 |
|-------|---------|------|
| **同步触发时机** | `sendMessages` 后延迟 1 秒调度 `fullSync` | 防抖，防止频繁同步 |
| **budgetType 特殊处理** | `applyMessages` 内直接调用 `setBudgetType` | 立即生效，无需等待下次加载 |
| **Merkle 树校验** | 客户端与服务端 Merkle 哈希对比 | 保证数据一致性 |
| **Lamport 时钟** | `Timestamp.send()` / `Timestamp.recv()` | 因果顺序，冲突解决 |
| **导入模式限制** | 仅禁止 `prefs` 写入，允许 `preferences` | 保留预算类型设置 |
| **原子性保证** | 数据库事务包裹所有写操作 | 要么全部成功，要么全部失败 |
| **递归同步** | `_fullSync` 发现不一致时递归同步更早消息 | 处理网络中断等异常场景 |
