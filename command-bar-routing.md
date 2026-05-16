# 全局命令面板与快捷键切页链路分析

## 目录
1. [概述](#概述)
2. [命令入口汇总](#命令入口汇总)
3. [两条并行的执行链路详解](#两条并行的执行链路详解)
4. [焦点状态拦截机制详解](#焦点状态拦截机制详解)
5. [页面跳转路由逻辑](#页面跳转路由逻辑)
6. [测试覆盖逐条对齐](#测试覆盖逐条对齐)
7. [核心文件索引](#核心文件索引)

---

## 概述

Actual Budget 实现了**两条完全独立、互不干涉**的快捷键执行链路：

### 链路 A：原生 keydown 事件链 (共 3 个独立监听器)
**代码证实**：通过 `document.addEventListener('keydown')` 直接绑定，完全绕过 `react-hotkeys-hook` 库
1. **命令面板触发** - `CommandBar.tsx` → `Ctrl/Cmd+K`
2. **全局快捷键** - `index.tsx` → `Ctrl/Cmd+O`, `Ctrl/Cmd+Z`, `Ctrl/Cmd+Shift+Z`
3. **Electron 数字键** - `GlobalKeys.ts` → `Ctrl/Cmd+1/2/3` (仅桌面端生效)

### 链路 B：useHotkeys Scope 受控链
**代码证实**：通过 `useHotkeys` Hook 注册，受 Scope 机制控制
- 预算页月份切换：`←/→/0`
- 账户页操作：`T` `Ctrl/Cmd+F` `J/K` `Enter/Tab` 等
- 所有快捷键声明：`scopes: ['app']`

> **关键修正**：KeyboardShortcutModal 仅为**快捷键说明弹窗**，不参与执行。真正的快捷键逻辑分散在各页面组件中。

---

## 命令入口汇总

### 1. 原生 keydown 链 - 触发页面切换的快捷键

| 快捷键 | 触发条件 | 切换目标 | 实现文件 | 生效环境 |
|-------|---------|---------|---------|---------|
| `Ctrl/Cmd + K` → 搜索 + Enter | 无条件 (modalStack.length === 0 时) | 任意页面 | `CommandBar.tsx:147-162` | 浏览器 + Electron |
| `Ctrl/Cmd + 1` | `Platform.isBrowser === false` | Budget 页面 | `GlobalKeys.ts:10-33` | **仅 Electron** |
| `Ctrl/Cmd + 2` | `Platform.isBrowser === false` | Reports 页面 | `GlobalKeys.ts:10-33` | **仅 Electron** |
| `Ctrl/Cmd + 3` | `Platform.isBrowser === false` | Accounts 页面 | `GlobalKeys.ts:10-33` | **仅 Electron** |
| `Ctrl/Cmd + ,` | `Platform.isBrowser === false && Platform.OS === 'mac'` | Settings 页面 | `GlobalKeys.ts:26-29` | **仅 Mac Electron** |
| `Ctrl/Cmd + O` | 无条件 | 关闭预算，返回预算列表 | `index.tsx:120-126` | 浏览器 + Electron |

---

### 2. useHotkeys Scope 链 - 触发页面切换的快捷键

| 快捷键 | 触发条件 | 切换目标 | 实现文件 | 生效环境 |
|-------|---------|---------|---------|---------|
| `←` (左箭头) | INPUT 无焦点 + Modal 未打开 | 预算页 - 上月视图 | `DynamicBudgetTable.tsx:83-93` | 浏览器 + Electron |
| `→` (右箭头) | INPUT 无焦点 + Modal 未打开 | 预算页 - 下月视图 | `DynamicBudgetTable.tsx:94-104` | 浏览器 + Electron |
| `0` (零键) | INPUT 无焦点 + Modal 未打开 | 预算页 - 返回当前月 | `DynamicBudgetTable.tsx:105-124` | 浏览器 + Electron |

---

### 3. 其他功能快捷键 (非切页类)

| 链路类型 | 快捷键 | 功能 | 实现文件 |
|---------|-------|------|---------|
| **原生 keydown** | `?` | 打开帮助菜单 | `HelpMenu.tsx:102` |
| **原生 keydown** | `Ctrl/Cmd + Z` | 撤销 | `index.tsx:128-139` |
| **原生 keydown** | `Ctrl/Cmd + Shift + Z` | 重做 | `index.tsx:128-139` |
| **useHotkeys** | `T` | 添加新交易 | `Header.tsx:255-263` |
| **useHotkeys** | `Ctrl/Cmd + F` | 聚焦搜索框 | `Header.tsx:236-247` |
| **useHotkeys** | `Ctrl/Cmd + B` | 银行同步 | `Header.tsx:264-275` |
| **useHotkeys** | `Ctrl/Cmd + I` | 导入交易 | `Header.tsx:256-263` |
| **原生事件(表格)** | `J / ↓` | 选中下移交易 | `table.tsx:1366-1378` |
| **原生事件(表格)** | `K / ↑` | 选中上移交易 | `table.tsx:1366-1378` |
| **原生事件(表格)** | `Enter` | 下移编辑行 | `table.tsx:1383-1396` |
| **原生事件(表格)** | `Shift + Enter` | 上移编辑行 | `table.tsx:1383-1396` |
| **原生事件(表格)** | `Tab` | 右移编辑列 | `table.tsx:1383-1396` |
| **原生事件(表格)** | `Shift + Tab` | 左移编辑列 | `table.tsx:1383-1396` |

---

## 两条并行的执行链路详解

### 关键前提：事件监听是并行的

**代码证实**：应用中存在**4 个独立的 keydown 监听器**，各自独立处理事件，互不影响：
1. `CommandBar.tsx:160` → `document.addEventListener('keydown', openEventListener)`
2. `GlobalKeys.ts:36` → `document.addEventListener('keydown', handleKeys)`
3. `index.tsx:120` → `document.addEventListener('keydown', e => { ... })`
4. `react-hotkeys-hook` 库内部维护的监听器

> **重要结论**：原生 keydown 监听器 **完全不受** `react-hotkeys-hook` 的 Scope 机制影响。

---

### 链路 A：原生 keydown 事件链 (以 Ctrl/Cmd+K 为例)

```
用户按下 Ctrl/Cmd+K
    │
    ├───────────────────────────────────────────────────────────┐
    │  事件冒泡到 document，触发 4 个独立监听器                  │
    └───────────────────────────────────────────────────────────┘
    │
    ▼
【CommandBar.tsx 监听器 - 代码 100% 证实】
    │
    ├─ 步骤 1: 检查按键匹配 → e.key === 'k' && (e.metaKey || e.ctrlKey)
    │       代码证实: CommandBar.tsx:149
    │
    ├─ 步骤 2: e.preventDefault() → 阻止浏览器默认行为
    │       代码证实: CommandBar.tsx:150
    │
    ├─ 步骤 3: 命令面板独立 Guard → modalStack.length > 0 ? return
    │       代码证实: CommandBar.tsx:152
    │       └─ Modal 打开时直接 return，不打开命令面板
    │
    └─ 步骤 4: setOpen(true) → 打开命令面板
            代码证实: CommandBar.tsx:153
```

---

### 链路 A-1：Electron 数字键分支 (Ctrl/Cmd+1/2/3)

```
用户按下 Ctrl/Cmd+1/2/3
    │
    ▼
【GlobalKeys.ts 监听器 - 代码 100% 证实】
    │
    ├─ 步骤 1: 平台检查 → if (Platform.isBrowser) return
    │       代码证实: GlobalKeys.ts:11-12
    │       └─ 【关键】浏览器环境下直接 return，完全不触发
    │
    ├─ 步骤 2: Meta 键检查 → e.metaKey (Mac 上的 Cmd 键)
    │       代码证实: GlobalKeys.ts:15
    │       └─ 注意: Ctrl 键在 Electron 中对应 metaKey?
    │
    ├─ 步骤 3: 匹配数字键 (1/2/3)
    │       代码证实: GlobalKeys.ts:16-25
    │
    └─ 步骤 4: 调用 navigate(path) → 页面跳转
            代码证实: GlobalKeys.ts:18/21/24

【平台差异总结 - 代码证实】
┌─────────────────┬─────────────────┬─────────────────────┐
│ 环境            │ Ctrl/Cmd+1 行为 │ 原因                 │
├─────────────────┼─────────────────┼─────────────────────┤
│ 浏览器 (Chrome) │ ❌ 完全不触发   │ Platform.isBrowser = true → return │
│ Electron (Mac)  │ ✅ Cmd+1 触发   │ Platform.isBrowser = false │
│ Electron (Win)  │ ✅ Ctrl+1 触发?  │ e.metaKey 在 Win Electron 行为待验证 │
└─────────────────┴─────────────────┴─────────────────────┘
```

---

### 链路 B：useHotkeys Scope 受控链 (以预算页左箭头为例)

```
用户按下 ← (左箭头)
    │
    ▼
【react-hotkeys-hook 库内部监听器 - 依赖库语义】
    │
    ├─ 步骤 1: 检查 activeScopes 是否包含 'app'
    │       依赖库语义: react-hotkeys-hook 的 Scope 匹配机制
    │       代码证实: 所有 useHotkeys 调用都声明 scopes: ['app']
    │
    ├─ 步骤 2: 检查 enableOnFormTags (默认 false)
    │       依赖库语义: INPUT/TEXTAREA/SELECT 有焦点时跳过
    │       代码证实: DynamicBudgetTable.tsx:88-90 未显式启用
    │
    ├─ 步骤 3: preventDefault: true → 阻止浏览器滚动
    │       代码证实: DynamicBudgetTable.tsx:89
    │
    ├─ 步骤 4: 执行回调 → _onMonthSelect(monthUtils.prevMonth(startMonth))
    │       代码证实: DynamicBudgetTable.tsx:86
    │
    └─ 步骤 5: 更新 React state → 组件重渲染（不改变 URL）
            代码证实: _onMonthSelect 内部 state 更新
```

---

### Modal 打开时两条链路的行为差异

| 链路类型 | Modal 打开时的行为 | 依据 | 可信度 |
|---------|------------------|------|-------|
| **原生 keydown 链** (Cmd+K) | ❌ 被拦截，不触发 | CommandBar.tsx:152 显式检查 `modalStack.length > 0` | **100% 代码证实** |
| **原生 keydown 链** (Cmd+O/Z) | ✅ 仍可触发 | index.tsx 未检查 modalStack | **100% 代码证实** |
| **原生 keydown 链** (Electron 数字键) | ✅ 仍可触发 | GlobalKeys.ts 未检查 modalStack | **100% 代码证实** |
| **useHotkeys Scope 链** (←/→/0) | ❌ 很可能被拦截 | Modal.tsx 调用 `enableScope(name)` + `disableScope(name)` | **依赖库语义推断，未 100% 证实** |

> **重要区分**：
> - ✅ **代码证实**：在 Actual 代码库中可直接找到对应逻辑
> - ⚠️ **依赖库语义推断**：行为依赖 `react-hotkeys-hook` 库的内部实现语义，代码中仅调用 API，未显式验证其效果

---

## 焦点状态拦截机制详解

### 拦截层级全景图

```
┌─────────────────────────────────────────────────────────────────────┐
│  优先级从高到低                                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. 【Scope 层 - 仅影响 useHotkeys 链】⚠️ 依赖库语义推断             │
│     ┌─────────────────────────────────────────────────────────┐   │
│     │ Modal 打开时:                                            │   │
│     │   enableScope(modalName)   → 激活该 Modal 的 scope      │   │
│     │   disableScope(name)?      → 是否禁用 'app' scope?      │   │
│     └─────────────────────────────────────────────────────────┘   │
│     状态: Modal.tsx:60-66 只调用 enableScope，未显式 disable 'app'  │
│     结论: enableScope/disableScope 精确语义需查阅库文档              │
│                                                                     │
├─────────────────────────────────────────────────────────────────────┤
│  2. 【命令面板独立 Guard - 仅影响 Cmd+K】✅ 代码证实                 │
│     ┌─────────────────────────────────────────────────────────┐   │
│     │ if (modalStack.length > 0) return;                       │   │
│     └─────────────────────────────────────────────────────────┘   │
│     文件: CommandBar.tsx:152                                        │
│                                                                     │
├─────────────────────────────────────────────────────────────────────┤
│  3. 【表单元素焦点 - 仅影响 useHotkeys 链】⚠️ 依赖库语义 + 代码证实  │
│     ┌─────────────────────────────────────────────────────────┐   │
│     │ enableOnFormTags: 默认 false                               │   │
│     │ → INPUT/TEXTAREA/SELECT 有焦点时，快捷键不触发            │   │
│     └─────────────────────────────────────────────────────────┘   │
│     例外: Ctrl+F 显式设置 enableOnFormTags: true → 输入框中可触发  │
│     文件: Header.tsx:249                                           │
│                                                                     │
├─────────────────────────────────────────────────────────────────────┤
│  4. 【表格编辑状态拦截 - 仅影响原生事件链】✅ 代码证实                │
│     ┌─────────────────────────────────────────────────────────┐   │
│     │ if (e.target.tagName !== 'INPUT') {                      │   │
│     │   onMove('up/down');                                      │   │
│     │ }                                                         │   │
│     └─────────────────────────────────────────────────────────┘   │
│     文件: table.tsx:1366-1378                                      │
│                                                                     │
├─────────────────────────────────────────────────────────────────────┤
│  5. 【自定义 enabled 属性 - 仅影响 useHotkeys 链】✅ 代码证实        │
│     ┌─────────────────────────────────────────────────────────┐   │
│     │ enabled: canSync && !isServerOffline                      │   │
│     └─────────────────────────────────────────────────────────┘   │
│     文件: Header.tsx:274                                           │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

### 关于 Modal Scope 的不确定性说明

**代码中可观察到的事实** (✅ 100% 证实)：
```typescript
// Modal.tsx:60-66
const { enableScope, disableScope } = useHotkeysContext();

useEffect(() => {
  enableScope(name);  // 例如: name = 'keyboard-shortcuts'
  return () => disableScope(name);
}, [enableScope, disableScope, name]);
```

**依赖库语义推断** (⚠️ 未 100% 证实)：
- 推测 1: `enableScope('modal-name')` 仅**添加**激活的 scope，不会自动禁用 'app'
- 推测 2: Modal 内部的快捷键需要声明 `scopes: ['modal-name']` 才能生效
- 推测 3: 'app' scope 的快捷键在 Modal 打开时**可能仍然有效**，除非显式禁用

**建议**：如需 100% 确定行为，需查阅 `react-hotkeys-hook` 库文档或编写测试验证。

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

### 2. 三种导航场景的对比

| 导航触发方式 | 链路类型 | 是否改变 URL | 使用场景 | 示例 |
|------------|---------|------------|---------|------|
| **命令面板导航** | 原生 keydown | ✅ 是 | 跨页面跳转 | Budget → Reports |
| **Electron 数字键** | 原生 keydown | ✅ 是 | 桌面端快速跳转 | Cmd+1 → Budget |
| **预算月份切换** | useHotkeys | ❌ 否 | 同一页面内状态更新 | 左箭头 → 上月 |

---

### 3. 路由配置结构

```typescript
// FinancesApp.tsx:243-390 ✅ 代码证实
<Routes>
  <Route path="/" element={<Navigate to="/budget" />} />
  <Route path="/budget" element={<BudgetPage />} />
  <Route path="/reports/*" element={<ReportsPage />} />
  <Route path="/reports/custom/:id" element={<CustomReport />} />
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
| **B-1. useHotkeys Scope 匹配** | ⚠️ 依赖库语义 | ❌ 未测试 | - | - |
| **B-2. Modal Scope 切换效果** | ⚠️ 依赖库语义 | ❌ 未测试 | Modal 打开时 ←/→ 应无效? | 无 |
| **B-3. enableOnFormTags 拦截** | ⚠️ 依赖库语义 | ❌ 未测试 | 输入框有焦点时 ←/→ 应无效 | 无 |
| **B-4. 预算页月份切换** | ✅ 代码证实 | ❌ 未测试 | 左/右箭头切月、数字 0 返回当月 | 无 |
| **命令面板搜索过滤** | ✅ 代码证实 | ✅ 基本测试 | 特殊字符过滤、空结果处理 | `command-bar.test.ts` |
| **命令面板导航跳转** | ✅ 代码证实 | ✅ 已测试 (Reports) | 所有导航项遍历、Account 详情 | `command-bar.test.ts` |
| **命令面板键盘导航** | ✅ 代码证实 | ✅ 基本测试 (ArrowDown) | Vim 按键绑定、上下键循环 | `command-bar.test.ts` |
| **useNavigate 智能返回** | ✅ 代码证实 | ❌ 未测试 | 目标与前一位置相同时应 go back | 无 |
| **帮助菜单快捷键** | ✅ 代码证实 | ✅ 已测试 | - | `help-menu.test.ts` |
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
| 点击 Help 按钮 → 菜单显示 → Escape 关闭 | 帮助菜单快捷键 | ✅ 完整覆盖 |
| 点击 Keyboard shortcuts → 模态框显示 | 快捷键模态框 | ✅ 完整覆盖 |
| 搜索框输入 "command" → 搜索结果显示 | 快捷键模态框搜索 | ✅ 基本覆盖 |
| Back 按钮返回分类列表 → 点击 Global 分类 | 快捷键模态框分类 | ✅ 仅测试 Global 分类 |

---

### 高优先级缺失测试建议

1. **Modal 打开时 Cmd+K 拦截测试** ✅ 逻辑已实现，仅缺测试
   - 打开任意 Modal → 按 Cmd+K → 命令面板不应打开

2. **浏览器环境数字键不触发测试** ✅ 逻辑已实现，仅缺测试
   - 浏览器环境 → 按 Cmd+1 → 不应跳转

3. **表单输入焦点对 useHotkeys 的影响** ⚠️ 需验证库语义
   - 聚焦金额输入框 → 按左右箭头 → 预算月份不应切换

4. **Modal Scope 切换效果验证** ⚠️ 需验证库语义
   - 打开 Modal → 按左右箭头 → 预算月份不应切换 (预期，需验证)

---

## 核心文件索引

| 文件路径 | 主要职责 | 关键代码行数 |
|---------|---------|-------------|
| `packages/desktop-client/src/index.tsx` | 全局原生 keydown 监听 (Undo/Redo/CloseBudget) | 120-139 |
| `packages/desktop-client/src/components/CommandBar.tsx` | 命令面板原生 keydown 监听 + 独立 Guard | 147-162 |
| `packages/desktop-client/src/components/GlobalKeys.ts` | Electron 数字键原生 keydown 监听 + 平台检查 | 10-33 |
| `packages/desktop-client/src/components/common/Modal.tsx` | Modal Scope 切换 (enableScope/disableScope) | 60-66 |
| `packages/desktop-client/src/components/budget/DynamicBudgetTable.tsx` | useHotkeys 预算页月份切换 | 83-124 |
| `packages/desktop-client/src/components/table.tsx` | 表格原生事件焦点拦截 | 1366-1378, 1383-1396 |
| `packages/desktop-client/src/components/accounts/Header.tsx` | Account 页面 useHotkeys 快捷键 + enableOnFormTags | 236-275 |
| `packages/desktop-client/src/hooks/useNavigate.ts` | 增强型导航 Hook | 1-54 |
| `packages/desktop-client/src/components/HelpMenu.tsx` | 帮助菜单快捷键 (useHotkeys) | 102 |
| `packages/desktop-client/src/components/modals/KeyboardShortcutModal.tsx` | **快捷键说明弹窗 (仅展示，不参与执行)** | 108-267 |
| `packages/desktop-client/e2e/command-bar.test.ts` | 命令面板 E2E 测试 | 1-79 |
| `packages/desktop-client/e2e/help-menu.test.ts` | 帮助菜单 E2E 测试 | 1-67 |

---

## 最终架构图 (双链路并行版)

```
用户按下快捷键
    │
    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                  事件到达 document，4 个监听器并行触发                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ 【链路 A: 原生 keydown 监听器】✅ 代码 100% 证实               │  │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐                    │  │
│  │  │Cmd+K     │  │Ctrl/Cmd+O│  │Electron  │                    │  │
│  │  │CommandBar│  │Z/Undo    │  │1/2/3数字键│                    │  │
│  │  └────┬─────┘  └────┬─────┘  └────┬─────┘                    │  │
│  │       │              │              │                           │  │
│  │       ▼              ▼              ▼                           │  │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐                    │  │
│  │  │检查Modal │  │检查焦点  │  │检查平台  │                    │  │
│  │  │Stack>0?  │  │inputFocus?│  │isBrowser?│                    │  │
│  │  └────┬─────┘  └────┬─────┘  └────┬─────┘                    │  │
│  │       │              │              │                           │  │
│  │       └────────────┐ │ ┌────────────┘                           │  │
│  │                    ▼ ▼ ▼                                           │  │
│  │                执行回调函数                                         │  │
│  └───────────────────────────────────────────────────────────────────┘
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ 【链路 B: useHotkeys Scope 受控链】⚠️ 部分依赖库语义            │  │
│  │  ┌─────────────────────────────────────────────────────┐   │  │
│  │  │ react-hotkeys-hook 库内部监听器                       │   │  │
│  │  └────────────────────┬────────────────────────────────┘   │  │
│  │                       │                                       │  │
│  │                       ▼                                       │  │
│  │  ┌─────────────────────────────────────────────────────┐   │  │
│  │  │ 1. activeScopes 包含 'app'? ⚠️ 依赖库语义             │   │  │
│  │  └────────────────────┬────────────────────────────────┘   │  │
│  │                       │                                       │  │
│  │                       ▼                                       │  │
│  │  ┌─────────────────────────────────────────────────────┐   │  │
│  │  │ 2. enableOnFormTags? INPUT有焦点? ⚠️ 依赖库语义      │   │  │
│  │  └────────────────────┬────────────────────────────────┘   │  │
│  │                       │                                       │  │
│  │                       ▼                                       │  │
│  │  ┌─────────────────────────────────────────────────────┐   │  │
│  │  │ 3. enabled 条件检查? ✅ 代码证实                       │   │  │
│  │  └────────────────────┬────────────────────────────────┘   │  │
│  │                       │                                       │  │
│  │                       ▼                                       │  │
│  │                   执行回调函数                                 │  │
│  └───────────────────────────────────────────────────────────────────┘
│                                                                     │
└───────────────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                  useNavigate Hook 智能路由处理                        │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 1. 与当前路径相同? → replace=true                            │   │
│  │ 2. 与前一位置相同? → navigate(-1) 智能返回                   │   │
│  │ 3. 记录 previousLocation 到 state                            │   │
│  └─────────────────────────────────────────────────────────────┘   │
└───────────────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    React Router <Routes> 匹配渲染                    │
└───────────────────────────────────────────────────────────────────────┘
```

---

*文档生成时间: 2026-05-16*  
*基于 Actual Budget 代码库分析*  
*可信度标记: ✅ 代码证实 / ⚠️ 依赖库语义推断*
