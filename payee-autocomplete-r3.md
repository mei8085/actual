# 账户名称（Payee）自动补全与筛选器联动机制分析报告（修订版 v3）

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

---

## 四、三处筛选入口的应用链路对比

### 4.1 三处筛选入口概览

| 入口场景 | 主要组件 | 状态管理 | 数据更新方式 | 典型使用场景 |
|---------|---------|---------|------------|-------------|
| **账户页** | `Account.tsx` (Class Component) | 组件内部 `state` | `pagedQuery` 分页订阅 | 单个账户的交易列表 |
| **报表页** | `CustomReport.tsx` + `useRuleConditionFilters` | Hook 状态管理 | `useReport` + spreadsheet 计算 | 自定义报表图表数据 |
| **公式查询器** | `QueryManager.tsx` + `Formula.tsx` | `useRef` 存储 + 版本号 | `useFormulaExecution` 异步执行 | 公式小部件的 QUERY 函数参数 |

### 4.2 账户页筛选应用链路 (`Account.tsx`)

**状态管理**: Class Component 内部状态
```typescript
type AccountInternalState = {
  filterConditions: ConditionEntity[];
  filterConditionsOp: 'and' | 'or';
  filterId?: SavedFilter;
  transactions: TransactionEntity[];
  balances: Record<string, IntegerAmount> | null;
  filteredAmount: null | number;
};
```

**完整更新流程**:
```
用户选择补全项
    ↓
PayeeAutocomplete.handleSelect()
    ├─ 新建 payee 时调用 createPayeeMutation
    └─ 调用 onSelect 回调
        ↓
PayeeFilter.onChange()
    ↓
FiltersMenu.dispatch({ type: 'set-value' })
    ↓
用户点击 Apply 按钮
    ↓
ConfigureField.onSubmit()
    ├─ unparse() 标准化条件格式
    ├─ rule-validate 后端验证
    └─ onValidateAndApply()
        ↓
AccountInternal.onAddFilter()
    ├─ 检查重复条件（isEqual 比较）
    ├─ 合并到 filterConditions 数组
    └─ applyFilters()
        ↓
AccountInternal.applyFilters() [Account.tsx:1522-1566]
    ├─ send('make-filters-from-conditions') 后端接口
    │   将 RuleConditionEntity → 查询过滤器
    ├─ 构建查询条件链（$and/$or）
    ├─ 更新 rootQuery.filter()
    ├─ setState({ filterConditions })
    └─ updateQuery() 执行查询
        ↓
AccountInternal.updateQuery() [Account.tsx:465-519]
    ├─ 订阅 pagedQuery（分页增量加载）
    ├─ ungroupTransactions 解包分组交易
    ├─ calculateBalances() 异步计算余额
    ├─ getFilteredAmount() 异步计算筛选汇总
    └─ setState 更新 transactions, balances, filteredAmount
        ↓
UI 更新：交易列表 + 汇总指标
```

**关键特性**:
- 使用 `pagedQuery` 实现分页增量加载
- 筛选模式下自动关闭所有拆分交易展开
- 同步事件（transactions/payee_mapping 表变更）自动触发重查
- 汇总金额 `filteredAmount` 与交易数据并行计算

### 4.3 报表页筛选应用链路 (`CustomReport.tsx`)

**状态管理**: `useRuleConditionFilters` Hook
```typescript
// useRuleConditionFilters.ts:5-75
const {
  conditions,
  conditionsOp,
  onApply,        // 添加筛选条件
  onDelete,       // 删除筛选条件
  onUpdate,       // 更新筛选条件
  onConditionsOpChange,
} = useRuleConditionFilters();
```

**完整更新流程**:
```
用户选择补全项
    ↓
PayeeAutocomplete.handleSelect()
    ↓
PayeeFilter.onChange()
    ↓
FiltersMenu.dispatch({ type: 'set-value' })
    ↓
用户点击 Apply 按钮
    ↓
ConfigureField.onSubmit()
    ├─ unparse() 标准化
    ├─ rule-validate 验证
    └─ onValidateAndApply()
        ↓
onApplyFilter(condition)  [useRuleConditionFilters.onApply]
    ├─ setConditions([...state, condition])
    └─ setSaved(null) 标记为已修改
        ↓
条件变更触发 useReport 重新计算
    ↓
报表 spreadsheet 计算（如 spending-spreadsheet.ts:46）
    ├─ send('make-filters-from-conditions') 转换条件
    ├─ 构建 $and/$or 条件链
    ├─ aqlQuery 执行查询获取资产/负债数据
    └─ 图表数据聚合计算
        ↓
UI 更新：报表图表 + 图例 + 汇总面板
```

**关键特性**:
- 筛选条件变更后自动触发报表数据重算
- 不同报表类型有独立的 spreadsheet 计算逻辑
- 支持保存筛选条件到报表配置中
- 筛选条件通过 sessionStorage 跨页面暂存

### 4.4 公式查询器筛选应用链路 (`QueryManager.tsx`)

**状态管理**: `useRef` 存储 + 版本号触发重计算
```typescript
// Formula.tsx:60-61
const queriesRef = useRef(widget?.meta?.queries || {});
const [queriesVersion, setQueriesVersion] = useState(0);
```

**完整更新流程**:
```
用户在 QueryManager 中添加 payee 筛选
    ↓
FilterButton → ConfigureField → onApply
    ↓
useRuleConditionFilters.onApply()
    ├─ 更新 conditions 数组
    └─ 触发 sendUpdate()
        ↓
QueryItem.sendUpdate() [QueryManager.tsx:323-349]
    ├─ 组装 QueryConfig: { conditions, conditionsOp, timeFrame }
    └─ 调用 onUpdate 回调
        ↓
QueryManager.handleUpdateQuery()
    └─ onQueriesChange(newQueries)
        ↓
Formula.handleQueriesChange() [Formula.tsx:105-111]
    ├─ queriesRef.current = newQueries
    └─ setQueriesVersion(v => v + 1)  // 版本号 +1 触发重执行
        ↓
useFormulaExecution 检测到 queriesVersion 变化
    ↓
公式重新执行
    ├─ 解析公式中的 QUERY("queryName") 引用
    ├─ 对每个查询调用 make-filters-from-conditions
    ├─ 执行 AQL 查询获取交易数据
    └─ HyperFormula 计算公式结果
        ↓
UI 更新：公式结果展示 + 颜色条件格式化
```

**关键特性**:
- 使用 `useRef` 存储查询配置，避免不必要的重渲染
- 通过版本号 `queriesVersion` 触发公式重执行
- 支持多个命名查询，每个查询有独立的筛选条件和时间范围
- 筛选条件持久化到 dashboard widget 的 meta 数据中

### 4.5 三处入口的核心差异对比

| 对比维度 | 账户页 | 报表页 | 公式查询器 |
|---------|-------|-------|-----------|
| **条件提交时机** | 点击 Apply 按钮后立即应用 | 点击 Apply 后立即应用 | Apply 后更新配置，版本号变化时触发 |
| **查询触发方式** | `pagedQuery` 订阅式，增量加载 | `useReport` 响应式，全量重算 | `useFormulaExecution` 按需执行 |
| **状态存储** | 组件 `state` | Hook `useState` | `useRef` + 版本号 |
| **数据更新粒度** | 分页增量，实时更新 | 全量重算，图表级更新 | 公式级，按需计算 |
| **汇总更新** | `filteredAmount` 并行计算 | 图表聚合内嵌计算 | 公式返回值即汇总 |
| **与 URL 关系** | 账户 ID 从 URL 获取 | 报表 ID 从 URL 获取 | 小部件 ID 从 URL 获取 |
| **持久化方式** | 临时状态，不保存 | 保存到报表配置 | 保存到小部件 meta |
| **筛选条件数量** | 多条件组合 | 多条件组合 | 每个查询独立多条件 |

### 4.6 条件转换的统一接口

三处入口最终都调用同一个后端接口进行条件转换：

```typescript
// 统一调用方式
const { filters } = await send('make-filters-from-conditions', {
  conditions: conditions.filter(cond => !cond.customName),
});

// 统一构建查询
const conditionsOpKey = conditionsOp === 'or' ? '$or' : '$and';
query = query.filter({
  [conditionsOpKey]: [...filters],
});
```

---

## 五、与全局命令面板的交互边界（严谨校对版）

### 5.1 全局命令面板 (`CommandBar.tsx`)

**位置**: `packages/desktop-client/src/components/CommandBar.tsx:93-375

**功能定位**: 全局导航与快速访问入口
- 触发方式: `Ctrl/Cmd + K` 键盘快捷键
- 技术实现: 基于 `cmdk` 库
- 包含内容: 导航链接、账户列表、报表列表、自定义报表

### 5.2 PayeeAutocomplete 的两种使用模式

PayeeAutocomplete 组件存在两种使用模式，这是理解交互边界的关键：

| 使用模式 | 触发方式 | 是否加入 modalStack | data-testid |
|---------|---------|-------------------|------------|
| **内联/嵌入式模式** | 直接渲染组件（如筛选器内） | ❌ 不加入 | `autocomplete` |
| **模态框模式** | `pushModal({ name: 'payee-autocomplete' })` | ✅ 加入 | `payee-autocomplete-modal` |

**内联模式使用场景**：
- 筛选器中的 `PayeeFilter.tsx:47-57`
- 交易表格中的 payee 字段编辑
- 规则编辑中的 payee 选择

**模态框模式使用场景**：
- 批量操作中的 payee 选择
- 移动端全屏选择器

### 5.3 Ctrl/Cmd+K 全局监听的实际传播路径

#### 5.3.1 命令面板的打开逻辑

**位置**: `CommandBar.tsx:147-157

```typescript
const openEventListener = useCallback(
  (e: KeyboardEvent) => {
    if (e.key === 'k' && (e.metaKey || e.ctrlKey)) {
      e.preventDefault();
      // 关键：仅检查 modalStack.length > 0
      if (modalStack.length > 0) return;
      setOpen(true);
    }
  },
  [modalStack.length],
);
```

**关键发现**：
- 命令面板的打开逻辑**仅检查 `modalStack.length > 0**
- **不检查** filters-menu-tooltip（筛选器弹出层）
- **不检查** autocomplete 弹出层（自动补全列表）
- 事件监听器注册在 `document` 级别

#### 5.3.2 Autocomplete 组件的键盘事件处理

**位置**: `Autocomplete.tsx:544-612

Autocomplete 组件的 `onKeyDown` 仅处理以下按键：
- `Enter` - 选择高亮项
- `Escape` - 关闭/重置
- `Tab` - 保存并移动焦点
- `ArrowUp/ArrowDown` - 导航建议项

**对 Ctrl/Cmd+K 无任何特殊处理**：
- 没有 `stopPropagation()` 调用
- 没有 `preventDefault()` 调用
- 事件会正常冒泡到 document 级别的命令面板监听器

#### 5.3.3 键盘事件的完整传播链

```
用户在 Payee 输入框按下 Ctrl/Cmd+K
    ↓
输入框原生 keydown 事件
    ↓
Autocomplete.onKeyDown()
    ├─ 检查是否为 Enter/Escape/Tab/方向键
    └─ 不是上述按键 → 不拦截，不阻止传播
        ↓
事件冒泡到父元素
    ↓
事件继续冒泡到 document
        ↓
CommandBar.openEventListener() 捕获事件
    ↓
检查 modalStack.length > 0?
    ├─ 是 → 忽略（已有模态框打开中）
    └─ 否 → 打开命令面板
```

### 5.4 filters popover 是否属于 modalStack 拦截范围？

**结论：** **filters popover（`filters-menu-tooltip`）**不属于** modalStack 拦截范围。

证据：
1. **modalStack 的定义** (`modalsSlice.ts:691-699`)：
   ```typescript
   type ModalsState = {
     modalStack: Modal[];  // 仅包含通过 pushModal 推入的模态框
     isHidden: boolean;
   };
   ```

2. **filters popover 的实现**：
   - 使用 `@actual-app/components/popover` 组件实现
   - 渲染在 React 组件树内
   - **不通过** `pushModal` 推入 Redux store
   - 有独立的 `data-testid="filters-menu-tooltip"` 标识

3. **唯一的特殊检查** (`modalsSlice.ts:717-728`)：
   ```typescript
   if (
     modal.name.endsWith('keyboard-shortcuts') &&
     (state.modalStack.length > 0 ||
       window.document.querySelector(
         'div[data-testid="filters-menu-tooltip"]',
       ) !== null)
   ) {
     return state;
   }
   ```
   这是**仅针对 `keyboard-shortcuts` 模态框**的特殊处理，不影响 CommandBar。

### 5.5 四种场景触发结果矩阵

#### 场景定义：
- **命令面板状态**：打开 / 关闭
- **补全列表状态**：打开 / 关闭

| 场景 | 命令面板 | 补全列表 | 触发 Ctrl/Cmd+K 结果 | 技术原因 |
|-----|---------|----------|---------------------|----------|
| **场景 1** | 关闭 | 关闭 | ✅ 命令面板正常打开 | `modalStack.length = 0`，无任何拦截 |
| **场景 2** | 关闭 | 打开（内联模式） | ✅ 命令面板打开<br>⚠️ 补全列表因失焦关闭 | 内联补全不在 modalStack 中<br>输入框失焦触发补全关闭 |
| **场景 3** | 关闭 | 打开（模态框模式） | ❌ 命令面板不打开 | 模态框模式下 `modalStack.length > 0`<br>CommandBar 逻辑拦截 |
| **场景 4** | 打开 | 关闭 | ⚠️ 命令面板已打开<br>输入焦点在命令面板搜索框 | 命令面板打开时 `modalStack.length > 0`<br>但 CommandBar 自身拦截 |
| **场景 5** | 打开 | 打开 | ❌ 理论上不可能同时存在 | 命令面板打开时 modalStack > 0<br>无法再打开 payee 模态框 |

### 5.6 交互边界的修正结论

**修正后的结论**：账户名称自动补全与全局命令面板**并非完全独立**，存在以下交互：

1. **模态框模式下的互斥**：
   - 当 PayeeAutocomplete 以模态框模式打开时，CommandBar 被拦截
   - 反之，CommandBar 打开时，无法打开 payee 模态框

2. **内联模式下的竞争**：
   - 内联补全列表打开时，Ctrl/Cmd+K 会打开命令面板
   - 补全列表因输入框失焦而自动关闭
   - 这是**预期行为**，不是 bug

3. **共享的键盘事件**：
   - 两者都监听键盘事件，但作用域不同
   - 没有显式的事件互斥处理

4. **间接关联**：
   - 命令面板可导航到账户页/报表页，这些页面内可使用 payee 筛选器

---

## 六、完整的更新链路总结

### 6.1 账户页更新链路（最完整）

```
用户选择 payee 补全项
    ↓
PayeeAutocomplete.handleSelect()
    ├─ 新建 payee: createPayeeMutation
    └─ 现有 payee: 直接返回 ID
        ↓
PayeeFilter.onChange(payeeId)
    ↓
FiltersMenu 本地状态更新
    ↓
[用户点击 Apply]
    ↓
ConfigureField.onSubmit()
    ├─ unparse() 标准化条件
    ├─ rule-validate 后端验证
    └─ onValidateAndApply(condition)
        ↓
AccountInternal.onAddFilter(condition)
    ├─ 重复检查 (isEqual)
    ├─ 合并到 filterConditions
    └─ applyFilters(conditions)
        ↓
AccountInternal.applyFilters()
    ├─ send('make-filters-from-conditions')
    │   conditions → filters[]
    ├─ 构建 $and/$or 查询
    ├─ rootQuery.filter({ [conditionsOpKey]: filters })
    ├─ setState({ filterConditions })
    └─ updateQuery(query, isFiltered=true)
        ↓
AccountInternal.updateQuery()
    ├─ pagedQuery 订阅
    │   ├─ 接收分页数据
    │   ├─ ungroupTransactions()
    │   └─ setState({ transactions })
    ├─ calculateBalances() → setState({ balances })
    └─ getFilteredAmount() → setState({ filteredAmount })
        ↓
渲染更新
    ├─ 交易列表 TransactionsTable
    ├─ 账户头部余额显示
    └─ 筛选汇总金额显示
```

### 6.2 报表页更新链路

```
用户选择 payee 补全项
    ↓
PayeeAutocomplete.handleSelect() → onSelect
    ↓
PayeeFilter.onChange()
    ↓
FiltersMenu 本地状态更新
    ↓
[用户点击 Apply]
    ↓
ConfigureField.onSubmit() → onValidateAndApply
    ↓
useRuleConditionFilters.onApply(condition)
    └─ setConditions([...prev, condition])
        ↓
条件变更触发 useReport 重算
    ↓
报表 spreadsheet 执行
    ├─ make-filters-from-conditions
    ├─ aqlQuery 查询数据
    └─ 聚合计算图表数据
        ↓
渲染更新
    ├─ 图表 (SVG/Canvas)
    ├─ ReportLegend 图例
    └─ ReportSummary 汇总
```

### 6.3 公式查询器更新链路

```
用户在 QueryManager 中配置 payee 筛选
    ↓
FilterButton → ConfigureField → onApply
    ↓
useRuleConditionFilters.onApply() → 更新 conditions
    ↓
QueryItem.sendUpdate() → 组装 QueryConfig
    ↓
QueryManager.onQueriesChange() → 更新 queries 对象
    ↓
Formula.handleQueriesChange()
    ├─ queriesRef.current = newQueries
    └─ setQueriesVersion(v => v + 1)
        ↓
useFormulaExecution 检测版本变化
    ├─ 解析公式中的 QUERY() 引用
    ├─ 每个查询独立调用 make-filters-from-conditions
    ├─ 执行 AQL 查询获取数据
    └─ HyperFormula 计算公式结果
        ↓
渲染更新
    ├─ FormulaResult 数值显示
    └─ 颜色条件格式化
```

---

## 七、关键代码位置索引

| 功能模块 | 文件路径 | 关键行号 |
|---------|---------|---------|
| Payee 自动补全主组件 | `packages/desktop-client/src/components/autocomplete/PayeeAutocomplete.tsx` | 363-620 |
| 建议项生成逻辑 | `packages/desktop-client/src/components/autocomplete/PayeeAutocomplete.tsx` | 56-96 |
| Payee 筛选器组件 | `packages/desktop-client/src/components/filters/PayeeFilter.tsx` | 31-59 |
| 筛选器菜单 | `packages/desktop-client/src/components/filters/FiltersMenu.tsx` | 87-541 |
| 账户页面筛选应用 | `packages/desktop-client/src/components/accounts/Account.tsx` | 1522-1566 |
| 报表页条件管理 Hook | `packages/desktop-client/src/hooks/useRuleConditionFilters.ts` | 5-75 |
| 自定义报表主组件 | `packages/desktop-client/src/components/reports/reports/CustomReport.tsx` | 119-650 |
| 公式查询管理器 | `packages/desktop-client/src/components/formula/QueryManager.tsx` | 55-400 |
| 公式执行 Hook | `packages/desktop-client/src/hooks/useFormulaExecution.ts` | 84-150 |
| 全局命令面板 | `packages/desktop-client/src/components/CommandBar.tsx` | 93-375 |
| 命令面板快捷键监听 | `packages/desktop-client/src/components/CommandBar.tsx` | 147-157 |
| modalStack 状态管理 | `packages/desktop-client/src/modals/modalsSlice.ts` | 691-748 |
| modalStack 拦截逻辑 | `packages/desktop-client/src/modals/modalsSlice.ts` | 717-728 |
| Payee 自动补全模态框 | `packages/desktop-client/src/components/modals/PayeeAutocompleteModal.tsx` | 24-90 |
| 常用账户 DB 查询 | `packages/loot-core/src/server/db/index.ts` | 653-680 |
| 附近账户服务端逻辑 | `packages/loot-core/src/server/payees/app.ts` | 216-351 |
| Payee 查询配置 | `packages/desktop-client/src/payees/queries.ts` | 16-76 |

---

## 八、设计特点总结

### 8.1 自动补全设计
1. **分层数据源策略**: 附近 → 收藏 → 常用 → 全部，优先展示用户最可能选择的选项
2. **智能新建检测**: 精确匹配时不显示新建选项，避免干扰
3. **地理位置增强**: 移动设备上提供附近账户建议，提升录入效率

### 8.2 筛选器设计
1. **无缝操作符切换**: 不同操作符间自动转换值格式，减少用户输入丢失
2. **复用性**: 同一 `PayeeFilter` 组件在三处场景复用，通过 props 适配不同需求
3. **统一转换接口**: `make-filters-from-conditions` 作为条件到查询过滤器的统一转换层

### 8.3 三处入口的设计差异
| 设计考量 | 账户页 | 报表页 | 公式查询器 |
|---------|-------|-------|-----------|
| **性能优先** | 分页增量加载，响应快 | 全量计算，数据完整性优先 | 按需执行，公式级缓存 |
| **交互模式** | 实时筛选，即时反馈 | 条件组合后重算 | 配置式，版本触发 |
| **状态设计** | Class 组件本地状态 | Hook 响应式状态 | Ref + 版本号（避免重渲染） |
| **数据流转** | 单向数据流 | 响应式依赖追踪 | 显式版本控制 |

### 8.4 与命令面板的交互设计
1. **模态框互斥**: 模态框模式下通过 modalStack 实现互斥
2. **内联模式共存**: 内联补全与命令面板可同时触发，补全因失焦关闭
3. **边界清晰**: 无直接组件依赖，仅通过 modalStack 间接关联
4. **无键盘冲突**: 键盘事件自然冒泡，无特殊拦截逻辑
