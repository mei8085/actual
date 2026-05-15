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

## 三、上下文初始化流程详解

### 3.1 初始化入口 (`CategoryTemplateContext.init`)

```typescript
static async init(
  templates: Template[],
  category: CategoryEntity,
  month: string,
  budgeted: number,
  skipAvailableClamp: boolean = false,
)
```

**输入参数说明**：

| 参数 | 类型 | 说明 |
|------|------|------|
| `templates` | `Template[]` | 该类别的所有模板定义（包括普通模板和目标） |
| `category` | `CategoryEntity` | 类别实体，包含 `id`, `name`, `group`, `is_income` |
| `month` | `string` | 预算月份（格式：YYYY-MM） |
| `budgeted` | `number` | 当前已预算金额（整数表示，单位：最小货币单位） |
| `skipAvailableClamp` | `boolean` | 是否跳过可用资金限制（用于预览计算） |

### 3.2 数据采集阶段

#### 3.2.1 获取上月结转数据

```typescript
const lastMonthSheet = monthUtils.sheetForMonth(
  monthUtils.subMonths(month, 1),
);
let fromLastMonth = await getSheetValue(
  lastMonthSheet,
  `leftover-${category.id}`,
);
const carryover = await getSheetBoolean(
  lastMonthSheet,
  `carryover-${category.id}`,
);
```

**数据处理逻辑**：

| 条件 | 处理方式 |
|------|----------|
| 上月余额 < 0 且无 carryover | `fromLastMonth = 0` |
| 收入类别（追踪预算） | `fromLastMonth = 0` |
| 追踪预算普通类别且无 carryover | `fromLastMonth = 0` |

#### 3.2.2 获取偏好设置

```typescript
const hideDecimal = await aqlQuery(
  q('preferences').filter({ id: 'hideFraction' }).select('*'),
);
const currencyPref = await aqlQuery(
  q('preferences').filter({ id: 'defaultCurrencyCode' }).select('*'),
);
```

**默认值处理**：
- `hideDecimal`: 默认 `false`
- `currencyCode`: 默认空字符串（后续使用系统默认货币）

#### 3.2.3 模板验证

```typescript
await CategoryTemplateContext.checkByAndScheduleAndSpend(templates, month);
await CategoryTemplateContext.checkPercentage(templates);
```

**验证内容**：

| 验证方法 | 验证对象 | 验证规则 |
|----------|----------|----------|
| `checkByAndScheduleAndSpend` | `by`/`schedule`/`spend` 模板 | 调度名称存在性、优先级一致性、目标日期有效性 |
| `checkPercentage` | `percentage` 模板 | 收入类别名称/ID有效性 |

### 3.3 私有构造函数处理

#### 3.3.1 模板分类

```typescript
templates.forEach(t => {
  if (t.directive === 'template' && t.type !== 'remainder' && t.type !== 'limit') {
    this.templates.push(t);
    if (t.priority !== null) this.priorities.add(t.priority);
  } else if (t.directive === 'template' && t.type === 'remainder') {
    this.remainder.push(t);
    this.remainderWeight += t.weight;
  } else if (t.directive === 'goal' && t.type === 'goal') {
    this.goals.push(t);
  }
});
```

**模板分类结果**：

| 分类 | 模板类型 | 存储属性 |
|------|----------|----------|
| 普通模板 | `simple`, `copy`, `periodic`, `spend`, `percentage`, `by`, `schedule`, `average`, `refill` | `templates[]` + `priorities<Set>` |
| 余数模板 | `remainder` | `remainder[]` + `remainderWeight` |
| 目标模板 | `goal` | `goals[]` |

#### 3.3.2 限制检查 (`checkLimit`)

```typescript
private checkLimit(templates: Template[])
```

**限制计算逻辑**：

| 限制周期 | 计算方式 |
|----------|----------|
| `daily` | `limit.amount * 当月天数` |
| `weekly` | `limit.amount * 当月周数`（需指定 `start` 日期） |
| `monthly` | `limit.amount` |

**超限处理**：
- `hold=true`: 保持金额，不分配新预算
- `hold=false`: 计算超额部分，从预算中扣除

---

## 四、输入数据组合机制

### 4.1 数据来源汇总

| 数据项 | 获取方式 | 存储位置 |
|--------|----------|----------|
| 上月剩余 | `getSheetValue(lastMonth, leftover-{catId})` | Sheet |
| Carryover 设置 | `getSheetBoolean(lastMonth, carryover-{catId})` | Sheet |
| 当前已预算 | `getSheetValue(currentMonth, budget-{catId})` | Sheet |
| 当前目标 | `getSheetValue(currentMonth, goal-{catId})` | Sheet |
| 隐藏小数偏好 | `aqlQuery(preferences.hideFraction)` | DB |
| 默认货币 | `aqlQuery(preferences.defaultCurrencyCode)` | DB |
| 活动调度 | `getActiveSchedules()` | DB |
| 收入类别列表 | `db.getCategories().filter(is_income)` | DB |

### 4.2 模板执行输入格式

```typescript
// 执行上下文核心状态
interface ExecutionContext {
  category: CategoryEntity;      // 类别信息
  month: string;                  // 预算月份
  fromLastMonth: number;          // 上月结转金额
  previouslyBudgeted: number;     // 当前已预算金额
  currency: Currency;             // 货币配置
  hideDecimal: boolean;           // 是否隐藏小数
  priorities: Set<number>;        // 优先级集合
  templates: Template[];          // 普通模板列表
  remainder: RemainderTemplate[]; // 余数模板列表
  goals: GoalTemplate[];          // 目标模板列表
  remainderWeight: number;        // 余数权重总和
  limitAmount: number;            // 限制金额
  limitCheck: boolean;            // 是否启用限制检查
  limitMet: boolean;              // 是否已达到限制
  limitExcess: number;            // 超额金额
}
```

---

## 五、数据缺失降级处理

### 5.1 降级处理策略

| 数据项 | 默认值 | 降级场景 | 影响范围 |
|--------|--------|----------|----------|
| 上月剩余 | `0` | Sheet 单元格不存在 | 预算计算起点 |
| Carryover 设置 | `false` | Sheet 单元格不存在 | 负余额处理 |
| 隐藏小数偏好 | `false` | 偏好未设置 | 金额显示精度 |
| 默认货币 | 系统默认 | 偏好未设置 | 货币转换 |
| 调度名称 | 抛出错误 | 调度不存在 | 模板执行失败 |
| 收入类别 | 抛出错误 | 类别不存在 | 模板执行失败 |

### 5.2 异常处理路径

```
模板执行异常处理流程
       │
       ▼
┌─────────────────┐
│ 模板验证阶段    │
└────────┬────────┘
         │
    验证失败?
         │
    ┌────┴────┐
    │         │
   Yes        No
    │         │
    ▼         ▼
┌─────────┐  ┌─────────────────┐
│ 抛出错误 │  │ 继续执行        │
│ 记录日志 │  │ 进入计算阶段    │
└─────────┘  └────────┬────────┘
                      │
                      ▼
              ┌─────────────────┐
              │ 计算阶段        │
              └────────┬────────┘
                       │
                  数据缺失?
                       │
                  ┌────┴────┐
                  │         │
                 Yes        No
                  │         │
                  ▼         ▼
          ┌─────────────┐  ┌─────────────────┐
          │ 使用默认值   │  │ 正常计算        │
          │ (如: 0)     │  └─────────────────┘
          └──────┬──────┘
                 │
                 ▼
          ┌─────────────────┐
          │ 完成计算        │
          └─────────────────┘
```

### 5.3 关键降级代码示例

#### 5.3.1 Sheet 值缺失处理

```typescript
export async function getSheetValue(
  sheetName: string,
  cell: string,
): Promise<number> {
  const node = sheet.getCell(sheetName, cell);
  return safeNumber(typeof node.value === 'number' ? node.value : 0);
}

export async function getSheetBoolean(
  sheetName: string,
  cell: string,
): Promise<boolean> {
  const node = sheet.getCell(sheetName, cell);
  return typeof node.value === 'boolean' ? node.value : false;
}
```

#### 5.3.2 偏好设置缺失处理

```typescript
const currencyCode =
  currencyPref.data.length > 0 ? currencyPref.data[0].value : '';
// 后续通过 getCurrency(currencyCode) 获取货币，空字符串返回系统默认
```

#### 5.3.3 模板验证失败处理

```typescript
try {
  const templateContext = await CategoryTemplateContext.init(
    templates,
    category,
    month,
    budgeted,
    skipAvailableClamp,
  );
  templateContexts.push(templateContext);
} catch (e) {
  errors.push(`${category.name}: ${e.message}`);
}
```

---

## 六、模板类型与数据依赖

### 6.1 模板类型数据依赖表

| 模板类型 | 依赖数据 | 获取方式 |
|----------|----------|----------|
| `simple` | 固定金额 / 限制 | 模板定义 |
| `refill` | 限制金额 | `checkLimit()` 计算 |
| `copy` | 历史预算 | `getSheetValue(budget-{catId})` |
| `periodic` | 周期配置 | 模板定义 |
| `spend` | 历史支出/预算 | `getSheetValue(sum-amount/budget-{catId})` |
| `percentage` | 收入/可用资金 | `getSheetValue(total-income/sum-amount-{catId})` |
| `by` | 目标日期 | 模板定义 |
| `schedule` | 调度数据 | `runSchedule()` |
| `average` | 历史支出 | `getSheetValue(sum-amount-{catId})` |
| `remainder` | 可用资金 | 外部传入 |
| `goal` | 目标金额 | 模板定义 |
| `limit` | 限制配置 | 模板定义 |

### 6.2 数据依赖关系图

```
                    ┌─────────────────┐
                    │  用户配置数据    │
                    │  (模板定义)     │
                    └────────┬────────┘
                             │
         ┌───────────────────┼───────────────────┐
         │                   │                   │
         ▼                   ▼                   ▼
┌───────────────┐   ┌───────────────┐   ┌───────────────┐
│   静态配置     │   │   动态计算     │   │   外部依赖     │
│ simple/refill │   │ by/spend      │   │ schedule      │
│ periodic      │   │ percentage    │   │ average       │
│ goal/limit    │   │ average       │   │ copy          │
└───────────────┘   └───────────────┘   └───────────────┘
         │                   │                   │
         └───────────────────┼───────────────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │  Sheet 存储层   │
                    │  (预算/支出数据)│
                    └─────────────────┘
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

## 八、完整数据流（按调用顺序）

### 8.1 入口：`computeTemplates`

```typescript
async function computeTemplates(
  month: string,
  force: boolean,
  categoryTemplates: Record<CategoryEntity['id'], Template[]>,
  categories: CategoryEntity[] = [],
  skipAvailableClamp: boolean = false
)
```

| 阶段 | 输入 | 输出 | 关键中间变量 | 操作说明 |
|------|------|------|--------------|----------|
| **初始获取** | `month` | `isTracking`, `availBudget` | `availBudget` | 调用 `getSheetValue(sheetForMonth(month), 'to-budget')` 获取当月可分配预算 |
| **类别循环** | `categories`, `categoryTemplates` | `templateContexts`, `errors`, `orphanGoals` | `templateContexts`, `errors` | 对每个类别执行初始化，捕获错误 |
| **条件判断** | `budgeted`, `force` | - | `budgeted` | 仅处理未预算(`budgeted=0`)或强制更新的类别 |
| **上下文初始化** | `templates`, `category`, `month`, `budgeted` | `templateContext` | `availBudget` | 调用 `CategoryTemplateContext.init()` 初始化，`availBudget += budgeted + getLimitExcess()` |
| **优先级执行** | `priorities`, `templateContexts` | `templateContexts` | `availBudget` | 按优先级排序后逐个调用 `runTemplatesForPriority()`, `availBudget -= budget` |
| **余数分配** | `templateContexts`, `availBudget` | `templateContexts` | `availBudget` | 调用 `distributeRemainder()` 分配剩余预算 |
| **返回结果** | `templateContexts`, `errors`, `orphanGoals` | - | - | - |

---

### 8.2 `CategoryTemplateContext.init()` 初始化流程

```typescript
static async init(
  templates: Template[],
  category: CategoryEntity,
  month: string,
  budgeted: number,
  skipAvailableClamp: boolean = false
)
```

| 阶段 | 输入 | 输出 | 关键中间变量 | 操作说明 |
|------|------|------|--------------|----------|
| **上月数据获取** | `month`, `category.id` | `fromLastMonth`, `carryover` | `lastMonthSheet` | `lastMonthSheet = sheetForMonth(subMonths(month, 1))` <br> `fromLastMonth = getSheetValue(lastMonthSheet, 'leftover-' + cat.id)` <br> `carryover = getSheetBoolean(lastMonthSheet, 'carryover-' + cat.id)` |
| **上月数据处理** | `fromLastMonth`, `carryover`, `category` | `fromLastMonth` | - | 条件：`(fromLastMonth<0且无carryover) 或 是收入类别 或 是追踪预算` → `fromLastMonth=0` |
| **模板验证** | `templates`, `month` | - | - | 调用 `checkByAndScheduleAndSpend()`, `checkPercentage()`，验证失败抛错 |
| **偏好获取** | - | `hideDecimal`, `currencyCode` | - | 查询 preferences 表获取 `hideFraction`, `defaultCurrencyCode`，无数据时 `hideDecimal=false`，`currencyCode=''` |
| **私有构造** | 全部参数 | `CategoryTemplateContext` 实例 | - | 调用受保护构造函数，内部调用 `checkLimit()`, `checkSpend()`, `checkGoal()` |

---

### 8.3 私有构造函数执行流程

```typescript
protected constructor(
  templates: Template[],
  category: CategoryEntity,
  month: string,
  fromLastMonth: number,
  budgeted: number,
  currencyCode: string,
  hideDecimal: boolean = false,
  skipAvailableClamp: boolean = false
)
```

| 阶段 | 输入 | 输出 | 关键中间变量 | 操作说明 |
|------|------|------|--------------|----------|
| **属性赋值** | 全部参数 | 实例属性 | - | `this.category`, `this.month`, `this.fromLastMonth`, `this.previouslyBudgeted`, `this.currency` (通过 `getCurrency(currencyCode)` 获取), `this.hideDecimal`, `this.skipAvailableClamp` |
| **模板分类** | `templates` | `templates[]`, `remainder[]`, `goals[]` | `priorities: Set<number>`, `remainderWeight` | 按 `directive` 和 `type` 分类：<br>- 普通模板：`type!='remainder' && type!='limit'` → `this.templates[]`，收集 `priority` → `priorities`<br>- 余数模板：`type='remainder'` → `this.remainder[]`，累加 `weight` → `remainderWeight`<br>- 目标模板：`directive='goal' && type='goal'` → `this.goals[]` |
| **限制检查** | `templates` | `limitAmount`, `limitCheck`, `limitHold`, `limitExcess`, `limitMet` | `limitAmount` | 调用 `checkLimit()`，计算 `limitAmount`，超限情况下设置 `limitMet=true` 和 `limitExcess` |
| **支出检查** | `templates` | - | - | 调用 `checkSpend()`，检查 `spend` 模板数量不超过 1，否则抛错 |
| **目标检查** | `templates` | - | - | 调用 `checkGoal()`，检查 `goal` 模板数量不超过 1，否则抛错 |

---

### 8.4 `runTemplatesForPriority()` 执行流程

```typescript
async runTemplatesForPriority(
  priority: number,
  budgetAvail: number,
  availStart: number
)
```

| 阶段 | 输入 | 输出 | 关键中间变量 | 操作说明 |
|------|------|------|--------------|----------|
| **前期检查** | `priority` | 返回 `0` | - | 无此优先级或 `limitMet=true` 时直接返回 0 |
| **模板过滤** | `templates`, `priority` | `t: Template[]` | `available`, `toBudget` | `t = templates.filter(...)`，初始化 `available=budgetAvail`, `toBudget=0` |
| **模板循环** | `t` | `toBudget` | `perTemplateLocal: Map<Template, number>`, `byFlag`, `scheduleFlag`, `byPerTemplate`, `schedulePerTemplate` | 逐个执行模板：<br> - 普通模板调用对应 `run*` 函数<br> - `by`/`schedule` 模板：首次执行完整逻辑，后续返回 0 |
| **批量重分配** | `perTemplateLocal`, `t` | `perTemplateLocal` | - | 调用 `redistributeBatch()` 将 `by`/`schedule` 批量总额按权重分配回各模板 |
| **限制检查** | `toBudget`, `fromLastMonth`, `toBudgetAmount` | `toBudget`, `limitMet`, `available` | `scale` | 若 `toBudget + toBudgetAmount + fromLastMonth >= limitAmount`，计算 `scale`，设置 `limitMet=true`，调整 `toBudget` 和 `available` |
| **小数隐藏** | `toBudget`, `hideDecimal` | `toBudget` | `scale` | 若 `hideDecimal=true`，将 `toBudget` 取整，更新 `scale` |
| **可用金额钳制** | `priority`, `available`, `toBudget`, `category.is_income`, `skipAvailableClamp` | `toBudget` | `fullAmount` | 若 `priority>0` 且 `available<0` 且非收入类别且未跳过钳制：<br>- `fullAmount += toBudget`<br>- `toBudget = max(0, toBudget + available)`<br>- 更新 `scale` |
| **按比例分配到模板** | `perTemplateLocal`, `toBudget`, `scale` | `this.perTemplateContribution` | `remaining` | 遍历模板，按 `scale` 分配，最后一个模板吸收剩余四舍五入差额，累积到 `this.perTemplateContribution` |
| **返回结果** | - | 最终 `toBudget` | - | 收入类别返回 `-toBudget`，否则返回 `toBudget`，同时 `this.toBudgetAmount += toBudget` |

---

### 8.5 `runRemainder()` 执行流程

```typescript
runRemainder(budgetAvail: number, perWeight: number)
```

| 阶段 | 输入 | 输出 | 关键中间变量 | 操作说明 |
|------|------|------|--------------|----------|
| **前期检查** | `remainder[]` | `0` | - | 无余数模板直接返回 0 |
| **初始计算** | `remainderWeight`, `perWeight` | `toBudget` | `smallest` | `toBudget = round(remainderWeight * perWeight)`，`smallest = hideDecimal ? 100 : 1` |
| **预算调整** | `toBudget`, `budgetAvail` | `toBudget` | - | 若 `toBudget>budgetAvail` 或 `budgetAvail-toBudget<=smallest`，`toBudget=budgetAvail` |
| **限制检查** | `toBudget`, `toBudgetAmount`, `fromLastMonth`, `limitAmount` | `toBudget`, `limitMet` | - | 若超限，`toBudget=limitAmount - toBudgetAmount - fromLastMonth`, `limitMet=true` |
| **模板分配** | `toBudget`, `remainder[]` | `this.perTemplateContribution` | `remaining` | 按 `weight` 比例分配，最后一个模板吸收剩余，累积到 `this.perTemplateContribution` |
| **累积返回** | `toBudget` | `toBudget` | - | `this.toBudgetAmount += toBudget`，返回 `toBudget` |

---

### 8.6 `getValues()` 执行流程

```typescript
getValues()
```

| 阶段 | 输入 | 输出 | 关键中间变量 | 操作说明 |
|------|------|------|--------------|----------|
| **目标计算** | `goals[]` | `isLongGoal`, `goalAmount` | - | 调用 `runGoal()`：<br>- 有目标：`isLongGoal=true`, `goalAmount=amountToInteger(goals[0].amount)`<br>- 无目标：`isLongGoal=null`, `goalAmount=fullAmount` |
| **返回结果** | - | `{ budgeted, goal, longGoal, perTemplateContribution }` | - | 最终返回：<br> - `budgeted: this.toBudgetAmount`<br> - `goal: this.goalAmount`<br> - `longGoal: this.isLongGoal`<br> - `perTemplateContribution: this.perTemplateContribution` |

---

## 九、模板类型字段映射表

| 模板类型 | Sheet 键读取 | DB 读取 | 读取位置（函数） | 读取后如何参与计算 |
|---------|-------------|---------|-----------------|-------------------|
| **`simple`** | 无 | 无 | `runSimple()` | 优先使用 `template.monthly` 转为整数，否则返回 `limitAmount - fromLastMonth` |
| **`refill`** | 无 | 无 | `runRefill()` | 直接返回 `limitAmount - fromLastMonth` |
| **`copy`** | `budget-{category.id}` | 无 | `runCopy()` | 从 `subMonths(month, template.lookBack)` 月份 Sheet 读取预算，原样返回 |
| **`spend`** | `sum-amount-{category.id}`, `leftover-{category.id}`, `budget-{category.id}` | 无 | `runSpend()` | 1. 从 `template.from` 开始读取历史数据：<br>   - 首月：`alreadyBudgeted = leftover - sum-amount`<br>   - 后续：`alreadyBudgeted += budget`<br>2. 计算：`(target - alreadyBudgeted) / (numMonths + 1)` |
| **`percentage`** | `total-income` 或 `sum-amount-{incomeCat.id}` | `db.getCategories()` (过滤收入类别) | `runPercentage()` | 1. 根据 `template.category` 决定来源：<br>   - 'all income'：`total-income`<br>   - 'available funds'：入参 `availableFunds`<br>   - 其他：查找收入类别 → `sum-amount-{cat.id}`<br>2. 可选读取上月数据（`previous=true`）<br>3. 计算：`max(0, round(income * percent/100))` |
| **`by`** | 无 | 无 | `runBy()` | 1. 计算各模板剩余月数，找到最短时间<br>2. 对每个模板计算所需资金：<br>   - 超过最短窗口且可重复：`(amount/period)*(period - numMonths + shortNumMonths)`<br>   - 超过最短窗口不可重复：`(amount/(numMonths+1))*(shortNumMonths+1)`<br>   - 其他：`amount`<br>3. 最终：`(totalNeeded - fromLastMonth) / (shortNumMonths + 1)` |
| **`schedule`** | `goal-{category.id}` (上月) | `schedules` 表 | `runSchedule()` 调用 `createScheduleList()` 调用 `getRuleForSchedule()`, `prefetchBalanceOfForTransaction()` | 1. 查询调度表获取规则<br>2. 执行规则计算 `target`<br>3. 分为 `payMonthOf` 和 `sinking` 两类<br>4. 根据上月余额、上月目标等条件决定预算策略<br>5. 最终 `to_budget` 加相关金额 |
| **`average`** | `sum-amount-{category.id}` (历史 `numMonths` 个月) | 无 | `runAverage()` | 1. 读取历史 `numMonths` 个月的支出总和<br>2. 计算平均：`-(sum / numMonths)`<br>3. 可选调整：<br>   - 百分比：`average *= (1 + adjustment/100)`<br>   - 固定值：`average += adjustment`<br>4. 取整返回 |
| **`remainder`** | 无 | 无 | `runRemainder()` | 按 `weight` 权重分配剩余预算 |

---

## 十、缺失数据处理矩阵

| 数据项 | 缺失场景 | 处理策略 | 默认值/回退值 | 错误类型 | 捕获位置 | 错误聚合方式 |
|-------|---------|---------|-------------|---------|---------|------------|
| **上月剩余 (`fromLastMonth`)** | Sheet 无 `leftover-{cat.id}` 单元格 | 降级处理 | `0` | 无 | `actions.getSheetValue()` → `safeNumber(node.value || 0)` | - |
| **`carryover` 标记** | Sheet 无 `carryover-{cat.id}` 单元格 | 降级处理 | `false` | 无 | `actions.getSheetBoolean()` → `node.value || false` | - |
| **隐藏小数偏好 (`hideFraction`)** | preferences 表无此记录 | 降级处理 | `false` | 无 | `CategoryTemplateContext.init()` → 空数组检查 | - |
| **默认货币 (`defaultCurrencyCode`)** | preferences 表无此记录 | 降级处理 | `''` → 系统默认货币 | 无 | `CategoryTemplateContext.init()` → 空数组检查 | - |
| **活动调度名称** | `schedule` 模板引用的调度名在 `schedules` 表不存在 | 抛错终止 | - | `Error("Schedule X does not exist")` | `CategoryTemplateContext.checkByAndScheduleAndSpend()` | 收集到 `errors[]`，上层 `computeTemplates()` try-catch |
| **收入类别名称/ID** | `percentage` 模板引用的收入类别不存在 | 抛错终止 | - | `Error("Category X is not found")` | `CategoryTemplateContext.checkPercentage()` | 收集到 `errors[]`，上层 `computeTemplates()` try-catch |
| **`by`/`spend` 目标月份** | 目标月份已过且不可重复 | 抛错终止 | - | `Error("Target month has passed")` | `CategoryTemplateContext.checkByAndScheduleAndSpend()` | 收集到 `errors[]`，上层 `computeTemplates()` try-catch |
| **周限起始日期** | `limit.period='weekly'` 但无 `start` | 抛错终止 | - | `Error("Weekly limit requires a start date")` | `CategoryTemplateContext.checkLimit()` | 收集到 `errors[]`，上层 `computeTemplates()` try-catch |
| **多限制定义** | 同类别有多个 `limit` 模板 | 抛错终止 | - | `Error("Only one 'up to' allowed per category")` | `CategoryTemplateContext.checkLimit()` | 收集到 `errors[]`，上层 `computeTemplates()` try-catch |
| **多支出模板** | 同类别有多个 `spend` 模板 | 抛错终止 | - | `Error("Only one spend template allowed")` | `CategoryTemplateContext.checkSpend()` | 收集到 `errors[]`，上层 `computeTemplates()` try-catch |
| **多目标模板** | 同类别有多个 `goal` 模板 | 抛错终止 | - | `Error("Only one #goal allowed per category")` | `CategoryTemplateContext.checkGoal()` | 收集到 `errors[]`，上层 `computeTemplates()` try-catch |
| **调度数据异常** | `schedule` 模板指向已完成或过期调度 | 软降级 | 该调度被跳过 | 非致命错误 | `createScheduleList()` → `t.filter(c => c.completed === 0)` | 收集到 `errors[]`，但继续处理其他模板 |
| **历史预算/支出** | `copy`/`spend`/`average` 模板读取历史月份无数据 | 降级处理 | `0` | 无 | `actions.getSheetValue()` → `safeNumber(node.value || 0)` | - |
| **类别配置** | 计算时找不到类别（仅 dryRun 可能） | 降级处理 | 返回 `{ budgeted: 0, perTemplate: [0...] }` | 无 | `dryRunCategoryTemplate()` → 空检查返回 | - |

---

## 附录：关键文件位置

| 文件 | 路径 | 说明 |
|------|------|------|
| `category-template-context.ts` | `packages/loot-core/src/server/budget/` | 核心上下文类 |
| `goal-template.ts` | `packages/loot-core/src/server/budget/` | 模板应用入口 |
| `actions.ts` | `packages/loot-core/src/server/budget/` | 数据读写操作 |
| `templates.ts` | `packages/loot-core/src/types/models/` | 模板类型定义 |
| `schedule-template.ts` | `packages/loot-core/src/server/budget/` | 调度模板执行 |
