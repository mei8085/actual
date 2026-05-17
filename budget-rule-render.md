# 自动化预算校验从规则到自然语言提示的完整链路分析

## 一、整体架构概览

自动化预算校验系统采用分层架构设计，从模板定义到用户可见的自然语言提示，经历以下核心阶段：

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│  模板定义与解析 │────▶│  规则校验与执行 │────▶│  结果渲染与展示 │
│  (Parsing)      │     │  (Execution)    │     │  (Rendering)    │
└─────────────────┘     └─────────────────┘     └─────────────────┘
          │                       │                       │
          ▼                       ▼                       ▼
  ┌──────────────┐       ┌──────────────┐       ┌──────────────┐
  │ PEG.js 语法  │       │ 优先级执行   │       │ i18n 多语言 │
  │ 解析器       │       │ 预算计算     │       │ 自然语言生成 │
  └──────────────┘       └──────────────┘       └──────────────┘
```

---

## 二、模板解析层：从文本到可执行规则

### 2.1 语法定义与解析器

**核心文件**：`packages/loot-core/src/server/budget/goal-template.pegjs`

系统使用 **PEG.js (Peggy)** 作为语法解析引擎，将用户编写的模板文本解析为结构化的 `Template` 对象。

#### 支持的模板类型（共12种）：

| 模板类型 | 语法示例 | 用途 |
|---------|---------|------|
| `simple` | `#template 500` | 每月固定预算金额 |
| `periodic` | `#template 100 repeat every 2 weeks starting 2024-01-01` | 周期性预算 |
| `percentage` | `#template 10% of Salary` | 按收入百分比预算 |
| `by` | `#template 1000 by 2024-12` | 截至日期储蓄目标 |
| `spend` | `#template 500 by 2024-06 spend from 2024-01` | 提前支出预算 |
| `schedule` | `#template schedule "Rent"` | 按账单日程预算 |
| `average` | `#template average 3 months` | 历史平均预算 |
| `copy` | `#template copy from 1 months ago` | 复制历史预算 |
| `remainder` | `#template remainder 2` | 剩余资金分配 |
| `limit` | `#template up to 2000` | 余额上限 |
| `refill` | `#template refill` | 自动补充到上限 |
| `goal` | `#goal 5000` | 长期目标 |

#### 解析器工作原理：

```pegjs
expr
  = template: template _ percent:percent _ of _ category: name
    { return { type: 'percentage', percent: +percent, category, ... }}
  / template: template _ amount: amount _ repeatEvery _ period: periodCount ...
    { return { type: 'periodic', amount, period, ... }}
  / ... 其他规则 ...
```

**关键特性**：
- 支持优先级标记：`#template-1 500` （优先级为1）
- 支持修饰符：`#template schedule "Rent" [increase 10%]`
- 支持限额：`#template 300 up to 2000`

### 2.2 模板存储与反序列化

**核心文件**：`packages/loot-core/src/server/budget/template-notes.ts`

#### 模板持久化：
- 模板存储在 `categories` 表的 `goal_def` 字段（JSON格式）
- 支持两种来源：`notes`（从备注解析）和 `ui`（从界面编辑器创建）

#### 解析流程（`storeNoteTemplates` 函数）：

```typescript
// 1. 从数据库读取带有模板备注的分类
const templateNotes = await getCategoriesWithTemplateNotes();

// 2. 逐行解析备注内容
note.split('\n').forEach(line => {
  const trimmedLine = line.substring(line.indexOf('#')).trim();
  if (trimmedLine.startsWith('#template') || trimmedLine.startsWith('#goal')) {
    try {
      const parsedTemplate = parse(trimmedLine);  // 调用PEG.js解析器
      parsedTemplates.push(parsedTemplate);
    } catch (e) {
      parsedTemplates.push({ type: 'error', line, error: e.message });
    }
  }
});

// 3. 存储到数据库
await storeTemplates({ categoriesWithTemplates, source: 'notes' });
```

#### 反序列化（`unparse` 函数）：
将结构化的 `Template` 对象转换回人类可读的文本格式，用于界面展示和编辑。

---

## 三、规则校验与执行层：与预算数据交互

### 3.1 模板上下文类

**核心文件**：`packages/loot-core/src/server/budget/category-template-context.ts`

`CategoryTemplateContext` 是模板执行的核心类，负责：
- 模板验证
- 预算金额计算
- 与预算工作表交互
- 优先级调度

#### 初始化流程：

```typescript
static async init(
  templates: Template[],
  category: CategoryEntity,
  month: string,
  budgeted: number,
  skipAvailableClamp: boolean = false,
) {
  // 1. 获取上月结余
  const fromLastMonth = await getSheetValue(lastMonthSheet, `leftover-${category.id}`);
  
  // 2. 运行前置校验
  await CategoryTemplateContext.checkByAndScheduleAndSpend(templates, month);
  await CategoryTemplateContext.checkPercentage(templates);
  
  // 3. 创建上下文实例
  return new CategoryTemplateContext(templates, category, month, fromLastMonth, ...);
}
```

### 3.2 模板校验机制

#### 分类级校验（静态校验）：

| 校验类型 | 检查内容 | 错误示例 |
|---------|---------|---------|
| Schedule校验 | 确保引用的日程存在 | `Schedule "Rent" does not exist` |
| By/Spend校验 | 确保目标日期未过期 | `Target month has passed` |
| Percentage校验 | 确保收入类别存在 | `Category "Bonus" is not found` |
| Limit校验 | 确保只有一个限额 | `Only one 'up to' allowed per category` |
| Goal校验 | 确保只有一个目标 | `Only one #goal is allowed per category` |

**代码位置**：`CategoryTemplateContext.checkByAndScheduleAndSpend()` (category-template-context.ts:465-517)

#### 全局冲突校验：

**核心文件**：`packages/desktop-client/src/components/budget/goals/validateAutomation.ts`

```typescript
// 检查百分比分配总和是否超过100%
export function validatePercentageAllocation(
  templates: readonly Template[],
): GlobalConflictKind | null {
  const percentBySource = new Map<string, number>();
  for (const t of templates) {
    if (t.type !== 'percentage') continue;
    const key = `${t.previous}|${t.category.toLowerCase()}`;
    percentBySource.set(key, (percentBySource.get(key) ?? 0) + t.percent);
  }
  const maxPercent = Math.max(0, ...percentBySource.values());
  return maxPercent > 100 ? { kind: 'percent-over-100', total: maxPercent } : null;
}
```

### 3.3 执行引擎：与预算数据交互

#### 预算工作表操作：

系统通过 `getSheetValue` 和 `setBudget`/`setGoal` 与预算数据交互：

```typescript
// 读取预算数据
const budgeted = await getSheetValue(sheetName, `budget-${categoryId}`);
const leftover = await getSheetValue(sheetName, `leftover-${categoryId}`);
const totalIncome = await getSheetValue(sheetName, `total-income`);

// 写入预算数据
await setBudget({ category, month, amount });
await setGoal({ month, category, goal, long_goal });
```

**核心文件**：`packages/loot-core/src/server/budget/actions.ts`

#### 优先级执行机制：

```typescript
// 按优先级排序执行
const priorities = new Int32Array([...prioritiesSet]).sort((a, b) => a - b);

for (const priority of priorities) {
  const availStart = availBudget;
  for (const templateContext of templateContexts) {
    // 执行该优先级下的所有模板
    const budget = await templateContext.runTemplatesForPriority(
      priority, availBudget, availStart
    );
    availBudget -= budget;
  }
}

// 最后分配剩余资金
distributeRemainder(templateContexts, availBudget);
```

#### 各模板类型的执行逻辑：

| 模板类型 | 执行函数 | 计算逻辑 |
|---------|---------|---------|
| `simple` | `runSimple()` | 直接返回固定金额 |
| `periodic` | `runPeriodic()` | 计算本月内发生的次数 × 单次金额 |
| `percentage` | `runPercentage()` | 收入金额 × 百分比 |
| `by` | `runBy()` | (目标金额 - 已存金额) / 剩余月数 |
| `spend` | `runSpend()` | 考虑提前支出期的分摊 |
| `schedule` | `runSchedule()` | 根据账单日程计算本月需求 |
| `average` | `runAverage()` | 历史N个月平均支出 |
| `copy` | `runCopy()` | 复制指定月份的预算 |

#### 限额处理机制：

```typescript
// 在每个优先级执行后检查限额
if (this.limitCheck) {
  if (toBudget + this.toBudgetAmount + this.fromLastMonth >= this.limitAmount) {
    const orig = toBudget;
    toBudget = this.limitAmount - this.toBudgetAmount - this.fromLastMonth;
    this.limitMet = true;
    available = available + orig - toBudget;
  }
}
```

### 3.4 模板贡献追踪

系统追踪每个模板对最终预算金额的贡献：

```typescript
// perTemplateContribution Map 记录每个模板的贡献
private perTemplateContribution = new Map<Template, number>();

// 在执行过程中累积
this.perTemplateContribution.set(template, existing + share);

// 最终通过 getValues() 返回
getValues() {
  return {
    budgeted: this.toBudgetAmount,
    goal: this.goalAmount,
    longGoal: this.isLongGoal,
    perTemplateContribution: this.perTemplateContribution,
  };
}
```

---

## 四、自然语言渲染层：从规则到用户可读句子

### 4.1 渲染架构

**核心文件**：`packages/desktop-client/src/components/budget/goals/TemplateSentence.tsx`

采用 **策略模式** 为每种模板类型提供专门的渲染组件：

```typescript
export function TemplateSentence({ template, categoryNameMap }: TemplateSentenceProps) {
  switch (template.type) {
    case 'limit':      return <LimitAutomationReadOnly template={template} />;
    case 'refill':     return <RefillAutomationReadOnly />;
    case 'periodic':   return <FixedAutomationReadOnly template={template} />;
    case 'schedule':   return <ScheduleAutomationReadOnly template={template} />;
    case 'percentage': return <PercentageAutomationReadOnly template={template} categoryNameMap={categoryNameMap} />;
    case 'average':
    case 'copy':       return <HistoricalAutomationReadOnly template={template} />;
    case 'by':
    case 'spend':      return <BySaveAutomationReadOnly template={template} />;
    case 'remainder':  return <RemainderAutomationReadOnly template={template} />;
    case 'goal':       return <LongTermGoalAutomationReadOnly template={template} />;
    default:           return <Trans>Unsupported template type: {{ type }}</Trans>;
  }
}
```

### 4.2 各类型渲染实现

#### 固定周期预算渲染（FixedAutomationReadOnly）：

**文件**：`packages/desktop-client/src/components/budget/goals/editor/FixedAutomationReadOnly.tsx`

```typescript
export function FixedAutomationReadOnly({ template }: FixedAutomationReadOnlyProps) {
  const format = useFormat();
  const amount = format(amountToInteger(template.amount, format.currency.decimalPlaces), 'financial');
  const periodAmount = template.period?.amount ?? 1;
  const periodUnit = template.period?.period ?? 'month';

  return (
    <Trans count={periodAmount}>
      Budget <FinancialText>{{ amount }}</FinancialText> every {{ count: periodAmount }} {periodUnit}s
    </Trans>
  );
}
```

**输出示例**：
- 英文：`Budget $500 every 2 weeks`
- 中文：`每2周预算 ¥500`

#### 百分比预算渲染（PercentageAutomationReadOnly）：

**文件**：`packages/desktop-client/src/components/budget/goals/editor/PercentageAutomationReadOnly.tsx`

```typescript
export const PercentageAutomationReadOnly = ({ template, categoryNameMap }) => {
  if (template.category === 'all income') {
    return template.previous ? (
      <Trans>Budget {{ percent: template.percent }}% of total income last month</Trans>
    ) : (
      <Trans>Budget {{ percent: template.percent }}% of total income this month</Trans>
    );
  }
  // ... 其他类别处理
};
```

**输出示例**：
- 英文：`Budget 10% of total income this month`
- 中文：`本月预算为总收入的10%`

#### 日程预算渲染（ScheduleAutomationReadOnly）：

**文件**：`packages/desktop-client/src/components/budget/goals/editor/ScheduleAutomationReadOnly.tsx`

```typescript
export const ScheduleAutomationReadOnly = ({ template }) => {
  if (template.full) {
    return (
      <Trans>
        Cover the occurrences of the schedule &lsquo;{{ name: template.name }}&rsquo; this month
      </Trans>
    );
  }
  return (
    <Trans>
      Save up for the schedule &lsquo;{{ name: template.name }}&rsquo;
    </Trans>
  );
};
```

**输出示例**：
- 英文：`Cover the occurrences of the schedule 'Rent' this month`
- 中文：`本月支付日程'房租'的费用`

#### 储蓄目标渲染（BySaveAutomationReadOnly）：

**文件**：`packages/desktop-client/src/components/budget/goals/editor/BySaveAutomationReadOnly.tsx`

```typescript
export const BySaveAutomationReadOnly = ({ template }) => {
  const format = useFormat();
  const amount = format(amountToInteger(template.amount, ...), 'financial');
  const month = formatMonthLabel(template.month, locale);
  
  return (
    <Trans>
      Save <FinancialText>{{ amount }}</FinancialText> by {{ month }}
    </Trans>
  );
};
```

**输出示例**：
- 英文：`Save $1,000 by December 2024`
- 中文：`在2024年12月前存够 ¥1,000`

### 4.3 国际化（i18n）机制

系统使用 `react-i18next` 实现多语言支持：

1. **Trans 组件**：用于包含变量的复杂句子
   ```typescript
   <Trans>
     Budget {{ percent: template.percent }}% of total income this month
   </Trans>
   ```

2. **t 函数**：用于简单文本
   ```typescript
   const { t } = useTranslation();
   const label = t('Budget');
   ```

3. **复数处理**：使用 `count` 属性
   ```typescript
   <Trans count={periodAmount}>
     Budget {{ amount }} every {{ count: periodAmount }} months
   </Trans>
   ```

---

## 五、界面渲染层与错误提示层协作

### 5.1 预算自动化模态框

**核心文件**：`packages/desktop-client/src/components/modals/BudgetAutomationsModal/`

#### 整体布局：

```
BudgetAutomationsModal
├── Header (标题 + 关闭按钮)
├── ConflictBanner (全局冲突警告)
├── AutomationListRow (自动化规则列表)
│   ├── 图标
│   ├── 类型标签
│   ├── TemplateSentence (自然语言描述)
│   ├── 贡献金额
│   └── 优先级标签
└── AutomationEditorPane (编辑器面板)
    ├── TypePicker (类型选择器)
    └── 各类型编辑器表单
```

### 5.2 自动化列表行（AutomationListRow）

**文件**：`packages/desktop-client/src/components/modals/BudgetAutomationsModal/AutomationListRow.tsx`

#### 状态驱动的视觉渲染：

```typescript
const borderColor = isActive
  ? theme.tableBorderSelected
  : error
    ? theme.errorBorder      // 错误状态：红色边框
    : 'transparent';

const backgroundColor = isActive
  ? theme.upcomingBackground
  : error
    ? theme.errorBackground  // 错误状态：红色背景
    : 'transparent';

const titleColor = error ? theme.errorText : theme.pageText;
```

#### 内容渲染逻辑：

```typescript
const subtitle = error ? (
  <AutomationErrorShort error={error} />    // 错误时显示错误信息
) : entry.template.type === 'limit' ? (
  <LimitAutomationShort template={entry.template} />
) : (
  <TemplateSentence                          // 正常时显示自然语言描述
    template={entry.template}
    categoryNameMap={categoryNameMap}
  />
);
```

### 5.3 错误提示系统

**核心文件**：`packages/desktop-client/src/components/budget/goals/automationMessages.tsx`

#### 三层错误信息设计：

| 组件 | 用途 | 显示位置 |
|-----|------|---------|
| `AutomationErrorTitle` | 简短错误标题 | 编辑器面板顶部 |
| `AutomationErrorShort` | 单行错误摘要 | 列表行副标题 |
| `AutomationErrorDetail` | 详细错误说明 + 修复建议 | 工具提示/帮助面板 |

#### 错误类型与消息映射：

```typescript
export function AutomationErrorShort({ error }) {
  switch (error.kind) {
    case 'schedule-not-found':
      return error.name 
        ? <Trans>No schedule named &ldquo;{{ name: error.name }}&rdquo;</Trans>
        : <Trans>Pick a schedule</Trans>;
    case 'percentage-out-of-range':
      return <Trans>{{ percent: error.percent }}% must be between 0 and 100</Trans>;
    case 'by-target-past':
      return <Trans>{{ month: formatMonthLabel(error.month, locale) }} has already passed</Trans>;
    // ... 其他错误类型
  }
}
```

#### 全局冲突提示（ConflictBanner）：

**文件**：`packages/desktop-client/src/components/modals/BudgetAutomationsModal/ConflictBanner.tsx`

用于展示跨规则的冲突，如总预算超过收入、百分比总和超过100%等。

### 5.4 只读展示组件

**核心文件**：`packages/desktop-client/src/components/budget/goals/BudgetAutomationReadOnly.tsx`

在预算表格单元格中展示自动化规则的摘要信息：

```typescript
export function BudgetAutomationReadOnly({ state, categoryNameMap, ... }) {
  let automationReadOnly;
  switch (state.displayType) {
    case 'fixed':
      automationReadOnly = <FixedAutomationReadOnly template={state.template} />;
    case 'schedule':
      automationReadOnly = <ScheduleAutomationReadOnly template={state.template} />;
    // ... 其他类型
  }
  
  return (
    <SpaceBetween gap={10}>
      <Text style={{ color: theme.tableText, fontSize: 13 }}>
        {automationReadOnly}
      </Text>
      {/* 编辑/删除按钮 */}
    </SpaceBetween>
  );
}
```

---

## 六、完整数据流示例

### 6.1 正常流程：从模板到展示

```
用户输入: "#template 10% of Salary"
        ↓
[PEG.js 解析器] goal-template.pegjs
        ↓
Template 对象: {
  type: 'percentage',
  percent: 10,
  category: 'Salary',
  priority: 0,
  directive: 'template'
}
        ↓
[存储] storeTemplates() → categories.goal_def
        ↓
[执行] CategoryTemplateContext.runPercentage()
        ↓
计算: 本月Salary收入 × 10% = 预算金额
        ↓
[渲染] PercentageAutomationReadOnly
        ↓
用户可见: "Budget 10% of 'Salary' this month"
```

### 6.2 错误流程：校验失败的展示

```
用户输入: "#template schedule 'NonExistent'"
        ↓
[PEG.js 解析] → 语法正确
        ↓
[校验] validateAutomation()
        ↓
错误: { kind: 'schedule-not-found', name: 'NonExistent' }
        ↓
[渲染] AutomationErrorShort
        ↓
用户可见: "No schedule named 'NonExistent'"
        ↓
[视觉反馈] 红色边框 + 警告图标 + 工具提示详情
```

---

## 七、关键设计模式与技术决策

### 7.1 设计模式

1. **解释器模式**：PEG.js 语法解析器将文本模板解释为可执行对象
2. **策略模式**：每种模板类型有独立的执行和渲染策略
3. **组合模式**：多个模板可以组合在一个分类下，按优先级执行
4. **状态模式**：根据校验结果（正常/错误）展示不同的UI状态

### 7.2 技术决策

| 决策 | 理由 | 权衡 |
|-----|------|-----|
| 使用 PEG.js 进行模板解析 | 语法灵活，支持自定义DSL | 学习成本较高，错误信息不够友好 |
| 模板与执行分离 | 同一模板可以在不同月份/上下文执行 | 需要维护上下文状态 |
| 优先级执行机制 | 确保重要预算优先分配 | 增加了执行复杂度 |
| i18next 作为国际化方案 | 支持复杂插值和复数 | 翻译文件维护成本 |
| 三层错误信息设计 | 不同场景展示不同粒度的错误 | 增加了组件数量 |

### 7.3 扩展点

1. **新模板类型**：只需添加 PEG.js 规则 + 执行函数 + 渲染组件
2. **新校验规则**：在 `validateAutomation` 或 `CategoryTemplateContext` 中添加
3. **新展示形式**：扩展 `TemplateSentence` 或创建新的只读组件

---

## 八、核心文件索引

| 层级 | 文件路径 | 主要职责 |
|-----|---------|---------|
| **解析层** | `packages/loot-core/src/server/budget/goal-template.pegjs` | PEG.js 语法定义 |
| | `packages/loot-core/src/server/budget/template-notes.ts` | 模板存储与解析入口 |
| **执行层** | `packages/loot-core/src/server/budget/category-template-context.ts` | 模板执行上下文 |
| | `packages/loot-core/src/server/budget/goal-template.ts` | 模板应用入口 |
| | `packages/loot-core/src/server/budget/actions.ts` | 预算工作表操作 |
| **校验层** | `packages/desktop-client/src/components/budget/goals/validateAutomation.ts` | 前端校验逻辑 |
| **渲染层** | `packages/desktop-client/src/components/budget/goals/TemplateSentence.tsx` | 自然语言渲染入口 |
| | `packages/desktop-client/src/components/budget/goals/editor/*ReadOnly.tsx` | 各类型只读组件 |
| **UI层** | `packages/desktop-client/src/components/modals/BudgetAutomationsModal/` | 自动化模态框 |
| | `packages/desktop-client/src/components/budget/goals/automationMessages.tsx` | 错误消息组件 |
| | `packages/desktop-client/src/components/budget/goals/BudgetAutomationReadOnly.tsx` | 预算表格展示 |
| **类型定义** | `packages/loot-core/src/types/models/templates.ts` | Template 类型定义 |
| | `packages/desktop-client/src/components/budget/goals/constants.ts` | 显示类型映射 |
