# 全局命令面板与快捷键切页链路分析

## 目录
1. [概述](#概述)
2. [命令入口汇总](#命令入口汇总)
3. [焦点状态管理机制](#焦点状态管理机制)
4. [页面跳转路由逻辑](#页面跳转路由逻辑)
5. [测试覆盖分析](#测试覆盖分析)
6. [核心文件索引](#核心文件索引)

---

## 概述

Actual Budget 实现了两套并行的命令/快捷键触发系统：
1. **全局命令面板 (Command Bar)** - 通过 `Ctrl/Cmd + K` 触发，提供搜索式导航
2. **键盘快捷键模态框 (Keyboard Shortcuts Modal)** - 通过 `?` 或 Help 菜单触发，展示所有可用快捷键

这两套系统都围绕 Redux 状态管理、React Router 路由和焦点控制构建了完整的交互链路。

---

## 命令入口汇总

### 1. 全局命令面板 (Command Bar)

#### 触发入口
| 触发方式 | 触发位置 | 实现代码 |
|---------|---------|---------|
| 键盘快捷键 | `Ctrl/Cmd + K` | `CommandBar.tsx:116-128` |
| 程序调用 | 直接渲染组件 | `FinancesApp.tsx:199` |

#### 命令数据源
命令面板的命令来自以下几个动态数据源：

**A. 静态导航项 (Navigation Items)**
- 位置: `CommandBar.tsx:92-114`
- 包含: Budget, Reports, Schedules, Payees, Rules, Tags, Settings, All Accounts
- 特点: 硬编码在组件内，使用 `useMemo` 缓存

**B. 账户列表 (Accounts)**
- 位置: `CommandBar.tsx:194-204`
- 数据源: `useAccounts()` hook
- 包含: On Budget, Off Budget, 以及所有未关闭的账户
- 特点: 动态数据，随账户变化更新

**C. 仪表盘页面 (Dashboard Pages)**
- 位置: `CommandBar.tsx:206-210`
- 数据源: `useDashboardPages()` hook
- 特点: 动态加载用户自定义仪表盘

**D. 自定义报告 (Custom Reports)**
- 位置: `CommandBar.tsx:212-216`
- 数据源: `useReports()` hook
- 特点: 动态加载用户创建的报告

#### 命令分组结构
```
┌─────────────────────────────────────────┐
│ Navigation                              │  (静态导航)
│   Budget, Reports, Schedules...         │
├─────────────────────────────────────────┤
│ Accounts                                │  (账户列表)
│   On Budget, Off Budget, Account 1...   │
├─────────────────────────────────────────┤
│ Reports                                 │  (仪表盘页面)
│   Dashboard 1, Dashboard 2...           │
├─────────────────────────────────────────┤
│ Custom Reports                          │  (自定义报告)
│   Report 1, Report 2...                 │
└─────────────────────────────────────────┘
```

---

### 2. 键盘快捷键模态框 (Keyboard Shortcuts Modal)

#### 触发入口
| 触发方式 | 触发位置 | 实现代码 |
|---------|---------|---------|
| 键盘快捷键 | `?` | `HelpMenu.tsx:102` |
| Help 菜单 | 点击 Help → Keyboard shortcuts | `HelpMenu.tsx:91-93` |
| Redux Action | `pushModal({ name: 'keyboard-shortcuts' })` | `modalsSlice.ts:717-730` |

#### 快捷键分类
快捷键按使用场景分为以下类别：

| 类别 | 位置 | 包含快捷键 |
|-----|------|-----------|
| Global | `KeyboardShortcutModal.tsx:108-135` | Help menu, Command Palette, Close budget, Toggle privacy, Undo, Redo |
| Budget Page | `KeyboardShortcutModal.tsx:136-152` | Move down/up, Current month, Previous month, Next month |
| Account Page General | `KeyboardShortcutModal.tsx:153-170` | Bank sync, Import transactions, Add transaction, Filter menu |
| Account Transaction Selection | `KeyboardShortcutModal.tsx:171-194` | Select all, Toggle selection, Move up/down |
| Account Transaction Editing | `KeyboardShortcutModal.tsx:195-214` | Move down/up, Add and close, Tab navigation |
| Account Transaction Management | `KeyboardShortcutModal.tsx:215-267` | Set date/payee/notes/category/amount/account, Toggle cleared, Link schedule, Filter, Delete, Duplicate, Merge, Make transfer |

---

## 焦点状态管理机制

### 1. 命令面板焦点控制

#### 打开限制 (Modal Guard)
命令面板不会在模态框已打开时打开，避免焦点冲突：

```typescript
// CommandBar.tsx:116-128
const openEventListener = useCallback(
  (e: KeyboardEvent) => {
    if (e.key === 'k' && (e.metaKey || e.ctrlKey)) {
      e.preventDefault();
      // 关键: 如果已有模态框打开，不打开命令面板
      if (modalStack.length > 0) return;
      setOpen(true);
    }
  },
  [modalStack.length],
);
```

#### 自动聚焦
命令面板使用 `cmdk` 库的 `Command.Input` 自动聚焦特性：
- 打开时输入框自动获得焦点
- 支持键盘上下箭头导航
- 支持 Vim 按键绑定 (`vimBindings` 属性)

---

### 2. 快捷键模态框焦点控制

#### 打开限制 (Modal Guard)
键盘快捷键模态框有更严格的打开限制：

```typescript
// modalsSlice.ts:717-730
pushModal(state, action: PayloadAction<PushModalPayload>) {
  const modal = action.payload.modal;
  // 特殊情况: 如果已有模态框打开或 filters-menu 打开，不打开快捷键模态框
  if (
    modal.name.endsWith('keyboard-shortcuts') &&
    (state.modalStack.length > 0 ||
      window.document.querySelector(
        'div[data-testid="filters-menu-tooltip"]',
      ) !== null)
  ) {
    return state;
  }
  state.modalStack = [...state.modalStack, modal];
}
```

#### 搜索框自动聚焦
```typescript
// KeyboardShortcutModal.tsx:490-493
<InitialFocus>
  <Search
    value={searchText}
    // ...
  />
</InitialFocus>
```

---

### 3. 全局焦点与键盘事件层级

```
键盘事件优先级 (从高到低):

┌─────────────────────────────────────────┐
│ 1. 活跃模态框 (Modal Dialog)            │  Highest priority - traps focus
│    - Command Bar Dialog                 │
│    - Keyboard Shortcuts Modal           │
│    - Other modals (Settings, etc.)      │
├─────────────────────────────────────────┤
│ 2. 输入框/表单元素                      │  Form inputs capture keys
│    - Transaction editing                │
│    - Budget amount editing              │
├─────────────────────────────────────────┤
│ 3. 页面级快捷键                         │  Context-aware shortcuts
│    - Account page shortcuts             │
│    - Budget page shortcuts              │
├─────────────────────────────────────────┤
│ 4. 全局快捷键                           │  Lowest priority
│    - Ctrl/Cmd + K (Command Bar)         │
│    - ? (Help menu)                      │
│    - Ctrl/Cmd + Z (Undo)                │
└─────────────────────────────────────────┘
```

---

## 页面跳转路由逻辑

### 1. 命令面板导航流程

```
用户输入搜索词
    ↓
[CommandBar.tsx]
    │
    ├─→ 实时过滤匹配项 (name.toLowerCase().includes(searchLower))
    │
    ├─→ 用户选择项 (点击或 Enter)
    │
    └─→ 触发 onSelect 回调
            │
            ├─→ Navigation Items → handleNavigate(item.path)
            │       例如: /budget, /reports, /accounts
            │
            ├─→ Accounts → handleNavigate(`/accounts/${id}`)
            │
            ├─→ Dashboard Pages → handleNavigate(`/reports/${id}`)
            │
            └─→ Custom Reports → handleNavigate(`/reports/custom/${id}`)
                    ↓
                    [useNavigate hook]
                        ↓
                        ├─→ 检查是否与当前路径相同
                        ├─→ 相同则 navigate(-1) (返回)
                        ├─→ 不同则 navigate(to, { state: { previousLocation } })
                        └─→ React Router 处理路由变化
                                ↓
                                [FinancesApp.tsx]
                                    ↓
                                    <Routes> 匹配对应组件
```

### 2. useNavigate Hook 增强实现

```typescript
// useNavigate.ts:1-54
export function useNavigate(): NavigateFunction {
  const location = useLocation();
  const navigate = useNavigateReactRouter();
  
  return useCallback(
    (to: To | number, options: NavigateOptions = {}) => {
      if (typeof to === 'number') {
        void navigate(to);
      } else {
        // 关键1: 记录前一个位置到 state 中
        const optionsWithPrevLocation: NavigateOptions = {
          replace: options.replace || isSamePath(to, location) ? true : undefined,
          ...options,
          state: {
            ...options?.state,
            previousLocation: location,
          },
        };

        const { previousLocation, ...previousOriginalState } = location.state || {};

        // 关键2: 如果目标与前一个位置相同，智能返回
        if (
          previousLocation == null ||
          !isSamePath(to, previousLocation) ||
          JSON.stringify(options?.state || {}) !==
            JSON.stringify(previousOriginalState)
        ) {
          void navigate(to, optionsWithPrevLocation);
        } else {
          // 智能返回: 与前一位置相同则直接 go back
          void navigate(-1);
        }
      }
    },
    [navigate, location],
  );
}
```

### 3. 路由配置结构

```typescript
// FinancesApp.tsx:243-390
<Routes>
  <Route path="/" element={<Navigate to="/budget" />} />
  <Route path="/reports/*" element={<Reports />} />
  <Route path="/budget" element={<Budget />} />
  <Route path="/schedules" element={<Schedules />} />
  <Route path="/payees" element={<Payees />} />
  <Route path="/rules" element={<Rules />} />
  <Route path="/accounts" element={<Accounts />} />
  <Route path="/accounts/:id" element={<Account />} />
  <Route path="/settings" element={<Settings />} />
  <Route path="/tags" element={<ManageTagsPage />} />
  {/* ... 更多路由 */}
</Routes>
```

---

## 测试覆盖分析

### 1. 命令面板测试 (command-bar.test.ts)

#### 测试用例 A: 视觉验证
- **测试名称**: `Check the command bar visuals`
- **测试内容**:
  1. 通过 `ControlOrMeta+k` 打开命令面板
  2. 验证命令面板可见性
  3. 截图对比 (主题一致性)
  4. 通过 `Escape` 关闭
  5. 验证关闭后不可见

#### 测试用例 B: 搜索与导航功能
- **测试名称**: `Check the command bar search works correctly`
- **测试内容**:
  1. 打开命令面板
  2. 输入 "reports" 搜索
  3. 按 Enter 导航
  4. 验证 Reports 页面加载
  5. 再次打开命令面板
  6. 使用箭头键选择第二个选项 (Schedules)
  7. 按 Enter 导航
  8. 验证 Schedules 页面可见

---

### 2. 帮助菜单与快捷键模态框测试 (help-menu.test.ts)

#### 测试用例 A: 帮助菜单视觉验证
- **测试名称**: `Check the help menu visuals`
- **测试内容**:
  1. 点击 Help 按钮
  2. 验证弹出菜单可见
  3. 验证 "Keyboard shortcuts" 选项可见
  4. 截图对比
  5. 通过 Escape 关闭

#### 测试用例 B: 快捷键模态框验证
- **测试名称**: `Check the keyboard shortcuts modal visuals`
- **测试内容**:
  1. 打开 Help 菜单并点击 Keyboard shortcuts
  2. 验证模态框可见
  3. 验证搜索框存在且为空
  4. 搜索 "command"
  5. 验证 "Open the Command Palette" 出现在结果中
  6. 点击 Back 按钮返回分类列表
  7. 点击 Global 分类
  8. 验证 "Open the help menu" 快捷键可见

---

### 3. 测试覆盖范围总结

| 功能模块 | 测试覆盖 | 未覆盖部分 |
|---------|---------|-----------|
| 命令面板打开/关闭 | ✅ 完整覆盖 | - |
| 命令面板搜索过滤 | ✅ 基本覆盖 | 边界情况(特殊字符、空结果) |
| 命令面板导航跳转 | ✅ 基本覆盖 | 所有导航项的完整遍历 |
| 键盘导航 (箭头键) | ✅ 基本覆盖 | Vim 按键绑定 |
| 帮助菜单触发 | ✅ 完整覆盖 | - |
| 快捷键模态框搜索 | ✅ 基本覆盖 | 所有快捷键搜索 |
| 快捷键分类切换 | ✅ 基本覆盖 | 所有分类切换 |
| 快捷键模态框打开限制 | ❌ 未覆盖 | 模态框已打开时的行为 |
| 命令面板打开限制 | ❌ 未覆盖 | 模态框已打开时的行为 |
| 智能返回导航 | ❌ 未覆盖 | useNavigate 的智能返回逻辑 |
| 账户余额显示 | ❌ 未覆盖 | Command Bar 中账户余额渲染 |

---

## 核心文件索引

| 文件路径 | 主要职责 | 关键行数 |
|---------|---------|---------|
| `packages/desktop-client/src/components/CommandBar.tsx` | 命令面板主组件 | 1-375 |
| `packages/desktop-client/src/components/modals/KeyboardShortcutModal.tsx` | 快捷键模态框 | 1-606 |
| `packages/desktop-client/src/components/HelpMenu.tsx` | 帮助菜单与快捷键触发 | 1-138 |
| `packages/desktop-client/src/modals/modalsSlice.ts` | 模态框状态管理与 Guard | 717-730 |
| `packages/desktop-client/src/hooks/useNavigate.ts` | 增强型导航 Hook | 1-54 |
| `packages/desktop-client/src/components/FinancesApp.tsx` | 主应用路由与组件挂载 | 199, 243-390 |
| `packages/desktop-client/src/components/App.tsx` | 应用根组件与 Provider | 198-240 |
| `packages/desktop-client/src/hooks/useModalState.ts` | 模态框状态访问 Hook | 1-43 |
| `packages/desktop-client/e2e/command-bar.test.ts` | 命令面板 E2E 测试 | 1-79 |
| `packages/desktop-client/e2e/help-menu.test.ts` | 帮助菜单 E2E 测试 | 1-67 |

---

## 架构图总结

```
┌─────────────────────────────────────────────────────────────────────┐
│                           用户输入层                                  │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐          │
│  │  Ctrl/Cmd+K  │    │      ?       │    │  Click Help  │          │
│  └──────┬───────┘    └──────┬───────┘    └──────┬───────┘          │
└─────────┼───────────────────┼─────────────────────┼──────────────────┘
          │                   │                     │
┌─────────▼───────────────────▼─────────────────────▼──────────────────┐
│                           触发守卫层                                  │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                    Modal Stack Guard                         │    │
│  │  IF modalStack.length > 0 → 阻止打开命令面板/快捷键模态框     │    │
│  └─────────────────────────────────────────────────────────────┘    │
└─────────┬────────────────────────────────────────────────────────────┘
          │
┌─────────▼────────────────────────────────────────────────────────────┐
│                           状态管理层                                  │
│  ┌──────────────────┐    ┌──────────────────────────┐               │
│  │  setOpen(true)   │    │  pushModal({ modal })    │               │
│  │  (CommandBar)    │    │  (Redux Action)          │               │
│  └────────┬─────────┘    └────────────┬─────────────┘               │
└───────────┼────────────────────────────┼─────────────────────────────┘
            │                            │
┌───────────▼────────────────────────────▼─────────────────────────────┐
│                           UI 渲染层                                   │
│  ┌──────────────────┐    ┌──────────────────────────┐               │
│  │  Command Bar     │    │ Keyboard Shortcuts Modal │               │
│  │  - Search Input  │    │  - Categories List       │               │
│  │  - Filtered List │    │  - Search Filter         │               │
│  │  - Auto Focus    │    │  - InitialFocus          │               │
│  └────────┬─────────┘    └────────────┬─────────────┘               │
└───────────┼────────────────────────────┼─────────────────────────────┘
            │                            │
┌───────────▼────────────────────────────▼─────────────────────────────┐
│                         导航动作层                                    │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                    useNavigate() Hook                        │    │
│  │  - Same path check → navigate(-1)                           │    │
│  │  - Different path → navigate(to, { state: { previous } })   │    │
│  └────────────────────────────┬────────────────────────────────┘    │
└───────────────────────────────┼──────────────────────────────────────┘
                                │
┌───────────────────────────────▼──────────────────────────────────────┐
│                           React Router                                │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  <Routes> → 匹配 path → 渲染对应页面组件                    │    │
│  └─────────────────────────────────────────────────────────────┘    │
└───────────────────────────────────────────────────────────────────────┘
```

---

*文档生成时间: 2026-05-13*
*基于 Actual Budget 代码库分析*
