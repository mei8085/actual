# Envelope 与 Tracking 预算模式分析报告

## 目录

1. [预算模式切换完整链路](#1-预算模式切换完整链路)
2. [budgetType 的持久化位置与运行时状态来源](#2-budgettype-的持久化位置与运行时状态来源)
3. [导入流程对 budgetType 的处理机制](#3-导入流程对-budgettype-的处理机制)
4. [两种模式的核心数据结构差异](#4-两种模式的核心数据结构差异)
5. [电子表格计算规则差异](#5-电子表格计算规则差异)
6. [同步行为差异](#6-同步行为差异)

---

## 1. 预算模式切换完整链路

### 1.1 前端触发与偏好写入

**位置**: `packages/desktop-client/src/components/settings/BudgetTypeSettings.tsx:14-28`

```typescript
export function BudgetTypeSettings() {
  // 1. 从 Redux store 读取当前预算类型，默认为 'envelope'
  const [budgetType = 'envelope', setBudgetType] = useSyncedPref('budgetType');
  const [isLoading, setIsLoading] = useState(false);

  async function onSwitchType() {
    setIsLoading(true);
    try {
      // 2. 切换预算类型
      const newBudgetType = budgetType === 'envelope' ? 'tracking' : 'envelope';
      setBudgetType(newBudgetType);

      // 3. 重置预算缓存，触发后端重建
      await send('reset-budget-cache');
    } finally {
      setIsLoading(false);
    }
  }
  // ...
}
```

**useSyncedPref hook 工作原理**:

**位置**: `packages/desktop-client/src/hooks/useSyncedPref.ts:1-29`

```typescript
export function useSyncedPref<K extends keyof SyncedPrefs>(
  prefName: K,
): [SyncedPrefs[K], SetSyncedPrefAction<K>] {
  const dispatch = useDispatch();
  const setPref = useCallback<SetSyncedPrefAction<K>>(
    value => {
      // 触发 saveSyncedPrefs action，最终调用后端 API 'preferences/save'
      void dispatch(saveSyncedPrefs({ prefs: { [prefName]: value } }));
    },
    [prefName, dispatch],
  );
  // 从 Redux store 读取值
  const pref = useSelector(state => state.prefs.synced[prefName]);

  return [pref, setPref];
}
```

### 1.2 后端偏好持久化

**位置**: `packages/loot-core/src/server/preferences/app.ts:38-53`

```typescript
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

  // 写入 preferences 数据库表
  await db.update('preferences', {
    id,
    value,
  });
}
```

### 1.3 同步消息处理

**位置**: `packages/loot-core/src/server/sync/index.ts:367-369`

当同步消息中检测到 budgetType 变更时，立即触发预算类型切换：

```typescript
// 同步过程中对 budgetType 的特殊处理
if (dataset === 'preferences' && row === 'budgetType') {
  void setBudgetType(value);  // 立即触发预算类型切换
}
```

### 1.4 预算重建流程

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

### 1.5 预算缓存重置

**位置**: `packages/loot-core/src/server/budgetfiles/app.ts:146-150`

```typescript
async function resetBudgetCache() {
  // 重新加载用户预算到电子表格
  await sheet.loadUserBudgets(db);
  // 强制重新计算所有单元格
  sheet.get().recomputeAll();
  // 等待电子表格计算完成
  await sheet.waitOnSpreadsheet();
}
```

---

## 2. budgetType 的持久化位置与运行时状态来源

### 2.1 三层存储结构

| 存储位置 | 类型 | 用途 | 代码位置 |
|---------|------|------|---------|
| **`preferences` 数据库表** | `SyncedPrefs` | 跨设备同步的主存储 | `packages/loot-core/src/server/budget/actions.ts:49-56` |
| **`sheet.meta().budgetType`** | 运行时内存 | 电子表格计算时的状态来源 | `packages/loot-core/src/server/budget/base.ts:15-18` |
| **`metadata.json` 文件** | `MetadataPrefs` | **不再存储 budgetType**（历史遗留） | `packages/loot-core/src/types/prefs.ts:66-77` |

### 2.2 数据库表存储（主存储）

**位置**: `packages/loot-core/src/server/budget/actions.ts:49-56`

```typescript
export function isTrackingBudget(): boolean {
  // 从 preferences 表读取 budgetType
  const budgetType = db.firstSync<Pick<db.DbPreference, 'value'>>(
    `SELECT value FROM preferences WHERE id = ?`,
    ['budgetType'],
  );
  // 默认值为 'envelope'
  const val = budgetType ? budgetType.value : 'envelope';
  return val === 'tracking';
}
```

**表选择逻辑**:

**位置**: `packages/loot-core/src/server/budget/actions.ts:45-47`

```typescript
function getBudgetTable(): BudgetTable {
  // Tracking 模式使用 reflect_budgets 表
  // Envelope 模式使用 zero_budgets 表
  return isTrackingBudget() ? 'reflect_budgets' : 'zero_budgets';
}
```

### 2.3 运行时状态（电子表格元数据）

**位置**: `packages/loot-core/src/server/budget/base.ts:15-18`

```typescript
export function getBudgetType() {
  // 从电子表格元数据读取当前预算类型
  const meta = sheet.get().meta();
  // 默认值为 'envelope'
  return meta.budgetType || 'envelope';
}
```

### 2.4 预算加载时的初始化

**位置**: `packages/loot-core/src/server/budgetfiles/app.ts:600-607`

```typescript
// 加载预算时，从数据库读取并初始化电子表格元数据
const { value: budgetType = 'envelope' } =
  (await db.first<Pick<db.DbPreference, 'value'>>(
    'SELECT value from preferences WHERE id = ?',
    ['budgetType'],
  )) ?? {};

// 设置到电子表格元数据中，作为运行时状态
sheet.get().meta().budgetType = budgetType as prefs.BudgetType;
await budget.createAllBudgets();
```

### 2.5 MetadataPrefs 类型定义（不含 budgetType）

**位置**: `packages/loot-core/src/types/prefs.ts:66-77`

```typescript
/**
 * Preferences that are stored in the `metadata.json` file along with the
 * core database.
 */
export type MetadataPrefs = Partial<{
  budgetName: string;      // 预算名称
  id: string;              // 预算ID
  lastUploaded: string;    // 上次上传时间
  cloudFileId: string;     // 云文件ID
  groupId: string;         // 同步组ID
  encryptKeyId: string;    // 加密密钥ID
  lastSyncedTimestamp: string;  // 上次同步时间戳
  resetClock: boolean;     // 重置时钟标志
  lastScheduleRun: string; // 上次计划运行时间
  userId: string;          // 用户ID（未使用）
}>;
```

**重要结论**: budgetType **不在** `metadata.json` 中存储，而是完全存储在 SQLite 数据库的 `preferences` 表中。这是在 `1723665565000_prefs.js` 迁移中完成的变更。

---

## 3. 导入流程对 budgetType 的处理机制

### 3.1 prefs 与 preferences 的核心区别

| 概念 | 存储位置 | 类型 | 包含 budgetType? |
|-----|---------|------|-----------------|
| **`prefs` (MetadataPrefs)** | `metadata.json` 文件 | 文件级元数据 | ❌ 不包含 |
| **`preferences`** | SQLite 数据库表 | 可同步的偏好设置 | ✅ 包含 |

### 3.2 导入模式下的同步限制

**位置**: `packages/loot-core/src/server/sync/index.ts:231-250`

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

        // ⚠️ 重要：导入模式下禁止设置 prefs (MetadataPrefs)
        if (dataset === 'prefs') {
          throw new Error('Cannot set prefs while importing');
        }
        // ✅ 注意：preferences 表（包含 budgetType）不受此限制，可以正常写入
      }
    }
  });
}
```

### 3.3 导入流程的完整处理

#### 3.3.1 启动导入

**位置**: `packages/loot-core/src/server/api.ts:300-316`

```typescript
handlers['api/start-import'] = async function ({ budgetName }) {
  // 1. 关闭当前预算
  await handlers['close-budget']();

  // 2. 创建新预算（使用默认的 'envelope' 类型）
  await handlers['create-budget']({ budgetName, avoidUpload: true });

  // 3. 清空默认的支出分类（导入时会重新创建）
  db.runQuery('DELETE FROM categories WHERE is_income = 0');
  db.runQuery('DELETE FROM category_groups WHERE is_income = 0');

  // 4. 切换到导入同步模式
  setSyncingMode('import');

  connection.send('start-import');
  IMPORT_MODE = true;
};
```

#### 3.3.2 Actual 格式导入（特殊处理）

**位置**: `packages/loot-core/src/server/importers/actual.ts:1-49`

```typescript
export async function importActual(_filepath: string, buffer: Buffer) {
  // 导入 Actual 文件是特殊情况：直接复制数据库文件
  await handlers['close-budget']();

  let id;
  try {
    // 1. 直接从导入的 zip 包中提取并复制 db.sqlite 和 metadata.json
    ({ id } = await cloudStorage.importBuffer(
      { cloudFileId: null, groupId: null },
      buffer,
    ));
  } catch (e) {
    // ...错误处理
  }

  // 2. 删除缓存数据，强制重新计算
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

  // 3. 重新加载预算（此时会从数据库读取原有的 budgetType）
  await handlers['load-budget']({ id });
  await handlers['get-budget-bounds']();
  await waitOnSpreadsheet();
  // ...
}
```

#### 3.3.3 完成导入

**位置**: `packages/loot-core/src/server/api.ts:318-339`

```typescript
handlers['api/finish-import'] = async function () {
  checkFileOpen();

  sheet.get().markCacheDirty();

  // 关闭并重新加载预算，确保所有状态正确初始化
  const { id } = prefs.getPrefs();
  await handlers['close-budget']();
  await handlers['load-budget']({ id });  // 此步骤会重新读取数据库中的 budgetType

  await handlers['get-budget-bounds']();
  await sheet.waitOnSpreadsheet();

  await cloudStorage.upload().catch(err => {
    logger.warn('cloudStorage.upload failed during finish-import', err);
  });

  connection.send('finish-import');
  IMPORT_MODE = false;
};
```

### 3.4 导入流程关键结论

1. **YNAB 格式导入**:
   - 创建新预算时使用默认的 `'envelope'` 类型
   - 导入过程中不改变预算类型
   - 导入完成后用户可手动切换

2. **Actual 格式导入**:
   - 直接复制原数据库文件
   - 保留原有的 `budgetType` 设置
   - 重新加载时从数据库读取正确值

3. **同步限制**:
   - 仅禁止设置 `prefs` (MetadataPrefs，存储在 metadata.json)
   - `preferences` 表（包含 budgetType）可以正常写入
   - 限制原因：避免导入时修改文件级元数据（如预算名称、云同步ID等）

---

## 4. 两种模式的核心数据结构差异

### 4.1 预算表结构对比

| 模式 | 表名 | 月度汇总表 | 主要字段 |
|------|------|------------|---------|
| **Envelope** | `zero_budgets` | `zero_budget_months` | `amount`, `carryover`, `goal`, `long_goal` |
| **Tracking** | `reflect_budgets` | 无 | `amount`, `carryover`, `goal`, `long_goal` |

### 4.2 Envelope 模式特有 - 月度缓冲

**位置**: `packages/loot-core/src/server/budget/actions.ts:174-186`

```typescript
// zero_budget_months 表存储月度级别的缓冲金额
export async function setBuffer(month: string, amount: unknown): Promise<void> {
  const existing = db.firstSync<Pick<db.DbZeroBudgetMonth, 'id'>>(
    `SELECT id FROM zero_budget_months WHERE id = ?`,
    [month],
  );
  if (existing) {
    return db.update('zero_budget_months', {
      id: existing.id,
      buffered: amount,
    });
  }
  return db.insert('zero_budget_months', { id: month, buffered: amount });
}
```

### 4.3 预算操作路由

**位置**: `packages/loot-core/src/server/budget/actions.ts:106-148`

```typescript
export function getBudget({
  category,
  month,
}: {
  category: string;
  month: string;
}): number {
  const table = getBudgetTable();  // 根据预算类型选择表
  const existing = db.firstSync<db.DbZeroBudget | db.DbReflectBudget>(
    `SELECT * FROM ${table} WHERE month = ? AND category = ?`,
    [dbMonth(month), category],  // month 格式转换为 YYYYMM
  );
  return existing ? existing.amount || 0 : 0;
}

export function setBudget({
  category,
  month,
  amount,
}: {
  category: CategoryEntity['id'];
  month: string;
  amount: unknown;
}): Promise<void> {
  amount = safeNumber(typeof amount === 'number' ? amount : 0);
  const table = getBudgetTable();  // 根据预算类型选择表

  const existing = db.firstSync<...>(
    `SELECT id FROM ${table} WHERE month = ? AND category = ?`,
    [dbMonth(month), category],
  );
  if (existing) {
    return db.update(table, { id: existing.id, amount });
  }
  return db.insert(table, {
    id: `${dbMonth(month)}-${category}`,
    month: dbMonth(month),
    category,
    amount,
  });
}
```

---

## 5. 电子表格计算规则差异

### 5.1 Envelope 模式计算规则

**位置**: `packages/loot-core/src/server/budget/envelope.ts`

#### 5.1.1 核心计算公式

```typescript
// 剩余金额计算 - 考虑结转
sheet.get().createDynamic(sheetName, `leftover-${cat.id}`, {
  initialValue: 0,
  dependencies: [
    `budget-${cat.id}`,           // 本月预算金额
    `sum-amount-${cat.id}`,       // 本月支出金额
    `${prevSheetName}!carryover-${cat.id}`,  // 上月结转标志
    `${prevSheetName}!leftover-${cat.id}`,   // 上月剩余
    `${prevSheetName}!leftover-pos-${cat.id}`,  // 上月正数剩余
  ],
  run: (budgeted, spent, prevCarryover, prevLeftover, prevLeftoverPos) => {
    return safeNumber(
      number(budgeted) +
        number(spent) +  // spent 是负数
        (prevCarryover ? number(prevLeftover) : number(prevLeftoverPos)),
    );
  },
});

// 正数剩余（用于非结转时传递到下月）
sheet.get().createDynamic(sheetName, 'leftover-pos-' + cat.id, {
  initialValue: 0,
  dependencies: [`leftover-${cat.id}`],
  run: leftover => {
    return leftover < 0 ? 0 : leftover;
  },
});
```

#### 5.1.2 月度汇总计算

```typescript
// 待预算金额 = 上月结转 + 本月收入 - 已分配 - 缓冲金额
sheet.get().createDynamic(sheetName, 'to-budget', {
  initialValue: 0,
  dependencies: [
    'total-income',      // 本月总收入
    'from-last-month',   // 上月结转金额
    'last-month-overspent',  // 上月超支
    'total-budgeted',    // 本月已分配总额
    'buffered-selected', // 缓冲金额
  ],
  run: (income, fromLastMonth, lastOverspent, totalBudgeted, buffered) => {
    return safeNumber(
      number(income) +
        number(fromLastMonth) +
        number(lastOverspent) +
        number(totalBudgeted) -
        number(buffered),
    );
  },
});

// 上月结转金额 = 上月 to-budget + 上月 buffered
sheet.get().createDynamic(sheetName, 'from-last-month', {
  initialValue: 0,
  dependencies: [
    `${prevSheetName}!to-budget`,
    `${prevSheetName}!buffered-selected`,
  ],
  run: (toBudget, buffered) =>
    safeNumber(number(toBudget) + number(buffered)),
});
```

#### 5.1.3 空白月份处理

Envelope 模式有"空白月份"概念，用于处理第一个月之前的预算结转：

```typescript
function createBlankMonth(categories, sheetName, months) {
  sheet.get().createStatic(sheetName, 'is-blank', true);
  sheet.get().createStatic(sheetName, 'to-budget', 0);
  sheet.get().createStatic(sheetName, 'buffered', 0);

  categories.forEach(cat => createBlankCategory(cat, months));
}

function createBlankCategory(cat, months) {
  if (months.length > 0) {
    const sheetName = getBlankSheet(months);
    sheet.get().createStatic(sheetName, `carryover-${cat.id}`, false);
    sheet.get().createStatic(sheetName, `leftover-${cat.id}`, 0);
    sheet.get().createStatic(sheetName, `leftover-pos-${cat.id}`, 0);
  }
}
```

### 5.2 Tracking 模式计算规则

**位置**: `packages/loot-core/src/server/budget/tracking.ts`

#### 5.2.1 核心计算公式

```typescript
// 剩余金额计算 - 简化逻辑
sheet.get().createDynamic(sheetName, `leftover-${cat.id}`, {
  dependencies: [
    `budget-${cat.id}`,
    `sum-amount-${cat.id}`,
    `${prevSheetName}!carryover-${cat.id}`,
    `${prevSheetName}!leftover-${cat.id}`,
  ],
  run: (budgeted, sumAmount, prevCarryover, prevLeftover) => {
    if (cat.is_income) {
      // 收入分类：预算 - 实际 + 结转
      return safeNumber(
        number(budgeted) -
          number(sumAmount) +
          (prevCarryover ? number(prevLeftover) : 0),
      );
    }
    // 支出分类：预算 + 支出（负数） + 结转
    return safeNumber(
      number(budgeted) +
        number(sumAmount) +
        (prevCarryover ? number(prevLeftover) : 0),
    );
  },
});

// Tracking 特有：含结转的支出计算（用于进度显示）
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

#### 5.2.2 月度汇总计算

```typescript
// 计划储蓄 = 收入预算总额 - 支出预算总额
sheet.get().createDynamic(sheetName, 'total-saved', {
  initialValue: 0,
  dependencies: ['total-budget-income', 'total-budgeted'],
  run: (income, budgeted) => {
    return income - budgeted;
  },
});

// 实际储蓄 = 实际收入 - 实际支出
sheet.get().createDynamic(sheetName, 'real-saved', {
  initialValue: 0,
  dependencies: ['total-income', 'total-spent'],
  run: (income, spent) => {
    return safeNumber(income - -spent);
  },
});
```

#### 5.2.3 隐藏分类过滤

Tracking 模式在汇总时排除隐藏分类：

```typescript
sheet.get().createDynamic(sheetName, 'group-sum-amount-' + group.id, {
  dependencies: group.categories
    .filter(cat => !cat.hidden)  // 过滤隐藏分类
    .map(cat => `sum-amount-${cat.id}`),
  run: sumAmounts,
});
```

### 5.3 预算创建时的分支逻辑

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

  // Envelope 模式：创建空白月份
  if (budgetType === 'envelope') {
    envelopeBudget.createBudget(meta, categories, months);
  }

  months.forEach(month => {
    if (!meta.createdMonths.has(month)) {
      const prevMonth = monthUtils.prevMonth(month);
      const { start, end } = monthUtils.bounds(month);
      const sheetName = monthUtils.sheetForMonth(month);
      const prevSheetName = monthUtils.sheetForMonth(prevMonth);

      // 创建分类单元格
      categories.forEach(cat => {
        createCategory(cat, sheetName, prevSheetName, start, end);
      });

      // 创建分类组汇总
      groups.forEach(group => {
        if (budgetType === 'envelope') {
          envelopeBudget.createCategoryGroup(group, sheetName);
        } else {
          trackingBudget.createCategoryGroup(group, sheetName);
        }
      });

      // 创建月度汇总
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

## 6. 同步行为差异

### 6.1 预算变更触发机制

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

        // 月度缓冲变更（仅 Envelope 模式）
        if (table === 'zero_budget_months') {
          handleBudgetMonthChange(newValue);
        }
        // 分类预算变更（两种模式共用）
        else if (table === 'zero_budgets' || table === 'reflect_budgets') {
          handleBudgetChange(newValue);
        }
        // 分类变更
        else if (table === 'categories') {
          if (budgetType === 'envelope') {
            envelopeBudget.handleCategoryChange(
              createdMonths,
              oldValue,
              newValue,
            );
          } else {
            trackingBudget.handleCategoryChange(
              createdMonths,
              oldValue,
              newValue,
            );
          }
        }
        // 分类组变更
        else if (table === 'category_groups') {
          if (budgetType === 'envelope') {
            envelopeBudget.handleCategoryGroupChange(
              createdMonths,
              oldValue,
              newValue,
            );
          } else {
            trackingBudget.handleCategoryGroupChange(
              createdMonths,
              oldValue,
              newValue,
            );
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

### 6.2 预算月度变更处理

**位置**: `packages/loot-core/src/server/budget/base.ts:124-127`

```typescript
function handleBudgetMonthChange(budget) {
  const sheetName = monthUtils.sheetForMonth(budget.id);
  // 更新 buffered 单元格值
  sheet.get().set(`${sheetName}!buffered`, budget.buffered);
}
```

### 6.3 分类预算变更处理

**位置**: `packages/loot-core/src/server/budget/base.ts:129-146`

```typescript
function handleBudgetChange(budget) {
  if (budget.category) {
    const sheetName = monthUtils.sheetForMonth(budget.month.toString());
    // 更新预算金额
    sheet.get().set(`${sheetName}!budget-${budget.category}`, budget.amount || 0);
    // 更新结转标志
    sheet.get().set(
      `${sheetName}!carryover-${budget.category}`,
      budget.carryover === 1 ? true : false,
    );
    // 更新目标
    sheet.get().set(`${sheetName}!goal-${budget.category}`, budget.goal);
    sheet.get().set(`${sheetName}!long-goal-${budget.category}`, budget.long_goal);
  }
}
```

### 6.4 同步消息编码

**位置**: `packages/loot-core/src/server/sync/encoder.ts`

同步时，预算数据与其他数据的处理方式相同，都是通过 CRDT 消息同步。预算类型的切换通过 `preferences` 表的变更触发特殊处理逻辑。

---

## 7. 历史迁移说明

### 7.1 偏好系统重构（迁移 1723665565000）

**位置**: `packages/loot-core/migrations/1723665565000_prefs.js`

```javascript
const SYNCED_PREF_KEYS = [
  'budgetType',  // 从 metadata.json 迁移到数据库表
  // ... 其他可同步偏好
];

export default async function runMigration(db, { fs, fileId }) {
  await db.execQuery(`
    CREATE TABLE preferences
       (id TEXT PRIMARY KEY,
        value TEXT);
  `);

  try {
    const budgetDir = fs.getBudgetDir(fileId);
    const fullpath = fs.join(budgetDir, 'metadata.json');
    const prefs = JSON.parse(await fs.readFile(fullpath));

    await Promise.all(
      Object.keys(prefs).map(async key => {
        if (!SYNCED_PREF_KEYS.find(...)) {
          return;
        }
        // 将 budgetType 从 metadata.json 迁移到 preferences 表
        db.runQuery('INSERT INTO preferences (id, value) VALUES (?, ?)', [
          key,
          String(prefs[key]),
        ]);
      }),
    );
  } catch {
    // 忽略错误
  }
}
```

### 7.2 预算类型命名标准化（迁移 1745425408000）

**位置**: `packages/loot-core/migrations/1745425408000_update_budgetType_pref.sql`

```sql
BEGIN TRANSACTION;

UPDATE preferences
SET value = CASE
  WHEN id = 'budgetType' AND value = 'report' THEN 'tracking'  // report -> tracking
  ELSE 'envelope'  // rollover -> envelope（其他值也重置为 envelope）
END
WHERE id = 'budgetType';

COMMIT;
```

---

## 8. 关键设计决策总结

| 决策点 | Envelope 模式 | Tracking 模式 | 设计意图 |
|-------|-------------|--------------|---------|
| **余额结转** | 完整结转，支持超支处理，区分正负余额 | 简化结转，每月重置 | 匹配不同的预算理念 |
| **收入处理** | 收入不参与预算，仅影响 to-budget 金额 | 收入可设预算，计算储蓄率 | Tracking 模式强调预测与规划 |
| **缓冲机制** | 支持 Hold/Buffer 功能，存储在 zero_budget_months 表 | 不支持 | Envelope 模式更注重流动性管理 |
| **隐藏分类** | 隐藏分类仍参与汇总计算 | 隐藏分类排除在汇总外 | Tracking 模式提供更灵活的视图选项 |
| **数据库表** | `zero_budgets` + `zero_budget_months` | `reflect_budgets` | 历史演进结果，未来可能统一 |
| **存储位置** | `preferences` 数据库表 | `preferences` 数据库表 | 支持跨设备同步 |

---

## 9. 常见问题解答

### Q: 切换预算模式会丢失数据吗？
**A**: 不会丢失原始交易数据，但会：
- 删除所有电子表格预算单元格（动态生成，可恢复）
- 保留预算金额和结转设置（存储在数据库表中）
- 重新创建预算时会重新读取并应用这些值

### Q: 为什么两种模式使用不同的数据库表？
**A**: 这是历史演进的结果。早期版本有完全不同的两种预算实现，统一后保留了原有表结构。`zero_budget_months` 表是 Envelope 模式特有的，用于存储月度缓冲金额。

### Q: 导入预算文件时 budgetType 如何处理？
**A**:
- **YNAB 导入**: 创建新预算时使用默认的 `'envelope'`，导入完成后用户可手动切换
- **Actual 格式导入**: 直接复制原数据库，保留原有的 `budgetType` 设置
- **关键限制**: 导入过程中仅禁止设置 `prefs` (metadata.json)，`preferences` 表可以正常写入

### Q: 同步过程中 budgetType 变更会立即生效吗？
**A**: 是的。同步消息处理器在检测到 `preferences` 表中 `budgetType` 变更时，会立即调用 `setBudgetType()` 触发完整的预算重建流程。

### Q: metadata.json 中还存储 budgetType 吗？
**A**: 不存储。在迁移 1723665565000_prefs.js 中，budgetType 已从 metadata.json 移动到 preferences 数据库表中，支持跨设备同步。
