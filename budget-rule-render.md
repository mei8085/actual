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

## 四（补充）：i18n 文案全链路深度分析

### 4.4 i18n 初始化与核心配置

**核心文件**：`packages/desktop-client/src/i18n.ts`

#### 4.4.1 初始化配置

```typescript
import { initReactI18next } from 'react-i18next';
import i18n from 'i18next';
import resourcesToBackend from 'i18next-resources-to-backend';

void i18n
  .use(initReactI18next)
  .use(resourcesToBackend(loadLanguage))
  .init({
    lng: 'en',                          // 默认语言
    nsSeparator: false,                 // 禁用命名空间分隔符
    keySeparator: false,                // 禁用 key 分隔符（关键设计）
    fallbackLng: false,                 // 禁用语言回退（关键设计）
    interpolation: {
      escapeValue: false,               // React 已自动转义
    },
    react: {
      transSupportBasicHtmlNodes: false,
    },
  });
```

**关键设计决策**：
- `keySeparator: false`：**不使用人工 key**，直接使用英文自然语言作为 key
- `fallbackLng: false`：**不启用语言回退**，缺 key 时直接显示英文原文
- `nsSeparator: false`：不使用命名空间，所有翻译在一个层级

#### 4.4.2 翻译资源加载机制

**核心文件**：
- `packages/desktop-client/src/languages.ts`
- `bin/package-browser`（构建脚本）
- `packages/desktop-client/bin/remove-untranslated-languages`（翻译质量过滤脚本）

##### 资源来源：translations 独立仓库

语言资源不存储在主代码仓中，而是来自独立的 translations 仓库：

```
GitHub: actualbudget/translations
    ↓
构建时 clone 到: packages/desktop-client/locale/
    ↓
包含文件: en.json, zh.json, fr.json, de.json, ...
```

**构建脚本逻辑**（`bin/package-browser`）：

```bash
# 默认会自动拉取翻译资源
if [ "$SKIP_TRANSLATIONS" = false ]; then
  echo "Updating translations..."
  if ! [ -d packages/desktop-client/locale ]; then
      git clone https://github.com/actualbudget/translations packages/desktop-client/locale
  fi
  pushd packages/desktop-client/locale > /dev/null
  git checkout .
  git pull
  popd > /dev/null
  
  # 删除翻译完成度低于50%的语言文件
  packages/desktop-client/bin/remove-untranslated-languages
fi
```

**翻译质量过滤**（`remove-untranslated-languages`）：

```javascript
// 比较各语言与英文的 key 数量，低于50%的语言会被删除
const percentage = (fileKeysCount / enKeysCount) * 100;
if (percentage < 50) {
  fs.unlinkSync(filePath);  // 删除不完整的翻译
  console.log(`Deleted ${file} due to insufficient keys.`);
}
```

##### 运行时加载机制

**代码**（`packages/desktop-client/src/languages.ts`）：

```typescript
// 使用 Vite 的 import.meta.glob 动态导入所有语言文件
// 路径: /locale/*.json → 对应构建时 clone 到的 packages/desktop-client/locale/
export const languages = import.meta.glob([
  '/locale/*.json',
  '!/locale/*_old.json',
]);

const loadLanguage = (language: string) => {
  if (!isLanguageAvailable(language)) {
    throw new Error(`Unknown locale ${language}`);
  }
  return languages[`/locale/${language}.json`]();
};
```

**资源加载流程**：

1. **构建时**：
   - 从 `actualbudget/translations` 仓库 clone 翻译文件到 `packages/desktop-client/locale/`
   - 运行 `remove-untranslated-languages` 过滤掉完成度 <50% 的语言
   - Vite 通过 `import.meta.glob` 收集 `/locale/*.json` 所有翻译文件
   - 每个语言文件被打包为独立的 chunk，实现按需加载

2. **运行时**：
   - 通过 `i18next-resources-to-backend` 插件按需懒加载语言资源
   - 语言文件以 JSON 格式存储，key 是英文原文，value 是对应语言的翻译
   - 只有用户切换语言时才会加载对应的语言包

3. **可用语言检测**：

   ```typescript
   // Playwright 测试环境下不加载任何语言（避免测试不稳定）
   export const availableLanguages = Platform.isPlaywright
     ? []
     : Object.keys(languages).map(path => path.split('/')[2].split('.')[0]);
   ```

#### 4.4.3 翻译资源提取流程

**核心文件**：`packages/desktop-client/i18next-parser.config.js`

```javascript
module.exports = {
  input: ['src/**/*.{js,jsx,ts,tsx}', '../loot-core/src/**/*.{js,jsx,ts,tsx}'],
  output: 'locale/$LOCALE.json',
  locales: ['en'],                    // 只提取英文
  sort: true,
  keySeparator: false,
  namespaceSeparator: false,
  defaultValue: (locale, ns, key, value) => {
    if (locale === 'en') {
      return value || key;            // 英文的 value 就是 key 本身
    }
    return '';
  },
};
```

**提取命令**：
```bash
yarn workspace @actual-app/web generate:i18n
```

**工作原理**：
1. 扫描 `src/` 和 `loot-core/src/` 下所有使用 `<Trans>` 或 `t()` 的代码
2. 提取所有英文文本作为 key
3. 生成 `locale/en.json` 文件，key 和 value 都是英文原文
4. 其他语言通过 Weblate 平台进行翻译（不接受 GitHub PR 直接修改翻译文件）

### 4.5 自然语言 Key 设计原理

#### 4.5.1 设计理念

**核心原则**：**英文原文即是 Key**

```
传统方案：
key: "budget_percentage_of_income"
value: "Budget {{percent}}% of total income this month"

Actual 方案：
key: "Budget {{percent}}% of total income this month"
value: "预算本月总收入的 {{percent}}%"
```

**优势**：
- 代码可读性强：直接在代码中看到英文原文
- 缺 key 时用户仍能看到有意义的英文，而不是无意义的 key
- 翻译人员直接看到完整的上下文句子
- 避免了维护两套文本（key + 默认值）的不一致问题

**权衡**：
- 修改英文原文时，所有语言的翻译 key 都会失效，需要重新翻译
- 长句子作为 key 会增加翻译文件的体积

#### 4.5.2 Key 提取规则

`i18next-parser` 会提取以下模式：

1. **Trans 组件**（最常用）：
   ```typescript
   <Trans>Budget {{ percent }}% of total income</Trans>
   // → Key: "Budget {{percent}}% of total income"
   ```

2. **t 函数**：
   ```typescript
   t('Save changes')
   // → Key: "Save changes"
   ```

3. **带插值的 t 函数**：
   ```typescript
   t('You selected {{count}} items', { count: 5 })
   // → Key: "You selected {{count}} items"
   ```

4. **复数处理**：
   ```typescript
   <Trans count={periodAmount}>
     Budget {{ amount }} every {{ count: periodAmount }} month
   </Trans>
   // → 英文会自动处理复数："month" / "months"
   ```

### 4.6 语言解析与回退策略

**核心文件**：`packages/desktop-client/src/i18n.ts` 中的 `resolveLanguage` 和 `setI18NextLanguage`

#### 4.6.1 语言选择优先级

```typescript
export const setI18NextLanguage = (language: string | null) => {
  // 1. 用户设置优先
  const defaultLanguages = Array.isArray(navigator.languages)
    ? navigator.languages
    : [navigator.language || 'en'];
  const languagesToTry = language ? [language] : defaultLanguages;

  // 2. 逐个尝试解析
  for (const lang of languagesToTry) {
    resolved = resolveLanguage(lang);
    if (resolved) break;
  }

  // 3. 最终兜底：英文
  if (!resolved) {
    resolved = 'en';
  }
};
```

#### 4.6.2 语言解析链（resolveLanguage）

```
输入语言: "zh-CN"
    ↓
1. 精确匹配: 检查是否有 "zh-CN.json"
    ├─ 有 → 使用 "zh-CN"
    └─ 无 → 继续
    ↓
2. 小写匹配: 尝试 "zh-cn" (如果输入有大写)
    ├─ 有 → 使用 "zh-cn"
    └─ 无 → 继续
    ↓
3. 基础语言匹配: 尝试 "zh" (去掉 "-" 后的部分)
    ├─ 有 → 使用 "zh"
    └─ 无 → 继续
    ↓
4. 最终兜底: 返回 undefined → 调用方使用 "en"
```

**代码实现**：
```typescript
const resolveLanguage = (language: string) => {
  if (language === 'en') return 'en';  // 英文总是可用
  
  if (isLanguageAvailable(language)) return language;
  
  // 尝试小写
  const lowercaseLanguage = language.toLowerCase();
  if (lowercaseLanguage !== language) {
    return resolveLanguage(lowercaseLanguage);
  }
  
  // 尝试基础语言（如 zh-CN → zh）
  if (language.includes('-')) {
    const fallback = language.split('-')[0];
    return resolveLanguage(fallback);
  }
  
  return undefined;  // 所有尝试都失败
};
```

### 4.7 缺 Key 兜底展示路径

#### 4.7.1 运行时缺 Key 处理

由于配置了 `fallbackLng: false`，i18next 的缺 Key 行为如下：

```
场景：中文翻译文件中缺少某个 key
    ↓
1. i18n 查找当前语言（中文）的 key
    └─ 未找到
    ↓
2. 由于 fallbackLng: false，不回退到英文
    ↓
3. i18next 默认行为：直接返回 key 本身（即英文原文）
    ↓
4. 用户看到：英文原文（而不是空白或错误）
```

**这是有意的设计决策**：显示英文总比显示一个无意义的 key 或空白要好。

#### 4.7.2 缺 Key 的用户体验路径

以预算自动化文案为例：

```
Trans 组件: <Trans>Budget {{ percent }}% of total income this month</Trans>
    ↓
中文翻译存在 → "本月预算为总收入的 {{percent}}%"
    ↓
中文翻译缺失 → "Budget {{ percent }}% of total income this month"
    ↓
用户界面：显示英文原文（虽然不是最佳，但仍可理解）
```

#### 4.7.3 开发时缺 Key 检测

ESLint 自定义规则 `no-untranslated-strings` 确保所有用户可见文本都被包裹：

**核心文件**：`packages/eslint-plugin-actual/lib/rules/no-untranslated-strings.js`

```
检测规则：
- 未被 <Trans> 或 t() 包裹的字符串字面量 → 警告
- JSX 中的纯文本 → 警告
- aria-label 等无障碍属性中的文本 → 警告
```

### 4.8 错误提示组件与 i18n 的协作

#### 4.8.1 协作架构图

```
┌─────────────────────────────────────────────────────────┐
│                    错误提示层                            │
├─────────────────────────────────────────────────────────┤
│  validateAutomation.ts                                   │
│  └─ 输出: AutomationErrorKind { kind, ...params }        │
│                           ↓                              │
│  automationMessages.tsx                                  │
│  ├─ AutomationErrorTitle(error)                          │
│  ├─ AutomationErrorShort(error)                          │
│  └─ AutomationErrorDetail(error)                         │
│     └─ 每个 kind 对应 <Trans>...</Trans>                 │
│                           ↓                              │
│  i18next (运行时翻译)                                    │
│  └─ 查找到翻译 → 显示译文                                 │
│     未找到翻译 → 显示英文原文 (key)                       │
└─────────────────────────────────────────────────────────┘
```

#### 4.8.2 错误类型与翻译 Key 的映射

**核心文件**：`packages/desktop-client/src/components/budget/goals/automationMessages.tsx`

每个错误 kind 对应三个层级的翻译 Key：

| 错误 kind | Title Key | Short Key | Detail Key |
|-----------|-----------|-----------|------------|
| `schedule-not-found` | "Schedule not found" | "No schedule named \"{{name}}\"" | "Pick an existing schedule..." |
| `percentage-out-of-range` | "Percentage out of range" | "{{percent}}% must be between 0 and 100" | "Set a value greater than 0%..." |
| `by-target-past` | "Target is in the past" | "{{month}} has already passed" | "Pick a future month..." |
| ... | ... | ... | ... |

**代码示例**：
```typescript
export function AutomationErrorShort({ error }) {
  const locale = useLocale();
  switch (error.kind) {
    case 'schedule-not-found':
      return error.name ? (
        // Key: "No schedule named \"{{name}}\""
        <Trans>No schedule named &ldquo;{{ name: error.name }}&rdquo;</Trans>
      ) : (
        // Key: "Pick a schedule"
        <Trans>Pick a schedule</Trans>
      );
    case 'percentage-out-of-range':
      // Key: "{{percent}}% must be between 0 and 100"
      return <Trans>{{ percent: error.percent }}% must be between 0 and 100</Trans>;
    case 'by-target-past':
      // Key: "{{month}} has already passed"
      return <Trans>{{ month: formatMonthLabel(error.month, locale) }} has already passed</Trans>;
    // ... 其他错误类型
  }
}
```

#### 4.8.3 插值变量的传递

错误消息支持动态插值，变量从 `AutomationErrorKind` 透传到 Trans 组件：

```
错误对象: { kind: 'schedule-not-found', name: 'NonExistent' }
    ↓
Trans 组件: <Trans>No schedule named "{{ name }}"</Trans>
    ↓
i18n 处理: 
  - Key: "No schedule named \"{{name}}\""
  - 插值: { name: 'NonExistent' }
    ↓
中文翻译: "未找到名为 \"{{name}}\" 的日程"
    ↓
最终输出: "未找到名为 \"NonExistent\" 的日程"
```

#### 4.8.4 错误提示的视觉渲染链路

```
validateAutomation() 输出错误
    ↓
AutomationListRow 接收到 error prop
    ├─ 视觉样式：红色边框、红色背景、警告图标
    └─ 内容渲染：
        ├─ error 存在 → 渲染 <AutomationErrorShort error={error} />
        │   └─ 内部 <Trans> 组件 → i18n 翻译
        └─ error 不存在 → 渲染 <TemplateSentence template={...} />
            └─ 内部 <Trans> 组件 → i18n 翻译
```

#### 4.8.5 全局冲突提示的 i18n 协作

`ConflictBanner` 组件展示跨规则冲突：

```typescript
export function GlobalConflictDetail({ conflict }) {
  const format = useFormat();
  switch (conflict.kind) {
    case 'over-income':
      // Key: "This month's automations ask for around {{total}} but only {{income}} is available..."
      return (
        <Trans>
          This month&rsquo;s automations ask for around {{ total: format(conflict.total, 'financial') }}
          but only {{ income: format(conflict.income, 'financial') }} is available to budget...
        </Trans>
      );
    case 'percent-over-100':
      // Key: "Your percent automations add up to more than 100% and will be capped at 100%."
      return <Trans>Your percent automations add up to more than 100% and will be capped at 100%.</Trans>;
  }
}
```

### 4.9 预算自动化文案全链路总结

#### 4.9.1 正常文案路径

```
Template 对象 (type: 'percentage', percent: 10, category: 'Salary')
    ↓
PercentageAutomationReadOnly 组件
    ↓
<Trans>Budget {{ percent }}% of &lsquo;{{ category }}&rsquo; this month</Trans>
    ↓
i18next:
  - Key: "Budget {{percent}}% of '{{category}}' this month"
  - 插值: { percent: 10, category: 'Salary' }
  - 查找当前语言翻译
    ├─ 找到 → 返回翻译文本
    └─ 未找到 → 返回 Key 本身 (英文)
    ↓
插值替换 → 最终展示给用户
```

#### 4.9.2 错误文案路径

```
validateAutomation() → { kind: 'schedule-not-found', name: 'Rent' }
    ↓
AutomationErrorShort 组件
    ↓
<Trans>No schedule named &ldquo;{{ name }}&rdquo;</Trans>
    ↓
i18next:
  - Key: "No schedule named \"{{name}}\""
  - 插值: { name: 'Rent' }
  - 查找翻译 → 显示结果
    ↓
用户可见: "未找到名为 \"Rent\" 的日程" (中文) 或 英文原文
```

#### 4.9.3 关键链路节点

| 节点 | 文件位置 | 职责 |
|-----|---------|------|
| i18n 初始化 | `packages/desktop-client/src/i18n.ts` | 配置 i18next，加载语言资源 |
| 语言资源 | `packages/desktop-client/locale/*.json` | 存储各语言翻译 |
| Key 提取 | `packages/desktop-client/i18next-parser.config.js` | 从代码中提取翻译 Key |
| Trans 渲染 | `*ReadOnly.tsx` 组件 | 将模板数据转换为 Trans 组件 |
| 错误消息 | `automationMessages.tsx` | 将错误类型映射为 Trans 文本 |
| 视觉渲染 | `AutomationListRow.tsx` | 根据错误状态选择渲染路径 |

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
| **i18n 层** | `packages/desktop-client/src/i18n.ts` | i18next 初始化与语言解析 |
| | `packages/desktop-client/src/languages.ts` | 语言资源动态加载 |
| | `packages/desktop-client/i18next-parser.config.js` | 翻译 Key 提取配置 |
| | `packages/desktop-client/locale/*.json` | 各语言翻译资源文件 |
| **UI层** | `packages/desktop-client/src/components/modals/BudgetAutomationsModal/` | 自动化模态框 |
| | `packages/desktop-client/src/components/budget/goals/automationMessages.tsx` | 错误消息组件 |
| | `packages/desktop-client/src/components/budget/goals/BudgetAutomationReadOnly.tsx` | 预算表格展示 |
| **类型定义** | `packages/loot-core/src/types/models/templates.ts` | Template 类型定义 |
| | `packages/desktop-client/src/components/budget/goals/constants.ts` | 显示类型映射 |
| **工具链** | `packages/eslint-plugin-actual/lib/rules/no-untranslated-strings.js` | 未翻译字符串检测 |
| | `packages/eslint-plugin-actual/lib/rules/prefer-trans-over-t.js` | Trans 组件优先规则 |
