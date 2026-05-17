# 交易筛选器复合表达式归约分析报告

## 1. 概述

本报告分析 Actual Budget 交易筛选器中复合表达式从用户界面到最终可执行查询的完整归约过程。整个流程分为三个核心阶段：

1.  **界面条件翻译**：将 UI 上的条件组合转换为内部布尔结构
2.  **表达式化简重写**：对内部表达式进行优化和规范化
3.  **查询映射与执行**：将表达式映射到本地查询，配合缓存与渲染协作

## 2. 界面条件到内部布尔结构的翻译

### 2.1 数据结构定义

筛选条件的核心数据结构是 `RuleConditionEntity`，定义于 `packages/loot-core/src/types/models/rule.ts:57-125`：

```typescript
type RuleConditionEntity = {
  field: string;           // 字段名 (account, category, amount, date 等)
  op: string;              // 操作符 (is, contains, gt, oneOf 等)
  value: unknown;          // 条件值
  options?: {              // 附加选项
    inflow?: boolean;      // 仅流入
    outflow?: boolean;     // 仅流出
    month?: boolean;
    year?: boolean;
  };
  conditionsOp?: 'and' | 'or';  // 条件间逻辑关系
  type?: 'id' | 'boolean' | 'date' | 'number' | 'string';
  customName?: string;     // 自定义筛选器名称
  queryFilter?: Record<string, { $oneof: string[] }>;
};
```

### 2.2 条件组合的 UI 表示

在 `Account.tsx:1522-1566` 的 `applyFilters` 方法中，界面条件被收集并发送到后端：

```typescript
applyFilters = async (conditions: ConditionEntity[]) => {
  if (conditions.length > 0) {
    const { filters: queryFilters } = await send(
      'make-filters-from-conditions',
      { conditions: ... }
    );
    const conditionsOpKey = this.state.filterConditionsOp === 'or' ? '$or' : '$and';
    this.currentQuery = this.rootQuery.filter({
      [conditionsOpKey]: [...queryFilters, ...customQueryFilters],
    });
  }
};
```

### 2.3 翻译过程关键点

1.  **顶层逻辑操作符**：`conditionsOp` 控制所有条件的组合方式（'and' 或 'or'），映射为 AQL 的 `$and` 或 `$or`
2.  **条件扁平化**：所有 UI 条件被收集到一维数组中，通过顶层操作符组合
3.  **自定义筛选器展开**：保存的筛选器（`TransactionFilterEntity`）在应用时会展开为具体条件数组

## 3. 化简与重写阶段

核心转换逻辑位于 `packages/loot-core/src/server/transactions/transaction-rules.ts:418-679` 的 `conditionsToAQL` 函数。

### 3.1 特殊情况处理

`conditionSpecialCases` 函数（`transaction-rules.ts:385-415`）处理隐式条件：

```typescript
function conditionSpecialCases(cond: Condition | null): Condition | null {
  // category is null 隐含：非转账、非父交易
  if (cond.op === 'is' && cond.field === 'category' && cond.value === null) {
    return new Condition('and', cond.field, [
      cond,
      new Condition('is', 'transfer', false, null),
      new Condition('is', 'parent', false, null),
    ], {});
  }
  // category isNot null 隐含：非父交易
  if (cond.op === 'isNot' && cond.field === 'category' && cond.value === null) {
    return new Condition('and', cond.field, [
      cond,
      new Condition('is', 'parent', false, null),
    ], {});
  }
  return cond;
}
```

### 3.2 操作符映射与展开

| UI 操作符 | AQL 操作符 | 说明 |
|----------|-----------|------|
| `is` | `$eq` | 等于 |
| `isNot` | `$ne` | 不等于 |
| `contains` | `$like` | 包含（自动添加 `%` 通配符） |
| `doesNotContain` | `$notlike` | 不包含 |
| `matches` | `$regexp` | 正则匹配 |
| `oneOf` | `$or` + `$eq` | 多值匹配展开为 OR |
| `notOneOf` | `$and` + `$ne` | 多值排除展开为 AND |
| `hasTags` | `$and` + `$regexp` | 标签匹配展开为多个正则 |
| `isapprox` | `$and` + `$gte` + `$lte` | 近似匹配展开为范围 |
| `isbetween` | `$and` + `$gte` + `$lte` | 区间匹配 |

### 3.3 日期条件展开

日期条件在 `mapConditionToActualQL` 中进行特殊处理（`transaction-rules.ts:506-556`）：

- **月份匹配**：`date is '2024-01'` 展开为 `date >= '2024-01-00' AND date <= '2024-01-99'`
- **年份匹配**：`date is '2024'` 展开为 `date >= '2024-00-00' AND date <= '2024-99-99'`
- **近似日期**：`date isapprox '2024-01-15'` 展开为前后 2 天范围
- **周期性日期**：重复规则的日期会展开为多个具体日期的 OR 组合

### 3.4 金额条件处理

金额条件根据 `options` 进行隐式条件添加（`transaction-rules.ts:472-490`）：

```typescript
if (options.outflow) {
  return {
    $and: [
      { amount: { $lt: 0 } },              // 金额小于0
      { [field]: { $transform: '$neg', [aqlOp]: value } },  // 取反后比较
    ],
  };
} else if (options.inflow) {
  return {
    $and: [
      { amount: { $gt: 0 } },              // 金额大于0
      { [field]: { [aqlOp]: value } },
    ],
  };
}
```

### 3.5 字符串空值处理

字符串空值匹配被特殊处理（`transaction-rules.ts:571-575`）：

```typescript
if (value === '') {
  return {
    $or: [apply(field, '$eq', null), apply(field, '$eq', '')],
  };
}
```

## 4. 本地查询映射、缓存与渲染协作

### 4.1 查询构建流程

1.  **根查询创建**：`makeRootTransactionsQuery()` 创建基础交易查询
2.  **过滤条件应用**：`applyFilters()` 将 AQL 表达式通过 `query.filter()` 添加
3.  **排序应用**：`applySort()` 添加排序条件
4.  **额外过滤**：`updateQuery()` 根据 UI 设置添加隐含过滤（如隐藏已对账交易）

### 4.2 缓存机制

系统使用两层缓存：

#### 4.2.1 TanStack Query 层（`transactionQueries.aql`）

```typescript
// packages/desktop-client/src/transactions/queries.ts:9-31
export const transactionQueries = {
  aql: ({ query, pageSize = 50 }) =>
    infiniteQueryOptions({
      queryKey: [...transactionQueries.all(), 'aql', query, pageSize],
      queryFn: async ({ pageParam }) => {
        const queryWithOffset = query
          .offset((pageParam as number) * pageSize)
          .limit(pageSize);
        const { data } = await aqlQuery(queryWithOffset);
        return data;
      },
      // ...
    }),
};
```

缓存键包含：查询对象、页面大小，确保相同查询能命中缓存。

#### 4.2.2 LiveQuery 实时更新层（`liveQuery.ts`）

```typescript
// packages/desktop-client/src/queries/liveQuery.ts:110-119
protected onUpdate = (tables: string[]) => {
  if (
    this._dependencies == null ||
    tables.find(table => this._dependencies.has(table))
  ) {
    void this.run();
  }
};
```

- 监听 `sync-event` 事件，当相关表变更时自动刷新
- 通过 `_dependencies` 跟踪查询依赖的表
- 支持乐观更新（`optimisticUpdate`）在本地立即应用更改

### 4.3 分页查询（`pagedQuery`）

```typescript
// Account.tsx:479-528
this.paged = pagedQuery(query.select('*'), {
  onData: async (groupedData, prevData) => {
    const data = ungroupTransactions([...groupedData]);
    // ... 更新状态并渲染
  },
  options: {
    pageCount: 150,    // 每页加载数量
    onlySync: true,    // 仅响应同步事件
  },
});
```

### 4.4 渲染协作流程

```
用户操作 → applyFilters → make-filters-from-conditions (RPC)
         ↓
conditionsToAQL (后端转换)
         ↓
AQL 表达式 → Query.filter() → TanStack Query 缓存键
         ↓
命中缓存？→ 是 → 返回缓存数据 → 渲染
         ↓ 否
aqlQuery (RPC) → SQLite 查询 → 结果缓存 → 渲染
         ↓
sync-event 触发 → 缓存失效 → 重新查询
```

### 4.5 渲染优化

1.  **乐观更新**：`pagedQuery.optimisticUpdate()` 在服务器响应前更新本地数据
2.  **分组展开控制**：筛选模式下自动折叠拆分交易（`Account.tsx:487-497`）
3.  **动画控制**：首次加载禁用行动画，避免视觉混乱

## 5. 容易与用户直觉不一致的组合方式

### 5.1 NULL 值的隐式条件

**问题**：`category is null` 实际上匹配的是：
```
category IS NULL AND transfer_id IS NULL AND is_parent = 0
```

用户可能期望只匹配分类为空的交易，但实际上自动排除了转账和父交易。

**影响场景**：
- 查找未分类交易时，转账交易会被排除
- 拆分交易的父项不会被匹配

### 5.2 OR 组合下的否定条件

**问题**：当 `conditionsOp = 'or'` 时，否定条件（`isNot`, `doesNotContain`, `notOneOf`）的语义可能与直觉不符。

**示例**：
- 条件：`[category isNot '食物', amount > 100]`，操作符：`OR`
- 实际语义：`category != '食物' OR amount > 100`
- 用户可能期望：`NOT (category = '食物' OR amount > 100)`

### 5.3 日期近似匹配的范围

**问题**：`date isapprox '2024-01-15'` 匹配的是 `2024-01-13` 到 `2024-01-17`（前后各2天），用户可能期望不同的误差范围。

### 5.4 空字符串匹配的双重语义

**问题**：`notes is ''` 同时匹配 `notes IS NULL` 和 `notes = ''`。

用户可能期望只匹配空字符串，但实际上也包含了 NULL 值。

### 5.5 金额 inflow/outflow 的隐含条件

**问题**：当选择"流入"或"流出"时，系统会自动添加金额符号判断：
- `amount (inflow) > 100` → `amount > 0 AND amount > 100`
- `amount (outflow) > 100` → `amount < 0 AND (-amount) > 100`

用户可能没有意识到这些附加条件，导致结果不符合预期。

### 5.6 `hasTags` 的正则展开

**问题**：`notes hasTags '#groceries'` 被展开为复杂的正则表达式：
```regexp
(?<!#)#groceries([\s#]|$)
```

这意味着：
- 不匹配 `##groceries`
- 不匹配 `#groceries2`
- 匹配 `#groceries,` 或 `#groceries#other`

用户可能期望简单的子字符串匹配。

### 5.7 排序与筛选的交互

**问题**：筛选时如果正在按非日期字段排序，余额计算会被禁用（`Account.tsx:648-664`）。

用户可能期望在筛选状态下仍能看到余额，但由于筛选后余额计算语义不明确，系统选择禁用。

### 5.8 保存筛选器的条件展开

**问题**：保存的筛选器在应用时会完全展开，修改保存的筛选器不会影响已应用的筛选。

用户可能期望已应用的筛选与保存的筛选器保持关联，但实际上是值拷贝关系。

## 6. 关键代码位置索引

| 功能模块 | 文件位置 | 关键函数/类 |
|---------|---------|------------|
| 条件类型定义 | `packages/loot-core/src/types/models/rule.ts` | `RuleConditionEntity` |
| 条件到 AQL 转换 | `packages/loot-core/src/server/transactions/transaction-rules.ts` | `conditionsToAQL`, `conditionSpecialCases` |
| 筛选器应用 | `packages/desktop-client/src/components/accounts/Account.tsx` | `applyFilters`, `updateQuery` |
| AQL 编译器 | `packages/loot-core/src/server/aql/compiler.ts` | `compileQuery`, `compileConditions` |
| 查询缓存 | `packages/desktop-client/src/transactions/queries.ts` | `transactionQueries` |
| 实时查询 | `packages/desktop-client/src/queries/liveQuery.ts` | `LiveQuery` |
| 分页查询 | `packages/desktop-client/src/queries/pagedQuery.ts` | `pagedQuery` |
| 字段映射 | `packages/loot-core/src/shared/rules.ts` | `mapField`, `friendlyOp` |
