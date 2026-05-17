# 账户名称（Payee）自动补全与筛选器联动机制分析报告

## 一、自动补全建议项的数据来源

### 1.1 核心数据源分层结构

`PayeeAutocomplete` 组件 (`packages/desktop-client/src/components/autocomplete/PayeeAutocomplete.tsx:363-620`) 通过以下四层数据源构建建议列表：

| 数据源类型 | 获取方式 | 优先级 | 说明 |
|-----------|---------|--------|------|
| **附近账户** (Nearby Payees) | `useNearbyPayees()` hook | 最高 | 基于地理位置服务，使用 Haversine 公式计算距离，返回最近 10 个有位置记录的账户 |
| **收藏账户** (Favorite Payees) | `usePayees()` → 过滤 `favorite=true` | 次高 | 用户标记为收藏的账户，按名称排序 |
| **常用账户** (Common Payees) | `useCommonPayees()` hook | 中等 | 最近 12 周内交易频次最高的前 10 个账户（不含转账账户） |
| **所有账户** (All Payees) | `usePayees()` hook | 最低 | 系统中所有活跃账户，排除已关闭账户对应的转账账户 |

### 1.2 常用账户算法 (`getCommonPayees`)

**位置**: `packages/loot-core/src/server/db/index.ts:653-680`

```sql
SELECT p.id, p.name, p.favorite, p.category, 
       TRUE as common, NULL as transfer_acct,
       count(*) as c, max(t.date) as latest
FROM payees p
LEFT JOIN v_transactions_internal_alive t ON t.payee == p.id
WHERE LENGTH(p.name) > 0
  AND p.tombstone = 0
  AND t.date > ${twelveWeeksAgo}  -- 最近12周
GROUP BY p.id
ORDER BY c DESC, p.transfer_acct IS NULL DESC, p.name COLLATE NOCASE
LIMIT 10
```

### 1.3 附近账户算法 (`getNearbyPayees`)

**位置**: `packages/loot-core/src/server/payees/app.ts:216-351`

- 使用 **Haversine 公式** 计算经纬度距离（单位：米）
- 通过 `ROW_NUMBER()` 窗口函数为每个账户选择最近的一个位置记录
- 默认最大距离：`DEFAULT_MAX_DISTANCE_METERS`（常量定义）
- 返回结果按距离升序排列，最多 10 条

### 1.4 活跃账户过滤 (`getActivePayees`)

**位置**: `packages/desktop-client/src/payees/queries.ts:78-92`

转账账户 (`transfer_acct` 非空) 需要检查对应账户是否已关闭：
```typescript
if (payee.transfer_acct) {
  const account = accountsById[payee.transfer_acct];
  return account != null && !account.closed;
}
```

---

## 二、自动补全建议项的分组与展示规则

### 2.1 建议项类型分类 (`determineItemType`)

**位置**: `PayeeAutocomplete.tsx:152-168`

| 类型标记 | 判定条件 | UI 分组标题 |
|---------|---------|------------|
| `account` | `item.transfer_acct` 为真 | "Transfer To/From" |
| `nearby_payee` | `isNearby = true` | "Nearby Payees" |
| `common_payee` | `isCommon = true` | "Suggested Payees" |
| `payee` | 其他情况 | "Payees" |

### 2.2 建议项排序逻辑 (`getPayeeSuggestions`)

**位置**: `PayeeAutocomplete.tsx:56-96`

1. **收藏账户** 优先展示，按名称排序
2. 若收藏账户不足 5 个，用 **常用账户** 补充到 5 个
3. 其余账户按普通账户展示
4. 转账账户单独分组排在最后

### 2.3 模糊搜索算法

使用 `fzf` 库进行模糊匹配：
```typescript
new Fzf(realSuggestions, {
  selector: item => item.name ?? '',
  limit: 100,
})
.find(value)
```

### 2.4 新建账户选项

当用户输入内容且无精确匹配时，在列表顶部显示 "Create payee \"XXX\"" 选项：
- 精确匹配判断：`getNormalisedString(firstFiltered.name) === getNormalisedString(value)`
- 转账模式下 (`focusTransferPayees = true`) 不显示新建选项

---

## 三、与筛选器的合并规则

### 3.1 PayeeFilter 组件 (`packages/desktop-client/src/components/filters/PayeeFilter.tsx`)

作为 `FiltersMenu` 的子组件，支持四种操作符：

| 操作符 | 类型 | 值格式 |
|-------|------|--------|
| `is` | 单选 | `string` (单个 payee ID) |
| `isNot` | 单选 | `string` (单个 payee ID) |
| `oneOf` | 多选 | `string[]` (payee ID 数组) |
| `notOneOf` | 多选 | `string[]` (payee ID 数组) |

**关键特性**:
- 强制 `showInactivePayees = true`，允许选择已关闭账户对应的转账账户
- 强制 `showMakeTransfer = false`，隐藏转账切换按钮
- 根据操作符自动切换单选/多选模式

### 3.2 操作符切换时的值转换逻辑

**位置**: `FiltersMenu.tsx:225-321`

#### 3.2.1 单值 ID ↔ 文本转换
```
is/isNot → contains/matches/doesNotContain:
  ID → 查找对应 payee 的 name 文本

contains/matches/doesNotContain → is/isNot:
  文本 → 精确匹配 name，唯一匹配时返回 ID，否则清空
```

#### 3.2.2 多值 ID ↔ 文本转换
```
oneOf/notOneOf → contains/matches/doesNotContain:
  数组长度为 1 时转换为文本，否则清空

contains/matches/doesNotContain → oneOf/notOneOf:
  精确匹配唯一结果时包装为数组 [ID]
```

### 3.3 多条件合并逻辑

筛选条件通过 `conditionsOp` 字段（`'and'` 或 `'or'`）进行合并：
- **AND 模式** (`$and`): 所有条件必须同时满足
- **OR 模式** (`$or`): 满足任一条件即可

**位置**: `Account.tsx:1522-1566`

```typescript
const conditionsOpKey = this.state.filterConditionsOp === 'or' ? '$or' : '$and';
this.currentQuery = this.rootQuery.filter({
  [conditionsOpKey]: [...queryFilters, ...customQueryFilters],
});
```

---

## 四、选择自动补全后的更新链路

### 4.1 完整更新流程

```
用户选择补全项
    ↓
PayeeAutocomplete.handleSelect() [PayeeAutocomplete.tsx:475-493]
    ├─ 若为新建 payee（id='new'）:
    │   调用 createPayeeMutation 创建新账户
    │   返回新创建的 payee ID
    └─ 调用 onSelect 回调
        ↓
PayeeFilter.onChange() [PayeeFilter.tsx:54-56]
    ↓
FiltersMenu.dispatch({ type: 'set-value' }) [FiltersMenu.tsx:522]
    ↓
用户点击 Apply 按钮
    ↓
ConfigureField.onSubmit() [FiltersMenu.tsx:440-477]
    ├─ 调用 unparse() 标准化条件格式
    ├─ 调用 rule-validate 后端验证
    └─ 调用 onValidateAndApply()
        ↓
AccountInternal.onAddFilter() [Account.tsx:1448-1487]
    ├─ 检查重复条件
    ├─ 合并到 filterConditions 数组
    └─ 调用 applyFilters()
        ↓
AccountInternal.applyFilters() [Account.tsx:1522-1566]
    ├─ 调用 make-filters-from-conditions 后端接口
    │   将 RuleConditionEntity 转换为查询过滤器
    ├─ 构建查询条件链（$and/$or）
    ├─ 更新 rootQuery.filter()
    ├─ setState({ filterConditions })
    └─ 调用 updateQuery() 执行查询
        ↓
AccountInternal.updateQuery() [Account.tsx:465-519]
    ├─ 订阅 pagedQuery
    ├─ 接收交易数据
    ├─ 调用 calculateBalances() 计算余额
    ├─ 调用 getFilteredAmount() 计算筛选汇总
    └─ setState 更新 transactions, balances, filteredAmount
        ↓
UI 更新：交易列表 + 汇总指标
```

### 4.2 交易列表更新 (`pagedQuery`)

**位置**: `Account.tsx:479-519`

分页查询通过 `pagedQuery` 实现增量加载：
- 新数据到达时自动解包分组交易 (`ungroupTransactions`)
- 首次加载时自动关闭拆分交易展开状态
- 筛选模式下 (`isFiltered = true`) 强制关闭所有拆分

### 4.3 汇总指标更新

筛选后的汇总金额通过 `getFilteredAmount()` 方法异步计算：
- 与交易数据加载并行执行
- 结果存储在 `state.filteredAmount`
- 在账户头部展示筛选后的总金额

---

## 五、与其他模块的相互影响

### 5.1 与全局状态的交互

#### 5.1.1 React Query 缓存层
```
usePayees()        → queryKey: ['payees', 'lists']
useCommonPayees()  → queryKey: ['payees', 'lists', 'common']
useNearbyPayees()  → queryKey: ['payees', 'nearby']
```
- 缓存策略：`staleTime: Infinity`（永久有效，手动失效）
- 同步事件触发时手动失效缓存

#### 5.1.2 Redux 状态
- 筛选条件存储在组件内部状态（非 Redux）
- 已保存的筛选器（Saved Filters）通过 `useTransactionFilters()` 从数据库查询

### 5.2 与同步系统的交互

**位置**: `Account.tsx:329-338`

```typescript
const maybeRefetch = (tables: string[]) => {
  if (tables.includes('transactions') ||
      tables.includes('category_mapping') ||
      tables.includes('payee_mapping')) {
    return this.refetchTransactions();
  }
};
```

当以下数据发生同步变更时自动重查：
- `transactions` 表：交易数据变更
- `payee_mapping` 表：账户合并/映射变更
- `category_mapping` 表：分类映射变更

### 5.3 与全局命令面板的关系

经代码搜索，**账户名称自动补全与全局命令面板无直接交互**。两者是独立的功能模块：

- **全局命令面板**: 主要通过键盘快捷键触发，提供全局操作入口
- **Payee 自动补全**: 是表单/筛选器内的嵌入式组件，仅在特定输入框聚焦时激活

潜在的间接关联：
- 命令面板可能包含导航到账户管理页面的命令
- 但不会直接触发或控制 `PayeeAutocomplete` 组件的行为

---

## 六、关键代码位置索引

| 功能模块 | 文件路径 | 关键行号 |
|---------|---------|---------|
| Payee 自动补全主组件 | `packages/desktop-client/src/components/autocomplete/PayeeAutocomplete.tsx` | 363-620 |
| 建议项生成逻辑 | `packages/desktop-client/src/components/autocomplete/PayeeAutocomplete.tsx` | 56-96 |
| Payee 筛选器组件 | `packages/desktop-client/src/components/filters/PayeeFilter.tsx` | 31-59 |
| 筛选器菜单 | `packages/desktop-client/src/components/filters/FiltersMenu.tsx` | 87-541 |
| 账户页面筛选应用 | `packages/desktop-client/src/components/accounts/Account.tsx` | 1522-1566 |
| 常用账户 DB 查询 | `packages/loot-core/src/server/db/index.ts` | 653-680 |
| 附近账户服务端逻辑 | `packages/loot-core/src/server/payees/app.ts` | 216-351 |
| Payee 查询配置 | `packages/desktop-client/src/payees/queries.ts` | 16-76 |

---

## 七、设计特点总结

1. **分层数据源策略**: 收藏 → 常用 → 附近 → 全部，优先展示用户最可能选择的选项
2. **智能新建检测**: 精确匹配时不显示新建选项，避免干扰
3. **无缝操作符切换**: 不同操作符间自动转换值格式，减少用户输入丢失
4. **响应式更新链路**: 选择 → 筛选 → 查询 → 渲染 全链路自动化
5. **同步感知**: 数据同步变更时自动刷新，保持视图一致性
6. **地理位置增强**: 移动设备上提供附近账户建议，提升录入效率
