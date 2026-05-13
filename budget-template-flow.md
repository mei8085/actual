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

预算模板配置有两种来源：UI 配置和笔记模板。系统通过 `template_settings.source` 字段标记来源，避免笔记覆盖 UI 保存的配置。

#### 1.1 UI 配置（推荐方式）

- **入口组件**：`CategoryAutomationButton.tsx`
  - 位置：`packages/desktop-client/src/components/budget/goals/CategoryAutomationButton.tsx`
  - 功能：在预算表格中显示自动化按钮，点击后打开编辑器
  - 触发条件：需要启用 `goalTemplatesEnabled` 和 `goalTemplatesUIEnabled` 特性标志
  - 限制：收入类别仅在 tracking 预算类型中支持模板

- **编辑器组件**：`BudgetAutomationsBody.tsx`
  - 位置：`packages/desktop-client/src/components/modals/BudgetAutomationsModal/BudgetAutomationsBody.tsx`
  - 核心功能：
    1. 实时预览计算：使用 `debounce` 调用 `budget/dry-run-category-template`
    2. 前端校验：`validateAutomation()` 检查模板合法性
    3. 保存时标记来源：`source: 'ui'`
  - 保存逻辑（`onSave` 函数）：
    ```typescript
    // BudgetAutomationsBody.tsx:213-228
    const onSave = async () => {
      const templatesToSave = entries.map(({ template }) => template);
      await send('budget/set-category-automations', {
        categoriesWithTemplates: [
          { id: categoryId, templates: templatesToSave },
        ],
        source: 'ui',  // 关键：标记为 UI 来源
      });
    };
    ```

- **编辑器面板**：`AutomationEditorPane.tsx`
  - 位置：`packages/desktop-client/src/components/modals/BudgetAutomationsModal/AutomationEditorPane.tsx`
  - 功能：提供图形化界面编辑单个模板
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

#### 1.3 来源标记与隔离机制

**关键机制：`template_settings.source` 字段**

系统通过在 `categories` 表的 `template_settings` 字段中标记来源，实现 UI 配置与笔记模板的隔离：

- `source: 'ui'`：通过图形界面保存的模板
- `source: 'notes'` 或 `null`：从笔记解析的模板（默认值）

**隔离逻辑实现**（`statements.ts`）：

1. **读取笔记模板时排除 UI 类别**：
   ```typescript
   // statements.ts:26-43
   SELECT c.id AS id, c.name as name, n.note AS note
   FROM notes n
          JOIN categories c ON n.id = c.id
   WHERE c.id = n.id
     AND c.tombstone = 0
     AND COALESCE(JSON_EXTRACT(c.template_settings, '$.source'), 'notes') <> 'ui'  -- 排除 UI 来源
     AND (lower(note) LIKE '%#template%' OR lower(note) LIKE '%#goal%')
   ```

2. **重置无模板类别时保留 UI 类别**：
   ```typescript
   // statements.ts:6-18
   UPDATE categories
   SET goal_def = NULL
   WHERE id NOT IN (SELECT n.id
                    FROM notes n
                    WHERE lower(note) LIKE '%#template%'
                       OR lower(note) LIKE '%#goal%')
     AND COALESCE(JSON_EXTRACT(template_settings, '$.source'), 'notes') <> 'ui'  -- 排除 UI 来源
   ```

**效果**：
- 一旦用户通过 UI 保存模板（`source: 'ui'`），该类别的笔记内容将不再被解析
- 即使后来删除了笔记中的模板，`goal_def` 也不会被自动清空
- 用户可以通过 "Un-migrate to text notes" 功能切换回笔记模式

### 2. 存储层

#### 2.1 配置存储

- **存储位置**：`categories` 表
- **关键字段**：
  - `goal_def`: 模板配置 JSON（Template 数组）
  - `template_settings`: JSON 对象，包含 `source` 字段标记来源

#### 2.2 存储操作函数

- **保存配置**：`storeTemplates()`
  - 文件：`packages/loot-core/src/server/budget/goal-template.ts:44-66`
  - 逻辑：将 Template 数组序列化为 JSON，更新 `categories` 表
  - 参数 `source`：标记来源为 'notes' 或 'ui'

- **读取配置**：`getTemplates()` / `getTemplatesForCategory()`
  - 文件：`packages/loot-core/src/server/budget/goal-template.ts:145-170`
  - 逻辑：查询 `categories` 表中 `goal_def` 不为空的记录，解析 JSON

#### 2.3 笔记模板同步

- **同步函数**：`storeNoteTemplates()`
  - 文件：`packages/loot-core/src/server/budget/template-notes.ts:22-28`
  - **真实触发时机**（按用户操作路径）：

| 触发场景 | 触发路径 | 目的 |
|---------|---------|------|
| **打开自动化面板时** | `useBudgetAutomations` hook 自动调用 | 加载可能存在的笔记模板到编辑器 |
| **执行模板应用动作时** | `applyTemplate()` / `overwriteTemplate()` 等函数自动调用 | 确保最新的笔记模板被解析 |
| **从 UI 切回笔记模式时** | `UnmigrateBudgetAutomationsModal` 显式调用 | 切换控制权：清空 UI 模板，让笔记成为数据源 |

  - **不触发的场景**：
    - 普通预算页面加载时**不会**自动同步
    - 切换月份时**不会**自动同步

#### 2.4 从自动化界面切回笔记模式（Un-migrate）

**功能描述**：用户可以通过 "Un-migrate to text notes" 功能，将 UI 管理的模板重新切换回笔记管理。

**触发组件**：`UnmigrateBudgetAutomationsModal.tsx`
  - 位置：`packages/desktop-client/src/components/modals/UnmigrateBudgetAutomationsModal.tsx`

**核心流程**：

```
1. 渲染预览
   ├─ useEffect 调用 'budget/render-note-templates'
   │    └─ template-notes.renderNoteTemplates()
   │         └─ unparse(templates)  → 转为文本语法
   │
2. 合并现有笔记
   │
3. 保存
   ├─ 第1步：send('notes-save-undoable')
   │         └─ 保存合并后的笔记到 notes 表
   │
   ├─ 第2步：send('budget/set-category-automations', {
   │         │         templates: [],
   │         │         source: 'notes'
   │         │       })
   │         └─ 清空 goal_def，标记 source: 'notes'
   │
   └─ 第3步：send('budget/store-note-templates')
             └─ 显式同步笔记模板
```

**为什么第 3 步显式调用的意义**：
- 第 2 步只是清空了 `goal_def` 并标记 `source: 'notes'`
- 但此时 `goal_def` 是 `null`，需要立即从笔记重新解析
- 确保控制权完整移交给笔记解析器
- 下次打开面板时会重新从笔记解析（因为 source 已改为 'notes'）

**与其他场景的关系**：

| 场景 | 调用方 | 调用时机 | 目的 |
|-----|-------|---------|------|
| **打开面板** | `useBudgetAutomations` hook | 模态框打开时 | 加载已有笔记模板 |
| **应用模板** | `applyTemplate()` 等 | 用户点击应用按钮 | 解析最新笔记 |
| **切回笔记** | `UnmigrateBudgetAutomationsModal` | 用户点击"Save notes & un-migrate" | 切换数据源 |

**关键区别**：

**场景对比**：

| 场景 | 方向 | 含义 |
|-----|------|------|
| **打开面板** | 笔记 → UI | 是"加载"动作：从笔记同步可能存在的模板到 UI 编辑器中，供用户查看和编辑。 |
| **应用模板** | 笔记/UI → Sheet | 是"解析+应用"动作：解析最新的笔记模板（排除 UI 来源），计算预算金额，写入月度电子表格。 |
| **切回笔记** | UI → 笔记 | 是"控制权移交"动作：先将 UI 模板导出到笔记、清空 UI 模板、切换 source 标记、最后显式同步笔记模板，确保笔记成为唯一数据源。 |

- **同步流程**：
  ```typescript
  // template-notes.ts:22-28
  export async function storeNoteTemplates(): Promise<void> {
    const categoriesWithTemplates = await getCategoriesWithTemplates();
    await storeTemplates({ categoriesWithTemplates, source: 'notes' });
    await resetCategoryGoalDefsWithNoTemplates();
  }
  ```

  三个关键步骤：
  1. `getCategoriesWithTemplates()`：从数据库读取含模板语法的笔记（排除 UI 来源类别）
  2. `storeTemplates()`：将解析后的模板保存到 `goal_def`，标记 `source: 'notes'`
  3. `resetCategoryGoalDefsWithNoTemplates()`：清理已删除笔记模板的 `goal_def`（排除 UI 来源类别）

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
  ├─ 获取所有可见类别（过滤隐藏类别和收入类别）
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

#### 3.3 模板校验失败分支

**校验时机**：
1. **前端校验**：`validateAutomation()` 在编辑时实时检查
2. **后端校验**：`CategoryTemplateContext.init()` 在计算前执行

**校验失败的返回路径**：

```
computeTemplates()
  │
  ├─ 遍历类别
  │   └─ CategoryTemplateContext.init()
  │       ├─ checkByAndScheduleAndSpend()  --> 检查 schedule/by/spend 模板
  │       └─ checkPercentage()            --> 检查百分比模板
  │       └─ 抛出异常 Error
  │
  ├─ 收集 errors 数组
  │
  ├─ if (errors.length > 0)
  │   └─ 立即返回 { contexts, errors, orphanGoals }
  │       │
  │       └─ processTemplate()
  │           └─ 返回错误通知:
  │              {
  │                sticky: true,
  │                message: 'There were errors interpreting some templates:',
  │                pre: errors.join('\n\n')
  │              }
  │
  └─ 成功继续执行
```

**前端处理**：`BudgetAutomationsBody.tsx:258-267`
- 使用 `validateAutomation()` 实时校验
- 有错误时禁用保存按钮
- 错误信息显示在编辑器面板中

#### 3.4 模板移除时的孤立目标清理

**场景**：某个类别之前有模板和目标，后来用户删除了所有模板

**清理流程**（`computeTemplates()` 中的 orphanGoals 处理）：

```typescript
// goal-template.ts:266-273
// do a reset of the goals that are orphaned
} else if (existingGoal !== null && !templates) {
  orphanGoals.push({
    category: id,
    goal: null,
    longGoal: null,
  });
}
```

**判断条件**：
- `existingGoal !== null`：该类别在电子表格中已有目标值
- `!templates`：该类别当前没有模板配置

**处理逻辑**：

```
processTemplate()
  │
  ├─ computeTemplates() 收集 orphanGoals
  │
  ├─ if (contexts.length === 0 && errors.length === 0)
  │   │
  │   └─ if (orphanGoals.length > 0)
  │       └─ setGoals(month, orphanGoals)
  │           └─ 将 goal 设置为 null，long_goal 设置为 null
  │
  └─ 正常情况下
      └─ goalList 包含 [...orphanGoals, ...contextGoals]
          └─ 一次性调用 setGoals()
```

**效果**：
- 删除模板后，电子表格中的目标值会被清理
- 侧边栏的目标指示器消失
- 避免用户困惑：为什么没有模板但还有目标？

#### 3.5 模板类型详解

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

#### 3.6 关键计算函数

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
1. storeNoteTemplates() - 同步笔记模板到数据库（排除 UI 来源类别）
2. getTemplates() - 读取所有模板配置
3. computeTemplates() - 计算每个类别的预算金额
   ├─ 校验模板（失败则返回错误）
   ├─ 按优先级计算
   ├─ 收集 orphanGoals（模板已移除的类别）
   └─ 返回 { contexts, errors, orphanGoals }
4. 处理返回
   ├─ if errors: 返回错误通知
   └─ else: setBudgets() + setGoals()（包含 orphanGoals 清理）
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

## 跨模块数据流向与调用链

### 完整流程图

```
┌─────────────────────────────────────────────────────────────────────┐
│                         前端层 (desktop-client)                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  CategoryAutomationButton (显示自动化图标)                           │
│         │                                                           │
│         │ 点击触发 pushModal('category-automations-edit')           │
│         v                                                           │
│  BudgetAutomationsModal (打开编辑窗口)                               │
│         │                                                           │
│         │ useBudgetAutomations hook                                  │
│         │   ├─ send('budget/store-note-templates')  [自动同步笔记]   │
│         │   └─ send('budget/get-category-automations', catId)       │
│         v                                                           │
│  BudgetAutomationsBody (主编辑器界面)                                │
│         │                                                           │
│         ├─ 实时预览: debounce send('budget/dry-run-category-...')   │
│         │                                                           │
│         └─ 保存: onSave()                                            │
│             └─ send('budget/set-category-automations', {            │
│                    categoriesWithTemplates: [...],                  │
│                    source: 'ui'  [标记 UI 来源]                      │
│                  })                                                 │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              │ send(...)
                              v
┌─────────────────────────────────────────────────────────────────────┐
│                         连接层 (Connection)                          │
├─────────────────────────────────────────────────────────────────────┤
│  - 序列化请求为 JSON                                                 │
│  - 通过 WebSocket 发送到后端                                         │
│  - 等待响应并反序列化                                                │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              v
┌─────────────────────────────────────────────────────────────────────┐
│                         后端层 (loot-core/server)                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  app.ts (预算应用路由注册)                                           │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  'budget/store-note-templates'                              │   │
│  │      └─ goalNoteActions.storeNoteTemplates()                │   │
│  │                                                             │   │
│  │  'budget/get-category-automations'                         │   │
│  │      └─ goalActions.getTemplatesForCategory(catId)         │   │
│  │                                                             │   │
│  │  'budget/set-category-automations'                         │   │
│  │      └─ goalActions.storeTemplates({ source })              │   │
│  │                                                             │   │
│  │  'budget/apply-goal-template'                              │   │
│  │      └─ goalActions.applyTemplate({ month })                │   │
│  │           ├─ storeNoteTemplates() [同步笔记]                │   │
│  │           ├─ getTemplates() [读取配置]                      │   │
│  │           └─ processTemplate() [计算并应用]                 │   │
│  │                                                             │   │
│  │  'budget/dry-run-category-template'                        │   │
│  │      └─ goalActions.dryRunCategoryTemplate()               │   │
│  │           └─ computeTemplates(skipAvailableClamp=true)     │   │
│  └─────────────────────────────────────────────────────────────┘   │
│         │                                                           │
│         v                                                           │
│  goal-template.ts (核心逻辑)                                        │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  storeTemplates()                                           │   │
│  │      ├─ storeNoteCleanups(catIds) [清理清理模板]            │   │
│  │      └─ db.updateWithSchema('categories', {                 │   │
│  │           goal_def: JSON.stringify(templates),              │   │
│  │           template_settings: { source }                     │   │
│  │         })                                                  │   │
│  │                                                             │   │
│  │  getTemplates()                                             │   │
│  │      └─ aqlQuery(q('categories')                            │   │
│  │           .filter({ goal_def: { $ne: null } })             │   │
│  │           .select('*'))                                    │   │
│  │                                                             │   │
│  │  computeTemplates()                                        │   │
│  │      ├─ 遍历 categories                                    │   │
│  │      ├─ CategoryTemplateContext.init()                     │   │
│  │      │   ├─ 校验模板（可能抛出异常）                        │   │
│  │      │   └─ 初始化上下文状态                               │   │
│  │      ├─ 收集 errors                                        │   │
│  │      ├─ 收集 orphanGoals（模板已移除）                      │   │
│  │      ├─ 按优先级执行 runTemplatesForPriority()             │   │
│  │      └─ distributeRemainder()                              │   │
│  │                                                             │   │
│  │  processTemplate()                                         │   │
│  │      ├─ computeTemplates()                                 │   │
│  │      ├─ if errors: return error notification               │   │
│  │      ├─ setBudgets(month, budgetList)                      │   │
│  │      └─ setGoals(month, [...orphanGoals, ...contextGoals]) │   │
│  └─────────────────────────────────────────────────────────────┘   │
│         │                                                           │
│         v                                                           │
│  category-template-context.ts (执行上下文)                          │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  CategoryTemplateContext.init()                             │   │
│  │      ├─ getSheetValue(lastMonth, `leftover-${catId}`)      │   │
│  │      ├─ checkByAndScheduleAndSpend(templates, month)       │   │
│  │      ├─ checkPercentage(templates)                         │   │
│  │      └─ 构造函数初始化状态                                  │   │
│  │                                                             │   │
│  │  runTemplatesForPriority(priority, budgetAvail, availStart)│   │
│  │      ├─ switch template.type                               │   │
│  │      │   ├─ runSimple()                                    │   │
│  │      │   ├─ runPeriodic()                                  │   │
│  │      │   ├─ runBy()                                        │   │
│  │      │   ├─ runPercentage()                                │   │
│  │      │   └─ ...                                            │   │
│  │      ├─ 检查 limit 限制                                    │   │
│  │      ├─ 检查可用预算（priority > 0 时）                    │   │
│  │      └─ 更新 toBudgetAmount                                │   │
│  │                                                             │   │
│  │  runRemainder(budgetAvail, perWeight)                      │   │
│  │      └─ 按权重分配剩余资金                                  │   │
│  │                                                             │   │
│  │  getValues()                                               │   │
│  │      ├─ runGoal()                                          │   │
│  │      └─ 返回 { budgeted, goal, longGoal, ... }             │   │
│  └─────────────────────────────────────────────────────────────┘   │
│         │                                                           │
│         v                                                           │
│  template-notes.ts (笔记模板)                                       │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  storeNoteTemplates()                                       │   │
│  │      ├─ getCategoriesWithTemplates()                       │   │
│  │      │   └─ SQL 排除 template_settings.source = 'ui'       │   │
│  │      ├─ storeTemplates({ source: 'notes' })                │   │
│  │      └─ resetCategoryGoalDefsWithNoTemplates()             │   │
│  │          └─ SQL 排除 template_settings.source = 'ui'       │   │
│  │                                                             │   │
│  │  getCategoriesWithTemplates()                              │   │
│  │      └─ 遍历笔记行，parse() 解析模板语法                   │   │
│  │                                                             │   │
│  │  unparse(templates)                                        │   │
│  │      └─ 将 Template 对象转换回文本语法                      │   │
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
│  │    ├─ name                                                  │   │
│  │    ├─ cat_group                                             │   │
│  │    ├─ goal_def (Template[] JSON)                            │   │
│  │    ├─ template_settings ({ source: 'ui' | 'notes' })        │   │
│  │    └─ ...                                                   │   │
│  │                                                             │   │
│  │  notes 表                                                   │   │
│  │    ├─ id (与 categories.id 对应)                           │   │
│  │    └─ note (文本内容，可能包含 #template 语法)              │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  电子表格 (Sheet)                                                    │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  每月一个 sheet (如: month-2024-01)                         │   │
│  │    ├─ budget-{catId}: 预算金额                              │   │
│  │    ├─ goal-{catId}: 目标金额                                │   │
│  │    ├─ long-goal-{catId}: 是否长期目标                       │   │
│  │    ├─ leftover-{catId}: 上月结转余额                        │   │
│  │    ├─ carryover-{catId}: 是否允许结转                       │   │
│  │    ├─ to-budget: 本月可用预算                               │   │
│  │    └─ total-income: 本月总收入                              │   │
│  └─────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

### 关键调用链

#### 调用链 1：UI 保存模板

```
BudgetAutomationsBody.onSave()
  │
  └─ send('budget/set-category-automations', {
       categoriesWithTemplates: [{ id, templates }],
       source: 'ui'
     })
       │
       └─ app.ts: 'budget/set-category-automations'
            │
            └─ goal-template.storeTemplates()
                 │
                 ├─ storeNoteCleanups(categoryIds)  // 清理清理模板
                 │
                 └─ db.updateWithSchema('categories', {
                      id,
                      goal_def: JSON.stringify(templates),
                      template_settings: { source: 'ui' }  // 关键标记
                    })
```

#### 调用链 2：加载自动化（含笔记同步）

```
useBudgetAutomations hook (categoryId, onLoaded)
  │
  ├─ send('budget/store-note-templates')  // 先同步笔记
  │    │
  │    └─ template-notes.storeNoteTemplates()
  │         │
  │         ├─ statements.getCategoriesWithTemplateNotes()
  │         │    └─ SQL: ... WHERE template_settings.source <> 'ui'
  │         │
  │         ├─ storeTemplates({ source: 'notes' })
  │         │
  │         └─ statements.resetCategoryGoalDefsWithNoTemplates()
  │              └─ SQL: ... WHERE template_settings.source <> 'ui'
  │
  └─ send('budget/get-category-automations', categoryId)
       │
       └─ goal-template.getTemplatesForCategory(categoryId)
            │
            └─ aqlQuery(q('categories')
                 .filter({ id: categoryId, goal_def: { $ne: null } })
                 .select('*'))
```

#### 调用链 3：应用模板到某月

```
applyTemplate({ month: '2024-01' })
  │
  ├─ storeNoteTemplates()  // 同步笔记（排除 UI 来源）
  │
  ├─ getTemplates()  // 读取所有模板配置
  │
  └─ processTemplate(month, force=false, categoryTemplates, [])
       │
       └─ computeTemplates(month, force, categoryTemplates, [])
            │
            ├─ 遍历 categories
            │    │
            │    └─ CategoryTemplateContext.init(templates, category, month, budgeted)
            │         │
            │         ├─ 校验: checkByAndScheduleAndSpend()
            │         │    └─ 失败: 抛出 Error
            │         │
            │         └─ 校验: checkPercentage()
            │              └─ 失败: 抛出 Error
            │
            ├─ 收集 errors[]
            │
            ├─ 收集 orphanGoals[]  // 有 goal 但无 template 的类别
            │
            ├─ if (errors.length > 0)
            │    └─ 立即返回 { contexts, errors, orphanGoals }
            │
            ├─ 按优先级执行 runTemplatesForPriority()
            │
            └─ distributeRemainder()
       │
       ├─ if (errors.length > 0)
       │    └─ 返回错误通知（不修改任何预算）
       │
       ├─ 构建 budgetList[] 和 goalList[]
       │    └─ goalList 包含 [...orphanGoals, ...contextGoals]
       │
       ├─ setBudgets(month, budgetList)
       │    └─ 批量写入 budget-{catId}
       │
       └─ setGoals(month, goalList)
            └─ 批量写入 goal-{catId} 和 long-goal-{catId}
                 └─ orphanGoals 会将 goal 设为 null
```

#### 调用链 4：预计算（Dry Run）

```
BudgetAutomationsBody 中 useEffect (debounce 200ms)
  │
  └─ send('budget/dry-run-category-template', {
       month, categoryId, templates
     })
       │
       └─ goal-template.dryRunCategoryTemplate()
            │
            └─ computeTemplates(
                 month,
                 force=true,
                 { [categoryId]: templates },
                 categoryData,
                 skipAvailableClamp=true  // 关键：不限制可用预算
               )
                 │
                 └─ 返回 { budgeted, perTemplate } 用于预览显示
```

#### 调用链 5：切回笔记模式（Un-migrate）

**说明**：这是跨模块的控制权移交过程，对应触发时机的第 3 条"从 UI 切回笔记模式时"。三步按严格顺序执行：保存笔记 → 切换来源 → 显式同步。

```
┌─────────────────────────────────────────────────────────────────────┐
│                       前端层 (desktop-client)                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  UnmigrateBudgetAutomationsModal.onSave(close)                      │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  Step 1: 保存合并后的笔记到 notes 表                         │   │
│  │                                                              │   │
│  │  send('notes-save-undoable', {                               │   │
│  │    id: categoryId,                                           │   │
│  │    note: editedNotes  // 现有笔记 + 导出的模板语法           │   │
│  │  })                                                          │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                           │                                         │
│                           v                                         │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  Step 2: 清空 UI 模板，切换来源标记为 notes                   │   │
│  │                                                              │   │
│  │  send('budget/set-category-automations', {                   │   │
│  │    categoriesWithTemplates: [{                               │   │
│  │      id: categoryId,                                         │   │
│  │      templates: []  // 清空 UI 模板                         │   │
│  │    }],                                                       │   │
│  │    source: 'notes'  // 标记为笔记来源                       │   │
│  │  })                                                          │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                           │                                         │
│                           v                                         │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  Step 3: 显式同步笔记模板（重新从笔记解析）                   │   │
│  │                                                              │   │
│  │  send('budget/store-note-templates')                         │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                           │                                         │
│                           v                                         │
│                      close()  // 关闭模态框                          │
└─────────────────────────────────────────────────────────────────────┘
                           │
                           v
┌─────────────────────────────────────────────────────────────────────┐
│                       后端层 (loot-core/server)                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  Step 1: 'notes-save-undoable'                             │   │
│  │      └─ 更新 notes 表，写入包含模板语法的笔记               │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  Step 2: 'budget/set-category-automations'                  │   │
│  │      └─ goal-template.storeTemplates({ source: 'notes' })   │   │
│  │           ├─ storeNoteCleanups([categoryId])               │   │
│  │           └─ db.updateWithSchema('categories', {            │   │
│  │                id: categoryId,                              │   │
│  │                goal_def: null,        // 清空              │   │
│  │                template_settings: {                         │   │
│  │                  source: 'notes'     // 切换来源           │   │
│  │                }                                            │   │
│  │              })                                             │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  Step 3: 'budget/store-note-templates'                      │   │
│  │      └─ template-notes.storeNoteTemplates()                 │   │
│  │           ├─ getCategoriesWithTemplates()                   │   │
│  │           │    └─ 现在 source='notes'，该类别会被包含       │   │
│  │           ├─ storeTemplates({ source: 'notes' })            │   │
│  │           │    └─ 重新解析笔记 → 填充 goal_def              │   │
│  │           └─ resetCategoryGoalDefsWithNoTemplates()         │   │
│  │                └─ 清理无模板的类别                          │   │
│  └─────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
                           │
                           v
┌─────────────────────────────────────────────────────────────────────┐
│                            数据层                                    │
├─────────────────────────────────────────────────────────────────────┤
│  notes 表: note 字段更新（Step 1）                                  │
│  categories 表: goal_def=null, template_settings.source='notes'（Step 2）│
│  categories 表: goal_def 被重新填充（Step 3，从笔记解析）           │
└─────────────────────────────────────────────────────────────────────┘
```

**前置调用链：渲染预览（弹窗打开时）**

```
UnmigrateBudgetAutomationsModal.useEffect([templates, categoryData])
  │
  ├─ 构建 idToName 映射（百分比模板的 category 是 ID，需转为名称）
  │
  ├─ sanitizePercentageCategoriesForNotes(templates, idToName)
  │    └─ 将 percentage 模板中的 category ID → category name
  │
  └─ send('budget/render-note-templates', sanitizedTemplates)
       │
       └─ app.ts: 'budget/render-note-templates'
            │
            └─ template-notes.renderNoteTemplates(templates)
                 │
                 └─ unparse(templates)  // Template[] → 文本语法
                      │
                      └─ 返回 "#template 100\n#goal 5000" 格式字符串
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

### 笔记模板同步触发时机

**真实触发**：

| 触发场景 | 调用方 | 目的 | 备注 |
|---------|-------|------|------|
| **1. 打开 UI 编辑器时** | `useBudgetAutomations` hook | 加载可能存在的笔记模板到编辑器中 | 只会处理 `source <> 'ui'` 的类别 |
| **2. 应用模板前** | `applyTemplate()` / `overwriteTemplate()` 等函数 | 确保最新的笔记模板被解析 | 排除 UI 来源类别 |
| **3. 从 UI 切回笔记模式时** | `UnmigrateBudgetAutomationsModal` | 切换控制权，确保笔记重新成为数据源 | 用户点击"Save notes & un-migrate"按钮后显式触发 |

**不会触发**：
1. 普通预算页面加载时
2. 切换月份时
3. 保存 UI 模板时（因为 UI 模板直接写入数据库，不经过笔记）

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
   - 资金不足时的需求展示（skipAvailableClamp=true）

2. **applyMultipleCategoryTemplates** - 多类别应用测试
   - 成功应用多个类别
   - 优先级资金分配（高优先级先获得资金）
   - 模板验证错误处理（by 模板目标月份已过）
   - 孤立目标清理（有 goal 但无 template）
   - 剩余资金按权重分配

3. **applyTemplate** - 全量应用测试
   - 跳过已预算类别（force=false）

---

## 关键文件索引

| 文件路径 | 职责 |
|---------|------|
| `packages/loot-core/src/server/budget/goal-template.ts` | 核心逻辑：存储、读取、计算、应用 |
| `packages/loot-core/src/server/budget/category-template-context.ts` | 计算引擎：模板执行上下文 |
| `packages/loot-core/src/server/budget/template-notes.ts` | 笔记模板：解析、同步、序列化、反序列化 |
| `packages/loot-core/src/server/budget/statements.ts` | SQL 语句：来源隔离、笔记查询 |
| `packages/loot-core/src/server/budget/app.ts` | API 路由：定义预算相关方法 |
| `packages/loot-core/src/types/models/templates.ts` | 类型定义：Template 类型系统 |
| `packages/desktop-client/src/hooks/useBudgetAutomations.ts` | 前端 Hook：加载自动化配置 |
| `packages/desktop-client/src/components/budget/goals/CategoryAutomationButton.tsx` | UI 入口：自动化按钮 |
| `packages/desktop-client/src/components/modals/BudgetAutomationsModal/BudgetAutomationsBody.tsx` | 编辑器主界面：保存、预览、校验 |
| `packages/desktop-client/src/components/modals/UnmigrateBudgetAutomationsModal.tsx` | 切回笔记模式：控制权移交、显式同步 |

---

## 扩展思考

### 系统优势

1. **灵活性**：支持多种模板类型，覆盖大部分预算场景
2. **优先级**：通过优先级机制确保重要支出先获得资金
3. **限制机制**：防止超预算，支持每日/每周/每月限制
4. **剩余分配**：智能分配剩余资金到弹性类别
5. **双输入**：支持 UI 配置和笔记语法两种方式
6. **来源隔离**：UI 保存的模板不会被笔记覆盖

### 潜在优化点

1. **批量计算优化**：当前逐类别计算，可考虑并行化
2. **缓存机制**：模板配置变化不频繁，可增加缓存
3. **增量计算**：只重新计算受影响的类别
4. **预测能力**：基于模板预测未来数月的预算需求

---

## 总结

预算模板系统是 Actual Budget 的核心功能之一，通过以下链路实现自动化预算：

1. **配置**：用户通过 UI 或笔记语法定义模板规则
2. **存储**：模板配置以 JSON 格式存储在 `categories.goal_def`，通过 `template_settings.source` 标记来源
3. **同步**：笔记模板自动同步到数据库，但不会覆盖 UI 保存的配置
4. **计算**：`CategoryTemplateContext` 执行复杂的预算计算，包含校验、优先级执行、限制检查
5. **应用**：计算结果写入月度电子表格，同时清理已移除模板的孤立目标
6. **展示**：前端从电子表格读取并展示预算数据

该系统设计优雅，通过优先级、限制、剩余分配、来源隔离等机制，实现了智能且灵活的自动化预算管理。
