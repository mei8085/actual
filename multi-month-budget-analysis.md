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

## 八、关键时序分析

### 8.1 预算页面月份切换主链路

**注意**：此链路为预算主页面（`/budget`）的月份切换流程，与报表页面（Reports）的 `fetchBudgetData` 是两条独立路径。

完整的月份切换时序如下：

```
用户点击月份导航 (MonthPicker / 左右箭头)
       │
       ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 1. onMonthSelect 回调触发 (budget/index.tsx:86-116)                   │
│    • setStartMonthPref(month) - 更新 localStorage 中的 startMonth       │
│    • 判断切换方向（左/右）决定预热目标月份                              │
└─────────────────────────────────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 2. 条件性执行 prewarmMonth 预热相邻月份 (budget/util.ts:181-196)        │
│    • 向左切换：预热 month-1                                            │
│    • 向右切换：预热 month + numDisplayed                               │
│    • 调用 API：'tracking-budget-month' 或 'envelope-budget-month'      │
│    • spreadsheet.prewarmCache(value.name, value) - 填充单元格缓存       │
└─────────────────────────────────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 3. React 状态更新触发重渲染                                           │
│    • startMonthPref 状态更新                                           │
│    • Budget 组件重新渲染，传递新的 startMonth 给 AutoSizingBudgetTable    │
└─────────────────────────────────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 4. BudgetTable 组件接收新 props 并渲染 (BudgetTable.tsx:57-307)        │
│    • prewarmStartMonth: 用于 BudgetSummaries 的预热                     │
│    • startMonth: 用于 BudgetTotals 和 BudgetCategories 的主数据         │
└─────────────────────────────────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 5. 双层 MonthsProvider 计算月份范围 (MonthsContext.tsx:38-54)          │
│    • 第一层：BudgetSummaries 使用 prewarmStartMonth                     │
│      → months = rangeInclusive(bounds.start, bounds.end)               │
│    • 第二层：BudgetTotals/BudgetCategories 使用 startMonth              │
│      → months = rangeInclusive(bounds.start, bounds.end)               │
└─────────────────────────────────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 6. 子组件消费 MonthsContext 获取月份数据                               │
│    • BudgetSummaries / BudgetTotals / BudgetCategories 读取 months      │
│    • 每个月份单元格通过 spreadsheet API 获取实时数据                     │
└─────────────────────────────────────────────────────────────────────────┘
```

**时序细节代码示例：

```typescript
// budget/index.tsx - onMonthSelect 实现
const onMonthSelect = async (month, numDisplayed) => {
  setStartMonthPref(month);
  const warmingMonth = month;
  // 优化：用户左/右按钮切换时预热相邻月份
  if (month < startMonth) {
    // 向左：预热前一个月
    await prewarmMonth(budgetType, spreadsheet, monthUtils.subMonths(month, 1));
  } else if (month > startMonth) {
    // 向右：预热后一个月
    await prewarmMonth(
      budgetType,
      spreadsheet,
      monthUtils.addMonths(month, numDisplayed)
    );
  }
  if (warmingMonth === month) {
    setStartMonthPref(month);
  }
};
```

```typescript
// budget/util.ts - prewarmMonth 实现
export async function prewarmMonth(
  budgetType,
  spreadsheet,
  month
) {
  const method = budgetType === 'tracking'
    ? 'tracking-budget-month'
    : 'envelope-budget-month';
  const values = await send(method, { month });
  for (const value of values) {
    spreadsheet.prewarmCache(value.name, value);
  }
}
```

### 8.2 报表查询链路（独立路径）

**`fetchBudgetData` 所属场景**：此函数专用于**报表页面**（`components/reports/spreadsheets/`），与预算主页面的月份切换无关。

```
报表页面加载 / 用户选择日期范围
       │
       ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 1. fetchBudgetData 被调用 (budgetDataQuery.ts:119-209)                │
│    • 参数：startDate, endDate, interval, categories, categoryGroups    │
│    • 过滤分类：filterCategoriesByConditions                            │
└─────────────────────────────────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 2. 并行获取多个月份数据                                                │
│    • months = rangeInclusive(startMonth, endMonth)                    │
│    • 并发数限制：monthFetchConcurrency = 8                            │
│    • 调用 API：'tracking-budget-month' 或 'envelope-budget-month'      │
└─────────────────────────────────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 3. 数据聚合构建 QueryDataEntity                                        │
│    • 按 interval 分组（月度/年度）                                     │
│    • 区分资产(amount > 0)和负债(amount < 0)                            │
│    • 返回：{ assets: QueryDataEntity[], debts: QueryDataEntity[] }     │
└─────────────────────────────────────────────────────────────────────────┘
```

### 8.3 银行同步链路

银行同步的完整链路如下：

```
用户触发银行同步
       │
       ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 1. 调用 accounts-bank-sync 或 simplefin-batch-sync API             │
│    (accounts/app.ts)                                                   │
│    • 查询所有已链接且未关闭的账户                                     │
└─────────────────────────────────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 2. 遍历账户调用 bankSync.syncAccount() (accounts/sync.ts)               │
│    • 获取同步起始日期（最早 90 天或账户最早交易日期）                  │
│    • 根据同步源下载交易（GoCardless/SimpleFin/PluggyAI/EnableBanking） │
│    • reconcileTransactions 协调新旧交易                                │
│    • 应用用户规则                                                  │
└─────────────────────────────────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 3. 数据变更写入数据库，触发 CRDT 消息生成                              │
└─────────────────────────────────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 4. applyMessages 执行 (sync/index.ts)                                │
│    • sheet.startCacheBarrier() - 启动缓存屏障                          │
│    • 比较消息与现有 CRDT（过滤已应用的消息）                          │
│    • 按时间戳排序                                                    │
│    • db.transaction() - 数据库事务                                    │
│      - 应用消息到数据库                                                │
│      - 插入 messages_crdt 表                                            │
│      - 更新 Merkle 树                                                 │
└─────────────────────────────────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 5. 触发预算变化计算与单元格重新计算                                      │
│    • sheet.startTransaction()                                         │
│    • triggerBudgetChanges(oldData, newData)                              │
│    • sheet.triggerDatabaseChanges(oldData, newData) - 标记相关 SQL 单元格为脏│
│    • 重新计算全局聚合单元格（如账户余额）                              │
│    • sheet.endTransaction()                                           │
└─────────────────────────────────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 6. 结束缓存屏障                                                      │
│    • sheet.endCacheBarrier()                                      │
│    • 标记缓存安全（如果没有待处理变更）                                │
└─────────────────────────────────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 7. 发送 sync-event 通知客户端 (accounts/app.ts)                   │
│    • connection.send('sync-event', { type: 'success', tables })        │
│    • tables 数组包含实际变更的表名列表                                  │
└─────────────────────────────────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 8. 前端接收 sync-event (sync-events.ts:20-391)                       │
│    • listenForSyncEvent 监听                                          │
│    • 根据 tables 数组判断需要失效的缓存                                  │
│    • queryClient.invalidateQueries() - 触发 TanStack Query 缓存失效    │
└─────────────────────────────────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 9. UI 组件重新渲染，显示最新数据                                    │
│    • useCategories / useAccounts / usePayees 重新获取数据               │
│    • BudgetTable 通过 Spreadsheet 缓存获取更新后的数据                   │
└─────────────────────────────────────────────────────────────────────────┘
```

**缓存屏障实现细节**（spreadsheet/spreadsheet.ts）：**

```typescript
startCacheBarrier() {
  this.cacheBarrier = true;
  this.markCacheDirty();
}

endCacheBarrier() {
  this.cacheBarrier = false;
  const pendingChange = this.running || this.computeQueue.length > 0;
  if (!pendingChange) {
    this.markCacheSafe();
  }
}

triggerDatabaseChanges(oldValues, newValues) {
  const tables = new Set([...oldValues.keys(), ...newValues.keys()]);
  this.startTransaction();
  this.nodes.forEach(node => {
    if (node.sql && node.sql.state.dependencies.some(dep => tables.has(dep))) {
      this._markDirty(node.name);
    }
  });
  this.endTransaction();
}
```

### 8.4 sync-event 前端失效条件精确对齐

根据 `sync-events.ts` 的实现，`tables` 数组的不同值会触发不同的缓存失效行为：

| tables 包含值 | 触发的失效操作 | 不触发的操作 |
|--------------|---------------|-------------|
| `prefs` | `store.dispatch(loadPrefs())` | 不触发任何 Query 失效 |
| `categories` | `queryClient.invalidateQueries({ queryKey: categoryQueries.lists() })` | 不影响 accounts/payees 查询 |
| `category_groups` | `queryClient.invalidateQueries({ queryKey: categoryQueries.lists() })` | 不影响 accounts/payees 查询 |
| `category_mapping` | `queryClient.invalidateQueries({ queryKey: categoryQueries.lists() })` | 不影响 accounts/payees 查询 |
| `accounts` | `queryClient.invalidateQueries({ queryKey: accountQueries.lists() })` + `payeeQueries.lists()` | 不影响 categories 查询 |
| `payees` | `queryClient.invalidateQueries({ queryKey: payeeQueries.lists() })` | 不影响 categories/accounts 查询 |
| `payee_mapping` | `queryClient.invalidateQueries({ queryKey: payeeQueries.lists() })` | 不影响 categories/accounts 查询 |
| **`transactions`** | **Spreadsheet 层单元格重新计算** | **不触发任何 TanStack Query 失效** |

**仅有 transactions 时的实际行为：**

当 `tables = ['transactions']` 时：
1. **不触发** `categoryQueries.lists()` 失效
2. **不触发** `accountQueries.lists()` 失效  
3. **不触发** `payeeQueries.lists()` 失效
4. **仅在** Spreadsheet 层：`triggerDatabaseChanges` 标记依赖 `transactions` 表的 SQL 单元格为脏，触发重新计算
5. BudgetTable 通过 Spreadsheet API 读取时自动获取更新后的数据

### 8.5 月份切换对照表

| 步骤 | 触发点 | 依赖模块 | 被更新的数据 | 缓存失效范围 |
|------|--------|----------|--------------|------------|
| 1 | 用户点击月份导航 | MonthPicker / 左右箭头组件 | React 状态 | 无 |
| 2 | `onMonthSelect(month, numDisplayed)` | `budget/index.tsx` | `startMonthPref` (localStorage) | 无 |
| 3 | `prewarmMonth(budgetType, spreadsheet, month)` | `budget/util.ts` | Spreadsheet 单元格缓存 | Spreadsheet 层：预热指定月份的单元格缓存 |
| 4 | React 状态更新 | React 调度器 | `startMonth` 状态 | 无 |
| 5 | BudgetTable 重渲染 | `BudgetTable.tsx` | 组件 props | 无 |
| 6 | MonthsProvider 计算 | `MonthsContext.tsx` | `months` 范围数组 | 无 |
| 7 | 子组件消费 context | `BudgetSummaries` / `BudgetTotals` / `BudgetCategories` | 渲染输出 | 无（从 Spreadsheet 获取实时数据） |

### 8.6 银行同步对照表

| 步骤 | 触发点 | 依赖模块 | 被更新的数据 | 缓存失效范围 |
|------|--------|----------|--------------|------------|
| 1 | 用户触发银行同步 | UI 组件（账户页/预算页） | 无 | 无 |
| 2 | `accounts-bank-sync` API 调用 | `accounts/app.ts` | 无 | 无 |
| 3 | `syncAccount()` | `accounts/sync.ts` | `transactions` 表新增/更新 | 无（事务中） |
| 4 | CRDT 消息生成 | CRDT 模块 | 无（消息待应用） | 无 |
| 5 | `applyMessages()` | `sync/index.ts` | `messages_crdt`, `messages_clock`, `merkle` 树 | Spreadsheet 层：`startCacheBarrier()` |
| 6 | `triggerBudgetChanges()` | `sync/index.ts` | 预算计算中间状态 | Spreadsheet 层：相关单元格标记为脏 |
| 7 | `triggerDatabaseChanges()` | `spreadsheet.ts` | SQL 依赖表相关单元格 | Spreadsheet 层：依赖变更表的单元格重新计算 |
| 8 | `endCacheBarrier()` | `sync/index.ts` | 缓存屏障状态 | Spreadsheet 层：缓存屏障关闭 |
| 9 | `sync-event` 发送 | `accounts/app.ts` | 无（事件通知） | 无 |
| 10 | `sync-event` 接收 | `sync-events.ts` | 无（触发失效） | TanStack Query：根据 `tables` 数组选择性失效 |
| 11 | UI 组件重渲染 | React 调度器 | 显示最新数据 | 无（数据已更新） |

---

## 九、参考文件清单

| 文件路径 | 说明 |
|----------|------|
| `packages/desktop-client/src/components/budget/BudgetTable.tsx` | 预算表格主组件 |
| `packages/desktop-client/src/components/budget/MonthsContext.tsx` | 月份上下文管理 |
| `packages/desktop-client/src/components/budget/index.tsx` | 预算页面主组件，onMonthSelect 实现 |
| `packages/desktop-client/src/components/budget/util.ts` | 预热函数 prewarmMonth / prewarmAllMonths |
| `packages/desktop-client/src/components/reports/spreadsheets/budgetDataQuery.ts` | 预算数据查询 |
| `packages/desktop-client/src/hooks/useCategories.ts` | 分类数据 Hook |
| `packages/desktop-client/src/sync-events.ts` | sync-event 处理 |
| `packages/loot-core/src/server/accounts/app.ts` | 账户服务 API |
| `packages/loot-core/src/server/accounts/sync.ts` | 银行同步实现 |
| `packages/loot-core/src/server/sync/index.ts` | CRDT 同步核心 |
| `packages/loot-core/src/server/spreadsheet/spreadsheet.ts` | Spreadsheet 缓存屏障实现 |
| `packages/crdt/src/crdt/timestamp.ts` | HULC 时间戳实现 |
| `packages/crdt/src/crdt/merkle.ts` | Merkle 树实现 |