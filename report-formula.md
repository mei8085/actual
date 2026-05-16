# 自定义报表公式从编辑到执行完整过程

## 概述

Actual Budget 的自定义报表公式系统提供了一个类似 Excel 的公式编辑和执行环境，支持两种主要模式（交易模式和查询模式），并通过多层次的错误处理确保系统的稳定性。

---

## 1. 公式模式切换

### 1.1 两种模式说明

系统支持两种公式模式，通过 `FormulaEditor` 组件的 `mode` 属性切换：

| 模式 | 用途 | 核心特性 |
|------|------|----------|
| `transaction` | 交易级公式计算 | 可访问交易字段变量（amount, date, notes 等） |
| `query` | 报表级聚合计算 | 可使用 QUERY、BUDGET_QUERY 等查询函数 |

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

**文件位置**：
- 查询模式函数：`packages/desktop-client/src/components/formula/queryModeFunctions.ts`
- 交易模式函数：`packages/desktop-client/src/components/formula/transactionModeFunctions.ts`

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

## 3. 非法表达式处理机制

### 3.1 界面层（编辑器）处理

**文件位置**：`packages/desktop-client/src/components/formula/codeMirror-excelLanguage.tsx`

#### 3.1.1 语法高亮与分类

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

#### 3.1.2 智能自动补全

```typescript
export function excelFormulaAutocomplete(
  mode: FormulaMode,
  queries?: Record<string, unknown>,
  variables?: Record<string, number | string>,
): Extension {
  // 1. 函数补全（按分类分组）
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

#### 3.1.3 悬停文档提示

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

### 3.2 执行层处理

**文件位置**：`packages/desktop-client/src/hooks/useFormulaExecution.ts:95-392`

#### 3.2.1 多层错误捕获

```typescript
async function executeFormula() {
  let hfInstance: HyperFormula | null = null;

  try {
    // 1. 基础语法检查
    if (!formula || !formula.startsWith('=')) {
      setResult(null);
      setError('Formula must start with =');
      return;
    }

    setIsLoading(true);
    setError(null);

    try {
      // 2. 查询执行错误捕获
      for (const queryName of queryNames) {
        const queryConfig = queries[queryName];
        if (!queryConfig) {
          console.warn(`Query "${queryName}" not found`);
          queryData[queryName] = 0;
          continue;
        }
        const data = await fetchQuerySum(queryConfig);
        queryData[queryName] = integerToAmount(data, 2);
      }

      // 3. 提取函数错误捕获
      for (const [funcName, regex] of Object.entries(extractionFunctions)) {
        const matches = Array.from(formula.matchAll(regex));
        for (const match of matches) {
          try {
            // 执行提取函数...
          } catch (err) {
            console.error(`Error evaluating ${funcName}(${queryName})`, err);
            extractionResults[funcName][key] = null;
          }
        }
      }

      // 4. BUDGET_QUERY 错误捕获
      for (const match of budgetMatches) {
        try {
          // 解析参数、执行查询...
        } catch (err) {
          console.error('Error evaluating BUDGET_QUERY', err);
        }
      }

      // 5. HyperFormula 执行
      hfInstance = HyperFormula.buildEmpty({ /* 配置 */ });
      // ... 设置单元格内容 ...
      
      const cellValue = hfInstance.getCellValue({ sheet: sheetId, col: 0, row: 0 });

      // 6. 检查 HyperFormula 返回的错误类型
      if (cellValue && typeof cellValue === 'object' && 'type' in cellValue) {
        setError(`Formula error: ${cellValue.type}`);
        setResult(null);
      } else {
        setResult(cellValue as number | string);
        setError(null);
      }

    } catch (err) {
      // 7. 执行过程中的未知错误
      console.error('Formula execution error:', err);
      setError(err instanceof Error ? err.message : 'Unknown error');
      setResult(null);
    } finally {
      // 8. 确保资源释放
      if (!cancelled) {
        setIsLoading(false);
      }
      try {
        hfInstance?.destroy();
      } catch (err) {
        console.error('Error destroying HyperFormula instance:', err);
        setError('Error destroying HyperFormula instance');
        setResult(null);
      }
    }
  }
}
```

#### 3.2.2 取消机制

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
    cancelled = true;  // 卸载时设置取消标志
  };
}, [formula, queriesVersion, locale, queries, namedExpressions]);
```

### 3.3 结果显示层处理

**文件位置**：`packages/desktop-client/src/components/reports/reports/Formula.tsx:79-83`

```typescript
const {
  result,
  isLoading: isExecuting,
  error,
} = useFormulaExecution(formula, queriesRef.current, queriesVersion);
```

错误状态传递给 `FormulaResult` 组件进行友好显示：
- 加载中：显示加载指示器
- 错误：显示错误信息（红色）
- 成功：显示计算结果

---

## 4. 完整数据流图示

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

## 5. 关键文件索引

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
