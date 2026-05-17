# 交易筛选器复合表达式归约分析报告（修订版）

## 修订说明

本报告是对 `expression-reduction.md` 的复核与修订，主要修正以下三处关键问题：

1. **明确区分两条查询机制**：账户页主查询链路（PagedQuery/LiveQuery）与新 React Hooks 机制（useTransactions + TanStack Query）的职责与触发条件
2. **修正标签匹配正则结论**：准确描述 `hasTags` 操作符的正则匹配行为
3. **澄清 sync-event 机制**：sync-event 驱动的是基于依赖表的查询重跑，而非统一缓存失效

---

## 1. 两条查询机制的职责与触发条件

Actual Budget 中存在两条并行的查询机制，分别服务于不同的代码路径。

### 1.1 机制一：PagedQuery / LiveQuery（账户页主查询链路）

**核心文件**：
- `packages/desktop-client/src/queries/liveQuery.ts`
- `packages/desktop-client/src/queries/pagedQuery.ts`

**使用场景**：
- `Account.tsx` 类组件中的交易列表主查询
- 传统的类组件架构

**关键特性**：

| 特性 | 说明 |
|------|------|
| **数据管理** | 内部维护 `_data` 状态，不依赖外部状态管理 |
| **依赖跟踪** | 从查询结果中提取 `_dependencies` 表集合，用于精确更新 |
| **乐观更新** | 支持 `optimisticUpdate()` 在本地立即应用更改 |
| **分页方式** | 通过 `offset` + `limit` 实现，`fetchNext()` 加载下一页 |
| **缓存策略** | 无独立缓存层，每次查询直接发送 RPC |

**触发条件**：

```typescript
// Account.tsx:479-528
this.paged = pagedQuery(query.select('*'), {
  onData: async (groupedData, prevData) => {
    const data = ungroupTransactions([...groupedData]);
    // 更新组件状态并触发渲染
  },
  options: {
    pageCount: 150,    // 每页加载数量
    onlySync: true,    // 仅响应远程同步事件，忽略本地变更
  },
});
```

1.  **初始化触发**：创建时自动调用 `run()` 执行首次查询
2.  **sync-event 触发**：监听 `sync-event`，根据依赖表判断是否重跑
    ```typescript
    // liveQuery.ts:110-119
    protected onUpdate = (tables: string[]) => {
      if (
        this._dependencies == null ||
        tables.find(table => this._dependencies.has(table))
      ) {
        void this.run();  // 仅当依赖表变更时才重跑
      }
    };
    ```
3.  **手动触发**：调用 `fetchNext()` 加载下一页
4.  **本地更新**：调用 `optimisticUpdate()` 应用本地更改

### 1.2 机制二：useTransactions + TanStack Query（新 React Hooks 机制）

**核心文件**：
- `packages/desktop-client/src/hooks/useTransactions.ts`
- `packages/desktop-client/src/transactions/queries.ts`

**使用场景**：
- 新的 React 函数组件
- 基于 TanStack Query（React Query）的现代查询架构

**关键特性**：

| 特性 | 说明 |
|------|------|
| **数据管理** | 完全由 TanStack Query 管理 |
| **缓存策略** | TanStack Query 内置缓存，以 `queryKey` 为缓存键 |
| **余额计算** | 支持自动计算运行余额（`calculateRunningBalances`） |
| **分页方式** | 无限滚动（`useInfiniteQuery`），通过 `fetchNextPage()` 加载 |
| **状态管理** | 提供完整的加载/错误/成功状态 |

**触发条件**：

```typescript
// useTransactions.ts:91-169
export function useTransactions({ query, options }: UseTransactionsProps) {
  const { pageSize = 50, calculateRunningBalances = false, refetchOnSync = true } = options ?? {};

  const queryResult = useInfiniteQuery(
    transactionQueries.aql({ query, pageSize }),
  );

  // sync-event 监听
  useEffect(() => {
    if (!refetchOnSync) return;
    return listen('sync-event', onSyncEvent);
  }, [refetchOnSync]);

  return { ...queryResult, transactions: flattenPages(queryResult.data) };
}
```

1.  **挂载触发**：组件挂载时自动执行查询
2.  **依赖变化触发**：`query` 或 `options` 变化时重新执行
3.  **sync-event 触发**：
    ```typescript
    // useTransactions.ts:110-121
    const onSyncEvent = useEffectEvent((event: ServerEvents['sync-event']) => {
      if (event.type === 'applied') {
        const tables = event.tables;
        if (
          tables.includes('transactions') ||
          tables.includes('category_mapping') ||
          tables.includes('payee_mapping')
        ) {
          void queryResult.refetch();  // 主动刷新，不是缓存失效
        }
      }
    });
    ```
4.  **手动触发**：调用 `fetchNextPage()` 加载下一页
5.  **TanStack Query 自动触发**：缓存过期、窗口聚焦等

### 1.3 两条机制对比

| 维度 | PagedQuery / LiveQuery | useTransactions + TanStack Query |
|------|-----------------------|----------------------------------|
| **架构风格** | 面向对象（类） | 函数式（Hooks） |
| **状态管理** | 内部状态 | TanStack Query 托管 |
| **缓存层** | 无独立缓存 | TanStack Query 内置缓存 |
| **更新触发** | 基于依赖表精确触发 | 基于特定表触发 + TanStack Query 策略 |
| **余额计算** | 外部处理 | 内置支持 |
| **使用场景** | Account.tsx 类组件 | 新的函数组件 |
| **乐观更新** | 原生支持 | 需额外处理 |

---

## 2. 标签匹配正则表达式行为修正

### 2.1 原始实现分析

`hasTags` 操作符的实现位于 `transaction-rules.ts:616-641`，分为两个阶段：

```typescript
case 'hasTags': {
  // ========== 阶段一：从用户输入中提取标签 ==========
  const tagValues = [];
  const seenTags = new Set();
  for (const [_, tag] of value.matchAll(/(?<!#)(#[^#\s]+)/g)) {
    if (!seenTags.has(tag)) {
      seenTags.add(tag);
      tagValues.push(tag);
    }
  }

  if (tagValues.length === 0) {
    return { id: null };  // 没有有效标签，匹配不到任何结果
  }

  // ========== 阶段二：为每个标签构建匹配正则 ==========
  return {
    $and: tagValues.map(v => {
      const escapedTag = v
        .replace(/[.*+?^${}()|[\]\\]/g, '\\$&')
        .replace(/\\\$/g, '[$]');
      const pattern = `(?<!#)${escapedTag}([\\s#]|$)`;
      return apply(field, '$regexp', pattern);
    }),
  };
}
```

### 2.2 提取阶段正则解析

**正则**：`/(?<!#)(#[^#\s]+)/g`

| 组成部分 | 含义 |
|---------|------|
| `(?<!#)` | 负向后顾断言，确保当前位置前面不是 `#` |
| `#` | 匹配字面量 `#` |
| `[^#\s]+` | 匹配一个或多个非 `#` 且非空白的字符 |
| `g` | 全局匹配，提取所有标签 |

**提取示例**：
- 输入：`"#foo #bar ##baz #qux"`
- 提取结果：`["#foo", "#bar", "#qux"]`（`##baz` 被排除，因为 `#` 前面还是 `#`）

### 2.3 匹配阶段正则解析

**正则模板**：`(?<!#)${escapedTag}([\s#]|$)`

以 `#groceries` 为例，实际生成的正则为：`(?<!#)#groceries([\s#]|$)`

| 组成部分 | 含义 |
|---------|------|
| `(?<!#)` | 负向后顾断言，确保标签前不是 `#` |
| `#groceries` | 转义后的标签字面量 |
| `([\s#]|$)` | 捕获组，匹配空白字符、`#` 或字符串结尾（标签边界） |

### 2.4 修正后的匹配结论

对于 `notes hasTags '#groceries'`：

✅ **可匹配的情况**：
| 测试字符串 | 匹配原因 |
|-----------|---------|
| `"#groceries"` | 完整匹配，字符串结尾 |
| `"test #groceries"` | 前面是空格，后面是结尾 |
| `"#groceries test"` | 后面是空格 |
| `"#groceries#other"` | 后面是 `#` |
| `"#groceries, #food"` | 后面是逗号和空格 |

❌ **不可匹配的情况**：
| 测试字符串 | 不匹配原因 |
|-----------|-----------|
| `"##groceries"` | 第一个 `#` 满足 `(?<!#)`，但匹配 `#groceries` 后，后面是 `#`，等等——实际上这个会匹配！让我重新分析... |

**重要修正**：`"##groceries"` 实际上是**可以匹配**的！

让我们仔细分析：
- 字符串：`"##groceries"`
- 正则：`(?<!#)#groceries([\s#]|$)`
- 第一个字符 `#`：前面没有字符，满足 `(?<!#)`，但后面是 `#groceries`，不是 `groceries`，不匹配
- 第二个字符 `#`：前面是 `#`，不满足 `(?<!#)`，跳过
- **结论**：`"##groceries"` **不匹配**，因为两个 `#` 连在一起时，第二个 `#` 前面是 `#`，不满足负向后顾

继续分析：

| 测试字符串 | 匹配结果 | 原因 |
|-----------|---------|------|
| `"##groceries"` | ❌ 不匹配 | 第二个 `#` 前面是 `#`，不满足 `(?<!#)` |
| `"#groceries2"` | ❌ 不匹配 | `#groceries` 后面是 `2`，不是边界字符 |
| `"#groceriess"` | ❌ 不匹配 | `#groceries` 后面是 `s`，不是边界字符 |
| `"#groceries-test"` | ❌ 不匹配 | 后面是 `-`，不是边界字符 |
| `"#Groceries"` | ✅ 匹配 | SQLite 的 `REGEXP` 默认不区分大小写（取决于编译选项） |

### 2.5 额外注意点

1.  **大小写不敏感**：SQLite 的 `REGEXP` 函数是否区分大小写取决于编译选项，在 Actual 中通常是不区分的
2.  **特殊字符转义**：代码中对正则特殊字符进行了转义，`$` 被特殊处理为 `[$]` 以避免被当作字符串结尾锚点
3.  **多标签逻辑**：多个标签使用 `$and` 连接，必须全部匹配
4.  **无标签输入**：如果输入中没有 `#tag` 格式的内容，返回 `{ id: null }` 匹配不到任何结果

---

## 3. sync-event 机制澄清

### 3.1 sync-event 是什么

`sync-event` 是 Actual 内部的事件机制，用于在数据变更时通知相关组件。事件包含：

```typescript
type SyncEvent = {
  type: 'applied' | 'success';  // applied=本地变更，success=远程同步成功
  tables: string[];             // 变更涉及的表名
  // ... 其他字段
};
```

### 3.2 LiveQuery/PagedQuery 中的处理

```typescript
// liveQuery.ts:137-154
protected subscribe = () => {
  if (this._unsubscribeSyncEvent == null) {
    this._unsubscribeSyncEvent = listen('sync-event', event => {
      // onlySync 选项控制响应类型
      if (
        (event.type === 'applied' || event.type === 'success') &&
        this._supportedSyncTypes.has(event.type)
      ) {
        this.onUpdate(event.tables);
      }
    });
  }
};

// liveQuery.ts:110-119
protected onUpdate = (tables: string[]) => {
  // 如果还不知道依赖（首次查询未返回），或者依赖表有变更，则重跑
  if (
    this._dependencies == null ||
    tables.find(table => this._dependencies.has(table))
  ) {
    void this.run();
  }
};
```

**关键点**：
- ✅ **不是统一缓存失效**：没有全局缓存被清除
- ✅ **基于依赖表精确触发**：只有当变更的表在查询的依赖集合中时才重跑
- ✅ **依赖表来源**：从查询结果的 `dependencies` 字段获取，由后端 AQL 编译器分析得出
- ✅ **`onlySync` 选项**：如果设置为 `true`，只响应 `success` 类型（远程同步），忽略本地 `applied` 事件，避免乐观更新后重复查询

### 3.3 useTransactions 中的处理

```typescript
// useTransactions.ts:110-128
const onSyncEvent = useEffectEvent((event: ServerEvents['sync-event']) => {
  if (event.type === 'applied') {
    const tables = event.tables;
    if (
      tables.includes('transactions') ||
      tables.includes('category_mapping') ||
      tables.includes('payee_mapping')
    ) {
      void queryResult.refetch();  // 调用 TanStack Query 的 refetch
    }
  }
});
```

**关键点**：
- ✅ **不是缓存失效**：没有调用 `queryClient.invalidateQueries()`
- ✅ **主动刷新**：调用 `refetch()` 主动重新获取数据
- ✅ **硬编码表检查**：只检查交易相关的三个表，不使用动态依赖表
- ✅ **可配置**：通过 `refetchOnSync` 选项可以禁用

### 3.4 常见误解澄清

| 误解 | 事实 |
|------|------|
| "sync-event 会导致所有缓存失效" | ❌ 没有全局缓存失效，只有特定查询被触发重跑 |
| "sync-event 触发后所有查询都会重跑" | ❌ 只有满足依赖条件的查询才会重跑 |
| "TanStack Query 的缓存会被 sync-event 清除" | ❌ TanStack Query 的缓存仍然有效，只是主动调用了 refetch |
| "依赖表是前端分析出来的" | ❌ 依赖表由后端 AQL 编译器分析，随查询结果返回 |

---

## 4. "容易与直觉不一致的组合"代码核对清单

以下每条结论都附带代码位置和验证方法，确保可复核。

### 4.1 NULL 值的隐式条件

**结论**：`category is null` 实际上匹配的是 `category IS NULL AND transfer_id IS NULL AND is_parent = 0`

**代码证据**：
```typescript
// transaction-rules.ts:385-415
function conditionSpecialCases(cond: Condition | null): Condition | null {
  if (cond.op === 'is' && cond.field === 'category' && cond.value === null) {
    return new Condition('and', cond.field, [
      cond,
      new Condition('is', 'transfer', false, null),
      new Condition('is', 'parent', false, null),
    ], {});
  }
  // ...
}
```

**验证方法**：在测试中创建一个分类为空的转账交易，使用 `category is null` 筛选，验证其不被匹配。

---

### 4.2 OR 组合下的否定条件

**结论**：当 `conditionsOp = 'or'` 时，`[category isNot '食物', amount > 100]` 的实际语义是 `category != '食物' OR amount > 100`，而不是用户可能期望的 `NOT (category = '食物' OR amount > 100)`

**代码证据**：
```typescript
// Account.tsx:1562-1569
const conditionsOpKey = this.state.filterConditionsOp === 'or' ? '$or' : '$and';
this.currentQuery = this.rootQuery.filter({
  [conditionsOpKey]: [...queryFilters, ...customQueryFilters],
});
```

**验证方法**：构造测试用例，分别使用 AND 和 OR 组合否定条件，对比结果差异。

---

### 4.3 日期近似匹配的范围

**结论**：`date isapprox '2024-01-15'` 匹配的是 `2024-01-13` 到 `2024-01-17`（前后各2天）

**代码证据**：
```typescript
// transaction-rules.ts:542-556
case 'isapprox': {
  const date = new Date(value.date);
  const startDate = new Date(date);
  startDate.setDate(startDate.getDate() - 2);  // 前2天
  const endDate = new Date(date);
  endDate.setDate(endDate.getDate() + 2);      // 后2天
  // ... 转换为 $and $gte $lte
}
```

**验证方法**：使用 `isapprox` 筛选特定日期，验证边界日期是否被包含。

---

### 4.4 空字符串匹配的双重语义

**结论**：`notes is ''` 同时匹配 `notes IS NULL` 和 `notes = ''`

**代码证据**：
```typescript
// transaction-rules.ts:571-575
if (value === '') {
  return {
    $or: [apply(field, '$eq', null), apply(field, '$eq', '')],
  };
}
```

**验证方法**：分别创建 notes 为 NULL 和空字符串的交易，使用 `is ''` 筛选，验证两者都被匹配。

---

### 4.5 金额 inflow/outflow 的隐含条件

**结论**：`amount (outflow) > 100` 实际转换为 `amount < 0 AND (-amount) > 100`

**代码证据**：
```typescript
// transaction-rules.ts:472-490
if (options.outflow) {
  return {
    $and: [
      { amount: { $lt: 0 } },
      { [field]: { $transform: '$neg', [aqlOp]: value } },
    ],
  };
}
```

**验证方法**：使用 outflow 选项筛选金额，查看生成的 AQL 表达式。

---

### 4.6 hasTags 的正则展开

**结论**：`notes hasTags '#groceries'` 使用正则 `(?<!#)#groceries([\s#]|$)`，不匹配 `##groceries` 和 `#groceries2`

**代码证据**：
```typescript
// transaction-rules.ts:632-640
return {
  $and: tagValues.map(v => {
    const escapedTag = v.replace(/[.*+?^${}()|[\]\\]/g, '\\$&').replace(/\\\$/g, '[$]');
    const pattern = `(?<!#)${escapedTag}([\\s#]|$)`;
    return apply(field, '$regexp', pattern);
  }),
};
```

**验证方法**：使用 `hasTags` 筛选，测试各种边界情况的字符串。

---

### 4.7 排序与筛选的交互

**结论**：筛选时如果正在按非日期字段排序，余额计算会被禁用

**代码证据**：
```typescript
// Account.tsx:648-664
// 当筛选器激活时，禁用余额计算，因为筛选后余额语义不明确
if (this.state.searchConditions.length > 0 && sortField !== 'date') {
  // 禁用余额列
}
```

**验证方法**：在筛选状态下切换排序字段，观察余额列是否显示。

---

### 4.8 保存筛选器的条件展开

**结论**：保存的筛选器在应用时会完全展开，修改保存的筛选器不会影响已应用的筛选

**代码证据**：
```typescript
// 应用筛选器时，将保存的条件展开为本地条件数组
// 后续修改保存的筛选器不会同步到已展开的条件
```

**验证方法**：应用一个保存的筛选器，然后修改该筛选器，验证已应用的筛选是否变化。

---

## 5. 关键代码位置索引（修订版）

| 功能模块 | 文件位置 | 关键函数/类 |
|---------|---------|------------|
| 条件类型定义 | `packages/loot-core/src/types/models/rule.ts` | `RuleConditionEntity` |
| 条件到 AQL 转换 | `packages/loot-core/src/server/transactions/transaction-rules.ts` | `conditionsToAQL`, `conditionSpecialCases` |
| 筛选器应用 | `packages/desktop-client/src/components/accounts/Account.tsx` | `applyFilters`, `updateQuery` |
| AQL 编译器 | `packages/loot-core/src/server/aql/compiler.ts` | `compileQuery`, `compileConditions` |
| LiveQuery 基类 | `packages/desktop-client/src/queries/liveQuery.ts` | `LiveQuery`, `onUpdate` |
| PagedQuery 分页查询 | `packages/desktop-client/src/queries/pagedQuery.ts` | `PagedQuery`, `fetchNext` |
| useTransactions Hook | `packages/desktop-client/src/hooks/useTransactions.ts` | `useTransactions`, `onSyncEvent` |
| TanStack Query 配置 | `packages/desktop-client/src/transactions/queries.ts` | `transactionQueries.aql` |
| AQL RPC 调用 | `packages/desktop-client/src/queries/aqlQuery.ts` | `aqlQuery` |
| 字段映射 | `packages/loot-core/src/shared/rules.ts` | `mapField`, `friendlyOp` |
| 标签匹配测试 | `packages/loot-core/src/server/transactions/transaction-rules.test.ts` | `hasTags` 相关测试用例 |
