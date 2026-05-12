# 周期计划模板全链路分析报告

本文档记录周期计划（Schedule Template）从创建到逐期求值的完整链路，便于后续代码评审和理解。

---

## 一、模板参数与时间/金额字段职责

### 1.1 模板类型定义

**位置：** `packages/loot-core/src/types/models/templates.ts:62-68`

```typescript
export type ScheduleTemplate = {
  type: 'schedule';
  name: string;           // 关联的计划名称（用于查找对应的 Schedule）
  full?: boolean;          // 是否全额预算（true = 到期月一次性预算，false = 分期储蓄）
  adjustment?: number;    // 调整金额（百分比或固定值）
  adjustmentType?: 'percent' | 'fixed';  // 调整类型
}
```

### 1.2 计划实体模型

**位置：** `packages/loot-core/src/types/models/schedule.ts:22-41`

```typescript
export type ScheduleEntity = {
  id: string;
  name?: string;
  rule: RuleEntity['id'];  // 关联的规则ID（Schedule 通过 Rule 存储条件和动作）
  next_date: string;       // 下一次执行日期（核心时间推进字段）
  completed: boolean;      // 是否完成（用于终止状态判断）
  posts_transaction: boolean; // 是否自动生成交易
  custom_upcoming_length?: string | null; // 自定义即将到期天数
  tombstone: boolean;

  // 从规则中提取的动态字段（运行时填充）
  _payee: PayeeEntity['id'];        // 收款人
  _account: AccountEntity['id'];    // 账户
  _amount: number | { num1: number; num2: number };  // 金额或金额范围
  _amountOp: string;     // 金额操作符（is / isapprox / isbetween）
  _date: RecurConfig | string;      // 日期配置（递归或固定日期）
  _conditions: RuleConditionEntity[];  // 规则条件
  _actions: Array<{ op: unknown }>;     // 规则动作
};
```

### 1.3 递归配置（时间推进核心）

**位置：** `packages/loot-core/src/types/models/schedule.ts:10-20`

```typescript
export type RecurConfig = {
  frequency: 'daily' | 'weekly' | 'monthly' | 'yearly';  // 重复频率
  interval?: number;          // 间隔次数（如"每2周"则 interval=2）
  patterns?: RecurPattern[];  // 月/周模式（如"每月第3个周一"）
  skipWeekend?: boolean;       // 是否跳过周末
  start: string;              // 开始日期
  endMode?: 'never' | 'after_n_occurrences' | 'on_date';  // 结束模式
  endOccurrences?: number;     // 结束次数（endMode=after_n_occurrences 时使用）
  endDate?: string;             // 结束日期（endMode=on_date 时使用）
  weekendSolveMode?: 'before' | 'after';  // 周末解决策略（提前到周五或延后到周一）
};
```

### 1.4 各字段职责汇总

| 字段 | 所在位置 | 职责 |
|------|---------|------|
| `name` | ScheduleTemplate | 通过名称关联 Schedule 实体 |
| `full` | ScheduleTemplate | 决定预算策略：全额 vs 分期储蓄 |
| `adjustment`/`adjustmentType` | ScheduleTemplate | 对计划金额进行调整 |
| `next_date` | ScheduleEntity | 记录下一期执行日期，时间推进的核心锚点 |
| `completed` | ScheduleEntity | 终止状态标记 |
| `posts_transaction` | ScheduleEntity | 是否自动生成交易 |
| `_date` (RecurConfig) | ScheduleEntity | 递归配置，控制时间推进规则 |
| `_amount` | ScheduleEntity | 金额或金额范围（范围取平均值） |
| `frequency`/`interval` | RecurConfig | 重复周期和间隔 |
| `start`/`endMode`/`end*` | RecurConfig | 起止条件 |
| `skipWeekend`/`weekendSolveMode` | RecurConfig | 周末处理策略 |

### 1.5 创建阶段 next_date 持久化细节

#### 1.5.1 schedules_next_date 表结构

**位置：** `packages/loot-core/migrations/1618975177358_schedules.sql:11-17`

```sql
CREATE TABLE schedules_next_date
  (id TEXT PRIMARY KEY,
   schedule_id TEXT,
   local_next_date INTEGER,   -- 本地调整后的日期
   local_next_date_ts INTEGER, -- 本地调整的时间戳
   base_next_date INTEGER,    -- 规则计算的原始日期
   base_next_date_ts INTEGER); -- 原始日期的时间戳
```

#### 1.5.2 两套字段的设计意图

`v_schedules` 视图通过时间戳比较决定使用哪套字段：

**位置：** `packages/loot-core/src/server/aql/schema/index.ts:331-336`

```sql
CASE
  WHEN _nd.local_next_date_ts = _nd.base_next_date_ts THEN _nd.local_next_date
  ELSE _nd.base_next_date
END
```

| 场景 | 比较结果 | 使用字段 | 说明 |
|------|---------|---------|------|
| 初始创建 | `local_ts = base_ts` | `local_next_date` | 两套字段一致 |
| 正常推进（reset=true） | `local_ts = base_ts` | `local_next_date` | 两套字段同时更新 |
| 跳过操作（reset=false） | `local_ts != base_ts` | `base_next_date` | 使用原始日期 |

#### 1.5.3 创建时的写入路径

**位置：** `packages/loot-core/src/server/schedules/app.ts:254-304`

```typescript
export async function createSchedule({
  schedule = null,
  conditions = [],
} = {}): Promise<ScheduleEntity['id']> {
  // ...
  const nextDate = getNextDate(dateCond);  // 计算首次 next_date
  const nextDateRepr = nextDate ? toDateRepr(nextDate) : null;

  // 创建规则
  const ruleId = await insertRule({
    stage: null,
    conditionsOp: 'and',
    conditions,
    actions: [{ op: 'link-schedule', value: scheduleId }],
  });

  const now = Date.now();

  // 写入 schedules_next_date：两套字段初始化为相同值
  await db.insertWithUUID('schedules_next_date', {
    schedule_id: scheduleId,
    local_next_date: nextDateRepr,    // 本地日期
    local_next_date_ts: now,         // 相同时间戳
    base_next_date: nextDateRepr,     // 基期日期
    base_next_date_ts: now,          // 相同时间戳
  });

  // 写入 schedules 主表
  await db.insertWithSchema('schedules', {
    ...schedule,
    id: scheduleId,
    rule: ruleId,
  });

  return scheduleId;
}
```

#### 1.5.4 reset=true 路径（更新计划时重置）

**位置：** `packages/loot-core/src/server/schedules/app.ts:219-232`

```typescript
await db.update(
  'schedules_next_date',
  reset
    ? {
        id: nd.id,
        base_next_date: toDateRepr(newNextDate),
        base_next_date_ts: Date.now(),  // 更新基期时间戳
      }
    : {
        id: nd.id,
        local_next_date: toDateRepr(newNextDate),
        local_next_date_ts: nd.base_next_date_ts,  // 保持基期时间戳不变
      },
);
```

**更新时机：** `packages/loot-core/src/server/schedules/app.ts:357-373`

```typescript
if (
  resetNextDate ||
  !areScheduleConditionsEqual(
    oldConditions.find(c => c.field === 'account'),
    newConditions.find(c => c.field === 'account'),
  ) ||
  !areConditionValuesEqual(
    stripType(oldConditions.find(c => c.field === 'date') || {}),
    stripType(newConditions.find(c => c.field === 'date') || {}),
  )
) {
  await setNextDate({
    id: schedule.id,
    conditions: newConditions,
    reset: true,  // 日期条件变化 → 重置两套字段
  });
}
```

#### 1.5.5 reset=false 路径（跳过操作、正常推进）

**跳过操作入口：** `packages/loot-core/src/server/schedules/app.ts:395-403`

```typescript
export async function skipNextDate({ id }) {
  return setNextDate({
    id,
    start: nextDate => {
      return d.addDays(parseDate(nextDate), 1);  // 从下一天开始计算
    },
    skipRequested: true,  // 标记为主动跳过
  });
}
```

**正常推进（paid 状态）：** `packages/loot-core/src/server/schedules/app.ts:546-555`

```typescript
if (schedule._date.frequency) {
  try {
    await setNextDate({ id: schedule.id });  // 无 reset 参数，默认为 false
  } catch {
    // 忽略规则损坏的计划
  }
}
```

#### 1.5.6 setNextDate 内部逻辑

**位置：** `packages/loot-core/src/server/schedules/app.ts:159-235`

```typescript
export async function setNextDate({
  id,
  start,
  conditions,
  reset,
  skipRequested,
}: {
  id: string;
  start?;
  conditions?;
  reset?: boolean;
  skipRequested?: boolean;
}) {
  // ...

  // 跳过操作的周末特殊处理
  if (skipRequested === true) {
    const skipWeekend: boolean = dateCond.value?.skipWeekend;
    const weekendSolveMode: string = dateCond.value?.weekendSolveMode;

    // 周末策略=before 时的特殊处理
    if (weekendSolveMode === 'before' && skipWeekend === true) {
      const parsedNextDate = parseDate(nextDate);
      if (d.isFriday(parsedNextDate) || d.isWeekend(parsedNextDate)) {
        // 强制推到下周一，避免 getNextDate 又拉回周五
        nextDate = dayFromDate(d.nextMonday(parsedNextDate));
      }
    }
  }

  // 计算新日期
  const newNextDate = getNextDate(
    dateCond,
    start ? start(nextDate) : new Date(),
  );

  if (newNextDate != null && newNextDate !== nextDate) {
    // 查询现有记录
    const nd = await db.first<...>(
      'SELECT id, base_next_date_ts FROM schedules_next_date WHERE schedule_id = ?',
      [id],
    );

    // 根据 reset 标志更新不同字段
    await db.update(
      'schedules_next_date',
      reset
        ? {
            id: nd.id,
            base_next_date: toDateRepr(newNextDate),
            base_next_date_ts: Date.now(),
          }
        : {
            id: nd.id,
            local_next_date: toDateRepr(newNextDate),
            local_next_date_ts: nd.base_next_date_ts,  // 复用基期时间戳
          },
    );
  }
}
```

#### 1.5.7 reset 与 skip 对应关系汇总

| 操作 | reset 参数 | 更新字段 | 时间戳变化 |
|------|-----------|---------|-----------|
| 初始创建 | N/A | `local + base` 都写 | `local_ts = base_ts = now()` |
| 日期条件变化 | `true` | `base_next_date` + `base_next_date_ts` | `base_ts` 更新 |
| 跳过操作 | `false`（默认） | `local_next_date` | `local_ts` 复用 `base_ts` |
| 正常推进（已付款） | `false`（默认） | `local_next_date` | `local_ts` 复用 `base_ts` |

**视图选择逻辑：**
- `local_ts == base_ts` → 使用 `local_next_date`（两套一致或刚 reset）
- `local_ts != base_ts` → 使用 `base_next_date`（local 被跳过操作更新过）

---

## 二、计划触发后当期值获取与落库

### 2.1 时间推进核心函数

**位置：** `packages/loot-core/src/shared/schedules.ts:356-389`

```typescript
export function getNextDate(
  dateCond,
  start = new Date(monthUtils.currentDay()),
  noSkipWeekend = false,
): string | null {
  start = d.startOfDay(start);

  const cond = new Condition(dateCond.op, 'date', dateCond.value, null);
  const value = cond.getValue();

  // 路径1：固定日期（单次计划）
  if (value.type === 'date') {
    return value.date;
  }

  // 路径2：递归计划（使用 rSchedule 库计算）
  else if (value.type === 'recur') {
    // 正向查找下一个日期
    let dates = value.schedule.occurrences({ start, take: 1 }).toArray();

    // 若没找到（可能已结束），反向查找最后一个日期
    if (dates.length === 0) {
      dates = value.schedule.occurrences({ reverse: true, take: 1 }).toArray();
    }

    if (dates.length > 0) {
      let date = dates[0].date;
      // 处理周末跳过
      if (value.schedule.data.skipWeekend && !noSkipWeekend) {
        date = getDateWithSkippedWeekend(
          date,
          value.schedule.data.weekendSolve,
        );
      }
      return monthUtils.dayFromDate(date);
    }
  }
  return null;
}
```

### 2.2 金额计算核心函数

**位置：** `packages/loot-core/src/shared/schedules.ts:407-420`

```typescript
export function getScheduledAmount(
  amount: number | { num1: number; num2: number },
  inverse: boolean = false,
): number {
  if (amount == null) return 0;

  // 固定金额
  if (typeof amount === 'number') {
    return inverse ? -amount : amount;
  }

  // 金额范围（取两个端点的平均值）
  const avg = (amount.num1 + amount.num2) / 2;
  return inverse ? -Math.round(avg) : Math.round(avg);
}
```

### 2.3 自动推进服务（写入交易表）

**入口：** `packages/loot-core/src/server/schedules/app.ts:514-594`

```typescript
async function advanceSchedulesService(syncSuccess) {
  // Step 1: 获取所有未完成、账户未关闭的计划
  const { data: schedules } = await aqlQuery(
    q('schedules')
      .filter({ completed: false, '_account.closed': false })
      .select('*'),
  );

  // Step 2: 检查每个计划是否已有匹配交易
  const { data: hasTransData } = await aqlQuery(
    getHasTransactionsQuery(schedules),
  );
  const hasTrans = new Set(hasTransData.filter(Boolean).map(row => row.schedule));

  for (const schedule of schedules) {
    // Step 3: 判断计划状态
    const status = getStatus(
      schedule.next_date,
      schedule.completed,
      hasTrans.has(schedule.id),
      schedule.custom_upcoming_length ?? upcomingLength,
    );

    // 状态判断逻辑见 schedules.ts:13-37
    // - completed: 已完成
    // - paid: 已有匹配交易（可能提前支付）
    // - due: 今天到期
    // - upcoming: 即将到期（默认7天内）
    // - missed: 逾期
    // - scheduled: 计划中

    // 分支A: 已付款 → 推进到下一期
    if (status === 'paid') {
      if (schedule._date) {
        // 递归计划：计算并更新 next_date
        if (schedule._date.frequency) {
          try {
            await setNextDate({ id: schedule.id });
          } catch {
            // 忽略规则损坏的计划
          }
        }
        // 单次计划：日期已过则标记完成
        else {
          if (schedule._date < currentDay()) {
            await updateSchedule({
              schedule: { id: schedule.id, completed: true },
            });
          }
        }
      }
    }

    // 分支B: 到期/逾期 + 自动发布 → 写入交易表
    else if (
      (status === 'due' || status === 'missed') &&
      schedule.posts_transaction &&
      schedule._account
    ) {
      if (syncSuccess) {
        await postTransactionForSchedule({ id: schedule.id });
        didPost = true;
      }
    }
  }
}
```

### 2.4 交易写入逻辑

**位置：** `packages/loot-core/src/server/schedules/app.ts:485-510`

```typescript
async function postTransactionForSchedule({
  id,
  today,
}: {
  id: string;
  today?: boolean;
}) {
  const { data } = await aqlQuery(q('schedules').filter({ id }).select('*'));
  const schedule = data[0];

  // 构建交易对象
  const transaction = {
    payee: schedule._payee,
    account: schedule._account,
    amount: getScheduledAmount(schedule._amount),  // 计算当期金额
    date: today ? currentDay() : schedule.next_date,  // 使用 next_date 或今天
    schedule: schedule.id,  // 关联计划ID
    cleared: false,
  };

  // 写入交易表
  if (transaction.account) {
    await addTransactions(transaction.account, [transaction]);
  }
}
```

### 2.5 预算视图求值（返回值，不直接写入）

**⚠️ 重要修正：** `runSchedule` **仅负责计算并返回** `to_budget` / `perScheduleMonthly`，**不直接写入**预算视图。实际写入由上层调用链汇总后批量处理。

#### 2.5.1 runSchedule 计算逻辑

**入口：** `packages/loot-core/src/server/budget/schedule-template.ts:305-395`

```typescript
export async function runSchedule(
  template_lines: Template[],
  current_month: string,
  balance: number,
  remainder: number,
  last_month_balance: number,
  to_budget: number,           // 传入的累计值
  errors: string[],
  category: CategoryEntity,
  currency: Currency,
) {
  // Step 1: 过滤出 schedule 类型模板
  const scheduleTemplates = template_lines.filter(t => t.type === 'schedule');

  // Step 2: 创建计划列表（计算每个计划的当期目标金额）
  const t = await createScheduleList(
    scheduleTemplates,
    current_month,
    category,
    currency,
  );

  // Step 3: 分类处理
  // pay-month-of: 当月到期或高频率计划 → 全额预算
  // sinking: 需分期储蓄的计划（如年度账单）→ 按月分摊
  const t_payMonthOf = t.t.filter(isPayMonthOf);
  const t_sinking = t.t
    .filter(c => !isPayMonthOf(c))
    .sort((a, b) => a.next_date_string.localeCompare(b.next_date_string));

  // Step 4: 计算预算金额
  if (
    balance >= totalSinking + totalPayMonthOf ||
    (lastMonthGoal < totalSinking + totalPayMonthOf &&
      lastMonthGoal !== 0 &&
      balance >= lastMonthGoal &&
      numSubMonthly > 0)
  ) {
    // 余额充足：全额预算 + 基础分摊
    to_budget += Math.round(totalPayMonthOf + totalSinkingBaseContribution);
    for (const c of t_payMonthOf) addContribution(c.name, c.target);
    for (const c of t_sinking) {
      addContribution(c.name, getMonthlyBaseContribution(c));
    }
  } else {
    // 余额不足：按顺序分配
    const { total: totalSinkingContribution, perSchedule: sinkingPerSchedule } =
      getSinkingContributionBreakdown(t_sinking, remainder, last_month_balance);
    to_budget += Math.round(totalPayMonthOf + totalSinkingContribution);
    for (const c of t_payMonthOf) addContribution(c.name, c.target);
    for (const [name, amount] of sinkingPerSchedule) {
      addContribution(name, amount);
    }
  }

  // 仅返回计算结果，不写入数据库
  return { to_budget, errors, remainder, perScheduleMonthly };
}
```

#### 2.5.2 CategoryTemplateContext 调用链

`runSchedule` 在 `CategoryTemplateContext.runTemplatesForPriority()` 中被调用，计算结果在上下文内汇总。

**位置：** `packages/loot-core/src/server/budget/category-template-context.ts:139-327`

```typescript
// runTemplatesForPriority 方法签名
async runTemplatesForPriority(
  priority: number,
  budgetAvail: number,
  availStart: number,
): Promise<number> {
  if (!this.priorities.has(priority)) return 0;
  if (this.limitMet) return 0;

  const t = this.templates.filter(
    t => t.directive === 'template' && t.priority === priority,
  );
  let available = budgetAvail || 0;
  let toBudget = 0;
  const perTemplateLocal = new Map<Template, number>();
  let remainder = 0;
  let scheduleFlag = false;
  let schedulePerTemplate: Map<string, number> | null = null;

  // 按模板类型循环处理
  for (const template of t) {
    let newBudget = 0;
    switch (template.type) {
      // ... 其他类型（simple, refill, copy, periodic, spend, percentage, by, average）

      case 'schedule': {
        if (!scheduleFlag) {
          const budgeted = this.fromLastMonth + toBudget;
          // 调用 runSchedule
          const ret = await runSchedule(
            t,                          // 所有同优先级模板
            this.month,
            budgeted,                   // 当前累计预算
            remainder,
            this.fromLastMonth,         // 上月余额
            toBudget,                   // 传入累计值
            [],
            this.category,
            this.currency,
          );
          // Schedules 假设 to_budget 是整体值，需要减去之前累计避免重复计算
          newBudget = ret.to_budget - toBudget;
          remainder = ret.remainder;
          schedulePerTemplate = ret.perScheduleMonthly;
          scheduleFlag = true;
        }
        break;
      }
      default: {
        break;
      }
    }

    available = available - newBudget;
    toBudget += newBudget;
    perTemplateLocal.set(
      template,
      (perTemplateLocal.get(template) ?? 0) + newBudget,
    );
  }

  // ... 后续处理：redistributeBatch 重新分配、limit 检查、四舍五入、available clamp 等

  // 将 per-template 贡献写入上下文的 perTemplateContribution
  const items = Array.from(perTemplateLocal);
  let remaining = Math.max(0, toBudget);
  items.forEach(([template, value], i) => {
    const isLast = i === items.length - 1;
    const share = isLast
      ? remaining
      : Math.max(0, Math.min(remaining, Math.round(value * perRowScale)));
    const existing = this.perTemplateContribution.get(template) ?? 0;
    this.perTemplateContribution.set(template, existing + share);
    remaining -= share;
  });

  return this.category.is_income ? -toBudget : toBudget;
}
```

#### 2.5.3 完整预算求值调用链

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    CategoryTemplateContext 调用链                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  1. 初始化                                                              │
│     CategoryTemplateContext.init()                                     │
│     ├─ 计算上月余额 fromLastMonth                                       │
│     ├─ 校验模板合法性（checkByAndScheduleAndSpend）                     │
│     └─ 分类模板：templates / remainder / goals                          │
│                                                                         │
│  2. 按优先级求值（runAll 或外部调用 runTemplatesForPriority）            │
│     runTemplatesForPriority(priority, budgetAvail, availStart)          │
│     ├─ 循环处理该优先级下的所有模板                                      │
│     │                                                                   │
│     │  ┌─ schedule 类型 ──────────────────────────────────────────────┐│
│     │  │                                                             ││
│     │  │  首次遇到 schedule 模板时调用 runSchedule()                   ││
│     │  │    ├─ createScheduleList() → 计算每个计划的当期目标            ││
│     │  │    ├─ 分类：pay-month-of vs sinking                          ││
│     │  │    ├─ 计算 to_budget 增量                                     ││
│     │  │    └─ 返回 { to_budget, errors, remainder, perScheduleMonthly }││
│     │  │                                                             ││
│     │  │  newBudget = ret.to_budget - toBudget (避免重复计算)          ││
│     │  │  schedulePerTemplate 用于后续 per-template 分配               ││
│     │  │                                                             ││
│     │  └───────────────────────────────────────────────────────────────┘│
│     │                                                                   │
│     ├─ 累加 toBudget                                                    │
│     ├─ redistributeBatch() → 按权重重新分配 per-template 贡献           │
│     │   └─ schedule 使用 perScheduleMonthly 作为权重                     │
│     ├─ limit 检查（超限时 clamp）                                       │
│     ├─ hideDecimal 四舍五入                                             │
│     ├─ available clamp（非收入类别不超额）                              │
│     └─ 写入 perTemplateContribution Map                                 │
│                                                                         │
│  3. remainder 模板处理（如果有）                                        │
│     runRemainder(budgetAvail, perWeight)                                │
│                                                                         │
│  4. 获取结果（不写入，由调用者决定）                                      │
│     getValues()                                                         │
│     ├─ runGoal() → 计算 goalAmount                                      │
│     └─ 返回 { budgeted, goal, longGoal, perTemplateContribution }       │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

#### 2.5.4 关键数据流向说明

| 阶段 | 数据流向 | 说明 |
|------|---------|------|
| runSchedule 内部 | 计算 `to_budget` 增量 | 基于传入的 `to_budget` 累计值，返回新的累计值 |
| runTemplatesForPriority | `newBudget = ret.to_budget - toBudget` | 计算本次 schedule 模板贡献的增量 |
| redistributeBatch | 按 `perScheduleMonthly` 权重分配 | 将批量预算分配到各个 schedule 模板行 |
| perTemplateContribution | 上下文内累积 | 按优先级逐步写入 Map，最后通过 getValues() 暴露 |
| 实际落库 | 由上层调用者处理 | 不在 runSchedule 或 CategoryTemplateContext 内写入数据库 |

### 2.6 当期目标金额计算（createScheduleList）

**位置：** `packages/loot-core/src/server/budget/schedule-template.ts:32-209`

```typescript
async function createScheduleList(
  templates: ScheduleTemplate[],
  current_month: string,
  category: CategoryEntity,
  currency: Currency,
) {
  const t: Array<ScheduleTemplateTarget> = [];
  const errors: string[] = [];

  for (const template of templates) {
    // Step 1: 通过名称查找计划
    const { id: sid, completed } = await db.first(...);
    const rule = await getRuleForSchedule(sid);
    const conditions = rule.serialize().conditions;

    // Step 2: 提取日期和金额条件
    const { date: dateConditions, amount: amountCondition } =
      extractScheduleConds(conditions);

    // Step 3: 计算基础金额
    let scheduleAmount =
      amountCondition.op === 'isbetween'
        ? Math.round(amountCondition.value.num1 + amountCondition.value.num2) / 2
        : amountCondition.value;

    // Step 4: 应用模板调整（百分比或固定值）
    if (template.adjustment !== undefined && template.adjustmentType) {
      switch (template.adjustmentType) {
        case 'percent': {
          const adjustmentFactor = 1 + template.adjustment / 100;
          scheduleAmount = scheduleAmount * adjustmentFactor;
          break;
        }
        case 'fixed': {
          const sign = scheduleAmount < 0 ? -1 : 1;
          scheduleAmount +=
            sign * amountToInteger(template.adjustment, currency.decimalPlaces);
          break;
        }
      }
    }

    scheduleAmount = Math.round(scheduleAmount);

    // Step 5: 获取下次日期
    const next_date_string = getNextDate(
      dateConditions,
      monthUtils._parse(current_month),
    );

    // Step 6: 执行规则动作（处理 BALANCE_OF 等公式）
    const formulaStrings = collectFormulasFromActions(rule.actions);
    const balanceOfPrefetched = await prefetchBalanceOfForTransaction(...);

    const { amount: postRuleAmount, subtransactions } = rule.execActions({
      ...scheduleRuleContext,
      _balanceOfPrefetched: balanceOfPrefetched,
    });

    // Step 7: 计算目标金额（考虑分类和子交易）
    const sign = category.is_income ? 1 : -1;
    const target =
      sign *
      (categorySubtransactions?.length
        ? categorySubtransactions.reduce((acc, t) => acc + t.amount, 0)
        : (postRuleAmount ?? scheduleAmount));

    // Step 8: 处理月度内多次发生的计划（如日度、周度）
    if (isRepeating) {
      let monthlyTarget = 0;
      const nextMonth = monthUtils.addMonths(
        current_month,
        t[t.length - 1].num_months + 1,
      );
      // 循环计算月度内所有发生日期并累加金额
      while (nextDate < nextMonth) {
        monthlyTarget += -target;
        // ... 推进到下一个日期
      }
      t[t.length - 1].target = -monthlyTarget;
    }
  }

  // 过滤已完成计划
  return { t: t.filter(c => c.completed === 0), errors };
}
```

### 2.7 基础分摊计算

**位置：** `packages/loot-core/src/server/budget/schedule-template.ts:254-288`

```typescript
function getMonthlyBaseContribution(schedule: ScheduleTemplateTarget) {
  let prevDate;
  let intervalMonths;
  switch (schedule.target_frequency) {
    case 'yearly':
      return schedule.target / schedule.target_interval / 12;
    case 'monthly':
      return schedule.target / schedule.target_interval;
    case 'weekly':
      prevDate = monthUtils.subWeeks(
        schedule.next_date_string,
        schedule.target_interval,
      );
      intervalMonths = monthUtils.differenceInCalendarMonths(
        schedule.next_date_string,
        prevDate,
      );
      if (intervalMonths === 0) intervalMonths = 1;
      return schedule.target / intervalMonths;
    case 'daily':
      prevDate = monthUtils.subDays(
        schedule.next_date_string,
        schedule.target_interval,
      );
      intervalMonths = monthUtils.differenceInCalendarMonths(
        schedule.next_date_string,
        prevDate,
      );
      if (intervalMonths === 0) intervalMonths = 1;
      return schedule.target / intervalMonths;
    default:
      return schedule.target / schedule.target_interval;
  }
}
```

---

## 三、特殊情况处理路径

### 3.1 跨日历边界

**场景：** 月度内多次发生的计划（如日度、周度）

**处理位置：** `packages/loot-core/src/server/budget/schedule-template.ts:153-200`

```typescript
if (isRepeating) {
  let monthlyTarget = 0;
  const nextMonth = monthUtils.addMonths(
    current_month,
    t[t.length - 1].num_months + 1,
  );
  let nextBaseDate = getNextDate(
    dateConditions,
    monthUtils._parse(current_month),
    true,  // 不跳过周末（用于精确计算边界）
  );
  let nextDate = dateConditions.value.skipWeekend
    ? monthUtils.dayFromDate(
        getDateWithSkippedWeekend(
          monthUtils._parse(nextBaseDate),
          dateConditions.value.weekendSolveMode,
        ),
      )
    : nextBaseDate;

  // 循环直到跨到下一月
  while (nextDate < nextMonth) {
    monthlyTarget += -target;
    const currentDate = nextBaseDate;
    const oneDayLater = monthUtils.addDays(nextBaseDate, 1);
    nextBaseDate = getNextDate(
      dateConditions,
      monthUtils._parse(oneDayLater),
      true,
    );
    nextDate = ...;  // 重新计算（考虑周末跳过）

    const diffDays = monthUtils.differenceInCalendarDays(
      nextBaseDate,
      currentDate,
    );
    if (!diffDays) {
      // 终止条件：日期不再推进（可能计划已结束）
      break;
    }
  }
  t[t.length - 1].target = -monthlyTarget;
}
```

**测试用例验证：**
- 日度计划（`schedule-template.test.ts:368-395`）：1月31天，每天$1 → 预算$31
- 6周间隔计划（`schedule-template.test.ts:517-549`）：按月份跨度分摊

### 3.2 跳过操作

**入口函数：** `packages/loot-core/src/server/schedules/app.ts:395-403`

```typescript
export async function skipNextDate({ id }) {
  return setNextDate({
    id,
    start: nextDate => {
      // 从下一天开始计算，跳过当前 next_date
      return d.addDays(parseDate(nextDate), 1);
    },
    skipRequested: true,  // 标记为主动跳过
  });
}
```

**周末特殊处理：** `packages/loot-core/src/server/schedules/app.ts:186-200`

```typescript
if (skipRequested === true) {
  const skipWeekend: boolean = dateCond.value?.skipWeekend;
  const weekendSolveMode: string = dateCond.value?.weekendSolveMode;

  // 问题场景：周末策略=before（提前到周五）
  // 如果当前 next_date 在周五或周末
  // 正常的 getNextDate 会把日期"拉回"周五，导致无法推进
  if (weekendSolveMode === 'before' && skipWeekend === true) {
    const parsedNextDate = parseDate(nextDate);
    if (d.isFriday(parsedNextDate) || d.isWeekend(parsedNextDate)) {
      // 强制推到下周一
      nextDate = dayFromDate(d.nextMonday(parsedNextDate));
    }
  }
}
```

**数据库更新路径：** 详见第一节 1.5.5 reset=false 路径。

### 3.3 提前支付

**状态判断：** `packages/loot-core/src/shared/schedules.ts:13-37`

```typescript
export function getStatus(...) {
  if (hasTrans) {
    return 'paid';  // 只要有匹配交易就认为已支付（包括提前支付）
  }
  // ...
}
```

**交易匹配查询（2天回溯窗口）：** `packages/loot-core/src/shared/schedules.ts:58-91`

```typescript
export function getHasTransactionsQuery(schedules) {
  const filters = schedules.map(schedule => {
    const dateCond = schedule._conditions?.find(c => c.field === 'date');
    return {
      $and: {
        schedule: schedule.id,
        date: {
          $gte:
            dateCond && dateCond.op === 'is'
              ? schedule.next_date           // 单次计划：精确日期
              : schedule.posts_transaction
                ? schedule.next_date         // 自动发布：精确日期
                : monthUtils.subDays(schedule.next_date, 2),  // 手动计划：2天回溯
        },
      },
    };
  });
  // ...
}
```

**已付款推进逻辑：** `packages/loot-core/src/server/schedules/app.ts:546-564`

```typescript
if (status === 'paid') {
  if (schedule._date.frequency) {
    // 递归计划：立即推进到下一期
    await setNextDate({ id: schedule.id });
  } else {
    if (schedule._date < currentDay()) {
      // 单次计划：日期已过则标记完成
      await updateSchedule({
        schedule: { id: schedule.id, completed: true },
      });
    }
  }
}
```

### 3.4 终止条件

**路径1：单次计划日期已过**

**位置：** `packages/loot-core/src/server/schedules/app.ts:556-563`

```typescript
if (schedule._date.frequency) {
  // 递归计划：正常推进
} else {
  // 单次计划（固定日期）
  if (schedule._date < currentDay()) {
    await updateSchedule({
      schedule: { id: schedule.id, completed: true },
    });
  }
}
```

**路径2：递归计划结束条件（rSchedule 内置）**

**配置转换：** `packages/loot-core/src/shared/schedules.ts:285-294`

```typescript
switch (config.endMode) {
  case 'after_n_occurrences':
    base.count = config.endOccurrences;  // 发生 N 次后自动结束
    break;
  case 'on_date':
    base.end = monthUtils.parseDate(config.endDate);  // 到指定日期后结束
    break;
  default:
    break;  // 永不结束
}
```

**getNextDate 中的处理：** `packages/loot-core/src/shared/schedules.ts:369-375`

```typescript
let dates = value.schedule.occurrences({ start, take: 1 }).toArray();

if (dates.length === 0) {
  // 正向查找失败 → 可能已结束
  // 尝试反向查找最后一个日期
  dates = value.schedule.occurrences({ reverse: true, take: 1 }).toArray();
}
// 如果 dates 仍然为空，返回 null
```

**路径3：预算模板中过滤已完成计划**

**位置：** `packages/loot-core/src/server/budget/schedule-template.ts:208`

```typescript
return { t: t.filter(c => c.completed === 0), errors };
```

**测试用例验证：** `schedule-template.test.ts:340-366`
- 已完成计划从预算中排除

**路径4：非递归过期计划**

**位置：** `packages/loot-core/src/server/budget/schedule-template.ts:137-140`

```typescript
const num_months = monthUtils.differenceInCalendarMonths(
  next_date_string,
  current_month,
);
if (num_months < 0) {
  // 非递归计划日期已过 → 记录错误
  errors.push(`Schedule ${template.name} is in the Past.`);
}
```

**测试用例验证：** `schedule-template.test.ts:441-482`
- 非递归过期计划记录 Past 错误，不参与预算计算

### 3.5 其他特殊情况

| 场景 | 处理位置 | 处理方式 |
|------|---------|---------|
| 余额超过目标 | `schedule-template.ts:585-614` | 本月不预算，吸收上期余额 |
| Tracking 预算模式 | `schedule-template.ts:616-647` | 全部视为 pay-month-of |
| 多个 sinking 计划 | `schedule-template.ts:397-439` | 按到期日期排序，先覆盖早到期的 |
| 计划名称空格 | `schedule-template.ts:360-365` | 自动 trim 后匹配 |

---

## 四、完整调用链图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        计划创建阶段                               │
├─────────────────────────────────────────────────────────────────────────┤
│  schedule/create (packages/loot-core/src/server/schedules/app.ts)    │
│                                                                       │
│  createSchedule()                                                    │
│    ├─ 提取条件：date, amount, payee, account                       │
│    ├─ getNextDate() → 计算首次 next_date                            │
│    ├─ insertRule() → 创建规则（含 link-schedule 动作）                │
│    │                                                                  │
│    ├─ db.insertWithUUID('schedules_next_date', {                     │
│    │    schedule_id,                                                  │
│    │    local_next_date: nextDateRepr, local_next_date_ts: now,      │
│    │    base_next_date: nextDateRepr,  base_next_date_ts: now,       │
│    │  }) → 两套字段初始化为相同值                                      │
│    │                                                                  │
│    └─ db.insertWithSchema('schedules', ...) → 写入计划表              │
└─────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────┐
│                        计划推进阶段（写入交易）                       │
├─────────────────────────────────────────────────────────────────────────┤
│  触发时机：sync 事件 + 每天执行一次（lastScheduleRun 检查）           │
│                                                                       │
│  advanceSchedulesService()                                           │
│    ├─ 查询所有未完成计划                                              │
│    ├─ getHasTransactionsQuery() → 检查是否有匹配交易                │
│    ├─ getStatus() → 判断状态 (paid/due/missed)                     │
│    │                                                                  │
│    ├─ [paid] 已付款                                                   │
│    │   ├─ 递归计划 → setNextDate({ reset: false })                  │
│    │   │    ├─ getNextDate() → 计算新日期                            │
│    │   │    └─ db.update schedules_next_date                          │
│    │   │         { local_next_date, local_next_date_ts: base_ts }    │
│    │   │         (保持 base 不变，仅更新 local)                      │
│    │   │                                                             │
│    │   └─ 单次计划 → 日期已过 → completed=true                       │
│    │                                                                  │
│    └─ [due/missed] + posts_transaction                              │
│        └─ postTransactionForSchedule()                              │
│            └─ addTransactions() → 写入交易表                        │
│                                                                       │
│  跳过操作入口：                                                        │
│  skipNextDate() → setNextDate({ reset: false, skipRequested: true }) │
└─────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────┐
│                        预算求值阶段（返回计算值）                       │
├─────────────────────────────────────────────────────────────────────────┤
│  触发时机：预算模板求值时                                              │
│                                                                       │
│  ⚠️ 重要说明：以下流程仅计算并返回值，不直接写入预算视图              │
│                                                                       │
│  CategoryTemplateContext.init(templates, category, month)            │
│    ├─ 计算上月余额 fromLastMonth                                       │
│    ├─ 校验：checkByAndScheduleAndSpend()                              │
│    └─ 分类：templates / remainder / goals                              │
│                                                                       │
│  runTemplatesForPriority(priority, budgetAvail, availStart)          │
│    ├─ 循环处理同优先级模板                                             │
│    │                                                                  │
│    │  ┌─ schedule 模板首次命中                                       ─┐│
│    │  │                                                             ││
│    │  │  runSchedule(...)  ← 传入累计 to_budget                    ││
│    │  │    ├─ createScheduleList() → 计算每个计划的当期目标            ││
│    │  │    ├─ 分类：pay-month-of vs sinking                          ││
│    │  │    └─ 返回 { to_budget, errors, remainder,                   ││
│    │  │             perScheduleMonthly }                            ││
│    │  │                                                             ││
│    │  │  newBudget = ret.to_budget - toBudget                       ││
│    │  │  (计算增量，避免重复累计)                                     ││
│    │  │                                                             ││
│    │  └───────────────────────────────────────────────────────────────┘│
│    │                                                                  │
│    ├─ 累加 toBudget += newBudget                                      │
│    ├─ redistributeBatch() → 按权重分配到各模板行                       │
│    │   (schedule 使用 perScheduleMonthly 作为权重)                     │
│    ├─ limit 检查 + clamp                                              │
│    ├─ hideDecimal 四舍五入                                             │
│    ├─ available clamp（非收入类别）                                  │
│    └─ 写入 perTemplateContribution Map                                │
│                                                                       │
│  runRemainder(budgetAvail, perWeight) → 处理剩余权重模板               │
│                                                                       │
│  getValues()                                                          │
│    ├─ runGoal() → 计算 goalAmount                                      │
│    └─ 返回 { budgeted, goal, longGoal, perTemplateContribution }      │
│                                                                       │
│  ⚠️ 实际落库：由上层调用者处理                                        │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 五、关键测试覆盖

已验证的测试场景（`packages/loot-core/src/server/budget/schedule-template.test.ts`）：

| 测试用例 | 行号 | 验证点 |
|---------|------|--------|
| 月度递归计划 | 108-138 | 正确计算全额预算 |
| 年度计划余额充足 | 140-170 | 按月分摊 |
| 名称空格处理 | 172-201 | trim 后正确匹配 |
| pay-month-of + sinking 混合 | 203-246 | 分别计算 |
| `full: true` | 248-276 | 到期前不预算 |
| 百分比调整 | 278-307 | `amount × 1.1` |
| 固定金额调整 | 309-338 | `amount + $5` |
| 已完成计划 | 340-366 | 从预算排除 |
| 日度计划跨月 | 368-395 | 计算月度内所有天数 |
| sinking 计划排序 | 397-439 | 按到期日期分配 |
| 非递归过期计划 | 441-482 | Past 错误 |
| 双月计划 | 484-515 | `target/interval` |
| 6周间隔 | 517-549 | 按月跨度分摊 |
| 60天间隔 | 551-583 | 按月跨度分摊 |
| 余额超目标 | 585-614 | 本月不预算 |
| Tracking 模式 | 616-647 | 全部 pay-month-of |

---

## 六、相关文件速查表

| 文件路径 | 核心职责 |
|---------|---------|
| `packages/loot-core/src/types/models/schedule.ts` | Schedule 类型定义 |
| `packages/loot-core/src/types/models/templates.ts` | ScheduleTemplate 类型定义 |
| `packages/loot-core/src/shared/schedules.ts` | 共享工具：getStatus, getNextDate, getScheduledAmount 等 |
| `packages/loot-core/src/server/schedules/app.ts` | 计划 CRUD + 自动推进服务 + schedules_next_date 更新逻辑 |
| `packages/loot-core/src/server/budget/schedule-template.ts` | 预算模板求值逻辑（runSchedule） |
| `packages/loot-core/src/server/budget/category-template-context.ts` | 分类模板上下文（调用 runSchedule 并汇总结果） |
| `packages/loot-core/src/server/util/rschedule.ts` | rSchedule 库封装 |
| `packages/loot-core/src/server/rules/rule.ts` | 规则执行（execActions） |
| `packages/loot-core/src/server/aql/schema/index.ts` | v_schedules 视图定义（next_date 选择逻辑） |
| `packages/loot-core/migrations/1618975177358_schedules.sql` | schedules_next_date 表结构 |
