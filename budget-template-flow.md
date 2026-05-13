# 预算模板（Budget Template）完整流程报告

## 概述

预算模板系统允许用户为预算类别定义自动化规则，系统根据这些规则自动计算月度预算金额。该系统支持多种模板类型，包括固定金额、周期性支出、收入百分比、目标储蓄等。

---

## 完整链路概览

### 数据流向图

```
┌─────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│  配置输入层     │────>│  存储层           │────>│  计算引擎层      │
│  (UI/Notes)     │     │  (Database)      │     │  (Server)        │
└─────────────────┘     └──────────────────┘     └──────────────────┘
                                                    │
                                                    v
                                            ┌──────────────────┐
                                            │  月度数据层       │
                                            │  (Sheet)         │
                                            └──────────────────┘
```

---

## 模块详解

### 1. 配置输入层

预算模板配置有两种来源：

#### 1.1 UI 配置（推荐方式）
- **入口组件**：`CategoryAutomationButton.tsx`
  - 位置：`packages/desktop-client/src/components/budget/goals/CategoryAutomationButton.tsx`
  - 功能：在预算表格中显示自动化按钮，点击后打开编辑器
  - 触发条件：需要启用 `goalTemplatesEnabled` 和 `goalTemplatesUIEnabled` 特性标志

- **编辑器组件**：`AutomationEditorPane.tsx`
  - 位置：`packages/desktop-client/src/components/modals/BudgetAutomationsModal/AutomationEditorPane.tsx`
  - 功能：提供图形化界面编辑模板
  - 支持的模板类型选择：`TypePicker.tsx`
  - 状态管理：使用 `templateReducer` 管理编辑状态

#### 1.2 笔记模板（传统方式）
- **语法示例**：
  - `#template 100` - 每月固定 100
  - `#template-1 500 by 2024-12` - 优先级 1，到 2024 年 12 月攒够 500
  - `#template 10% of Salary` - 收入的 10%
  - `#goal 10000` - 设置目标金额

- **解析器**：`goal-template.pegjs`
  - 位置：`packages/loot-core/src/server/budget/goal-template.pegjs`
  - 功能：将文本语法解析为结构化 Template 对象

### 2. 存储层

#### 2.1 配置存储
- **存储位置**：`categories` 表的 `goal_def` 字段
  - 类型：JSON 字符串
  - 存储内容：Template 数组

- **相关字段**：
  - `goal_def`: 模板配置 JSON
  - `template_settings`: 模板来源信息（'notes' 或 'ui'）

#### 2.2 存储操作函数
- **读取配置**：`getTemplates()` / `getTemplatesForCategory()`
  - 文件：`packages/loot-core/src/server/budget/goal-template.ts:145-170`
  - 逻辑：查询 `categories` 表中 `goal_def` 不为空的记录，解析 JSON

- **保存配置**：`storeTemplates()`
  - 文件：`packages/loot-core/src/server/budget/goal-template.ts:44-66`
  - 逻辑：将 Template 数组序列化为 JSON，更新 `categories` 表

#### 2.3 笔记模板同步
- **同步函数**：`storeNoteTemplates()`
  - 文件：`packages/loot-core/src/server/budget/template-notes.ts:22-28`
  - 功能：从类别笔记中解析模板并同步到 `goal_def` 字段
  - 调用时机：
    - 前端加载自动化时自动调用（`useBudgetAutomations` hook）
    - 应用模板前手动调用

### 3. 计算引擎层（核心）

计算引擎是整个系统的核心，负责将模板配置转换为具体的月度预算金额。

#### 3.1 核心类：`CategoryTemplateContext`
- **文件**：`packages/loot-core/src/server/budget/category-template-context.ts`
- **职责**：
  1. 初始化上下文（获取上月结转、货币设置等）
  2. 验证模板配置
  3. 按优先级执行模板计算
  4. 处理限制和剩余资金分配
  5. 汇总最终预算金额

#### 3.2 核心流程：`computeTemplates()`
- **文件**：`packages/loot-core/src/server/budget/goal-template.ts:214-297`
- **执行步骤**：

```
Step 1: 初始化
  ├─ 获取所有可见类别
  ├─ 获取本月可用预算（to-budget）
  └─ 遍历每个类别

Step 2: 上下文初始化（CategoryTemplateContext.init）
  ├─ 获取上月结转余额
  ├─ 检查 carryover 设置
  ├─ 获取货币和小数显示偏好
  ├─ 验证模板配置（checkByAndScheduleAndSpend, checkPercentage）
  └─ 分类模板（普通模板、目标模板、剩余分配模板）

Step 3: 检查限制（checkLimit）
  ├─ 解析 limit 模板
  ├─ 计算每日/每周/每月限制金额
  └─ 检查是否已达到限制

Step 4: 按优先级执行模板
  ├─ 收集所有优先级级别并排序
  ├─ 对每个优先级：
  │   ├─ 执行该优先级的所有模板
  │   ├─ 检查可用预算
  │   ├─ 应用限制
  │   └─ 更新可用预算
  └─ 支持的模板类型：
      ├─ simple: 固定金额
      ├─ periodic: 周期性支出
      ├─ by: 目标日期前攒够
      ├─ spend: 分期消费
      ├─ percentage: 收入百分比
      ├─ schedule: 计划账单
      ├─ average: 历史平均
      ├─ copy: 复制历史预算
      └─ refill: 补充到限制金额

Step 5: 分配剩余资金
  ├─ 收集所有 remainder 模板
  ├─ 按权重分配剩余可用预算
  └─ 应用限制

Step 6: 汇总结果
  ├─ 计算目标金额（runGoal）
  └─ 返回预算金额、目标金额等
```

#### 3.3 模板类型详解

| 模板类型 | 关键字 | 计算逻辑 | 典型用例 |
|---------|--------|---------|---------|
| Simple | `#template 100` | 固定金额 | 每月固定支出 |
| Periodic | `#template 50 repeat every 3 months` | 按周期计算 | 季度订阅、年度会员 |
| By | `#template 1000 by 2024-12` | 目标日期前平均分配 | 假期储蓄、大额采购 |
| Spend | `#template 500 by 2024-06 spend from 2024-01` | 从指定月份开始分摊 | 旅行预算 |
| Percentage | `#template 10% of Salary` | 收入百分比 | 储蓄比例、投资计划 |
| Schedule | `#template schedule "Netflix"` | 基于账单计划 | 自动匹配账单金额 |
| Average | `#template average 3 months` | 过去 N 月平均支出 | 可变支出预估 |
| Copy | `#template copy from 1 months ago` | 复制历史 | 与上月相同 |
| Remainder | `#template remainder 3` | 按权重分配剩余资金 | 弹性支出类别 |
| Limit | `#template up to 500` | 设置上限 | 控制消费上限 |
| Refill | `#template refill` | 补充到限制金额 | 保持余额充足 |
| Goal | `#goal 10000` | 设置长期目标 | 储蓄目标显示 |

#### 3.4 关键计算函数

**固定金额 (simple)**：
```typescript
// 文件: category-template-context.ts:648-660
static runSimple(template, context): number {
  if (template.monthly != null) {
    return amountToInteger(template.monthly, context.currency.decimalPlaces);
  } else {
    return context.limitAmount - context.fromLastMonth;
  }
}
```

**周期性支出 (periodic)**：
```typescript
// 文件: category-template-context.ts:682-736
static runPeriodic(template, context): number {
  // 计算周期内需要的金额
  // 支持日、周、月、年周期
  // 考虑起始日期
}
```

**目标储蓄 (by)**：
```typescript
// 文件: category-template-context.ts:894-974
static runBy(context): { toBudget: number; perTemplateNeed: Map } {
  // 找到最短时间范围
  // 计算每月需要储蓄的金额
  // 考虑已有余额
}
```

**收入百分比 (percentage)**：
```typescript
// 文件: category-template-context.ts:807-851
static runPercentage(template, availableFunds, context): number {
  // 支持 "all income"、"available funds" 或特定收入类别
  // 支持上月收入或本月收入
}
```

**剩余资金分配 (remainder)**：
```typescript
// 文件: category-template-context.ts:329-372
runRemainder(budgetAvail, perWeight): number {
  // 按权重分配剩余可用预算
  // 考虑限制金额
}
```

### 4. 应用层

#### 4.1 应用模板函数

- **全量应用**：`applyTemplate({ month })`
  - 文件：`goal-template.ts:68-77`
  - 特性：只填充未预算的类别（`force=false`）
  - 适用场景：月初自动填充空白类别

- **覆盖应用**：`overwriteTemplate({ month })`
  - 文件：`goal-template.ts:79-88`
  - 特性：强制重新计算所有类别（`force=true`）
  - 适用场景：修改模板后重新计算

- **单类别应用**：`applySingleCategoryTemplate({ month, category })`
  - 文件：`goal-template.ts:113-132`
  - 适用场景：单个类别模板修改后应用

- **多类别应用**：`applyMultipleCategoryTemplates({ month, categoryIds })`
  - 文件：`goal-template.ts:90-111`
  - 适用场景：批量更新多个类别

#### 4.2 应用流程

```
1. storeNoteTemplates() - 同步笔记模板到数据库
2. getTemplates() - 读取所有模板配置
3. computeTemplates() - 计算每个类别的预算金额
4. setBudgets() - 批量保存预算金额
5. setGoals() - 批量保存目标金额
```

#### 4.3 保存操作

- **保存预算**：`setBudgets()`
  - 文件：`goal-template.ts:177-187`
  - 调用 `actions.setBudget()` 更新电子表格

- **保存目标**：`setGoals()`
  - 文件：`goal-template.ts:195-206`
  - 调用 `actions.setGoal()` 更新电子表格

### 5. 月度数据层

#### 5.1 数据存储结构

预算数据存储在电子表格（Sheet）系统中，每个月一个 sheet。

**关键单元格**：
- `budget-{categoryId}`: 预算金额
- `goal-{categoryId}`: 目标金额
- `long-goal-{categoryId}`: 是否为长期目标
- `leftover-{categoryId}`: 上月结转余额
- `carryover-{categoryId}`: 是否允许结转
- `to-budget`: 本月可用预算
- `total-income`: 本月总收入

#### 5.2 数据读取函数

- **获取单元格值**：`getSheetValue(sheetName, cellName)`
  - 文件：`packages/loot-core/src/server/budget/actions.ts`
  - 用途：读取现有预算、上月余额等

- **设置单元格值**：`setBudget()` / `setGoal()`
  - 文件：`packages/loot-core/src/server/budget/actions.ts`
  - 用途：写入计算后的预算和目标

---

## 跨模块数据流向

### 完整流程图

```
┌─────────────────────────────────────────────────────────────────────┐
│                         前端层 (desktop-client)                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  CategoryAutomationButton                                           │
│         │                                                           │
│         v                                                           │
│  BudgetAutomationsModal (编辑模板)                                  │
│         │                                                           │
│         │ send('budget/set-category-automations', ...)              │
│         v                                                           │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                    连接层 (Connection)                      │    │
│  │  - 序列化请求                                               │    │
│  │  - 发送到后端                                               │    │
│  └─────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              v
┌─────────────────────────────────────────────────────────────────────┐
│                         后端层 (loot-core/server)                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  app.ts (预算应用)                                                  │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  'budget/set-category-automations'                          │   │
│  │  'budget/get-category-automations'                          │   │
│  │  'budget/apply-goal-template'                               │   │
│  │  'budget/overwrite-goal-template'                           │   │
│  │  'budget/apply-single-template'                             │   │
│  └─────────────────────────────────────────────────────────────┘   │
│         │                                                           │
│         v                                                           │
│  goal-template.ts (核心逻辑)                                        │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  storeTemplates() - 保存配置                                │   │
│  │  getTemplates() - 读取配置                                  │   │
│  │  computeTemplates() - 计算引擎                              │   │
│  │  applyTemplate() - 应用模板                                 │   │
│  └─────────────────────────────────────────────────────────────┘   │
│         │                                                           │
│         v                                                           │
│  category-template-context.ts (执行上下文)                          │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  CategoryTemplateContext.init()                             │   │
│  │  runTemplatesForPriority() - 按优先级执行                   │   │
│  │  runRemainder() - 剩余资金分配                              │   │
│  │  getValues() - 获取结果                                     │   │
│  └─────────────────────────────────────────────────────────────┘   │
│         │                                                           │
│         v                                                           │
│  template-notes.ts (笔记模板)                                       │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  storeNoteTemplates() - 同步笔记到数据库                     │   │
│  │  parse() - 解析 PEG.js 语法                                 │   │
│  │  unparse() - 序列化回文本                                   │   │
│  └─────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              v
┌─────────────────────────────────────────────────────────────────────┐
│                         数据层                                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  数据库 (SQLite)                                                    │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  categories 表                                              │   │
│  │    ├─ id (主键)                                             │   │
│  │    ├─ goal_def (模板配置 JSON)                              │   │
│  │    ├─ template_settings (来源信息)                          │   │
│  │    └─ ...                                                   │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  电子表格 (Sheet)                                                    │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  每月一个 sheet (如: month-2024-01)                         │   │
│  │    ├─ budget-{catId}: 预算金额                              │   │
│  │    ├─ goal-{catId}: 目标金额                                │   │
│  │    ├─ leftover-{catId}: 结转余额                            │   │
│  │    └─ to-budget: 可用预算                                   │   │
│  └─────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

### 关键数据结构

#### Template 类型定义
```typescript
// 文件: packages/loot-core/src/types/models/templates.ts

type Template =
  | PercentageTemplate    // 百分比模板
  | PeriodicTemplate      // 周期性模板
  | ByTemplate            // 目标日期模板
  | SpendTemplate         // 消费分期模板
  | SimpleTemplate        // 简单固定模板
  | ScheduleTemplate      // 账单计划模板
  | RemainderTemplate     // 剩余分配模板
  | AverageTemplate       // 历史平均模板
  | GoalTemplate          // 目标模板
  | CopyTemplate          // 复制历史模板
  | RefillTemplate        // 补充模板
  | LimitTemplate         // 限制模板
  | ErrorTemplate;        // 错误模板
```

#### CategoryTemplateContext 状态
```typescript
// 关键状态字段
private templates: Template[] = [];           // 普通模板
private remainder: RemainderTemplate[] = [];  // 剩余分配模板
private goals: GoalTemplate[] = [];           // 目标模板
private priorities: Set<number> = new Set();  // 优先级集合
private toBudgetAmount: number = 0;           // 计算出的预算金额
private goalAmount: number | null = null;     // 目标金额
private fromLastMonth: number = 0;            // 上月结转
private limitAmount: number = 0;              // 限制金额
private limitCheck: boolean = false;          // 是否有限制
private limitMet: boolean = false;            // 是否已达限制
```

---

## 触发时机

### 自动触发
1. **加载预算页面**：自动同步笔记模板到数据库
2. **切换月份**：可能触发模板应用（取决于用户设置）

### 手动触发
1. **点击自动化按钮**：打开编辑器查看/修改模板
2. **保存模板**：保存配置到数据库
3. **应用模板**：点击"应用"按钮触发计算
4. **检查模板**：验证模板语法正确性

---

## 测试覆盖

测试文件：`packages/loot-core/src/server/budget/goal-template.test.ts`

### 关键测试场景

1. **dryRunCategoryTemplate** - 预计算测试
   - 单模板计算验证
   - 多模板叠加和限制验证
   - 类别不存在时的处理
   - 资金不足时的需求展示

2. **applyMultipleCategoryTemplates** - 多类别应用测试
   - 成功应用多个类别
   - 优先级资金分配（高优先级先获得资金）
   - 模板验证错误处理
   - 孤立目标清理
   - 剩余资金按权重分配

3. **applyTemplate** - 全量应用测试
   - 跳过已预算类别（force=false）

---

## 关键文件索引

| 文件路径 | 职责 |
|---------|------|
| `packages/loot-core/src/server/budget/goal-template.ts` | 核心逻辑：存储、读取、计算、应用 |
| `packages/loot-core/src/server/budget/category-template-context.ts` | 计算引擎：模板执行上下文 |
| `packages/loot-core/src/server/budget/template-notes.ts` | 笔记模板：解析、同步、序列化 |
| `packages/loot-core/src/server/budget/app.ts` | API 路由：定义预算相关方法 |
| `packages/loot-core/src/types/models/templates.ts` | 类型定义：Template 类型系统 |
| `packages/desktop-client/src/hooks/useBudgetAutomations.ts` | 前端 Hook：加载自动化配置 |
| `packages/desktop-client/src/components/budget/goals/CategoryAutomationButton.tsx` | UI 入口：自动化按钮 |
| `packages/desktop-client/src/components/modals/BudgetAutomationsModal/` | 编辑器：模板编辑界面 |

---

## 扩展思考

### 系统优势
1. **灵活性**：支持多种模板类型，覆盖大部分预算场景
2. **优先级**：通过优先级机制确保重要支出先获得资金
3. **限制机制**：防止超预算，支持每日/每周/每月限制
4. **剩余分配**：智能分配剩余资金到弹性类别
5. **双输入**：支持 UI 配置和笔记语法两种方式

### 潜在优化点
1. **批量计算优化**：当前逐类别计算，可考虑并行化
2. **缓存机制**：模板配置变化不频繁，可增加缓存
3. **增量计算**：只重新计算受影响的类别
4. **预测能力**：基于模板预测未来数月的预算需求

---

## 总结

预算模板系统是 Actual Budget 的核心功能之一，通过以下链路实现自动化预算：

1. **配置**：用户通过 UI 或笔记语法定义模板规则
2. **存储**：模板配置以 JSON 格式存储在数据库中
3. **计算**：`CategoryTemplateContext` 执行复杂的预算计算
4. **应用**：计算结果写入月度电子表格
5. **展示**：前端从电子表格读取并展示预算数据

该系统设计优雅，通过优先级、限制、剩余分配等机制，实现了智能且灵活的自动化预算管理。
