# 自定义报表公式从编辑到执行完整过程

## 概述

Actual Budget 的自定义报表公式系统提供了一个类似 Excel 的公式编辑和执行环境，支持两种主要模式（交易模式和查询模式），并通过多层次的错误处理确保系统的稳定性。

---

## 1. 公式模式切换

### 1.1 两种模式说明与使用场景

系统支持两种公式模式，通过 `FormulaEditor` 组件的 `mode` 属性切换：

| 模式 | 使用场景 | 组件 | 核心特性 |
|------|---------|------|---------|
| `transaction` | 规则编辑器 | `FormulaActionEditor.tsx:46` | 交易字段级计算，可访问单条交易的属性变量（amount, date, notes, category_name, payee_name 等） |
| `query` | 报表公式页 | `Formula.tsx:315`, `FormulaCard.tsx:44` | 报表级聚合计算，可使用 QUERY、BUDGET_QUERY 等查询函数调用已定义的查询配置 |

#### 各场景模式配置详情：

**1. 规则编辑器（FormulaActionEditor）**
- 文件：`packages/desktop-client/src/components/rules/FormulaActionEditor.tsx:43-50`
```typescript
<FormulaEditor
  value={value}
  onChange={handleChange}
  mode="transaction"     // 固定使用交易模式
  disabled={disabled}
  singleLine             // 强制单行显示
  showLineNumbers={false}
/>
```
- 用途：对单条交易执行计算或条件判断，如自动分类、设置标志等
- 可用函数：数学、逻辑、文本、日期函数 + 交易字段变量

**2. 报表公式页（Formula）**
- 文件：`packages/desktop-client/src/components/reports/reports/Formula.tsx:311-319`
```typescript
<FormulaEditor
  value={formula}
  onChange={setFormula}
  mode="query"           // 使用查询模式
  queries={queriesRef.current}
  singleLine={false}
  showLineNumbers
/>
```
- 用途：对交易数据进行多维聚合、统计、趋势分析
- 可用函数：全量函数集 + QUERY 系列查询函数

**3. 公式卡片（FormulaCard）**
- 文件：`packages/desktop-client/src/components/reports/reports/FormulaCard.tsx:44-48`
- 用途：在仪表盘展示公式计算结果
- 实现方式：不使用 FormulaEditor 组件，不存在 `mode` 模式属性，直接复用通用的 `useFormulaExecution` hook 执行链路
- 能力范围：该 hook 是通用执行引擎，既能预处理 QUERY、BUDGET_QUERY、QUERY_COUNT、QUERY_EXTRACT_* 等特殊查询函数，也能直接将 SUM、AVERAGE、IF 等普通公式透传给 HyperFormula 计算引擎处理
- 注意：没有"默认查询模式"的概念，mode 属性仅存在于 FormulaEditor 用于语法高亮和自动补全，执行层本身无模式限制

### 1.2 模式切换实现

**文件位置**：`packages/desktop-client/src/components/formula/FormulaEditor.tsx:12-319`

```typescript
type FormulaMode = 'transaction' | 'query';

export function FormulaEditor({
  value,
  onChange,
  mode,  // 模式通过此属性传入
  // ...
}: FormulaEditorProps) {
  // 根据模式加载不同的语言扩展
  const extensions = useMemo(
    () => [
      // ...
      ...excelFormulaExtension(mode, queries, isDarkTheme, variables),
      // ...
    ],
    [mode, queries, isDarkTheme, disabled, singleLine, variables],
  );
}
```

### 1.3 不同模式的函数集

每种模式都有专门的函数分类：
- 📊 数学函数（SUM, AVERAGE, COUNT 等）
- 🔀 逻辑函数（IF, AND, OR, NOT 等）
- 📝 文本函数（TEXT, CONCATENATE, LEFT 等）
- 📅 日期函数（DATE, TODAY, YEAR, MONTH 等）
- 🔍 查询函数（QUERY, QUERY_COUNT, BUDGET_QUERY 等）
- 💰 交易字段（仅 transaction 模式：amount, date, notes, category_name 等）

---

## 2. 查询参数落到底层的完整流程

### 2.1 查询定义与管理

**文件位置**：`packages/desktop-client/src/components/formula/QueryManager.tsx`

查询配置结构：
```typescript
type QueryConfig = {
  conditions?: RuleConditionEntity[];    // 过滤条件
  conditionsOp?: 'and' | 'or';           // 条件逻辑运算
  timeFrame?: {                           // 时间范围
    start: string;
    end: string;
    mode: 'sliding-window' | 'static' | 'full' | 'lastMonth' | 'yearToDate';
  };
};
```

查询管理器负责：
- 添加/删除查询
- 配置查询条件（过滤器）
- 设置时间范围（静态/滑动窗口/预设范围）
- 导出/导入查询配置

### 2.2 公式执行核心流程

**文件位置**：`packages/desktop-client/src/hooks/useFormulaExecution.ts:84-402`

#### 阶段 1：公式预处理与查询提取

```typescript
// 1. 提取 QUERY() 和 QUERY_COUNT() 函数调用
const queryMatches = Array.from(
  formula.matchAll(/QUERY\s*\(\s*["']([^"']+)["']\s*\)/gi),
);
const queryCountMatches = Array.from(
  formula.matchAll(/QUERY_COUNT\s*\(\s*["']([^"']+)["']\s*\)/gi),
);

// 2. 提取 QUERY_EXTRACT_* 函数（用于 BUDGET_QUERY 参数）
const extractionFunctions = {
  QUERY_EXTRACT_CATEGORIES: /.../,
  QUERY_EXTRACT_TIMEFRAME_START: /.../,
  QUERY_EXTRACT_TIMEFRAME_END: /.../,
};
```

#### 阶段 2：构建过滤查询

```typescript
async function buildFilteredTransactionsQuery(config: QueryConfig) {
  // 1. 将条件转换为查询过滤器
  const { filters: queryFilters } = await send('make-filters-from-conditions', {
    conditions,
  });

  // 2. 处理时间范围
  if (timeFrame && timeFrame.mode) {
    if (timeFrame.mode === 'sliding-window') {
      // 滑动窗口：始终以当前日期为终点
      const liveEndMonth = monthUtils.currentMonth();
      const liveStartMonth = monthUtils.subMonths(liveEndMonth, offset);
      startDate = monthUtils.firstDayOfMonth(liveStartMonth);
      endDate = monthUtils.currentDay();
    } else if (timeFrame.mode === 'static') {
      // 静态模式：使用用户设定的具体日期
      startDate = isMonthOnlyDate(timeFrame.start) 
        ? timeFrame.start + '-01' 
        : timeFrame.start;
      endDate = isMonthOnlyDate(timeFrame.end)
        ? monthUtils.getMonthEnd(timeFrame.end + '-01')
        : timeFrame.end;
    } else {
      // 预设范围：lastMonth, yearToDate, full 等
      const [calculatedStart, calculatedEnd] = getLiveRange(
        timeFrameModeToCondition(timeFrame.mode),
        earliestDate,
        latestDate,
        true,
      );
    }
  }

  // 3. 构建最终查询
  let transQuery = q('transactions')
    .filter({ /* 日期范围 */ })
    .filter({ [conditionsOpKey]: queryFilters });

  return transQuery;
}
```

#### 阶段 3：执行查询并获取结果

```typescript
// QUERY() - 获取交易金额总和
async function fetchQuerySum(config: QueryConfig): Promise<number> {
  const transQuery = await buildFilteredTransactionsQuery(config);
  const summedQuery = transQuery.calculate({ $sum: '$amount' });
  const { data } = await send('query', summedQuery.serialize());
  return data || 0;
}

// QUERY_COUNT() - 获取交易数量
async function fetchQueryCount(config: QueryConfig): Promise<number> {
  const transQuery = await buildFilteredTransactionsQuery(config);
  const countQuery = transQuery.calculate({ $count: '*' });
  const { data } = await send('query', countQuery.serialize());
  return data || 0;
}
```

#### 阶段 4：BUDGET_QUERY 特殊处理

BUDGET_QUERY 支持参数提取函数：
```typescript
// 解析参数类型：extraction 或 literal
function parseBudgetParam(param: string) {
  // 1. 提取函数调用：QUERY_EXTRACT_CATEGORIES("queryName")
  // 2. 数组字面量：{"id1";"id2"}
  // 3. 字符串字面量："value"
}

// 执行预算维度查询
async function fetchBudgetDimensionValueDirect(
  dimension: string,        // budgeted, spent, balance_start, balance_end, goal
  categoryIds: string[],    // 分类ID列表
  startMonth: string,       // 开始月份
  endMonth: string,         // 结束月份
): Promise<number> {
  // 遍历每个月份和分类，累加维度值
  for (const month of intervals) {
    const monthData = await send('envelope-budget-month', { month });
    for (const catId of categoryIds) {
      total += getMonthDataValue(monthData, fieldPattern, catId);
    }
  }
  return integerToAmount(total, 2);
}
```

#### 阶段 5：公式替换与最终执行

```typescript
// 1. 将查询函数调用替换为实际数值
let processedFormula = formula;
for (const [queryName, value] of Object.entries(queryData)) {
  processedFormula = processedFormula.replace(
    new RegExp(`QUERY\\s*\\(\\s*["']${escapeRegExp(queryName)}["']\\s*\\)`, 'gi'),
    String(value),
  );
}

// 2. 替换 BUDGET_QUERY 和提取函数结果
// ...

// 3. 使用 HyperFormula 执行最终计算
const hfInstance = HyperFormula.buildEmpty({
  licenseKey: 'gpl-v3',
  language: 'enUS',
  localeLang: locale,
  dateFormats: ['DD/MM/YYYY', 'YYYY-MM-DD', 'YYYY/MM/DD'],
});

const sheetName = hfInstance.addSheet('Sheet1');
const sheetId = hfInstance.getSheetId(sheetName);

// 设置命名变量（如 RESULT, theme_* 等）
if (namedExpressions) {
  for (const [name, value] of Object.entries(namedExpressions)) {
    hfInstance.addNamedExpression(name, value);
  }
}

// 执行公式
hfInstance.setCellContents({ sheet: sheetId, col: 0, row: 0 }, [
  [processedFormula],
]);

const cellValue = hfInstance.getCellValue({ sheet: sheetId, col: 0, row: 0 });
```

---

## 3. 颜色公式的独立执行链路

颜色公式是一个独立的计算链路，用于根据主公式结果动态设置显示颜色。它与主公式共享查询配置，但有独立的执行上下文和变量集。

### 3.1 完整执行流程

**触发点**：`Formula.tsx` 和 `FormulaCard.tsx` 中独立的 `useFormulaExecution` 调用

#### 阶段 1：变量准备（颜色公式专属）

**文件位置**：`packages/desktop-client/src/components/reports/reports/Formula.tsx:85-97`
```typescript
const colorVariables = useMemo(
  () => ({
    RESULT: result ?? 0,                    // 主公式计算结果（最核心变量）
    ...Object.entries(themeColors).reduce(  // 主题颜色变量集
      (acc, [key, value]) => {
        acc[`theme_${key}`] = value;
        return acc;
      },
      {} as Record<string, string>,
    ),
  }),
  [result, themeColors],
);
```

**可用变量示例**：
- `RESULT`: 主公式的数值结果
- `theme_pageText`: 页面文本颜色
- `theme_errorText`: 错误文本颜色（红色）
- `theme_positiveText`: 正数文本颜色（绿色）
- `theme_warningText`: 警告文本颜色（橙色）

#### 阶段 2：依赖驱动的响应式执行

**文件位置**：`packages/desktop-client/src/components/reports/reports/FormulaCard.tsx:44-68`
```typescript
// 第一步：主公式独立执行，不受颜色公式影响
const { result, isLoading, error } = useFormulaExecution(formula, ...);

// 第二步：主公式 result 变化触发颜色变量的重新计算
const colorVariables = useMemo(
  () => ({ RESULT: result ?? 0, ...themeColors }),  // RESULT 来自主公式结果
  [result, themeColors],
);

// 第三步：颜色变量变化触发颜色公式重新执行（独立 Hook 实例）
const { result: colorResult, error: colorError } = useFormulaExecution(
  colorFormula,
  meta?.queries || {},
  meta?.queriesVersion,
  colorVariables,
);
```

> **触发机制说明**：颜色公式与主公式并非固定串行关系，而是通过 React 响应式依赖链形成级联更新：
> 1. 两个 `useFormulaExecution` 是独立的 Hook 实例，各自维护状态
> 2. 首次渲染时，主公式 Hook 先调用，颜色公式 Hook 后调用（代码顺序）
> 3. 主公式异步计算完成 → 更新 `result` 状态 → 触发 `colorVariables` 的 useMemo 重新计算
> 4. `colorVariables` 引用变化 → 触发颜色公式的 `useFormulaExecution` 重新执行
> 5. 这是一个**依赖驱动的瀑布式更新**，而非固定的串行等待

#### 阶段 3：结果传递与渲染决策

**文件位置**：`packages/desktop-client/src/components/reports/reports/Formula.tsx:188-189` 和 `FormulaCard.tsx:71-72`
```typescript
// 颜色结果判定逻辑
const customColor =
  colorFormula && !colorError && colorResult 
    ? String(colorResult)   // 公式有效且有结果：使用计算出的颜色值
    : null;                 // 无公式、有错误或无结果：回退到默认颜色
```

#### 阶段 4：最终渲染应用

**文件位置**：`packages/desktop-client/src/components/reports/FormulaResult.tsx:153-158`
```typescript
// 颜色优先级：自定义颜色 > 错误颜色 > 默认文本颜色
const color = customColor
  ? customColor                    // 最高优先级：颜色公式的结果
  : error
    ? theme.errorText              // 次高优先级：错误状态的红色
    : theme.pageText;              // 默认：常规文本颜色
```

### 3.2 颜色公式典型用例

```excel
// 示例1：盈亏指示
=IF(RESULT > 0, "green", IF(RESULT < 0, "red", "gray"))

// 示例2：三级阈值
=IF(RESULT > 10000, theme_positiveText, IF(RESULT > 0, theme_warningText, theme_errorText))

// 示例3：精确颜色代码
=IF(ABS(RESULT) > 5000, "#ff5722", "#4caf50")
```

### 3.3 链路独立性保障

| 特性 | 主公式链路 | 颜色公式链路 |
|------|-----------|-------------|
| 独立 Hook实例 | ✓ | ✓ |
| 独立错误状态 | ✓ | ✓ |
| 独立变量集 | ✗（基础变量） | ✓（含 RESULT + theme_*） |
| 共享查询配置 | ✓ | ✓ |
| 错误互不影响 | - | ✓（颜色公式错误不影响主结果显示 |

---

## 4. 非法表达式处理机制

### 4.1 界面层（编辑器）处理

**文件位置**：`packages/desktop-client/src/components/formula/codeMirror-excelLanguage.tsx`

#### 4.1.1 语法高亮与分类

通过 StreamLanguage 解析器实现语法高亮，不同类型的函数使用不同颜色：

```typescript
// 函数分类集合
const MATH_FUNCTIONS = new Set(['SUM', 'AVERAGE', 'COUNT', ...]);
const LOGICAL_FUNCTIONS = new Set(['IF', 'AND', 'OR', ...]);
const TEXT_FUNCTIONS = new Set(['TEXT', 'CONCATENATE', ...]);
const DATE_FUNCTIONS = new Set(['DATE', 'TODAY', ...]);
const QUERY_FUNCTIONS = new Set(['QUERY', 'QUERY_COUNT', 'BUDGET_QUERY', ...]);

// 解析器根据函数类型返回不同的 token
if (MATH_FUNCTIONS.has(word)) return 'keyword';        // 蓝色
if (LOGICAL_FUNCTIONS.has(word)) return 'className';   // 青色
if (TEXT_FUNCTIONS.has(word)) return 'namespace';      // 紫色
if (DATE_FUNCTIONS.has(word)) return 'typeName';       // 绿色
if (QUERY_FUNCTIONS.has(word)) return 'propertyName';  // 红色
```

#### 4.1.2 智能自动补全

```typescript
export function excelFormulaAutocomplete(
  mode: FormulaMode,
  queries?: Record<string, unknown>,
  variables?: Record<string, number | string>,
): Extension {
  // 1. 函数补全（按分类分组分组）
  const functionCompletions = getFunctionCompletions(mode);
  
  // 2. 已定义查询的快捷补全
  const queryCompletions = queries 
    ? Object.keys(queries).flatMap(queryName => [
        { label: `QUERY("${queryName}")`, type: 'function', boost: 15 },
        { label: `QUERY_COUNT("${queryName}")`, type: 'function', boost: 14 },
        { label: `BUDGET_QUERY(..., "${queryName}")`, type: 'function', boost: 13 },
      ])
    : [];

  // 3. 变量补全（如 RESULT, theme_*）
  const variableCompletions = variables 
    ? Object.entries(variables).map(([varName, value]) => ({
        label: varName, type: 'variable', boost: 20,
      }))
    : [];

  // 4. 交易字段补全（仅 transaction 模式）
  if (mode === 'transaction') {
    suggestions.push(...transactionFields);
  }

  // 5. 排序策略：变量 > 查询函数 > 数学函数 > ...
  const sortedSuggestions = suggestions.sort((a, b) => {
    const sectionOrder = {
      '🔢 Variables': -1,
      '🔍 Query Functions': 0,
      '📊 Math Functions': 1,
      // ...
    };
    // 同 section 内按 boost 降序，再按字母升序
  });
}
```

#### 4.1.3 悬停文档提示

```typescript
export function excelFormulaHover(mode: FormulaMode): Extension {
  return hoverTooltip((view, pos) => {
    const word = view.state.wordAt(pos);
    if (!word) return null;
    
    const w = view.state.doc.sliceString(word.from, word.to);
    const funcDef = functions[w.toUpperCase()];
    
    if (funcDef) {
      // 显示函数文档：名称、描述、参数列表
      return {
        pos: word.from,
        end: word.to,
        above: true,
        create() {
          const dom = document.createElement('div');
          const root = createRoot(dom);
          root.render(
            <FunctionTooltip
              name={upper}
              description={funcDef.description}
              parameters={funcDef.parameters}
            />,
          );
          return { dom, destroy: () => root.unmount() };
        },
      };
    }
  });
}
```

### 4.2 执行层处理

**文件位置**：`packages/desktop-client/src/hooks/useFormulaExecution.ts:95-392`

#### 4.2.1 各类错误的处理策略与表现

系统采用**容错优先**的错误处理策略，不同类型的错误有不同的降级处理方式：

| 错误类型 | 界面表现 | 执行层处理 | 对用户的影响 |
|---------|---------|-----------|-------------|
| 公式格式错误（无 `=` 开头） | 显示红色错误文本 | 立即返回，设置 `error` 状态 | 明确提示修正公式 |
| 查询名称不存在 | 继续计算，结果为 0 | 仅 `console.warn`，回退值为 0 | 可能导致计算结果不准确但不中断 |
| 查询执行失败（数据库错误） | 继续计算，结果为 0 | try-catch 捕获，回退值为 0 | 静默失败，需查看控制台 |
| QUERY_EXTRACT 函数异常 | 参数解析失败，BUDGET_QUERY 可能失效 | try-catch 捕获，结果设为 `null` | 部分功能失效，整体继续 |
| BUDGET_QUERY 执行异常 | 该函数结果为空，公式继续计算 | 仅 `console.error`，无状态上报 | 静默失败，需查看控制台 |
| HyperFormula 语法错误 | 显示红色错误代码 | 检测 `{ type: error }` 对象，设置 `error` 状态 | 明确提示错误类型 |
| 颜色公式错误 | 使用默认颜色，不影响主结果 | 独立错误状态 `colorError`，不影响主公式 | 颜色回退默认，数值正常显示 |

---

##### 错误类型 1：查询缺失（名称不存在）

**代码位置**：`useFormulaExecution.ts:192-196`
```typescript
for (const queryName of queryNames) {
  const queryConfig = queries[queryName];

  if (!queryConfig) {
    console.warn(`Query "${queryName}" not found in queries config`);
    queryData[queryName] = 0;  // 静默回退为 0
    continue;
  }
  // ... 正常执行
}
```

**界面表现**：
- ✅ 不显示任何错误提示
- ✅ 公式继续执行，该 QUERY 函数返回 0
- ⚠️ 仅在浏览器控制台输出警告日志
- ❗ 潜在风险：用户可能误删查询后未察觉，导致后续计算基于错误的 0 值

---

##### 错误类型 2：查询执行失败（数据库异常）

**代码位置**：`useFormulaExecution.ts:538-547`（fetchQuerySum）和 `550-559`（fetchQueryCount）
```typescript
async function fetchQuerySum(config: QueryConfig): Promise<number> {
  try {
    const transQuery = await buildFilteredTransactionsQuery(config);
    const summedQuery = transQuery.calculate({ $sum: '$amount' });
    const { data } = await send('query', summedQuery.serialize());
    return data || 0;
  } catch (err) {
    console.error('Error fetching query sum:', err);
    return 0;  // 数据库查询失败，回退为 0
  }
}
```

**界面表现**：
- ✅ 不显示任何错误提示
- ✅ 公式继续执行，返回 0
- ⚠️ 控制台记录完整错误栈
- ❗ 潜在风险：数据库连接问题被隐藏，用户无法感知

---

##### 错误类型 3：预算查询异常（BUDGET_QUERY）的两阶段处理

预算查询异常采用**两阶段处理模型**，不同场景下的用户可见性完全不同：

---

**场景 A：仅记录日志（静默失败，无用户提示）**

**触发条件**：发生在 BUDGET_QUERY 的预处理和执行阶段

| 具体场景 | 界面层表现 | 执行层处理 |
|---------|-----------|-----------|
| ✅ 参数解析失败 | 无任何提示 | 仅 `console.error`，执行 `continue` 跳过当前 BUDGET_QUERY 匹配 |
| ✅ 参数验证失败 | 无任何提示 | 仅 `console.error`，执行 `continue` 跳过当前 BUDGET_QUERY 匹配 |
| ✅ 查询执行抛出异常 | 无任何提示 | try-catch 捕获，仅 `console.error`，不替换公式 |
| **共同结果** | **公式继续执行，用户无感** | **BUDGET_QUERY 函数调用原封不动保留在公式字符串中** |

**代码位置**：`useFormulaExecution.ts:241-285`
```typescript
try {
  const param1 = resolveBudgetParam(parseBudgetParam(param1Str), ...);
  if (!Array.isArray(param1) || ...) {  // 参数验证失败
    console.error('Failed to resolve BUDGET_QUERY parameters:', ...);
    continue;  // 仅日志，不替换公式
  }
  const val = await fetchBudgetDimensionValueDirect(dimension, param1, ...);
  processedFormula = processedFormula.replace(match[0], String(val));
} catch (err) {
  console.error('Error evaluating BUDGET_QUERY', err);  // 仅日志
}
```

---

**场景 B：触发 Formula error 提示（用户可见错误）**

**触发条件**：阶段 A 中 BUDGET_QUERY 未被替换，公式进入 HyperFormula 执行阶段

| 具体场景 | 界面层表现 | 执行层处理 |
|---------|-----------|-----------|
| ❌ HyperFormula 检测到未知函数名 | 显示红色错误文本 `Formula error: #NAME?` | 接收到引擎返回的 `{ type: '#NAME?' }` 错误对象 |
| ❌ 结果状态 | 计算结果区域清空，无数值显示 | 调用 `setError("Formula error: #NAME?")` + `setResult(null)` |
| **用户感知** | **明确看到公式异常，可定位问题** | **完整的错误状态上报到组件层** |

**代码位置**：`useFormulaExecution.ts:367-373`
```typescript
// BUDGET_QUERY 未被替换 → HyperFormula 将其视为未知函数名
if (cellValue && typeof cellValue === 'object' && 'type' in cellValue) {
  setError(`Formula error: ${cellValue.type}`);  // 触发用户可见的错误提示
  setResult(null);
}
```

---

**完整异常链路总结**：
```
BUDGET_QUERY(...) 执行
    │
    ├─ 成功 → 替换为数值 → 公式正常执行
    │
    └─ 失败（参数错误/查询异常）→ 仅日志，不替换
           │
           └─ BUDGET_QUERY 保留在公式字符串中
                  │
                  └─ HyperFormula 执行
                         │
                         └─ 未知函数名 → #NAME? 错误
                                │
                                └─ setError() → 界面显示红色 Formula error
```

---

##### 错误类型 4：提取函数执行异常（QUERY_EXTRACT_*）

**代码位置**：`useFormulaExecution.ts:166-172`
```typescript
try {
  if (funcName === 'QUERY_EXTRACT_CATEGORIES') {
    extractionResults[funcName][key] = await extractQueryCategories(queryName, queries);
  } // ... 其他提取函数
} catch (err) {
  console.error(`Error evaluating ${funcName}(${queryName})`, err);
  extractionResults[funcName][key] = null;  // 回退为 null
}
```

**连锁影响**：
- 提取函数返回 `null` 后，作为参数传递给 BUDGET_QUERY
- BUDGET_QUERY 参数验证失败（期望数组/字符串，实际为 null
- BUDGET_QUERY 执行失效，但仍然只记录日志不报错

---

##### 错误类型 5：HyperFormula 计算引擎错误

**代码位置**：`useFormulaExecution.ts:367-373`
```typescript
// 检查 HyperFormula 返回的错误对象
if (cellValue && typeof cellValue === 'object' && 'type' in cellValue) {
  setError(`Formula error: ${cellValue.type}`);  // 明确上报错误
  setResult(null);
} else {
  setResult(cellValue as number | string);
  setError(null);
}
```

**常见错误类型**：
- `#DIV/0!`：除零错误
- `#VALUE!`：参数类型错误
- `#NAME?`：函数名未定义
- `#REF!`：引用错误
- `#N/A`：值不可用

**界面表现**：
- ❌ 显示红色错误文本：`Formula error: #DIV/0!`
- ❌ 计算结果清空为 `null`
- ✅ 用户明确感知公式问题，可及时修正

---

##### 错误类型 6：颜色公式错误（独立链路容错）

**代码位置**：`Formula.tsx:188-189` 和 `FormulaCard.tsx:71-72`
```typescript
// 颜色公式错误时，customColor 为 null，回退到默认颜色
const customColor =
  colorFormula && !colorError && colorResult ? String(colorResult) : null;
```

**容错特性**：
- 颜色公式的错误完全与主公式隔离
- 即使颜色公式返回错误，主结果仍正常显示
- 仅颜色回退为默认值（普通文本色 / 主公式错误时的红色）

---

#### 4.2.2 取消机制

使用 React useEffect 的清理函数实现组件卸载时的执行取消：

```typescript
useEffect(() => {
  let cancelled = false;

  async function executeFormula() {
    // ... 执行过程中检查 cancelled 标志
    if (cancelled) return;
    
    // 获取结果后再次检查
    if (cancelled) return;
  }

  void executeFormula();

  return () => {
    cancelled = true;  // 卸载时设置取消标志，防止内存泄漏
  };
}, [formula, queriesVersion, locale, queries, namedExpressions]);
```

### 4.3 结果显示层处理

**文件位置**：`packages/desktop-client/src/components/reports/reports/Formula.tsx:79-83`

```typescript
const {
  result,
  isLoading: isExecuting,
  error,
} = useFormulaExecution(formula, queriesRef.current, queriesVersion);
```

**最终渲染逻辑**（FormulaResult.tsx:57-70）：
```typescript
const displayValue = useMemo(() => {
  if (error) {
    return error;              // 优先显示错误信息
  } else if (value === null || value === undefined) {
    return '';                 // 空值显示空白
  } else if (typeof value === 'number') {
    return format(amountToInteger(value, format.currency.decimalPlaces), 'financial');
  } else {
    return String(value);      // 其他类型转字符串
  }
}, [error, value, format]);
```

---

## 5. 错误处理设计权衡

### 优点
1. **高可用性**：多数错误采用静默降级，不中断用户操作
2. **隔离性**：颜色公式与主公式错误互不影响
3. **渐进式**：语法层面的错误明确提示，数据层面的错误静默降级

### 潜在问题
1. **可观测性不足**：查询失败、预算查询失败等关键错误仅日志记录，用户无感知
2. **调试困难**：静默失败可能导致结果不符合预期，但用户难以定位原因
3. **错误累积**：多个查询同时失败时，结果可能严重失真，但无任何提示

### 改进建议
- 增加"警告"状态（非阻塞的黄色提示），用于非致命错误
- 提供查询执行状态的可视化反馈（如每个 QUERY 函数旁边的状态指示器）
- 增加公式调试面板，展示每个子查询的执行结果和耗时

---

## 6. 完整数据流图示

```
用户输入公式
    ↓
[FormulaEditor 组件]
    ├─ CodeMirror 编辑器
    │   ├─ 语法高亮（按函数分类着色）
    │   ├─ 自动补全（变量 > 查询 > 函数 > 字段）
    │   └─ 悬停提示（函数文档）
    ↓
[Formula 页面]
    ├─ 保存查询配置到 queriesRef
    ├─ 更新 queriesVersion 触发重新执行
    ↓
[useFormulaExecution Hook]
    ├─ 阶段1：提取 QUERY, QUERY_COUNT, BUDGET_QUERY, QUERY_EXTRACT_*
    ├─ 阶段2：构建过滤查询（条件 + 时间范围）
    ├─ 阶段3：执行数据库查询（send('query', ...)）
    ├─ 阶段4：替换公式中的查询函数为实际值
    ├─ 阶段5：HyperFormula 执行最终计算
    └─ 返回 { result, isLoading, error }
    ↓
[FormulaResult 组件]
    ├─ 加载状态 → 显示加载动画
    ├─ 错误状态 → 显示红色错误信息
    └─ 成功状态 → 显示计算结果（支持字号调整）
```

---

## 7. 关键文件索引

| 功能模块 | 文件路径 | 主要职责 |
|---------|---------|---------|
| 公式编辑器 | `packages/desktop-client/src/components/formula/FormulaEditor.tsx` | CodeMirror 封装、模式切换 |
| 语言扩展 | `packages/desktop-client/src/components/formula/codeMirror-excelLanguage.tsx` | 语法高亮、自动补全、悬停提示 |
| 查询模式函数 | `packages/desktop-client/src/components/formula/queryModeFunctions.ts` | 查询模式可用函数定义 |
| 交易模式函数 | `packages/desktop-client/src/components/formula/transactionModeFunctions.ts` | 交易模式可用函数定义 |
| 查询管理器 | `packages/desktop-client/src/components/formula/QueryManager.tsx` | 查询配置UI、条件管理 |
| 执行核心 | `packages/desktop-client/src/hooks/useFormulaExecution.ts` | 公式解析、查询执行、HyperFormula 集成 |
| 公式页面 | `packages/desktop-client/src/components/reports/reports/Formula.tsx` | 公式编辑页面主UI |
| 公式卡片 | `packages/desktop-client/src/components/reports/reports/FormulaCard.tsx` | 仪表盘中的公式卡片 |
