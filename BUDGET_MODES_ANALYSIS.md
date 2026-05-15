# Envelope 与 Tracking 预算模式分析报告

## 1. 概述

Actual Budget 支持两种核心预算模式：**Envelope（信封预算）** 和 **Tracking（跟踪预算）**。本文档详细分析这两种模式在数据形态、记账规则、跨模块同步行为上的根本差异。

## 2. 数据形态差异

### 2.1 存储表结构

| 模式 | 预算表 | 月度汇总表 | 核心字段 |
|------|--------|------------|----------|
| **Envelope** | `zero_budgets` | `zero_budget_months` | `amount`, `carryover`, `buffered` |
| **Tracking** | `reflect_budgets` | 无 | `amount`, `carryover`, `goal`, `long_goal` |

**表选择逻辑** (`packages/loot-core/src/server/budget/actions.ts:45-56`)：
```typescript
function getBudgetTable(): BudgetTable {
  return isTrackingBudget() ? 'reflect_budgets' : 'zero_budgets';
}

export function isTrackingBudget(): boolean {
  const budgetType = db.firstSync<Pick<db.DbPreference, 'value'>>(
    `SELECT value FROM preferences WHERE id = ?`,
    ['budgetType'],
  );
  const val = budgetType ? budgetType.value : 'envelope';
  return val === 'tracking';
}
```

### 2.2 偏好存储位置

预算类型存储在两个位置：

1. **`preferences` 数据库表** - 用于同步和运行时查询
   - `id = 'budgetType'`
   - `value = 'envelope' | 'tracking'`

2. **`metadata.json` 文件** - 预算文件级元数据
   - 位于预算目录根目录
   - 包含 `budgetType` 字段

## 3. 预算模式切换完整链路

### 3.1 前端触发

**位置**: `packages/desktop-client/src/components/settings/BudgetTypeSettings.tsx`

```typescript
async function onSwitchType() {
  const newBudgetType = budgetType === 'envelope' ? 'tracking' : 'envelope';
  setBudgetType(newBudgetType);
  
  // 重置预算缓存，确保服务器端预算系统重新计算
  await send('reset-budget-cache');
}
```

### 3.2 后端处理

**位置**: `packages/loot-core/src/server/budget/base.ts:321-346`

```typescript
export async function setType(type) {
  const meta = sheet.get().meta();
  if (type === meta.budgetType) {
    return;
  }

  meta.budgetType = type;
  meta.createdMonths = new Set();  // 重置已创建月份集合

  // 强制删除所有预算相关单元格
  const nodes = sheet.get().getNodes();
  db.transaction(() => {
    for (const name of nodes.keys()) {
      const [sheetName, cellName] = name.split('!');
      if (sheetName.match(/^budget\d+/)) {
        sheet.get().deleteCell(sheetName, cellName);
      }
    }
  });

  // 重新创建所有预算
  sheet.get().startCacheBarrier();
  void sheet.loadUserBudgets(db);
  const bounds = await createAllBudgets();
  sheet.get().endCacheBarrier();

  return bounds;
}
```

### 3.3 预算缓存重置

**位置**: `packages/loot-core/src/server/budgetfiles/app.ts:146-150`

```typescript
async function resetBudgetCache() {
  await sheet.loadUserBudgets(db);
  sheet.get().recomputeAll();
  await sheet.waitOnSpreadsheet();
}
```

## 4. 电子表格计算规则差异

### 4.1 Envelope 预算模式

**位置**: `packages/loot-core/src/server/budget/envelope.ts`

#### 核心计算字段

| 字段 | 计算逻辑 |
|------|----------|
| **to-budget** | 可分配预算 = 上月结转 + 本月收入 - 已分配 - 缓冲金额 |
| **from-last-month** | 上月结转金额 = 上月 to-budget + 上月 buffered |
| **buffered** | 缓冲金额（Hold 功能） |
| **total-budgeted** | 总已分配 = Σ 各类别预算金额（取负值） |
| **last-month-overspent** | 上月超支（未结转的负数余额） |
| **leftover** | 类别余额 = 预算金额 + 支出金额 + 结转金额 |
| **leftover-pos** | 类别正数余额（仅用于结转） |

#### 创建分类时的特殊逻辑

```typescript
// Envelope 模式为每个分类创建：
sheet.get().createStatic(sheetName, `budget-${cat.id}`, 0);
sheet.get().createStatic(sheetName, `carryover-${cat.id}`, false);

// 动态计算 leftover
sheet.get().createDynamic(sheetName, `leftover-${cat.id}`, {
  dependencies: [
    `budget-${cat.id}`,
    `sum-amount-${cat.id}`,
    `${prevSheetName}!carryover-${cat.id}`,
    `${prevSheetName}!leftover-${cat.id}`,
    `${prevSheetName}!leftover-pos-${cat.id}`,
  ],
  run: (budgeted, spent, prevCarryover, prevLeftover, prevLeftoverPos) => {
    return safeNumber(
      number(budgeted) +
        number(spent) +
        (prevCarryover ? number(prevLeftover) : number(prevLeftoverPos)),
    );
  },
});
```

#### 空月份特殊处理

Envelope 模式有 "空白月份" 概念，用于处理第一个月之前的预算结转：

```typescript
function createBlankMonth(categories, sheetName, months) {
  sheet.get().createStatic(sheetName, 'is-blank', true);
  sheet.get().createStatic(sheetName, 'to-budget', 0);
  sheet.get().createStatic(sheetName, 'buffered', 0);

  categories.forEach(cat => createBlankCategory(cat, months));
}
```

### 4.2 Tracking 预算模式

**位置**: `packages/loot-core/src/server/budget/tracking.ts`

#### 核心计算字段

| 字段 | 计算逻辑 |
|------|----------|
| **total-saved** | 计划储蓄 = 收入预算总额 - 支出预算总额 |
| **real-saved** | 实际储蓄 = 实际收入 - 实际支出 |
| **total-budgeted** | 总已分配（不含负值转换） |
| **spent-with-carryover** | 含结转的支出（用于显示进度） |

#### 创建分类时的特殊逻辑

```typescript
// Tracking 模式为每个分类创建：
sheet.get().createStatic(sheetName, `budget-${cat.id}`, 0);
sheet.get().createStatic(sheetName, `carryover-${cat.id}`, false);

// 动态计算 leftover（逻辑不同）
sheet.get().createDynamic(sheetName, `leftover-${cat.id}`, {
  dependencies: [
    `budget-${cat.id}`,
    `sum-amount-${cat.id}`,
    `${prevSheetName}!carryover-${cat.id}`,
    `${prevSheetName}!leftover-${cat.id}`,
  ],
  run: (budgeted, sumAmount, prevCarryover, prevLeftover) => {
    if (cat.is_income) {
      return safeNumber(
        number(budgeted) -
          number(sumAmount) +
          (prevCarryover ? number(prevLeftover) : 0),
      );
    }
    return safeNumber(
      number(budgeted) +
        number(sumAmount) +
        (prevCarryover ? number(prevLeftover) : 0),
    );
  },
});

// Tracking 特有字段：spent-with-carryover
sheet.get().createDynamic(sheetName, `spent-with-carryover-${cat.id}`, {
  dependencies: [
    `budget-${cat.id}`,
    `sum-amount-${cat.id}`,
    `carryover-${cat.id}`,
  ],
  run: (budgeted, sumAmount, carryover) => {
    return carryover
      ? Math.max(0, safeNumber(number(budgeted) + number(sumAmount)))
      : sumAmount;
  },
});
```

#### 分类组汇总差异

```typescript
// Tracking 模式在汇总时过滤隐藏分类
sheet.get().createDynamic(sheetName, 'group-sum-amount-' + group.id, {
  dependencies: group.categories
    .filter(cat => !cat.hidden)
    .map(cat => `sum-amount-${cat.id}`),
  run: sumAmounts,
});
```

## 5. 跨模块同步行为

### 5.1 同步消息处理

**位置**: `packages/loot-core/src/server/sync/index.ts:368-369`

```typescript
// 同步过程中对 budgetType 的特殊处理
if (dataset === 'preferences' && row === 'budgetType') {
  void setBudgetType(value);  // 立即触发预算类型切换
}
```

### 5.2 同步监听器

预算变更通过 `triggerBudgetChanges` 函数触发电子表格重新计算：

**位置**: `packages/loot-core/src/server/budget/base.ts:148-211`

```typescript
export function triggerBudgetChanges(oldValues, newValues) {
  const { createdMonths = new Set() } = sheet.get().meta();
  const budgetType = getBudgetType();
  sheet.startTransaction();

  try {
    newValues.forEach((items, table) => {
      const old = oldValues.get(table);

      items.forEach(newValue => {
        const oldValue = old && old.get(newValue.id);

        if (table === 'zero_budget_months') {
          handleBudgetMonthChange(newValue);
        } else if (table === 'zero_budgets' || table === 'reflect_budgets') {
          handleBudgetChange(newValue);
        } else if (table === 'categories') {
          if (budgetType === 'envelope') {
            envelopeBudget.handleCategoryChange(createdMonths, oldValue, newValue);
          } else {
            trackingBudget.handleCategoryChange(createdMonths, oldValue, newValue);
          }
        } else if (table === 'category_groups') {
          if (budgetType === 'envelope') {
            envelopeBudget.handleCategoryGroupChange(createdMonths, oldValue, newValue);
          } else {
            trackingBudget.handleCategoryGroupChange(createdMonths, oldValue, newValue);
          }
        }
        // ... 其他表处理
      });
    });
  } finally {
    sheet.endTransaction();
  }
}
```

### 5.3 导入模式下的特殊处理

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

        if (dataset === 'prefs') {
          throw new Error('Cannot set prefs while importing');
        }
      }
    }
  });
}
```

**注意**: 导入模式下**禁止设置偏好**（包括预算类型），导入完成后用户需手动切换预算模式。

## 6. 操作行为差异

### 6.1 预算操作路由

**位置**: `packages/loot-core/src/server/budget/actions.ts`

| 操作 | Envelope 模式 | Tracking 模式 |
|------|--------------|---------------|
| **设置预算** | 使用 `zero_budgets` 表 | 使用 `reflect_budgets` 表 |
| **Hold 缓冲** | 支持 `setBuffer` 操作 | 不支持 |
| **覆盖超支** | 支持 `coverOverspending` | 不适用（每月重置） |
| **复制上月** | 仅复制支出分类预算 | 复制所有分类（包括收入） |
| **设置为零** | 仅支出分类设零 | 所有分类设零 |
| **结转开关** | 完整结转逻辑 | 简化结转逻辑 |

### 6.2 复制上月预算差异

```typescript
// Envelope 模式：跳过收入分类
if (prevBudget.is_income === 1 && !isTrackingBudget()) {
  return;
}

// Tracking 模式：复制所有分类（包括收入）
```

### 6.3 结转重置差异

**Envelope 模式重置收入结转**:
```typescript
export async function resetIncomeCarryover({ month }) {
  const table = getBudgetTable();
  const categories = await db.all(
    'SELECT * FROM v_categories WHERE is_income = 1 AND tombstone = 0',
  );

  await batchMessages(async () => {
    for (const category of categories) {
      await setCarryover(table, category.id, dbMonth(month).toString(), false);
    }
  });
}
```

## 7. 前端组件差异

### 7.1 组件路由

**位置**: `packages/desktop-client/src/components/budget/index.tsx`

```typescript
// 根据预算类型动态加载不同组件
const BudgetComponents = budgetType === 'envelope' 
  ? EnvelopeBudgetComponents 
  : TrackingBudgetComponents;
```

### 7.2 Envelope 模式特有组件

- `ToBudget` - 待分配金额显示
- `HoldMenu` - 缓冲金额设置菜单
- `BalanceMovementMenu` - 余额移动菜单
- `BudgetSummary` - 预算汇总（含上月结转）

### 7.3 Tracking 模式特有组件

- `BudgetTotal` - 预算总额显示
- `ExpenseProgress` - 支出进度条
- 简化的分类余额显示（无结转警告）

## 8. 迁移与兼容性

### 8.1 历史迁移

**位置**: `packages/loot-core/migrations/1745425408000_update_budgetType_pref.sql`

```sql
BEGIN TRANSACTION;

UPDATE preferences
SET value = CASE 
  WHEN id = 'budgetType' AND value = 'report' THEN 'tracking' 
  ELSE 'envelope' 
END
WHERE id = 'budgetType';

COMMIT;
```

**说明**: 早期版本使用 `'report'` 作为跟踪预算的标识，现已统一为 `'tracking'`。

### 8.2 电子表格单元格迁移

**位置**: `packages/loot-core/migrations/1567699552727_budget.sql`

- 清理非预算相关的电子表格单元格
- 标准化单元格命名格式（下划线转连字符）
- 修复 carryover 单元格的表达式格式

## 9. 关键设计决策总结

| 决策点 | Envelope 模式 | Tracking 模式 | 设计意图 |
|--------|--------------|---------------|----------|
| **余额结转** | 完整结转，支持超支处理 | 简化结转，每月重置 | 匹配不同的预算理念 |
| **收入处理** | 收入不参与预算，仅影响 to-budget | 收入可设预算，计算储蓄率 | Tracking 模式强调预测 |
| **缓冲机制** | 支持 Hold/Buffer 功能 | 不支持 | Envelope 模式更注重流动性管理 |
| **隐藏分类** | 隐藏分类仍参与汇总计算 | 隐藏分类排除在汇总外 | Tracking 模式更灵活 |
| **数据库表** | `zero_budgets` + `zero_budget_months` | `reflect_budgets` | 历史演进结果 |

## 10. 常见问题

### Q: 切换预算模式会丢失数据吗？
A: 不会丢失原始交易数据，但会：
- 删除所有电子表格预算单元格
- 重置预算金额（需要重新设置）
- 重置结转标志

### Q: 为什么两种模式使用不同的数据库表？
A: 这是历史演进的结果。早期版本有完全不同的两种预算实现，统一后保留了原有表结构。未来可能会统一为单表结构。

### Q: 导入预算文件时预算类型如何处理？
A: 导入过程中不允许设置预算类型偏好。导入完成后，用户需要在设置页面手动切换预算模式，系统会自动重新构建所有预算计算。

### Q: 同步过程中预算类型变更会立即生效吗？
A: 是的，同步消息处理器在检测到 `budgetType` 变更时会立即调用 `setBudgetType()` 触发完整的预算重建流程。
