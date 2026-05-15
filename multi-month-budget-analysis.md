# 跨月份预算表格数据构建与模块协作分析

## 一、架构概览

跨月份预算表格的渲染涉及多个模块的协同工作，主要包括：

| 模块 | 职责 | 关键文件 |
|------|------|----------|
| **账户服务层** | 账户管理、银行同步协调 | `packages/loot-core/src/server/accounts/app.ts` |
| **Bank Sync 接入组件** | 第三方银行数据拉取、交易协调 | `packages/loot-core/src/server/accounts/sync.ts` |
| **CRDT 模块** | 分布式数据同步、版本控制 | `packages/crdt/src/` |
| **预算表格组件** | UI 渲染、月份上下文管理 | `packages/desktop-client/src/components/budget/` |

---

## 二、核心模块分析

### 2.1 预算表格组件架构

**BudgetTable 组件** (`BudgetTable.tsx`) 是多月视图的核心容器：

```typescript
export function BudgetTable({
  type,
  prewarmStartMonth,
  startMonth,
  numMonths,
  monthBounds,
  // ... 回调函数
}: BudgetTableProps) {
  const { data: { grouped: categoryGroups } = { grouped: [] } } = useCategories();
  // ...
}
```

**关键设计特点：**

1. **双层 MonthsProvider**：
   - 第一层：用于 `BudgetSummaries`（预热数据）
   - 第二层：用于 `BudgetTotals` 和 `BudgetCategories`（主数据）

2. **月份范围计算** (`MonthsContext.tsx`)：
   ```typescript
   const endMonth = monthUtils.addMonths(startMonth, numMonths - 1);
   const bounds = getValidMonthBounds(monthBounds, startMonth, endMonth);
   const months = monthUtils.rangeInclusive(bounds.start, bounds.end);
   ```

3. **数据预取策略**：通过 `prewarmStartMonth` 实现数据预热，提升滚动性能。

### 2.2 账户服务层

**账户服务层** (`accounts/app.ts`) 提供银行同步的统一入口：

```typescript
// 核心 API Handler
app.method('accounts-bank-sync', accountsBankSync);
app.method('simplefin-batch-sync', simpleFinBatchSync);
```

**同步流程：**

1. **账户查询**：获取所有已链接且未关闭的账户
2. **批量同步**：遍历账户执行银行同步
3. **结果处理**：收集新增/更新的交易，通知客户端

```typescript
async function accountsBankSync({ ids = [] }: { ids: Array<AccountEntity['id']> }) {
  const accounts = db.runQuery<db.DbAccount & { bankId: db.DbBank['bank_id'] }>(
    `SELECT a.*, b.bank_id as bankId
     FROM accounts a LEFT JOIN banks b ON a.bank = b.id
     WHERE a.tombstone = 0 AND a.closed = 0
     ${ids.length ? `AND a.id IN (${ids.map(() => '?').join(', ')})` : ''}
     ORDER BY a.offbudget, a.sort_order`,
    ids,
    true,
  );
  
  for (const acct of accounts) {
    if (acct.bankId && acct.account_id) {
      const syncResponse = await bankSync.syncAccount(
        userId, userKey, acct.id, acct.account_id, acct.bankId
      );
      // 处理同步结果
    }
  }
}
```

### 2.3 Bank Sync 接入组件

**Bank Sync 模块** (`accounts/sync.ts`) 处理具体的银行数据拉取：

**支持的同步提供商：**
- GoCardless
- SimpleFin
- PluggyAI
- EnableBanking

**核心同步流程：**

```typescript
export async function syncAccount(
  userId: string | undefined,
  userKey: string | undefined,
  id: string,      // Actual 账户 ID
  acctId: string,  // 银行账户 ID
  bankId: string,  // 银行 ID
  customStartingDate?: string,
  customStartingBalance?: number,
) {
  // 1. 获取同步起始日期（最早 90 天或账户最早交易日期）
  const syncStartDate = customStartingDate ?? (await getAccountSyncStartDate(id));
  
  // 2. 根据同步源下载交易
  if (acctRow.account_sync_source === 'simpleFin') {
    download = await downloadSimpleFinTransactions(acctId, syncStartDate);
  } else if (acctRow.account_sync_source === 'goCardless') {
    download = await downloadGoCardlessTransactions(...);
  }
  
  // 3. 处理下载的数据
  return processBankSyncDownload(download, id, acctRow, newAccount, ...);
}
```

**交易协调机制** (`reconcileTransactions`)：

1. **严格匹配**：通过 `imported_id` 精确匹配
2. **模糊匹配**：基于日期（±7天）和金额进行匹配
3. **规则引擎**：应用用户定义的交易规则

```typescript
async function matchTransactions(acctId, transactions, isBankSyncAccount, ...) {
  // 第一阶段：规则执行和精确匹配
  // 第二阶段：基于 Payee ID 的模糊匹配
  // 第三阶段：最后的模糊匹配（日期和金额）
}
```

### 2.4 CRDT 模块

**CRDT（无冲突复制数据类型）** 模块实现分布式数据同步：

**时间戳系统** (`timestamp.ts`)：

使用 **HULC（Hybrid Unique Logical Clock）** 算法：

```typescript
export class Timestamp {
  _state: { millis: number; counter: number; node: string };
  
  toString() {
    return [
      new Date(this.millis()).toISOString(),      // ISO 时间
      ('0000' + this.counter().toString(16)).slice(-4),  // 计数器
      ('0000000000000000' + this.node()).slice(-16),     // 节点 ID
    ].join('-');
    // 示例: 2015-04-24T22:23:42.123Z-1000-A219E7A71CC18912
  }
}
```

**Merkle Trie** (`merkle.ts`)：

用于高效检测数据差异：
- 每个节点存储子节点哈希的组合
- 支持快速同步检测
- 支持修剪历史数据

**同步核心流程** (`sync/index.ts`)：

```typescript
export const applyMessages = sequential(async (messages: Message[]) => {
  // 1. 比较消息与现有 CRDT（过滤已应用的消息）
  messages = await compareMessages(messages);
  
  // 2. 按时间戳排序
  messages = [...messages].sort((m1, m2) => 
    m1.timestamp.toString().localeCompare(m2.timestamp.toString())
  );
  
  // 3. 数据库事务应用
  db.transaction(() => {
    for (const msg of messages) {
      apply(msg);  // INSERT 或 UPDATE
      db.runQuery('INSERT INTO messages_crdt ...');  // 记录到 CRDT 表
      currentMerkle = merkle.insert(currentMerkle, timestamp);  // 更新 Merkle
    }
  });
  
  // 4. 触发预算变化
  triggerBudgetChanges(oldData, newData);
});
```

---

## 三、多月视图数据构建流程

### 3.1 数据加载顺序

```
用户触发月份切换
       │
       ▼
┌─────────────────────────────────────────────────────────────┐
│ 1. UI 层：更新 startMonth/numMonths 状态                    │
│    • BudgetTable 组件重新渲染                                │
│    • MonthsProvider 计算新的月份范围                          │
└─────────────────────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────────────┐
│ 2. 分类数据加载                                              │
│    • useCategories() hook 触发查询                           │
│    • 通过 TanStack Query 缓存机制                            │
└─────────────────────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────────────┐
│ 3. 月份数据并行获取                                          │
│    • fetchBudgetData() 并发获取多个月份数据                   │
│    • 使用 mapWithConcurrency 控制并发数（默认 8）              │
│    • 调用 'envelope-budget-month' 或 'tracking-budget-month' │
└─────────────────────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────────────┐
│ 4. 数据聚合与渲染                                            │
│    • 按类别分组数据                                          │
│    • 构建 QueryDataEntity 数组                              │
│    • 区分资产/负债类型                                       │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 月份切换时的数据加载顺序

```typescript
// budgetDataQuery.ts - 多月数据并行获取
export async function fetchBudgetData({
  startDate, endDate, interval, categories, ...
}) {
  const months = monthUtils.rangeInclusive(
    monthUtils.getMonth(startDate),
    monthUtils.getMonth(endDate),
  );
  
  const monthFetchConcurrency = 8;
  
  const monthDataList = await mapWithConcurrency(
    months,
    monthFetchConcurrency,
    async month => ({
      month,
      monthData: await send(endpointName, { month }),
    }),
  );
  
  // 聚合数据...
}
```

**关键优化策略：**
- **并发限制**：避免同时发起过多请求
- **月份预热**：`prewarmStartMonth` 提前加载相邻月份数据
- **缓存复用**：通过 TanStack Query 缓存已加载的月份数据

---

## 四、缓存失效策略

### 4.1 缓存层次结构

| 层级 | 缓存机制 | 失效条件 |
|------|----------|----------|
| **UI 层** | TanStack Query | 查询键变化、手动失效 |
| **Spreadsheet 层** | 单元格缓存 | 相关数据变更 |
| **CRDT 层** | Merkle 树 | 数据同步时更新 |
| **数据库层** | SQLite 查询缓存 | 表数据变更 |

### 4.2 同步事件驱动的缓存失效

**Bank Sync 完成后的通知机制：**

```typescript
// accounts/app.ts - 同步成功后发送事件
if (updatedAccounts.length > 0) {
  connection.send('sync-event', {
    type: 'success',
    tables: ['transactions'],
  });
}
```

**响应式数据更新流程：**

```typescript
// sync/index.ts - 消息应用后的触发器
export const applyMessages = sequential(async (messages: Message[]) => {
  // ... 消息应用 ...
  
  // 触发预算变化计算
  if (sheet.get()) {
    sheet.startTransaction();
    triggerBudgetChanges(oldData, newData);
    sheet.get().triggerDatabaseChanges(oldData, newData);
    sheet.endTransaction();
    
    // 显式重新计算全局聚合单元格
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
  
  // 通知监听器
  _syncListeners.forEach(func => func(oldData, newData));
  
  // 触发应用级事件
  app.events.emit('sync', {
    type: 'applied',
    tables,
    data: newData,
    prevData: oldData,
  });
});
```

### 4.3 缓存屏障机制

```typescript
// 同步开始时启动缓存屏障
if (sheet.get()) {
  sheet.get().startCacheBarrier();
}

// 消息应用（数据库事务）
db.transaction(() => {
  // ... 所有数据库操作 ...
});

// 同步完成后结束缓存屏障
if (sheet.get()) {
  sheet.get().endCacheBarrier();
}
```

**缓存屏障的作用：**
- 在同步期间阻止缓存使用
- 确保所有变更完成后才允许后续查询使用缓存
- 避免中间状态被读取

### 4.4 TanStack Query 缓存策略

```typescript
// hooks/useCategories.ts
export function useCategories() {
  return useQuery(categoryQueries.list());
}
```

**缓存失效场景：**
1. **手动失效**：通过 `queryClient.invalidateQueries()`
2. **时间过期**：配置的 staleTime 过期
3. **后台刷新**：配置的 cacheTime 过期后后台重新获取

---

## 五、模块协作关系图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        用户界面层 (UI)                                  │
│  ┌─────────────────────┐                                               │
│  │   BudgetTable       │                                               │
│  │  - MonthsProvider   │                                               │
│  │  - useCategories    │                                               │
│  │  - BudgetCategories │                                               │
│  └─────────┬───────────┘                                               │
│            │ 数据查询                                                   │
├────────────┼───────────────────────────────────────────────────────────┤
│                        业务逻辑层 (Core)                                │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                    账户服务层                                    │   │
│  │  ┌─────────────┐  ┌───────────────────────────────────────┐   │   │
│  │  │  accounts   │  │           Bank Sync                   │   │   │
│  │  │   app.ts    │→│  sync.ts                               │   │   │
│  │  │  (API层)    │  │  - syncAccount()                      │   │   │
│  │  └─────────────┘  │  - reconcileTransactions()            │   │   │
│  │                   │  - download*Transactions()            │   │   │
│  │                   └───────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                              │                                         │
│                              │ 数据变更消息                             │
├──────────────────────────────┼─────────────────────────────────────────┤
│                        数据同步层 (CRDT)                               │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                    CRDT 同步引擎                               │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐   │   │
│  │  │ Timestamp   │  │   Merkle    │  │   Sync Index        │   │   │
│  │  │  (HULC)     │  │   Trie      │  │  - applyMessages()  │   │   │
│  │  │  - send()   │  │  - insert() │  │  - receiveMessages()│   │   │
│  │  │  - recv()   │  │  - prune()  │  │  - batchMessages()  │   │   │
│  │  └─────────────┘  └─────────────┘  └─────────────────────┘   │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                              │                                         │
│                              │ 持久化                                  │
├──────────────────────────────┼─────────────────────────────────────────┤
│                        数据存储层 (Database)                            │
│              ┌───────────────────────────────┐                         │
│              │         SQLite                │                         │
│              │  - messages_crdt (CRDT日志)  │                         │
│              │  - messages_clock (时钟状态)  │                         │
│              │  - transactions (交易数据)   │                         │
│              │  - categories (分类数据)     │                         │
│              └───────────────────────────────┘                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 六、关键设计要点总结

### 6.1 月份切换优化

1. **预加载策略**：`prewarmStartMonth` 提前加载相邻月份
2. **并发控制**：`mapWithConcurrency` 限制同时请求数
3. **缓存复用**：TanStack Query 自动缓存已查询的数据

### 6.2 数据一致性保障

1. **CRDT 时间戳**：HULC 算法保证全局唯一性和单调性
2. **Merkle 树差异检测**：高效识别需要同步的数据
3. **数据库事务**：确保所有变更原子性提交

### 6.3 缓存失效机制

1. **事件驱动**：同步完成后主动触发缓存失效
2. **缓存屏障**：同步期间阻止缓存读取
3. **分层失效**：UI层、业务层、存储层各自管理缓存

### 6.4 扩展性设计

1. **多银行支持**：通过 `account_sync_source` 字段扩展
2. **模块化架构**：各模块职责清晰，接口明确
3. **异步处理**：批量消息处理避免阻塞主线程

---

## 七、潜在优化方向

1. **增量加载**：当前实现为全量加载月份数据，可考虑按需加载可见月份
2. **缓存预热**：根据用户习惯预加载常用月份范围
3. **数据压缩**：减少跨月份查询的数据传输量
4. **智能失效**：根据数据变更类型选择性失效相关缓存

---

## 八、参考文件清单

| 文件路径 | 说明 |
|----------|------|
| `packages/desktop-client/src/components/budget/BudgetTable.tsx` | 预算表格主组件 |
| `packages/desktop-client/src/components/budget/MonthsContext.tsx` | 月份上下文管理 |
| `packages/desktop-client/src/components/reports/spreadsheets/budgetDataQuery.ts` | 预算数据查询 |
| `packages/desktop-client/src/hooks/useCategories.ts` | 分类数据 Hook |
| `packages/loot-core/src/server/accounts/app.ts` | 账户服务 API |
| `packages/loot-core/src/server/accounts/sync.ts` | 银行同步实现 |
| `packages/loot-core/src/server/sync/index.ts` | CRDT 同步核心 |
| `packages/crdt/src/crdt/timestamp.ts` | HULC 时间戳实现 |
| `packages/crdt/src/crdt/merkle.ts` | Merkle 树实现 |