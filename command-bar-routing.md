# 全局命令面板与快捷键切页链路分析

## 目录
1. [概述](#概述)
2. [命令入口汇总（按实现方式分类）](#命令入口汇总按实现方式分类)
3. [两条并行的执行链路详解](#两条并行的执行链路详解)
4. [焦点状态拦截机制详解](#焦点状态拦截机制详解)
5. [页面跳转路由逻辑](#页面跳转路由逻辑)
6. [测试覆盖逐条对齐](#测试覆盖逐条对齐)
7. [核心文件索引](#核心文件索引)

---

## 概述

Actual Budget 实现了**两条完全独立、互不干涉**的快捷键执行机制，所有实现均有代码可追溯：

### 链路 A: 原生 `document.addEventListener('keydown')` 链 ✅ 代码 100% 证实
- 共 **3 处独立监听器**，完全绕过 `react-hotkeys-hook` 库

### 链路 B: `useHotkeys` Scope 受控链 ✅ 代码 100% 证实
- 共 **8 个文件，总计 26 处 `useHotkeys` 调用**，通过 `react-hotkeys-hook` 库注册
- **25 处** 明确声明 `scopes: ['app']`
- **1 处** (HelpMenu) 未声明 scopes

> **关键修正**：`KeyboardShortcutModal` 仅为**快捷键说明弹窗**，不参与执行。真正的快捷键逻辑分散在各页面组件中。

---

## 命令入口汇总（按实现方式分类）

### 链路 A: 原生 keydown 监听器（共 3 处）

| 序号 | 文件 | 快捷键 | 触发条件 | 功能 |
|-----|------|-------|---------|------|
| A-1 | `CommandBar.tsx:147-162` | `Ctrl/Cmd + K` | `modalStack.length === 0` | 打开命令面板 |
| A-2 | `index.tsx:123-126` | `Ctrl/Cmd + O` | 无条件 | 关闭预算，返回预算列表 |
| A-3 | `index.tsx:127-139` | `Ctrl/Cmd + Z`、`Ctrl/Cmd + Shift + Z` | `!inputFocused(e)` | 撤销/重做 |
| A-4 | `GlobalKeys.ts:16-19` | `Ctrl/Cmd + 1` | `!Platform.isBrowser` | 跳转到 Budget 页面 |
| A-5 | `GlobalKeys.ts:20-22` | `Ctrl/Cmd + 2` | `!Platform.isBrowser` | 跳转到 Reports 页面 |
| A-6 | `GlobalKeys.ts:23-25` | `Ctrl/Cmd + 3` | `!Platform.isBrowser` | 跳转到 Accounts 页面 |
| A-7 | `GlobalKeys.ts:26-29` | `Ctrl/Cmd + ,` | `!Platform.isBrowser && Platform.OS === 'mac'` | 跳转到 Settings 页面 |

**注**：`GlobalKeys.ts` 共 4 个快捷键，均在同一个监听器中处理。

---

### 链路 B: useHotkeys 注册（共 26 处，8 个文件）

#### B-1: 明确声明 `scopes: ['app']`（共 25 处）

| 序号 | 文件 | 行号 | 快捷键 | enabled 条件 | enableOnFormTags |
|-----|------|------|-------|-------------|-----------------|
| 1 | `DynamicBudgetTable.tsx` | 83-93 | `←` (左箭头) | 无 | 默认 false |
| 2 | `DynamicBudgetTable.tsx` | 94-104 | `→` (右箭头) | 无 | 默认 false |
| 3 | `DynamicBudgetTable.tsx` | 105-124 | `0` (零键) | 无 | 默认 false |
| 4 | `Header.tsx` | 231-250 | `Ctrl/Cmd + F` | 无 | **true** |
| 5 | `Header.tsx` | 251-259 | `T` | 无 | 默认 false |
| 6 | `Header.tsx` | 260-267 | `Ctrl/Cmd + I` | 无 | 默认 false |
| 7 | `Header.tsx` | 268-277 | `Ctrl/Cmd + B` | `canSync && !isServerOffline` | 默认 false |
| 8 | `TransactionsTable.tsx` | 180-188 | `Ctrl/Cmd + A` | 无 | 默认 false |
| 9 | `SelectedTransactionsButton.tsx` | 230-233 | `F` | `types.trans` | 默认 false |
| 10 | `SelectedTransactionsButton.tsx` | 234-237 | `U` | `types.trans` | 默认 false |
| 11 | `SelectedTransactionsButton.tsx` | 238-241 | `D` | `types.trans` | 默认 false |
| 12 | `SelectedTransactionsButton.tsx` | 242-245 | `E` | `types.trans` | 默认 false |
| 13 | `SelectedTransactionsButton.tsx` | 246-249 | `A` | `types.trans` | 默认 false |
| 14 | `SelectedTransactionsButton.tsx` | 250-253 | `P` | `types.trans` | 默认 false |
| 15 | `SelectedTransactionsButton.tsx` | 254-257 | `N` | `types.trans` | 默认 false |
| 16 | `SelectedTransactionsButton.tsx` | 258-261 | `C` | `types.trans` | 默认 false |
| 17 | `SelectedTransactionsButton.tsx` | 262-265 | `L` | `types.trans` | 默认 false |
| 18 | `SelectedTransactionsButton.tsx` | 266-274 | `S` | 无 | 默认 false |
| 19 | `SelectedTransactionsButton.tsx` | 276-281 | `M` | `types.trans && !canMerge` | 默认 false |
| 20 | `SelectedTransactionsButton.tsx` | 283-288 | `G` | `types.trans && canMerge` | 默认 false |
| 21 | `SelectedTransactionsButton.tsx` | 290-295 | `R` | `types.trans && showMakeTransfer && canBeTransfer` | 默认 false |
| 22 | `FiltersMenu.tsx` | 688-690 | `F` | 无 | 默认 false |
| 23 | `Titlebar.tsx` | 78-88 | `Shift+Ctrl+P` | 无 | 默认 false |
| 24 | `Titlebar.tsx` | 201-210 | `Ctrl/Cmd + S` | 无 | **true** |
| 25 | `Overview.tsx` | 191-198 | `Ctrl/Cmd + Z` | 无 | 默认 false |

#### B-2: 未声明 scopes（共 1 处）

| 序号 | 文件 | 行号 | 快捷键 | 说明 |
|-----|------|------|-------|------|
| 26 | `HelpMenu.tsx` | 102 | `?` | 仅传 `{ useKey: true }`，未声明 scopes |

---

### 链路 C: 组件内原生 `onKeyDown` 事件处理（非全局）

| 序号 | 文件 | 行号 | 快捷键 | 触发条件 | 功能 |
|-----|------|------|-------|---------|------|
| C-1 | `table.tsx` | 1366-1378 | `J / ↓`、`K / ↑` | `e.target.tagName !== 'INPUT'` | 表格上下移动选中行 |
| C-2 | `table.tsx` | 1383-1396 | `Enter`、`Shift+Enter`、`Tab`、`Shift+Tab` | 编辑模式 | 表格单元格导航 |

---

## 两条并行的执行链路详解

### 关键前提：事件监听是并行的

**代码证实**：应用中存在 **3 个全局原生 keydown 监听器 + 1 个 react-hotkeys-hook 内部监听器**，各自独立处理事件，互不影响：
1. `CommandBar.tsx:160` → `document.addEventListener('keydown', openEventListener)`
2. `GlobalKeys.ts:36` → `document.addEventListener('keydown', handleKeys)`
3. `index.tsx:120` → `document.addEventListener('keydown', e => { ... })`
4. `react-hotkeys-hook` 库内部维护的监听器

> **重要结论**：原生 keydown 监听器 **完全不受** `react-hotkeys-hook` 的 Scope 机制影响。

---

### 链路 A: 原生 keydown 事件链详解

#### A-1: CommandBar 命令面板触发流程

```
用户按下 Ctrl/Cmd+K
    │
    ▼
【CommandBar.tsx:147-162 ✅ 代码 100% 证实
    │
    ├─ 步骤 1: 匹配按键 → e.key === 'k' && (e.metaKey || e.ctrlKey)
    │       代码位置: CommandBar.tsx:149
    │
    ├─ 步骤 2: e.preventDefault() → 阻止浏览器默认行为
    │       代码位置: CommandBar.tsx:150
    │
    ├─ 步骤 3: 独立 Guard 检查 → modalStack.length > 0 ? return
    │       代码位置: CommandBar.tsx:152
    │       └─ Modal 打开时直接 return，不打开命令面板
    │
    └─ 步骤 4: setOpen(true) → 打开命令面板
            代码位置: CommandBar.tsx:153
```

#### A-2: Electron 数字键跳转流程

```
用户按下 Ctrl/Cmd+1/2/3
    │
    ▼
【GlobalKeys.ts:10-33 ✅ 代码 100% 证实
    │
    ├─ 步骤 1: 平台检查 → if (Platform.isBrowser) return
    │       代码位置: GlobalKeys.ts:11-12
    │       └─ 【关键】浏览器环境下直接 return，完全不触发
    │
    ├─ 步骤 2: Meta 键检查 → e.metaKey
    │       代码位置: GlobalKeys.ts:15
    │
    ├─ 步骤 3: 匹配数字键 (1/2/3/,)
    │       代码位置: GlobalKeys.ts:16-29
    │
    └─ 步骤 4: 调用 navigate(path) → 页面跳转
            代码位置: GlobalKeys.ts:18/21/24/27
```

**平台差异 - 代码 100% 证实**：
| 环境 | Ctrl/Cmd+1 行为 | 原因 |
|-----|-----------------|------|
| 浏览器 | ❌ 完全不触发 | `Platform.isBrowser === true → return` |
| Electron (Mac) | ✅ Cmd+1 触发 | `Platform.isBrowser === false` |
| Electron (Win) | ⚠️ 行为待定 | Ctrl 键在 Win Electron 中对应 metaKey 需验证 |

---

### 链路 B: useHotkeys Scope 受控链详解

#### B-1: Scope 配置逐处验证统计

| 配置情况 | 数量 | 占比 |
|---------|------|-----|
| 明确声明 `scopes: ['app']` | 25 处 | 96.2% |
| 未声明 scopes (HelpMenu `?`) | 1 处 | 3.8% |
| **总计** | **26 处** | **100%** |

> **代码证实的结论**：除 `HelpMenu.tsx` 外，其余 25 处 useHotkeys 均明确声明 `scopes: ['app']`。**无一处声明其他 scope。**

---

#### B-2: 预算页月份切换执行流程（以左箭头为例）

```
用户按下 ← (左箭头)
    │
    ▼
【react-hotkeys-hook 库内部监听器
    │
    ├─ 步骤 1: 检查 activeScopes 包含 'app'?
    │       ⚠️ 依赖库语义推断
    │       代码证据: DynamicBudgetTable.tsx:90 声明 scopes: ['app']
    │
    ├─ 步骤 2: 检查 enableOnFormTags（默认 false）
    │       ⚠️ 依赖库语义
    │       代码证据: 未显式启用
    │       └─ INPUT/TEXTAREA/SELECT 有焦点时快捷键不触发
    │
    ├─ 步骤 3: preventDefault: true → 阻止浏览器滚动
    │       ✅ 代码证实: DynamicBudgetTable.tsx:89
    │
    ├─ 步骤 4: 执行回调 → _onMonthSelect(monthUtils.prevMonth(startMonth))
    │       ✅ 代码证实: DynamicBudgetTable.tsx:86
    │
    └─ 步骤 5: 更新 React state → 组件重渲染（不改变 URL）
            ✅ 代码证实: _onMonthSelect 内部 state 更新
```

---

### Modal 打开时两条链路的行为差异

| 链路类型 | 快捷键 | Modal 打开时行为 | 依据 | 可信度 |
|---------|-------|----------------|------|-------|
| **原生 keydown** | Cmd+K | ❌ 被拦截，不触发 | CommandBar.tsx:152 显式检查 `modalStack.length > 0` | **100% 代码证实** |
| **原生 keydown** | Cmd+O/Z | ✅ 仍可触发 | index.tsx 未检查 modalStack | **100% 代码证实** |
| **原生 keydown** | Cmd+1/2/3/, | ✅ 仍可触发 | GlobalKeys.ts 未检查 modalStack | **100% 代码证实** |
| **useHotkeys 'app' scope** | ←/→/0 等 25 处 | ⚠️ 行为依赖库语义 | Modal.tsx 仅调用 `enableScope(name)`，未显式 disable 'app' | **依赖库语义推断** |
| **useHotkeys 无 scope** | `?` (HelpMenu) | ⚠️ 行为依赖库语义 | HelpMenu.tsx:102 未声明 scopes | **依赖库语义推断** |

---

## 焦点状态拦截机制详解

### 拦截层级全景图

```
┌─────────────────────────────────────────────────────────────────────┐
│  优先级从高到低                                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. 【Modal Scope 切换 - 仅影响 useHotkeys 链】⚠️ 依赖库语义       │
│     ┌─────────────────────────────────────────────────────────┐     │
│     │ Modal.tsx:60-66 代码证实的事实：                       │     │
│     │   ✓ enableScope(name) // e.g. 'keyboard-shortcuts'      │     │
│     │   ✓ cleanup: disableScope(name)                         │     │
│     │   ✗ 未调用 disableScope('app') → 未显式禁用 app scope   │     │
│     └─────────────────────────────────────────────────────────┘     │
│     依赖库语义推断：                                                  │
│       → enableScope 是"添加"还是"替换"？需查阅库文档               │
│       → 无 scope 声明的快捷键是否不受影响？                         │
│                                                                     │
├─────────────────────────────────────────────────────────────────────┤
│  2. 【命令面板独立 Guard - 仅影响 Cmd+K】✅ 代码证实                │
│     ┌─────────────────────────────────────────────────────────┐     │
│     │ if (modalStack.length > 0) return;                       │     │
│     └─────────────────────────────────────────────────────────┘     │
│     文件: CommandBar.tsx:152                                        │
│                                                                     │
├─────────────────────────────────────────────────────────────────────┤
│  3. 【表单元素焦点 - 仅影响 useHotkeys 链】⚠️ 依赖库语义 + 代码证实 │
│     ┌─────────────────────────────────────────────────────────┐     │
│     │ enableOnFormTags 默认 false → INPUT 有焦点时不触发              │     │
│     └─────────────────────────────────────────────────────────┘     │
│     例外（显式启用，共 2 处）：                                        │
│       → Ctrl+F (Header.tsx:245): enableOnFormTags: true         │     │
│       → Ctrl+S (Titlebar.tsx:205): enableOnFormTags: true       │     │
│                                                                     │
├─────────────────────────────────────────────────────────────────────┤
│  4. 【表格编辑状态拦截 - 仅影响原生 onKeyDown】✅ 代码证实          │
│     ┌─────────────────────────────────────────────────────────┐     │
│     │ if (e.target.tagName !== 'INPUT') {                        │     │
│     │   onMove('up/down');                                       │     │
│     │ }                                                           │     │
│     └─────────────────────────────────────────────────────────┘     │
│     文件: table.tsx:1366-1378                                      │
│                                                                     │
├─────────────────────────────────────────────────────────────────────┤
│  5. 【自定义 enabled 属性 - 仅影响 useHotkeys 链】✅ 代码证实        │
│     ┌─────────────────────────────────────────────────────────┐     │
│     │ enabled: canSync && !isServerOffline (Header.tsx:272)    │     │
│     │ enabled: types.trans 等条件判断 (共 10 处)                │     │
│     └─────────────────────────────────────────────────────────┘     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 页面跳转路由逻辑

### 1. useNavigate Hook 增强实现

```typescript
// useNavigate.ts:1-54 ✅ 代码证实
export function useNavigate(): NavigateFunction {
  const location = useLocation();
  const navigate = useNavigateReactRouter();
  
  return useCallback(
    (to: To | number, options: NavigateOptions = {}) => {
      if (typeof to === 'number') {
        void navigate(to);  // 历史记录导航: navigate(-1)
      } else {
        const optionsWithPrevLocation: NavigateOptions = {
          replace: options.replace || isSamePath(to, location) ? true : undefined,
          ...options,
          state: {
            ...options?.state,
            previousLocation: location,  // 记录前一位置
          },
        };

        const { previousLocation } = location.state || {};

        if (previousLocation == null || !isSamePath(to, previousLocation)) {
          void navigate(to, optionsWithPrevLocation);
        } else {
          // 智能返回：目标与前一位置相同，直接 go back
          void navigate(-1);
        }
      }
    },
    [navigate, location],
  );
}
```

---

### 2. 页面跳转触发源对比

| 触发方式 | 链路类型 | 是否改变 URL | 典型场景 |
|---------|---------|------------|---------|
| **命令面板选择** | 原生 keydown + 后续交互 | ✅ 是 | 点击搜索结果 / 按 Enter |
| **Electron 数字键** | 原生 keydown | ✅ 是 | Cmd+1 → Budget |
| **预算页月份切换** | useHotkeys | ❌ 否 | 左/右箭头切换月份视图 |

---

### 3. 路由配置结构

```typescript
// FinancesApp.tsx:243-390 ✅ 代码证实
<Routes>
  <Route path="/" element={<Navigate to="/budget" />} />
  <Route path="/budget" element={<BudgetPage />} />
  <Route path="/reports/*" element={<ReportsPage />} />
  <Route path="/accounts" element={<AccountsPage />} />
  <Route path="/accounts/:id" element={<AccountPage />} />
  <Route path="/settings" element={<SettingsPage />} />
  <Route path="/schedules" element={<SchedulesPage />} />
  <Route path="/rules" element={<RulesPage />} />
  <Route path="/tags" element={<ManageTagsPage />} />
  <Route path="/transactions/:transactionId" element={<TransactionEdit />} />
</Routes>
```

---

## 测试覆盖逐条对齐

### 链路节点 vs 测试覆盖对照表

| 链路节点 | 可信度 | 已测场景 | 未测/缺失场景 | 相关测试文件 |
|---------|-------|---------|--------------|------------|
| **A-1. Cmd+K 触发命令面板** | ✅ 代码证实 | ✅ 已测试 | - | `command-bar.test.ts` |
| **A-2. Cmd+K Modal Guard** | ✅ 代码证实 | ❌ 未测试 | Modal 已打开时按 Cmd+K 不应打开 | 无 |
| **A-3. Electron 数字键** | ✅ 代码证实 | ❌ 未测试 | Electron 环境 Cmd+1/2/3 跳转 | 无 (仅 Electron) |
| **A-4. 浏览器禁用数字键** | ✅ 代码证实 | ❌ 未测试 | 浏览器环境 Cmd+1 不应触发 | 无 |
| **B-1. useHotkeys 'app' scope (25 处)** | ✅ 代码证实 | ❌ 未测试 | - | - |
| **B-2. Modal Scope 切换效果** | ⚠️ 依赖库语义 | ❌ 未测试 | Modal 打开时 ←/→ 应无效? | 无 |
| **B-3. enableOnFormTags 拦截** | ⚠️ 依赖库语义 | ❌ 未测试 | 输入框有焦点时 ←/→ 应无效 | 无 |
| **B-4. 预算页月份切换** | ✅ 代码证实 | ❌ 未测试 | 左/右箭头切月、数字 0 返回当月 | 无 |
| **B-5. HelpMenu `?` 快捷键** | ✅ 代码证实 | ❌ 未测试 | 按 `?` 应打开帮助菜单 | 无 |
| **命令面板搜索过滤** | ✅ 代码证实 | ✅ 基本测试 | 特殊字符过滤、空结果处理 | `command-bar.test.ts` |
| **命令面板导航跳转** | ✅ 代码证实 | ✅ 已测试 (Reports) | 所有导航项遍历、Account 详情 | `command-bar.test.ts` |
| **命令面板键盘导航** | ✅ 代码证实 | ✅ 基本测试 (ArrowDown) | Vim 按键绑定、上下键循环 | `command-bar.test.ts` |
| **useNavigate 智能返回** | ✅ 代码证实 | ❌ 未测试 | 目标与前一位置相同时应 go back | 无 |
| **帮助菜单按钮点击** | ✅ 代码证实 | ✅ 已测试 | - | `help-menu.test.ts` |
| **快捷键模态框显示** | ✅ 代码证实 | ✅ 已测试 | - | `help-menu.test.ts` |
| **快捷键模态框搜索** | ✅ 代码证实 | ✅ 已测试 | 所有快捷键搜索、空结果 | `help-menu.test.ts` |
| **快捷键模态框分类** | ✅ 代码证实 | ✅ 已测试 (Global) | Budget、Account 等分类切换 | `help-menu.test.ts` |
| **表格 J/K 导航** | ✅ 代码证实 | ❌ 未测试 | J/K 键上下移动选中行 | 无 |
| **表格 Enter/Tab 导航** | ✅ 代码证实 | ❌ 未测试 | Enter/Tab 键导航单元格 | 无 |

---

### 现有测试详情

#### 命令面板测试 (`command-bar.test.ts`)

| 测试用例 | 覆盖链路节点 | 备注 |
|---------|------------|------|
| 视觉测试 (Ctrl+K 打开 → Escape 关闭) | A-1 | ✅ 完整覆盖 |
| 搜索 "reports" → Enter 导航 | 命令面板导航 | ✅ 基本覆盖，仅测试 Reports |
| ArrowDown 选择第二项 → Enter 导航 | 命令面板键盘导航 | ✅ 基本覆盖，仅测试向下 |

#### 帮助菜单测试 (`help-menu.test.ts`)

| 测试用例 | 覆盖链路节点 | 备注 |
|---------|------------|------|
| 点击 Help 按钮 → 菜单显示 → Escape 关闭 | 帮助菜单按钮点击 | ✅ 完整覆盖 |
| 点击 Keyboard shortcuts → 模态框显示 | 快捷键模态框 | ✅ 完整覆盖 |
| 搜索框输入 "command" → 搜索结果显示 | 快捷键模态框搜索 | ✅ 基本覆盖 |
| Back 按钮返回分类列表 → 点击 Global 分类 | 快捷键模态框分类 | ✅ 仅测试 Global 分类 |

---

## 核心文件索引

| 文件路径 | 主要职责 | 关键代码行数 | useHotkeys 数量 |
|---------|---------|-------------|----------------|
| `packages/desktop-client/src/index.tsx` | 全局原生 keydown 监听 (Undo/Redo/CloseBudget) | 120-139 | 0 |
| `packages/desktop-client/src/components/CommandBar.tsx` | 命令面板原生 keydown 监听 + 独立 Guard | 147-162 | 0 |
| `packages/desktop-client/src/components/GlobalKeys.ts` | Electron 数字键原生 keydown 监听 + 平台检查 | 10-33 | 0 |
| `packages/desktop-client/src/components/common/Modal.tsx` | Modal Scope 切换 (enableScope/disableScope) | 60-66 | 0 |
| `packages/desktop-client/src/components/budget/DynamicBudgetTable.tsx` | useHotkeys 预算页月份切换 (scopes: ['app']) | 83-124 | 3 |
| `packages/desktop-client/src/components/table.tsx` | 表格原生 onKeyDown 焦点拦截 | 1366-1378, 1383-1396 | 0 |
| `packages/desktop-client/src/components/accounts/Header.tsx` | Account 页 useHotkeys 快捷键 + enableOnFormTags | 231-277 | 4 |
| `packages/desktop-client/src/components/transactions/TransactionsTable.tsx` | Ctrl+A 全选交易 (scopes: ['app']) | 180-188 | 1 |
| `packages/desktop-client/src/components/transactions/SelectedTransactionsButton.tsx` | 交易操作快捷键 (scopes: ['app']) | 230-295 | 13 |
| `packages/desktop-client/src/components/filters/FiltersMenu.tsx` | 筛选菜单快捷键 (scopes: ['app']) | 688-690 | 1 |
| `packages/desktop-client/src/components/Titlebar.tsx` | 隐私模式切换 + 同步快捷键 (scopes: ['app']) | 78-88, 201-210 | 2 |
| `packages/desktop-client/src/components/reports/Overview.tsx` | 关闭导入通知快捷键 (scopes: ['app']) | 191-198 | 1 |
| `packages/desktop-client/src/components/HelpMenu.tsx` | 帮助菜单快捷键 (useHotkeys, 无 scope 声明) | 102 | 1 |
| `packages/desktop-client/src/hooks/useNavigate.ts` | 增强型导航 Hook | 1-54 | 0 |
| `packages/desktop-client/src/components/modals/KeyboardShortcutModal.tsx` | **快捷键说明弹窗 (仅展示，不参与执行)** | 108-267 | 0 |
| `packages/desktop-client/e2e/command-bar.test.ts` | 命令面板 E2E 测试 | 1-79 | 0 |
| `packages/desktop-client/e2e/help-menu.test.ts` | 帮助菜单 E2E 测试 | 1-67 | 0 |

**useHotkeys 总计**: 3 + 4 + 1 + 13 + 1 + 2 + 1 + 1 = **26 处**

---

## 最终架构图（双链路并行版）

```
用户按下快捷键
    │
    ▼
┌─────────────────────────────────────────────────────────────────────┐
│          事件到达 document，4 个监听器并行处理                          │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ 【链路 A: 3 个原生 keydown 监听器】✅ 代码 100% 证实               │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │  │
│  │  │ CommandBar   │  │ index.tsx  │  │ GlobalKeys │     │  │
│  │  │ Cmd+K        │  │ Cmd+O/Z    │  │ 1/2/3/, 数字键│     │  │
│  │  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘     │  │
│  │         │              │              │                 │  │
│  │         ▼              ▼              ▼                 │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │  │
│  │  │检查 ModalStack│  │检查焦点    │  │检查平台    │     │  │
│  │  │>0?         │  │inputFocused?│  │isBrowser?  │     │  │
│  │  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘     │  │
│  │         │              │              │                 │  │
│  │         └────────────┐ │ ┌────────────┘                 │  │
│  │                      ▼ ▼ ▼                                    │  │
│  │                  执行回调函数                                  │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ 【链路 B: useHotkeys 库内部监听器】✅ 代码证实 26 处调用        │  │
│  │  ┌───────────────────────────────────────────────────┐   │  │
│  │  │ 1. 匹配声明的 scopes 是否在 activeScopes 中？       │   │  │
│  │  │    → 25 处 scopes: ['app']                     │   │  │
│  │  │    → 1 处 (HelpMenu ?) 未声明 scope             │   │  │
│  │  └──────────────────────┬────────────────────────────┘   │  │
│  │                         │                              │  │
│  │                         ▼                              │  │
│  │  ┌───────────────────────────────────────────────────┐   │  │
│  │  │ 2. enableOnFormTags? INPUT 有焦点？默认不触发      │   │  │
│  │  │    → 例外 (2 处): Ctrl+F, Ctrl+S 显式启用       │   │  │
│  │  └──────────────────────┬────────────────────────────┘   │  │
│  │                         │                              │  │
│  │                         ▼                              │  │
│  │  ┌───────────────────────────────────────────────────┐   │  │
│  │  │ 3. enabled 条件检查？(共 10 处)                     │   │  │
│  │  └──────────────────────┬────────────────────────────┘   │  │
│  │                         │                              │  │
│  │                         ▼                              │  │
│  │                     执行回调函数                          │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                  useNavigate Hook 智能路由处理 ✅ 代码证实              │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 1. 与当前路径相同? → replace=true                         │   │
│  │ 2. 与前一位置相同? → navigate(-1) 智能返回                │   │
│  │ 3. 记录 previousLocation 到 state                            │   │
│  └─────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    React Router <Routes> 匹配渲染 ✅ 代码证实      │
└─────────────────────────────────────────────────────────────────────┘
```

---

*文档生成时间: 2026-05-16*  
*基于 Actual Budget 代码库分析*  
*可信度标记: ✅ 代码证实 / ⚠️ 依赖库语义推断*  
*useHotkeys 统计: 8 文件 / 26 处调用 / 25 处 scopes: ['app'] / 1 处无 scope*
