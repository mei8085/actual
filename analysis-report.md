# 分类模板执行上下文构建分析报告

## 一、概述

分类模板（Category Template）在执行预算计算前，需要构建一个完整的执行上下文（`CategoryTemplateContext`）。该上下文整合了预算文件、历史交易和类别配置等多源数据，为模板计算提供统一的输入格式。

---

## 二、上下文构建模块架构

### 2.1 核心模块关系图

```
┌─────────────────────────────────────────────────────────────────────┐
│                    CategoryTemplateContext                          │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                    初始化阶段 (init)                          │  │
│  │  ┌──────────┐  ┌─────────────┐  ┌─────────────────┐         │  │
│  │  │  actions │  │    aql      │  │    statements   │         │  │
│  │  │ getSheet │  │ preferences │  │ getActiveSchedules │       │  │
│  │  └────┬─────┘  └──────┬──────┘  └────────┬────────┘         │  │
│  │       │               │                   │                  │  │
│  │       ▼               ▼                   ▼                  │  │
│  │  ┌───────────────────────────────────────────────────┐       │  │
│  │  │           数据验证阶段                            │       │  │
│  │  │  checkByAndScheduleAndSpend / checkPercentage   │       │  │
│  │  └───────────────────────────────────────────────────┘       │  │
│  │                           │                                  │  │
│  │                           ▼                                  │  │
│  │  ┌───────────────────────────────────────────────────┐       │  │
│  │  │           私有构造函数 (分类模板)                    │       │  │
│  │  │  checkLimit / checkSpend / checkGoal             │       │  │
│  │  └───────────────────────────────────────────────────┘       │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                              │                                      │
│                              ▼                                      │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                    执行阶段                                    │  │
│  │  runTemplatesForPriority / runRemainder / getValues         │  │
│  └───────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.2 模块职责划分

| 模块 | 职责 | 关键方法 | 数据源 |
|------|------|----------|--------|
| **actions** | 预算数据读写 | `getSheetValue`, `getSheetBoolean`, `isTrackingBudget` | Sheet存储 |
| **aql** | 偏好设置查询 | `aqlQuery` | 数据库 preferences 表 |
| **statements** | 调度信息获取 | `getActiveSchedules` | 数据库 schedules 表 |
| **db** | 类别信息查询 | `getCategories` | 数据库 categories 表 |
| **monthUtils** | 日期计算 | `sheetForMonth`, `subMonths`, `differenceInCalendarMonths` | 日期工具 |
| **currencies** | 货币处理 | `getCurrency` | 货币配置 |
| **schedule-template** | 调度模板执行 | `runSchedule` | 调度引擎 |

---

## 三、主链路数据流分析

### 3.1 执行链路时序图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           主链路数据流                                       │
│                                                                             │
│  applyTemplate / overwriteTemplate                                          │
│         │                                                                    │
│         ▼                                                                    │
│  ┌─────────────────┐                                                       │
│  │  getTemplates() │  ← 从数据库读取 categories.goal_def JSON               │
│  └────────┬────────┘                                                       │
│           │                                                                 │
│           ▼                                                                 │
│  ┌─────────────────────────────────────────────────────────────┐           │
│  │                    computeTemplates(month)                   │           │
│  │  ┌──────────────────────────────────────────────────────┐  │           │
│  │  │ Step 1: 初始化准备                                    │  │           │
│  │  │  • 获取 to-budget（本月可用预算）                     │  │           │
│  │  │  • 遍历所有类别，获取 budget-{id}, goal-{id}         │  │           │
│  │  │  • 过滤出需要处理的类别（budgeted===0 || force）     │  │           │
│  │  └──────────────────────────────────────────────────────┘  │           │
│  │                           │                                   │           │
│  │                           ▼                                   │           │
│  │  ┌──────────────────────────────────────────────────────┐  │           │
│  │  │ Step 2: CategoryTemplateContext.init()               │  │           │
│  │  │  • 获取 fromLastMonth (上月 leftover-{id})           │  │           │
│  │  │  • 获取 carryover 设置                              │  │           │
│  │  │  • 验证模板（checkByScheduleSpend, checkPercentage）│  │           │
│  │  │  • 获取偏好（hideDecimal, currencyCode）             │  │           │
│  │  │  • 分类模板（templates/remainder/goals）             │  │           │
│  │  │  • 计算 limitAmount                                 │  │           │
│  │  └──────────────────────────────────────────────────────┘  │           │
│  │                           │                                   │           │
│  │                           ▼                                   │           │
│  │  ┌──────────────────────────────────────────────────────┐  │           │
│  │  │ Step 3: 优先级执行循环                               │  │           │
│  │  │  for (priority : prioritiesSorted) {                 │  │           │
│  │  │    for (context : templateContexts) {               │  │           │
│  │  │      runTemplatesForPriority(priority, availBudget)  │  │           │
│  │  │      availBudget -= budget                           │  │           │
│  │  │    }                                                  │  │           │
│  │  │  }                                                    │  │           │
│  │  └──────────────────────────────────────────────────────┘  │           │
│  │                           │                                   │           │
│  │                           ▼                                   │           │
│  │  ┌──────────────────────────────────────────────────────┐  │           │
│  │  │ Step 4: distributeRemainder()                         │  │           │
│  │  │  • 按权重分配剩余可用预算                              │  │           │
│  │  │  • 调用各 context.runRemainder()                      │  │           │
│  │  └──────────────────────────────────────────────────────┘  │           │
│  │                           │                                   │           │
│  │                           ▼                                   │           │
│  │  ┌──────────────────────────────────────────────────────┐  │           │
│  │  │ Step 5: setBudgets() / setGoals()                    │  │           │
│  │  │  • 批量写入预算结果到数据库                           │  │           │
│  │  └──────────────────────────────────────────────────────┘  │           │
│  └─────────────────────────────────────────────────────────────┘           │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.2 computeTemplates 详细数据流

```typescript
async function computeTemplates(month, force, categoryTemplates, categories, skipAvailableClamp) {
  // === 数据采集阶段 ===
  const isTracking = isTrackingBudget();
  const allCategories = await getCategories();  // 获取所有类别（含 is_income）

  // 获取本月可用预算（全局 to-budget）
  let availBudget = await getSheetValue(
    monthUtils.sheetForMonth(month),
    `to-budget`
  );

  // === 类别迭代初始化 ===
  for (const category of allCategories) {
    const sheetName = monthUtils.sheetForMonth(month);

    // 读取当前类别的已预算金额和目标
    const budgeted = await getSheetValue(sheetName, `budget-${id}`);
    const existingGoal = await getSheetValue(sheetName, `goal-${id}`);

    if ((budgeted === 0 || force) && templates) {
      // === 调用 init 构建上下文 ===
      const templateContext = await CategoryTemplateContext.init(
        templates,
        category,
        month,
        budgeted,
        skipAvailableClamp
      );

      // 累加已预算金额到可用预算（排除纯目标类别）
      if (!templateContext.isGoalOnly()) {
        availBudget += budgeted;
      }

      // 累加超额限制金额
      availBudget += templateContext.getLimitExcess();

      // 收集所有优先级
      templateContext.getPriorities().forEach(p => prioritiesSet.add(p));
      templateContexts.push(templateContext);
    }
  }

  // === 优先级执行阶段 ===
  const priorities = new Int32Array([...prioritiesSet]).sort((a, b) => a - b);

  for (const priority of priorities) {
    const availStart = availBudget;  // 记录该优先级开始时的可用金额
    for (const templateContext of templateContexts) {
      const budget = await templateContext.runTemplatesForPriority(
        priority,
        availBudget,    // 当前可用预算（递减）
        availStart,    // 初始可用预算（不变）
      );
      availBudget -= budget;  // 扣除已分配预算
    }
  }

  // === 余数分配阶段 ===
  distributeRemainder(templateContexts, availBudget);

  return { contexts, errors, orphanGoals };
}
```

### 3.3 CategoryTemplateContext.init 数据流

```typescript
static async init(templates, category, month, budgeted, skipAvailableClamp) {
  // === 1. 上月结转数据获取 ===
  const lastMonthSheet = monthUtils.sheetForMonth(
    monthUtils.subMonths(month, 1)
  );

  // 读取上月剩余 + carryover 标志
  let fromLastMonth = await getSheetValue(lastMonthSheet, `leftover-${category.id}`);
  const carryover = await getSheetBoolean(lastMonthSheet, `carryover-${category.id}`);

  // === 2. 条件性重置 ===
  if ((fromLastMonth < 0 && !carryover) ||   // overspend no carryover
      category.is_income ||                     // tracking budget income
      (isTrackingBudget() && !carryover)) {    // tracking budget regular
    fromLastMonth = 0;
  }

  // === 3. 模板验证 ===
  await CategoryTemplateContext.checkByAndScheduleAndSpend(templates, month);
  await CategoryTemplateContext.checkPercentage(templates);

  // === 4. 偏好设置获取 ===
  const hideDecimal = await aqlQuery(...);     // hideFraction 偏好
  const currencyCode = await aqlQuery(...);     // defaultCurrencyCode 偏好

  // === 5. 构造上下文对象 ===
  return new CategoryTemplateContext(
    templates, category, month,
    fromLastMonth,      // 上月结转
    budgeted,           // 当前已预算
    currencyCode,
    hideDecimal,
    skipAvailableClamp
  );
}
```

### 3.4 runTemplatesForPriority 数据流

```typescript
async runTemplatesForPriority(priority, budgetAvail, availStart) {
  // === 1. 筛选该优先级的模板 ===
  const t = this.templates.filter(
    template => template.directive === 'template' && template.priority === priority
  );

  let available = budgetAvail || 0;
  let toBudget = 0;

  // === 2. 按类型逐个执行 ===
  for (const template of t) {
    let newBudget = 0;

    switch (template.type) {
      case 'simple':    newBudget = runSimple(template, this); break;
      case 'refill':    newBudget = runRefill(template, this); break;
      case 'copy':      newBudget = await runCopy(template, this); break;
      case 'periodic':  newBudget = runPeriodic(template, this); break;
      case 'spend':     newBudget = await runSpend(template, this); break;
      case 'percentage':newBudget = await runPercentage(template, availStart, this); break;
      case 'by':        /* 批量执行 */ break;
      case 'schedule':  /* 批量执行 */ break;
      case 'average':   newBudget = await runAverage(template, this); break;
    }

    available -= newBudget;
    toBudget += newBudget;
    perTemplateLocal.set(template, newBudget);
  }

  // === 3. 批量模板重分配（by/schedule） ===
  redistributeBatch(perTemplateLocal, t, 'by', weightFn);
  redistributeBatch(perTemplateLocal, t, 'schedule', weightFn);

  // === 4. 限制检查 ===
  if (this.limitCheck) {
    if (toBudget + this.toBudgetAmount + this.fromLastMonth >= this.limitAmount) {
      toBudget = this.limitAmount - this.toBudgetAmount - this.fromLastMonth;
      this.limitMet = true;
    }
  }

  // === 5. 舍入处理 ===
  if (this.hideDecimal) {
    toBudget = this.removeFraction(toBudget);
  }

  // === 6. 可用资金限制（非收入类别） ===
  if (priority > 0 && available < 0 && !this.category.is_income && !this.skipAvailableClamp) {
    toBudget = Math.max(0, toBudget + available);  // 裁剪到可用金额
  }

  this.toBudgetAmount += toBudget;
  return this.category.is_income ? -toBudget : toBudget;
}
```

### 3.5 runRemainder 数据流

```typescript
runRemainder(budgetAvail, perWeight) {
  if (this.remainder.length === 0) return 0;

  // === 1. 按权重计算初步分配额 ===
  let toBudget = Math.round(this.remainderWeight * perWeight);

  // === 2. 舍入处理 ===
  if (this.hideDecimal) {
    toBudget = this.removeFraction(toBudget);
  }

  // === 3. 不超过可用预算 ===
  if (toBudget > budgetAvail || budgetAvail - toBudget <= smallest) {
    toBudget = budgetAvail;
  }

  // === 4. 限制检查 ===
  if (this.limitCheck) {
    if (toBudget + this.toBudgetAmount + this.fromLastMonth >= this.limitAmount) {
      toBudget = this.limitAmount - this.toBudgetAmount - this.fromLastMonth;
      this.limitMet = true;
    }
  }

  // === 5. 按权重分配给各余数模板 ===
  if (toBudget > 0 && this.remainderWeight > 0) {
    let remaining = toBudget;
    for (const template of this.remainder) {
      const share = Math.round(toBudget * (template.weight / this.remainderWeight));
      this.perTemplateContribution.set(template, share);
      remaining -= share;
    }
  }

  this.toBudgetAmount += toBudget;
  return toBudget;
}

// distributeRemainder 在 goal-template.ts 中
function distributeRemainder(templateContexts, availBudget) {
  let remainderContexts = templateContexts.filter(c => c.hasRemainder());

  while (availBudget > 0 && remainderContexts.length > 0) {
    let remainderWeight = 0;
    remainderContexts.forEach(c => remainderWeight += c.getRemainderWeight());

    const perWeight = availBudget / remainderWeight;
    const beforePass = availBudget;

    remainderContexts.forEach(context => {
      availBudget -= context.runRemainder(availBudget, perWeight);
    });

    if (availBudget === beforePass) break;
    remainderContexts = templateContexts.filter(c => c.hasRemainder());
  }

  return availBudget;  // 返回未分配的剩余金额
}
```

---

## 四、各模板类型 Sheet 字段清单

### 4.1 字段总览

| 字段前缀 | 含义 | 示例 |
|----------|------|------|
| `budget-{categoryId}` | 某月某类别的预算金额 | `budget-cat-123` |
| `leftover-{categoryId}` | 某月某类别的结余金额 | `leftover-cat-123` |
| `sum-amount-{categoryId}` | 某月某类别的支出汇总（负数表示支出） | `sum-amount-cat-123` |
| `total-income` | 某月所有收入类别汇总 | `total-income` |
| `to-budget` | 某月可分配预算总额 | `to-budget` |
| `carryover-{categoryId}` | 某月某类别是否结转 | `carryover-cat-123` |
| `goal-{categoryId}` | 某月某类别的目标金额 | `goal-cat-123` |

### 4.2 各模板类型详细字段依赖

#### simple（固定金额模板）

| 读取字段 | Sheet 位置 | 必需 | 缺失处理 |
|----------|------------|------|----------|
| `limitAmount` | 由 `checkLimit()` 在构造时计算 | 否 | 使用模板内 `monthly` 字段 |

**计算逻辑**：
```typescript
if (template.monthly != null) {
  return amountToInteger(template.monthly);  // 直接使用模板定义
} else {
  return this.limitAmount - this.fromLastMonth;  // 补足到限制金额
}
```

**数据缺失处理**：默认降级
- 若无 `monthly` 且无 `limitAmount`：`fromLastMonth` 为 0 时返回 0
- 若 `limitAmount` 未设置：取决于 `limit` 模板定义

---

#### refill（补充模板）

| 读取字段 | Sheet 位置 | 必需 | 缺失处理 |
|----------|------------|------|----------|
| `limitAmount` | 由 `checkLimit()` 在构造时计算 | 否 | 默认降级为 0 |

**计算逻辑**：
```typescript
return this.limitAmount - this.fromLastMonth;
```

**数据缺失处理**：默认降级
- refill 本身**不校验** `limit` 是否存在
- 若无 `limit` 定义，`this.limitAmount` 保持为 0（由 `checkLimit` 中 `continue` 跳过）
- `runRefill` 返回 `0 - fromLastMonth`（即负的 fromLastMonth 或 0）
- 不会抛错，继续计算

**代码依据**：[category-template-context.ts:562-566](file:///d:/fz/0508-1/solo-dogfeeding/code/119-actual/packages/loot-core/src/server/budget/category-template-context.ts#L562-L566)
```typescript
} else {
  if (template.limit) {
    limitDef = template.limit;
  } else {
    continue; // may not have a limit defined in the template
  }
}
```

---

#### copy（复制模板）

| 读取字段 | Sheet 位置 | 必需 | 缺失处理 |
|----------|------------|------|----------|
| `budget-{categoryId}` | 上 N 个月的预算表 | 是 | 默认降级为 0 |

**计算逻辑**：
```typescript
const sheetName = monthUtils.sheetForMonth(
  monthUtils.subMonths(templateContext.month, template.lookBack)
);
return await getSheetValue(sheetName, `budget-${templateContext.category.id}`);
```

**数据缺失处理**：默认降级
- Sheet 单元格不存在时：`getSheetValue` 返回 0
- 即：复制上个月预算为 0

---

#### periodic（周期模板）

| 读取字段 | Sheet 位置 | 必需 | 缺失处理 |
|----------|------------|------|----------|
| 无 | 纯模板定义计算 | - | - |

**计算逻辑**：纯日期计算，根据 `template.amount` 和周期类型计算

**数据缺失处理**：不涉及 Sheet 数据
- 依赖模板定义完整性

---

#### spend（支出目标模板）

| 读取字段 | Sheet 位置 | 必需 | 缺失处理 |
|----------|------------|------|----------|
| `sum-amount-{categoryId}` | 从起始月到当前月遍历 | 是 | 默认降级为 0 |
| `leftover-{categoryId}` | 首月余额 | 是 | 默认降级为 0 |
| `budget-{categoryId}` | 中间月份预算 | 是 | 默认降级为 0 |

**计算逻辑**：
```typescript
// 遍历 fromMonth 到 当前月
for (let m = fromMonth; differenceInMonths(currentMonth, m) > 0; m = addMonths(m, 1)) {
  const sheetName = sheetForMonth(m);
  if (firstMonth) {
    const spent = await getSheetValue(sheetName, `sum-amount-${categoryId}`);
    const balance = await getSheetValue(sheetName, `leftover-${categoryId}`);
    alreadyBudgeted = balance - spent;  // 从余额反推预算
    firstMonth = false;
  } else {
    alreadyBudgeted += await getSheetValue(sheetName, `budget-${categoryId}`);
  }
}
return Math.round((target - alreadyBudgeted) / (numMonths + 1));
```

**数据缺失处理**：默认降级
- 任一 Sheet 值缺失：按 0 处理
- 可能导致 `alreadyBudgeted` 计算不准确

---

#### percentage（百分比模板）

| 读取字段 | Sheet 位置 | 必需 | 缺失处理 |
|----------|------------|------|----------|
| `total-income` | 当前月或上月收入汇总 | 是（条件） | 默认降级为 0 |
| `sum-amount-{incomeCategoryId}` | 特定收入类别汇总 | 是（条件） | 默认降级为 0 |
| 收入类别列表 | 数据库 categories 表 | 是 | 抛错终止 |

**计算逻辑**：
```typescript
if (cat === 'all income') {
  monthlyIncome = await getSheetValue(sheetName, `total-income`);
} else if (cat === 'available funds') {
  monthlyIncome = availableFunds;  // 来自参数传入
} else {
  // 查找收入类别
  const incomeCat = await db.getCategories().find(
    c => c.is_income && (c.id === cat || c.name.toLowerCase() === cat)
  );
  if (!incomeCat) {
    throw new Error(`Income category not found`);  // 抛错
  }
  monthlyIncome = await getSheetValue(sheetName, `sum-amount-${incomeCat.id}`);
}
return Math.max(0, Math.round(monthlyIncome * (percent / 100)));
```

**数据缺失处理**：
- `total-income` / `sum-amount-{id}` 缺失：默认降级为 0
- 收入类别不存在：**抛错终止**

---

#### by（目标日期模板）

| 读取字段 | Sheet 位置 | 必需 | 缺失处理 |
|----------|------------|------|----------|
| 无 | 纯模板定义计算 | - | - |

**计算逻辑**：纯模板定义计算
```typescript
// 基于 template.month, template.amount, template.repeat 计算
// 使用 this.fromLastMonth（来自 init 阶段）
const toBudget = Math.round(
  (totalNeeded - templateContext.fromLastMonth) / (shortNumMonths + 1)
);
```

**数据缺失处理**：不涉及 Sheet 数据
- `fromLastMonth` 在 init 阶段获取，缺失时为 0

---

#### schedule（调度模板）

| 读取字段 | Sheet 位置 | 必需 | 缺失处理 |
|----------|------------|------|----------|
| `fromLastMonth` | 上月 `leftover-{categoryId}` | 是 | 默认降级为 0 |
| `budgeted`（累计） | 当前模板累计预算 | 是 | 默认降级为 0 |
| 活动调度列表 | 数据库 schedules 表 | 是 | 抛错终止 |

**计算逻辑**：
```typescript
const budgeted = this.fromLastMonth + toBudget;  // 已累计预算
const ret = await runSchedule(
  t, this.month, budgeted, remainder,
  this.fromLastMonth, toBudget, [], this.category, this.currency
);
newBudget = ret.to_budget - toBudget;  // 本次增量
```

**异常路径可核实事实链**：

### createScheduleList 分支写入 facts

**代码位置**：[schedule-template.ts:137-208](file:///d:/fz/0508-1/solo-dogfeeding/code/119-actual/packages/loot-core/src/server/budget/schedule-template.ts#L137-L208)

**Fact 1**：`if (num_months < 0)` 分支执行 `errors.push()` → [schedule-template.ts:139](file:///d:/fz/0508-1/solo-dogfeeding/code/119-actual/packages/loot-core/src/server/budget/schedule-template.ts#L139)

**Fact 2**：`if (completed)` 分支执行 `errors.push()` → [schedule-template.ts:202-204](file:///d:/fz/0508-1/solo-dogfeeding/code/119-actual/packages/loot-core/src/server/budget/schedule-template.ts#L202-L204)

**Fact 3**：`return { t: t.filter(c => c.completed === 0), errors }` → [schedule-template.ts:208](file:///d:/fz/0508-1/solo-dogfeeding/code/119-actual/packages/loot-core/src/server/budget/schedule-template.ts#L208)
- 返回值包含 `errors` 字段
- 返回值包含过滤后的 `t` 数组（已完成调度的条目被移除）

---

### runSchedule 传递 facts

**代码位置**：[schedule-template.ts:305-324](file:///d:/fz/0508-1/solo-dogfeeding/code/119-actual/packages/loot-core/src/server/budget/schedule-template.ts#L305-L324)

**Fact 4**：`runSchedule` 调用 `createScheduleList()` → [schedule-template.ts:318-323](file:///d:/fz/0508-1/solo-dogfeeding/code/119-actual/packages/loot-core/src/server/budget/schedule-template.ts#L318-L323)

**Fact 5**：`errors = errors.concat(t.errors)` → [schedule-template.ts:324](file:///d:/fz/0508-1/solo-dogfeeding/code/119-actual/packages/loot-core/src/server/budget/schedule-template.ts#L324)
- `runSchedule` 的 `errors` 参数被就地修改，追加了 `createScheduleList` 返回的 `t.errors`

**Fact 6**：`runSchedule` 返回值 **不包含** `errors` 字段，仅返回计算结果 → [schedule-template.ts](file:///d:/fz/0508-1/solo-dogfeeding/code/119-actual/packages/loot-core/src/server/budget/schedule-template.ts)

---

### CategoryTemplateContext 调用点消费 facts

**代码位置**：[category-template-context.ts:210-225](file:///d:/fz/0508-1/solo-dogfeeding/code/119-actual/packages/loot-core/src/server/budget/category-template-context.ts#L210-L225)

**Fact 7**：`const ret = await runSchedule(...)` → [category-template-context.ts:210](file:///d:/fz/0508-1/solo-dogfeeding/code/119-actual/packages/loot-core/src/server/budget/category-template-context.ts#L210)
- 调用时传入 `[]` 作为 `errors` 参数

**Fact 8**：`ret.to_budget` 被读取并使用 → [category-template-context.ts:219](file:///d:/fz/0508-1/solo-dogfeeding/code/119-actual/packages/loot-core/src/server/budget/category-template-context.ts#L219)
- `newBudget = ret.to_budget - toBudget`

**Fact 9**：`ret.errors` 未被任何语句读取或赋值给任何变量

**可核实结论**：
- `createScheduleList` 在两个条件分支写入 `errors` 数组
- `runSchedule` 将 `createScheduleList` 返回的 `errors` 合并到自己的 `errors` 参数中（修改了传入的数组引用）
- `runSchedule` 的返回值对象中 **不包含** `errors` 字段
- `CategoryTemplateContext` 调用点只消费了 `ret.to_budget`，未消费 `ret.errors`
- `CategoryTemplateContext` 调用 `runSchedule` 时传入 `[]`，意味着 `errors` 参数的修改操作作用在这个空数组上，该数组在函数返回后无引用

---

### 硬抛错 facts（独立于上述事实链）

**代码位置**：[category-template-context.ts:476-487](file:///d:/fz/0508-1/solo-dogfeeding/code/119-actual/packages/loot-core/src/server/budget/category-template-context.ts#L476-L487)

**Fact 10**：`if (!scheduleNames.includes(t.name.trim()))` 时执行 `throw new Error()` → [category-template-context.ts:482-484](file:///d:/fz/0508-1/solo-dogfeeding/code/119-actual/packages/loot-core/src/server/budget/category-template-context.ts#L482-L484)

**Fact 11**：`if (priority !== currentPriority)` 时执行 `throw new Error()` → [category-template-context.ts:485-487](file:///d:/fz/0508-1/solo-dogfeeding/code/119-actual/packages/loot-core/src/server/budget/category-template-context.ts#L485-L487)

**Fact 12**：上述 `throw` 发生在 `CategoryTemplateContext.init()` 内部，`init` 被 `computeTemplates` 调用时外层有 `try-catch` 包裹 → [goal-template.ts](file:///d:/fz/0508-1/solo-dogfeeding/code/119-actual/packages/loot-core/src/server/budget/goal-template.ts) 中 `computeTemplates` 调用点

**可核实结论**：硬抛错在 `init` 阶段被抛出，被 `computeTemplates` 的 `catch` 捕获并添加到返回的 `errors` 列表。

---

#### average（平均模板）

| 读取字段 | Sheet 位置 | 必需 | 缺失处理 |
|----------|------------|------|----------|
| `sum-amount-{categoryId}` | 前 N 个月的支出汇总 | 是 | 默认降级为 0 |

**计算逻辑**：
```typescript
let sum = 0;
for (let i = 1; i <= template.numMonths; i++) {
  const sheetName = monthUtils.sheetForMonth(
    monthUtils.subMonths(templateContext.month, i)
  );
  sum += await getSheetValue(sheetName, `sum-amount-${templateContext.category.id}`);
}
let average = -(sum / template.numMonths);  // 取反（支出为负）

// 可选调整
if (template.adjustmentType === 'percent') {
  average *= (1 + template.adjustment / 100);
} else if (template.adjustmentType === 'fixed') {
  average += template.adjustment;
}
return Math.round(average);
```

**数据缺失处理**：默认降级
- 任一月份 `sum-amount` 缺失：该月按 0 处理
- 可能导致平均值偏差

---

#### remainder（余数模板）

| 读取字段 | Sheet 位置 | 必需 | 缺失处理 |
|----------|------------|------|----------|
| `limitAmount` | 由 `checkLimit()` 在构造时计算 | 否 | 默认降级处理 |

**计算逻辑**：
```typescript
// 权重分配
let toBudget = Math.round(this.remainderWeight * perWeight);

// 限制检查
if (this.limitCheck) {
  if (toBudget + this.toBudgetAmount + this.fromLastMonth >= this.limitAmount) {
    toBudget = this.limitAmount - this.toBudgetAmount - this.fromLastMonth;
    this.limitMet = true;
  }
}
```

**数据缺失处理**：默认降级
- 无 `limitAmount` 时：不执行限制检查

---

### 4.3 字段依赖矩阵

| 模板类型 | budget-{id} | leftover-{id} | sum-amount-{id} | total-income | to-budget | goal-{id} | 外部依赖 |
|----------|-------------|---------------|-----------------|--------------|-----------|-----------|----------|
| **simple** | - | - | - | - | - | - | - |
| **refill** | - | read:fromLastMonth | - | - | - | - | - |
| **copy** | read (历史月) | - | - | - | - | - | - |
| **periodic** | - | - | - | - | - | - | - |
| **spend** | read (中间月) | read (首月) | read (首月) | - | - | - | - |
| **percentage** | - | - | read (收入类别) | read | - | - | db.getCategories |
| **by** | - | read:fromLastMonth | - | - | - | - | - |
| **schedule** | - | read:fromLastMonth | - | - | - | - | db.schedules |
| **average** | - | - | read (多个月) | - | - | - | - |
| **remainder** | - | - | - | - | - | - | - |

---

## 五、优先级执行与余数分配传递逻辑

### 5.1 优先级执行数据传递

```
availBudget (可用预算)
     │
     ├─── 优先级 0 执行 ───┬─── runTemplatesForPriority(0, availBudget, availStart)
     │                     │
     │                     ▼
     │               toBudget (该优先级分配额)
     │                     │
     │                     ▼
     │               availBudget -= toBudget  (递减)
     │
     ├─── 优先级 1 执行 ───┬─── runTemplatesForPriority(1, availBudget, availStart)
     │                     │
     │                     ▼
     │               toBudget (该优先级分配额)
     │                     │
     │                     ▼
     │               availBudget -= toBudget  (递减)
     │
     └─── ... 继续递减 ...
```

**关键传递参数**：
- `budgetAvail`：当前可用预算（随分配递减）
- `availStart`：该优先级开始时的可用预算（不变，用于百分比模板计算）

### 5.2 批量模板重分配逻辑

`by` 和 `schedule` 模板执行后，其总预算需要按权重分配给同类型的多个模板：

```typescript
// by 模板权重：按需分配额（back-interpolated）
redistributeBatch(perTemplateLocal, t, 'by', template => {
  return Math.max(0, byPerTemplate?.get(template) ?? 0);
});

// schedule 模板权重：按实际月均额
redistributeBatch(perTemplateLocal, t, 'schedule', template => {
  const monthly = schedulePerTemplate?.get(template.name.trim()) ?? 0;
  return Math.max(0, monthly);
});

function redistributeBatch(perTemplateLocal, templates, type, weightOf) {
  const siblings = templates.filter(t => t.type === type);
  if (siblings.length < 2) return;

  let total = siblings.reduce((sum, s) => sum + (perTemplateLocal.get(s) ?? 0), 0);

  const totalWeight = siblings.reduce((sum, s) => sum + weightOf(s), 0);

  if (totalWeight <= 0) {
    // 无可用权重时：平均分配
    siblings.forEach((sibling, i) => {
      const share = Math.round(total / siblings.length);
      perTemplateLocal.set(sibling, share);
    });
  } else {
    // 按权重分配
    siblings.forEach((sibling) => {
      const share = Math.round(total * weightOf(sibling) / totalWeight);
      perTemplateLocal.set(sibling, share);
    });
  }
}
```

### 5.3 余数分配数据传递

```
availBudget (优先级执行后的剩余)
     │
     ▼
distributeRemainder(templateContexts, availBudget)
     │
     ├─── 计算总余数权重 ───→ remainderWeight
     │
     ├─── 计算每权重配额 ───→ perWeight = availBudget / remainderWeight
     │
     └─── 迭代分配 ───┬─── context1.runRemainder(availBudget, perWeight)
                      │
                      ├─── context2.runRemainder(availBudget, perWeight)
                      │
                      └─── ...
```

**迭代终止条件**：
1. `availBudget === 0`（预算分完）
2. `remainderContexts.length === 0`（无剩余模板）
3. `availBudget === beforePass`（本轮无进展）

---

## 六、数据缺失处理策略汇总

### 6.1 默认降级清单

| 数据项 | 默认值 | 适用模板 | 影响 |
|--------|--------|----------|------|
| Sheet 单元格不存在 | `0` | `copy`, `spend`, `average`, `percentage` | 计算结果可能偏低 |
| Sheet 布尔值不存在 | `false` | `carryover` | 按无结转处理 |
| 偏好设置不存在 | `false` | `hideDecimal` | 使用精确计算 |
| 货币代码为空 | 系统默认货币 | 金额转换 | 使用 USD 等 |

### 6.2 抛错终止清单

| 验证项 | 错误条件 | 适用模板 | 错误消息示例 |
|--------|----------|----------|--------------|
| 调度名称不存在 | `schedule.name` 不在活动调度列表 | `schedule` | `Schedule XXX does not exist` |
| 收入类别不存在 | `percentage.category` 非有效收入类别 | `percentage` | `Income category "XXX" not found` |
| 周期性周期无效 | `periodic.period` 非 day/week/month/year | `periodic` | `Unrecognized periodic period` |
| 优先级不一致 | `by` 和 `schedule` 混合不同优先级 | `by`, `schedule` | `Schedule and By templates must be the same priority level` |
| 目标日期已过 | `by`/`spend` 目标月已过且无 repeat | `by`, `spend` | `Target month has passed, remove or update the target month` |
| 多目标模板 | 同一类别多个 `#goal` | `goal` | `Only one #goal is allowed per category` |
| 多支出模板 | 同一类别多个 `spend` | `spend` | `Only one spend template is allowed per category` |
| 多限制定义 | 同一类别多个 `up to` | `limit` | `Only one \`up to\` allowed per category` |
| 周限制缺起始日 | `limit.period === 'weekly'` 但无 `start` | `limit` | `Weekly limit requires a start date (YYYY-MM-DD)` |
| 无效限制周期 | `limit.period` 非 daily/weekly/monthly | `limit` | `Invalid limit period. Check template syntax` |

**remainder 数量行为**：代码中**不存在**对多个 remainder 模板的任何校验或警告机制，完全允许多个 remainder 模板共存。

**代码依据**：[category-template-context.ts:436-438](file:///d:/fz/0508-1/solo-dogfeeding/code/119-actual/packages/loot-core/src/server/budget/category-template-context.ts#L436-L438) 对 remainder 仅做累加权重处理：
```typescript
} else if (t.directive === 'template' && t.type === 'remainder') {
  this.remainder.push(t);
  this.remainderWeight += t.weight;
}
```

**实际行为**：多个 remainder 模板会被全部收集到 `this.remainder` 数组中，权重累加后在余数分配阶段按各自权重比例分配剩余预算。

### 6.3 降级 vs 抛错决策树

```
数据可用性检查
      │
      ▼
┌─────────────────┐
│ 数据项类型？     │
└────────┬────────┘
         │
    ┌────┴────┬─────────────┐
    │         │             │
 Sheet单元格  偏好设置    外部引用
    │         │             │
    ▼         ▼             ▼
┌─────────┐ ┌─────────┐ ┌───────────┐
│ 默认 0  │ │ 默认false│ │ 查找失败  │
│ / false │ │ / 空    │ │ 抛错终止  │
└─────────┘ └─────────┘ └───────────┘
```

---

## 七、总结

### 7.1 上下文构建核心要点

1. **多源数据整合**：Sheet 存储（历史数据）+ 数据库（配置数据）+ 模板定义（用户配置）
2. **分层验证机制**：初始化前验证 → 构造时验证 → 执行时检查
3. **优雅降级策略**：数据缺失时使用合理默认值，关键验证失败时记录错误并跳过
4. **优先级执行模型**：按优先级有序执行，余数分配作为最后阶段

### 7.2 架构优势

| 特性 | 优势 |
|------|------|
| **单一入口** | `CategoryTemplateContext.init()` 统一初始化 |
| **职责分离** | 数据获取、验证、计算清晰分离 |
| **可扩展性** | 新增模板类型只需添加处理函数 |
| **容错性** | 局部失败不影响整体执行 |

### 7.3 潜在改进点

1. **缓存优化**：当前每次初始化都重新查询偏好设置，可引入缓存机制
2. **批量查询**：可合并多个 Sheet 查询以提升性能
3. **异步并行**：部分数据获取可并行执行

---

## 附录：关键文件位置

| 文件 | 路径 | 说明 |
|------|------|------|
| `category-template-context.ts` | `packages/loot-core/src/server/budget/` | 核心上下文类 |
| `goal-template.ts` | `packages/loot-core/src/server/budget/` | 模板应用入口 |
| `actions.ts` | `packages/loot-core/src/server/budget/` | 数据读写操作 |
| `templates.ts` | `packages/loot-core/src/types/models/` | 模板类型定义 |
