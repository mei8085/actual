# 全局命令面板与快捷键切页链路分析

## 目录
1. [概述](#概述)
2. [命令入口汇总](#命令入口汇总)
3. [快捷键切页完整执行链路](#快捷键切页完整执行链路)
4. [焦点状态拦截机制](#焦点状态拦截机制)
5. [页面跳转路由逻辑](#页面跳转路由逻辑)
6. [测试覆盖逐条对齐](#测试覆盖逐条对齐)
7. [核心文件索引](#核心文件索引)

---

## 概述

Actual Budget 实现了**三套并行**的快捷键触发系统，围绕 `react-hotkeys-hook` 的 **Scope 机制** 和 **表单输入拦截** 构建了完整的交互链路：

1. **全局命令面板 (Command Bar)** - 搜索式导航入口
2. **react-hotkeys-hook 快捷键系统** - 基于 Scope 的细粒度快捷键控制
3. **GlobalKeys 原生事件监听** - Electron 专用的数字键页面切换

> **重要修正**：KeyboardShortcutModal 仅为**快捷键说明弹窗**，不参与实际执行。真正的快捷键逻辑分散在各页面组件中通过 `useHotkeys` Hook 实现。

---

## 命令入口汇总

### 1. 触发页面切换的快捷键清单

| 快捷键 | 触发位置 | 切换目标 | 实现文件 |
|-------|---------|---------|---------|
| `Ctrl/Cmd + K` → 搜索 + Enter | 全局 | 任意页面 (Accounts, Reports, Settings) | `CommandBar.tsx` |
| `Ctrl/Cmd + 1` | Electron 专用 | Budget 页面 | `GlobalKeys.ts:24-26` |
| `Ctrl/Cmd + 2` | Electron 专用 | Reports 页面 | `GlobalKeys.ts:27-29` |
| `Ctrl/Cmd + 3` | Electron 专用 | Accounts 页面 | `GlobalKeys.ts:30-32` |
| `← / →` (左右箭头) | Budget 页面 | 切换上/下月视图 | `DynamicBudgetTable.tsx:83-103` |
| `0` (零键) | Budget 页面 | 返回当前月 | `DynamicBudgetTable.tsx:105-124` |
| `Ctrl/Cmd + O` | 全局 | 关闭预算，返回预算列表 | (UI 实现) |

---

### 2. 其他功能快捷键（非切页类）

| 类别 | 快捷键 | 功能 | 实现文件 |
|-----|-------|------|---------|
| **全局** | `?` | 打开帮助菜单 | `HelpMenu.tsx:102` |
| **全局** | `Ctrl/Cmd + Z` | 撤销 | (undo 模块) |
| **全局** | `Ctrl/Cmd + Shift + Z` | 重做 | (undo 模块) |
| **Account 页面** | `T` | 添加新交易 | `Header.tsx:255-263` |
| **Account 页面** | `Ctrl/Cmd + F` | 聚焦搜索框 | `Header.tsx:236-247` |
| **Account 页面** | `Ctrl/Cmd + B` | 银行同步 | `Header.tsx:264-275` |
| **Account 页面** | `Ctrl/Cmd + I` | 导入交易 | `Header.tsx:256-263` |
| **Account 页面** | `J / ↓` | 选中下移交易 | `TransactionsTable.tsx:180-188` |
| **Account 页面** | `K / ↑` | 选中上移交易 | `table.tsx:1366-1372` |
| **表格编辑** | `Enter` | 下移编辑行 | `table.tsx:1383-1396` |
| **表格编辑** | `Shift + Enter` | 上移编辑行 | `table.tsx:1383-1396` |
| **表格编辑** | `Tab` | 右移编辑列 | `table.tsx:1383-1396` |
| **表格编辑** | `Shift + Tab` | 左移编辑列 | `table.tsx:1383-1396` |

---

## 快捷键切页完整执行链路

### 链路总览

```
用户按下快捷键
    ↓
[1. HotkeysProvider 初始化]
    ↓
[2. Scope 有效性检查]
    ↓
[3. 表单输入拦截 (enableOnFormTags)]
    ↓
[4. preventDefault 阻止浏览器行为]
    ↓
[5. 执行回调函数 (页面跳转逻辑)]
    ↓
[6. useNavigate Hook 智能路由处理]
    ↓
[7. React Router 路由匹配]
    ↓
页面渲染完成
```

---

### 详细步骤拆解

#### 步骤 1: HotkeysProvider 初始化

```typescript
// App.tsx:200
<HotkeysProvider initiallyActiveScopes={['app']}>
  {/* 整个应用 */}
</HotkeysProvider>
```

- 全局仅初始化一次
- 默认激活 scope: `'app'`
- 所有 `useHotkeys` 声明的快捷键都需要指定 scope

---

#### 步骤 2: Scope 有效性检查 - Modal 打开时的拦截

**核心机制**：Modal 打开时，会**禁用 app scope**，启用自己的 scope：

```typescript
// Modal.tsx:60-66
const { enableScope, disableScope } = useHotkeysContext();

useEffect(() => {
  enableScope(name);  // 例如：'keyboard-shortcuts', 'goal-templates'
  return () => disableScope(name);
}, [enableScope, disableScope, name]);
```

**Scope 切换效果**：
- Modal 打开前: `activeScopes = ['app']` → 全局快捷键有效
- Modal 打开后: `activeScopes = ['modal-name']` → **全局快捷键失效**
- Modal 关闭后: scope 恢复为 `['app']`

**命令面板的额外 Guard**：
```typescript
// CommandBar.tsx:116-128
if (modalStack.length > 0) return;  // 有任何 Modal 打开时，Cmd+K 也无法打开命令面板
```

---

#### 步骤 3: 表单输入拦截 (enableOnFormTags)

**默认行为**：`useHotkeys` 默认 `enableOnFormTags = false`，即输入框获得焦点时快捷键**不触发**。

| 场景 | 快捷键行为 | 原因 |
|-----|-----------|------|
| 输入框 (`<input>`) 有焦点 | 快捷键 **不触发** | `enableOnFormTags` 默认 false |
| 文本域 (`<textarea>`) 有焦点 | 快捷键 **不触发** | 同上 |
| `contenteditable` 元素有焦点 | 快捷键 **不触发** | `enableOnContentEditable` 默认 false |
| 无焦点在表单元素上 | 快捷键正常触发 | - |

**例外情况**：部分快捷键显式启用了表单标签支持：
```typescript
// Header.tsx:236-247 (Ctrl+F 聚焦搜索框)
useHotkeys('ctrl+f, cmd+f, meta+f', e => {
  // 在输入框中按下 Ctrl+F 时依然可以触发
}, {
  enableOnFormTags: true,  // ← 显式启用表单标签支持
  preventDefault: false,
  scopes: ['app'],
});
```

---

#### 步骤 4: preventDefault 阻止浏览器默认行为

大多数快捷键都设置了 `preventDefault: true`，避免触发浏览器默认行为：

```typescript
// DynamicBudgetTable.tsx:88-90
{
  preventDefault: true,  // 阻止浏览器滚动页面
  scopes: ['app'],
}
```

---

#### 步骤 5: 执行回调函数

根据快捷键类型，执行不同的跳转逻辑：

**类型 A: 命令面板导航**
```typescript
// CommandBar.tsx:160-180
onSelect: ({ id }) => {
  const item = navigationItems.find(item => item.id === id);
  if (item) handleNavigate(item.path);  // path 如 '/budget', '/reports'
}
```

**类型 B: Electron 全局数字键**
```typescript
// GlobalKeys.ts:24-32
if (e.metaKey) {
  switch (e.key) {
    case '1': void navigate('/budget'); break;
    case '2': void navigate('/reports'); break;
    case '3': void navigate('/accounts'); break;
  }
}
```

**类型 C: 预算页面月份切换**
```typescript
// DynamicBudgetTable.tsx:83-92
useHotkeys('left', () => {
  _onMonthSelect(monthUtils.prevMonth(startMonth));  // 不改变 URL，仅更新内部 state
}, { preventDefault: true, scopes: ['app'] });
```

---

## 焦点状态拦截机制

### 拦截层级优先级（从高到低）

```
┌──────────────────────────────────────────────────────────┐
│ 1. Modal Scope 拦截 (Highest Priority)                    │
│    → Modal 打开时: activeScopes 从 ['app'] → [modal-name] │
│    → 所有声明 scopes: ['app'] 的快捷键全部失效             │
├──────────────────────────────────────────────────────────┤
│ 2. 命令面板独立 Guard                                     │
│    → CommandBar 监听 keydown 事件时检查 modalStack.length │
│    → Modal 打开时即使 scope 未切换，Cmd+K 也无法触发       │
├──────────────────────────────────────────────────────────┤
│ 3. 表单元素焦点拦截 (enableOnFormTags)                    │
│    → INPUT/TEXTAREA/SELECT 有焦点时默认拦截所有快捷键      │
│    → 可被 enableOnFormTags: true 覆盖                      │
├──────────────────────────────────────────────────────────┤
│ 4. 表格编辑状态拦截                                        │
│    → 原生 keydown 事件中检查 e.target.tagName !== 'INPUT'  │
│    → 输入框有焦点时 J/K 方向键不触发选中移动                │
├──────────────────────────────────────────────────────────┤
│ 5. 自定义 enabled 属性                                     │
│    → useHotkeys({ enabled: canSync && !isServerOffline }) │
│    → 根据业务状态动态启用/禁用快捷键                        │
└──────────────────────────────────────────────────────────┘
```

---

### 表格编辑状态的特殊拦截

在 `table.tsx` 中，使用**原生事件监听**而非 `useHotkeys`，有独立的拦截逻辑：

```typescript
// table.tsx:1366-1378
case 'ArrowUp':
case 'k':
  if (e.target.tagName !== 'INPUT') {  // ← 关键：输入框有焦点时不触发
    e.preventDefault();
    onMove('up');
  }
  break;

case 'ArrowDown':
case 'j':
  if (e.target.tagName !== 'INPUT') {  // ← 关键：输入框有焦点时不触发
    e.preventDefault();
    onMove('down');
  }
  break;
```

---

## 页面跳转路由逻辑

### 1. useNavigate Hook 增强实现

```typescript
// useNavigate.ts:1-54
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

| 导航触发方式 | 是否改变 URL | 使用场景 | 示例 |
|------------|------------|---------|------|
| **命令面板导航** | ✅ 是 | 跨页面跳转 | Budget → Reports |
| **GlobalKeys 数字键** | ✅ 是 | Electron 快速跳转 | Ctrl+1 → Budget |
| **预算月份切换** | ❌ 否 | 同一页面内的状态更新 | 左箭头 → 上月 |

---

### 3. 路由配置结构

```typescript
// FinancesApp.tsx:243-390
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

| 链路节点 | 已测场景 | 未测/缺失场景 | 相关测试文件 |
|---------|---------|--------------|------------|
| **1. HotkeysProvider 初始化** | ❌ 无直接测试 | - | - |
| **2. Modal Scope 切换拦截** | ❌ 未测试 | Modal 打开时 Cmd+K 应无效<br>Modal 打开时全局快捷键应失效 | 无 |
| **3. 命令面板独立 Guard** | ❌ 未测试 | Modal 已打开时按 Cmd+K 不应打开命令面板 | 无 |
| **4. 表单输入拦截** | ❌ 未测试 | 输入框有焦点时按 J/K 不应改变选中<br>输入框有焦点时左右箭头不应改变月份 | 无 |
| **5. 命令面板触发** | ✅ 已测试 | - | `command-bar.test.ts` |
| **6. 命令面板搜索过滤** | ✅ 基本测试 | 特殊字符过滤<br>空结果处理 | `command-bar.test.ts` |
| **7. 命令面板导航跳转** | ✅ 已测试 (Reports) | 所有导航项的完整遍历测试<br>Account 详情页跳转<br>Custom Report 跳转 | `command-bar.test.ts` |
| **8. 命令面板键盘导航** | ✅ 基本测试 (ArrowDown) | Vim 按键绑定测试<br>上下键循环选择 | `command-bar.test.ts` |
| **9. 命令面板关闭** | ✅ 已测试 (Escape) | 点击外部关闭<br>导航后自动关闭 | `command-bar.test.ts` |
| **10. 预算页面月份切换** | ❌ 未测试 | 左/右箭头切月<br>数字 0 返回当月 | 无 |
| **11. GlobalKeys 数字键** | ❌ 未测试 | Electron 环境 Ctrl+1/2/3 跳转 | 无 (仅 Electron) |
| **12. 帮助菜单快捷键** | ✅ 已测试 | - | `help-menu.test.ts` |
| **13. 快捷键模态框搜索** | ✅ 已测试 | 所有快捷键搜索<br>空搜索结果 | `help-menu.test.ts` |
| **14. 快捷键模态框分类** | ✅ 已测试 (Global) | 其他分类切换 (Budget, Account) | `help-menu.test.ts` |
| **15. useNavigate 智能返回** | ❌ 未测试 | 目标与前一位置相同时应触发 go back | 无 |
| **16. 表格编辑快捷键** | ❌ 未测试 | Enter/Tab 导航<br>J/K 选中移动 | 无 |

---

### 现有测试详情

#### 命令面板测试 (`command-bar.test.ts`)

| 测试用例 | 覆盖链路节点 | 备注 |
|---------|------------|------|
| 视觉测试 (Ctrl+K 打开 → Escape 关闭) | 5, 9 | ✅ 完整覆盖 |
| 搜索 "reports" → Enter 导航 | 6, 7 | ✅ 基本覆盖，仅测试 Reports |
| ArrowDown 选择第二项 → Enter 导航 | 8 | ✅ 基本覆盖，仅测试向下 |

#### 帮助菜单测试 (`help-menu.test.ts`)

| 测试用例 | 覆盖链路节点 | 备注 |
|---------|------------|------|
| 点击 Help 按钮 → 菜单显示 → Escape 关闭 | 13 | ✅ 完整覆盖 |
| 点击 Keyboard shortcuts → 模态框显示 | 13, 14 | ✅ 完整覆盖 |
| 搜索框输入 "command" → 搜索结果显示 | 13 | ✅ 基本覆盖 |
| Back 按钮返回分类列表 → 点击 Global 分类 | 14 | ✅ 仅测试 Global 分类 |

---

### 高优先级缺失测试建议

1. **Modal 打开时快捷键拦截测试**
   - 打开任意 Modal → 按 Cmd+K → 命令面板不应打开
   - 打开 Modal → 按左右箭头 → Budget 页面不应切换月份

2. **表单输入焦点拦截测试**
   - 聚焦金额输入框 → 按 J/K → 选中行不应改变
   - 聚焦搜索输入框 → 按左右箭头 → 不应切换预算月份

3. **useNavigate 智能返回测试**
   - Budget → Reports → CommandBar 选择 Budget → 应触发 go back (navigate(-1))

---

## 核心文件索引

| 文件路径 | 主要职责 | 关键代码行数 |
|---------|---------|-------------|
| `packages/desktop-client/src/components/App.tsx` | HotkeysProvider 初始化 | 200 |
| `packages/desktop-client/src/components/CommandBar.tsx` | 命令面板主组件 | 116-128, 160-216 |
| `packages/desktop-client/src/components/GlobalKeys.ts` | Electron 全局数字键导航 | 1-42 |
| `packages/desktop-client/src/components/common/Modal.tsx` | Modal Scope 切换逻辑 | 60-66 |
| `packages/desktop-client/src/components/budget/DynamicBudgetTable.tsx` | 预算页面月份切换快捷键 | 83-124 |
| `packages/desktop-client/src/components/table.tsx` | 表格编辑状态拦截逻辑 | 1354-1430 |
| `packages/desktop-client/src/components/accounts/Header.tsx` | Account 页面快捷键 | 236-275 |
| `packages/desktop-client/src/hooks/useNavigate.ts` | 增强型导航 Hook | 1-54 |
| `packages/desktop-client/src/components/HelpMenu.tsx` | 帮助菜单与快捷键说明入口 | 102 |
| `packages/desktop-client/src/components/modals/KeyboardShortcutModal.tsx` | **快捷键说明弹窗 (仅展示)** | 108-267 |
| `packages/desktop-client/e2e/command-bar.test.ts` | 命令面板 E2E 测试 | 1-79 |
| `packages/desktop-client/e2e/help-menu.test.ts` | 帮助菜单 E2E 测试 | 1-67 |

---

## 架构图总结

```
┌─────────────────────────────────────────────────────────────────────┐
│                        用户按下快捷键                                │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────┐  │
│  │ Ctrl/Cmd+K   │  │ Ctrl/Cmd+1   │  │ ←/→ (预算页)             │  │
│  │ 命令面板     │  │ /2/3 数字键  │  │ J/K (账户页)             │  │
│  └──────┬───────┘  └──────┬───────┘  └────────────┬───────────┘  │
└─────────┼───────────────────┼───────────────────────┼───────────────┘
          │                   │                       │
┌─────────▼───────────────────▼───────────────────────▼───────────────┐
│ 1. Scope 检查 (react-hotkeys-hook)                                  │
│    ├─ 检查 activeScopes 是否包含 'app'                              │
│    └─ Modal 打开时 activeScopes 切换 → 'app' 失效                    │
└─────────┬─────────────────────────────────────────────────────────────┘
          │
┌─────────▼─────────────────────────────────────────────────────────────┐
│ 2. 表单输入拦截 (enableOnFormTags)                                    │
│    ├─ 默认: INPUT/TEXTAREA/SELECT 有焦点时 → 拦截                    │
│    ├─ 例外: Ctrl+F 搜索 (enableOnFormTags=true)                      │
│    └─ 表格原生事件额外检查 e.target.tagName !== 'INPUT'              │
└─────────┬─────────────────────────────────────────────────────────────┘
          │
┌─────────▼─────────────────────────────────────────────────────────────┐
│ 3. 命令面板独立 Guard (仅 Ctrl/Cmd+K)                                 │
│    └─ modalStack.length > 0 → 直接 return，不触发                    │
└─────────┬─────────────────────────────────────────────────────────────┘
          │
┌─────────▼─────────────────────────────────────────────────────────────┐
│ 4. preventDefault 阻止浏览器默认行为                                   │
│    └─ preventDefault: true → 阻止浏览器滚动/历史记录等                │
└─────────┬─────────────────────────────────────────────────────────────┘
          │
┌─────────▼─────────────────────────────────────────────────────────────┐
│ 5. 执行回调                                                           │
│    ├─ 命令面板 → handleNavigate(path)                                 │
│    ├─ 数字键 → navigate('/budget')                                    │
│    └─ 月份切换 → onMonthSelect(prev/nextMonth) [不改变URL]           │
└─────────┬─────────────────────────────────────────────────────────────┘
          │
┌─────────▼─────────────────────────────────────────────────────────────┐
│ 6. useNavigate Hook                                                   │
│    ├─ 检查是否与当前路径相同 → replace=true                          │
│    ├─ 检查是否与前一位置相同 → navigate(-1) 智能返回                 │
│    └─ 记录 previousLocation 到 state                                 │
└─────────┬─────────────────────────────────────────────────────────────┘
          │
┌─────────▼─────────────────────────────────────────────────────────────┐
│ 7. React Router <Routes> 匹配                                         │
│    └─ path 匹配 → 渲染对应页面组件                                   │
└───────────────────────────────────────────────────────────────────────┘
```

---

*文档生成时间: 2026-05-16*  
*基于 Actual Budget 代码库分析*  
*关键修正：区分了快捷键说明弹窗与实际执行链路*
