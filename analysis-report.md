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

## 附录：关键文件位置

| 文件 | 路径 | 说明 |
|------|------|------|
| `category-template-context.ts` | `packages/loot-core/src/server/budget/` | 核心上下文类 |
| `goal-template.ts` | `packages/loot-core/src/server/budget/` | 模板应用入口 |
| `actions.ts` | `packages/loot-core/src/server/budget/` | 数据读写操作 |
| `templates.ts` | `packages/loot-core/src/types/models/` | 模板类型定义 |
