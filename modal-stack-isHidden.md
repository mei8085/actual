# isHidden 机制与 Hotkeys Scope 行为核实报告

> 本文档深入分析两个具体问题：
> 1. `isHidden` 字段从加载状态到弹窗 opacity 的完整串联路径
> 2. 核实 `react-hotkeys-hook` v5.x 中 `enableScope` / `disableScope` 的实际行为

---

## 一、isHidden 字段完整串联机制

### 1.1 触发源：全局事件 → setAppState

**起点**在 [global-events.ts#L142-L165](file:///d:/fz/0601-1/solo-dogfeeding/code/74-actual/packages/desktop-client/src/global-events.ts#L142-L165)，通过监听后端发来的 IPC 事件来控制加载状态：

```typescript
const unlistenStartLoad = listen('start-load', () => {
  void store.dispatch(closeBudgetUI());
  store.dispatch(setAppState({ loadingText: '' }));   // ← 开始加载
});

const unlistenFinishLoad = listen('finish-load', () => {
  store.dispatch(closeModal());
  store.dispatch(setAppState({ loadingText: null })); // ← 加载完成
  void store.dispatch(loadPrefs());
});

const unlistenStartImport = listen('start-import', () => {
  void store.dispatch(closeBudgetUI());
});

const unlistenFinishImport = listen('finish-import', () => {
  store.dispatch(closeModal());
  store.dispatch(setAppState({ loadingText: null })); // ← 导入完成
  void store.dispatch(loadPrefs());
});

const unlistenShowBudgets = listen('show-budgets', () => {
  void store.dispatch(closeBudgetUI());
  store.dispatch(setAppState({ loadingText: null })); // ← 显示预算列表
});
```

**关键观察**：
- `start-load`：`loadingText = ''`（空字符串，不是 null）
- `finish-load` / `finish-import` / `show-budgets`：`loadingText = null`
- 这些是**后端驱动的全局事件**，不是前端组件主动 dispatch 的

### 1.2 中转站：appSlice → modalsSlice extraReducers

`setAppState` 定义在 `appSlice` 中，但 `modalsSlice` 通过 `extraReducers` 监听它（[modalsSlice.ts#L749-L755](file:///d:/fz/0601-1/solo-dogfeeding/code/74-actual/packages/desktop-client/src/modals/modalsSlice.ts#L749-L755)）：

```typescript
extraReducers: builder => {
  builder.addCase(setAppState, (state, action) => {
    state.isHidden = action.payload.loadingText !== null;
    //                              ↑
    // 关键：'' !== null → true    (isHidden = true，弹窗隐藏)
    //      null !== null → false   (isHidden = false，弹窗显示)
  });
  builder.addCase(signOut.fulfilled, () => initialState);
  builder.addCase(resetApp, () => initialState);
},
```

**状态类型定义**（[modalsSlice.ts#L691-L699](file:///d:/fz/0601-1/solo-dogfeeding/code/74-actual/packages/desktop-client/src/modals/modalsSlice.ts#L691-L699)）：
```typescript
type ModalsState = {
  modalStack: Modal[];  // ← 弹窗栈（不受 isHidden 影响）
  isHidden: boolean;    // ← 弹窗是否隐藏（纯视觉标记）
};

const initialState: ModalsState = {
  modalStack: [],
  isHidden: false,
};
```

**核心洞察**：`isHidden` 和 `modalStack` 是**完全独立**的两个字段。`isHidden = true` **不会**改变 `modalStack` 的内容。

### 1.3 消费端：useModalState → Modal 组件 → opacity

**第一步：Hook 透传**（[useModalState.ts#L15-L42](file:///d:/fz/0601-1/solo-dogfeeding/code/74-actual/packages/desktop-client/src/hooks/useModalState.ts#L15-L42)）：

```typescript
export function useModalState(): ModalState {
  const modalStack = useSelector(state => state.modals.modalStack);
  const isHidden = useSelector(state => state.modals.isHidden);  // ← 读取
  const dispatch = useDispatch();
  // ...
  return {
    onClose: popModalCallback,
    modalStack,
    activeModal: lastModal?.name,
    isActive,
    isHidden,  // ← 直接透出
  };
}
```

**第二步：Modal 组件设置 opacity**（[Modal.tsx#L68-L68](file:///d:/fz/0601-1/solo-dogfeeding/code/74-actual/packages/desktop-client/src/components/common/Modal.tsx#L68-L68) 和 [L134](file:///d:/fz/0601-1/solo-dogfeeding/code/74-actual/packages/desktop-client/src/components/common/Modal.tsx#L134)）：

```typescript
const { isHidden, isActive, onClose: closeModal } = useModalState();
// ...
<ModalContentContainer
  style={{
    flex: 1,
    padding: 10,
    // ...
    opacity: isHidden ? 0 : 1,  // ← 关键：isHidden=true 时完全透明
    // ...
  }}
>
```

### 1.4 串联路径全景图

```
后端 IPC 事件
    │
    ├─ start-load     → setAppState({ loadingText: '' })
    ├─ finish-load    → setAppState({ loadingText: null })
    ├─ finish-import  → setAppState({ loadingText: null })
    └─ show-budgets   → setAppState({ loadingText: null })
             │
             ▼
┌──────────────────────────────────────────────┐
│  modalsSlice.extraReducers                   │
│  state.isHidden = (loadingText !== null)     │
│                                              │
│  注意：modalStack 不变！                      │
└──────────────────────────┬───────────────────┘
                           │
                           ▼
              useModalState() Hook
              透传 isHidden 字段
                           │
                           ▼
┌──────────────────────────────────────────────┐
│  <Modal> 组件 × N（每层弹窗各一个）           │
│  每个都读取同一个全局 isHidden 值             │
│                                              │
│  ModalContentContainer:                      │
│    opacity: isHidden ? 0 : 1                 │
└──────────────────────────────────────────────┘
```

### 1.5 关键设计：为什么用 opacity 而不是卸载？

**选择 opacity = 0 的效果**（对比卸载组件）：

| 维度 | opacity: 0（当前实现） | 卸载组件 / display: none |
|------|-----------------------|-------------------------|
| **DOM 存在性** | ✅ DOM 完全保留 | ❌ DOM 从渲染树移除 |
| **React Aria ModalOverlay 实例** | ✅ 保留 | ❌ 销毁 |
| **焦点陷阱** | ✅ 仍然有效 | ❌ 消失 |
| **触发元素引用** | ✅ 保留在组件实例中 | ❌ 丢失 |
| **Hotkeys Scope 状态** | ✅ 保持 enableScope | ❌ 触发 useEffect cleanup 调用 disableScope |
| **modalStack 状态** | ✅ 不变 | ❌ 需要同步清空 |
| **动画过渡** | ✅ 可配合 transition | ❌ 做不到平滑过渡 |
| **用户视觉感知** | ✅ 弹窗完全不可见 | ✅ 同样不可见 |

**实际应用场景**：
1. 用户正在某个弹窗中操作，触发了后端加载（如文件导入、数据同步）
2. `start-load` 事件触发 → 所有弹窗 opacity 归零
3. 用户此时看不到弹窗，但**弹窗状态完整保留**（包括输入的内容、焦点位置）
4. 加载完成 → `finish-load` 事件触发 → `closeModal()` 清空弹窗栈 + `setAppState({ loadingText: null })` 恢复 opacity

**注意**：`finish-load` 同时调用了 `closeModal()`（清空整个栈），所以实际上 opacity 恢复的同时弹窗也被关闭了。`isHidden` 机制更多是为了确保**加载过程中弹窗不会干扰加载 UI 的显示**，而不是为了加载后恢复弹窗。

### 1.6 加载中各层系统的状态

当 `isHidden = true` 时，四层系统的状态如下：

| 系统层 | 状态变化 |
|--------|---------|
| **Redux modalStack** | 不变，弹窗数据完整保留 |
| **Hotkeys Scope** | 不变，各弹窗的 scope 仍然活跃 |
| **React Aria 焦点陷阱** | 不变，焦点陷阱仍在 DOM 中工作 |
| **React 组件实例** | 不变，所有 state/ref 保留 |
| **DOM 元素** | 存在，但 `opacity: 0`，视觉上不可见 |
| **键盘事件** | 仍由焦点陷阱捕获，Tab 键循环仍在弹窗内 |
| **鼠标事件** | opacity: 0 的元素仍接收事件（与 pointer-events: none 不同） |

---

## 二、react-hotkeys-hook enableScope 行为核实

### 2.1 核实依据

使用的版本：`react-hotkeys-hook: ^5.2.4`（[desktop-client/package.json#L188](file:///d:/fz/0601-1/solo-dogfeeding/code/74-actual/packages/desktop-client/package.json#L188)）

参考官方文档 v5.x（https://react-hotkeys-hook.vercel.app/ 及 npm README）：

### 2.2 官方文档明确的 Context API

来自 HotkeysProvider 官方文档：

| API 方法 | 签名 | 官方说明 |
|---------|------|---------|
| **activeScopes** | `string[]` | 当前活跃的 scope 名称**数组**。热键只有当它属于活跃 scope 之一或 wildcard scope `'*'` 时才会触发。 |
| **enableScope** | `(scope: string) => void` | **启用**指定 scope，将其**添加**到活跃 scopes 列表。如果 wildcard scope `'*'` 当前活跃，则会**被替换**为指定 scope。 |
| **disableScope** | `(scope: string) => void` | **禁用**指定 scope，将其**从活跃列表移除**。该 scope 内的热键将不再触发。 |
| **toggleScope** | `(scope: string) => void` | 切换指定 scope 的开关状态 |

### 2.3 之前分析的说法 vs 实际行为

**之前分析中的说法**（需要更正）：
> `enableScope(name)` 会**自动禁用其他所有 scope**（包括 `'app'`）

**实际行为**（根据官方文档）：
> `enableScope(name)` 会将 name **追加**到活跃 scopes 数组中，**不会**自动移除其他已存在的 scope。仅存在一个例外：当 wildcard scope `'*'` 在活跃列表中时，调用 `enableScope` 会将 `'*'` **替换**为指定 scope。

### 2.4 Actual Budget 初始化场景推演

Actual Budget 初始化代码（[App.tsx#L200](file:///d:/fz/0601-1/solo-dogfeeding/code/74-actual/packages/desktop-client/src/components/App.tsx#L200)）：
```tsx
<HotkeysProvider initiallyActiveScopes={['app']}>
```

注意：初始值是 `['app']`，不是默认的 `['*']`。

**推演场景 A：打开第一个弹窗**

```
初始状态:
  activeScopes = ['app']

调用 enableScope('confirm-delete'):
  → 规则：追加（因为 '*' 不在活跃列表中，不触发替换）
  → activeScopes = ['app', 'confirm-delete']
          ↑ 'app' 仍然在列表中！

此时热键触发规则：
  - scopes: ['app'] 的快捷键          → 触发（'app' 活跃）
  - scopes: ['confirm-delete'] 的快捷键 → 触发（'confirm-delete' 活跃）
  - scopes: ['*'] 的快捷键             → 始终触发（wildcard 特殊规则）
```

**推演场景 B：打开第二个弹窗（叠加）**

```
当前状态:
  activeScopes = ['app', 'confirm-delete']

调用 enableScope('edit-rule'):
  → 规则：追加
  → activeScopes = ['app', 'confirm-delete', 'edit-rule']

此时热键触发规则：
  - scopes: ['app'] 的快捷键          → 仍然触发！
  - scopes: ['confirm-delete'] 的快捷键 → 仍然触发！
  - scopes: ['edit-rule'] 的快捷键      → 触发
```

**推演场景 C：关闭顶层弹窗**

```
当前状态:
  activeScopes = ['app', 'confirm-delete', 'edit-rule']

edit-rule 弹窗卸载 → cleanup 调用 disableScope('edit-rule'):
  → 规则：从列表移除
  → activeScopes = ['app', 'confirm-delete']
```

### 2.5 与代码注释的矛盾

在 [Modal.tsx#L62-L65](file:///d:/fz/0601-1/solo-dogfeeding/code/74-actual/packages/desktop-client/src/components/common/Modal.tsx#L62-L65) 中：

```typescript
// This deactivates any key handlers in the "app" scope
useEffect(() => {
  enableScope(name);
  return () => disableScope(name);
}, [enableScope, disableScope, name]);
```

**注释写的是**："这会禁用 'app' scope 中的所有键处理器"

**但实际行为是**（根据文档）：`'app'` scope 仍然在活跃列表中，不会被禁用。

**矛盾分析**：

| 可能性 | 说明 |
|--------|------|
| **1. 库版本变更** | v4 版本的行为可能不同，升级到 v5 后注释未更新 |
| **2. 作者理解偏差** | 代码作者可能基于错误的理解写了注释，但实际未验证 |
| **3. 隐含的互斥逻辑** | 可能所有弹窗热键都没有设置 scope（默认 `'*'`），页面热键都设置了 `scopes: ['app']`，而其他机制（焦点陷阱）阻止了页面热键实际生效 |
| **4. '*' 的特殊位置** | 如果某些路径下活跃列表包含 '*'，则 enableScope 会替换它 |

**最大可能的解释**：
在实际运行中，虽然 `'app'` scope 从库的角度仍然活跃，但**焦点陷阱**阻止了用户与页面元素的交互，且 React Aria ModalOverlay 的 `isDismissable` 配置会拦截 ESC 等键。因此即使 app scope 的热键在理论上可以触发，实际上也不会造成问题——因为用户的焦点被限制在弹窗内。

### 2.6 修正后的 Hotkeys Scope 状态链

修正之前的分析（假设使用 v5.x 文档描述的行为）：

```
┌─────────────────────────────────────────────────────┐
│  初始状态                                            │
│  activeScopes = ['app']                             │
│  页面快捷键（scopes: ['app']）→ ✅ 可触发           │
└─────────────────────────────────────────────────────┘
                           ↓ pushModal('confirm-delete')
                           ↓ enableScope('confirm-delete')
┌─────────────────────────────────────────────────────┐
│  弹窗 A 挂载                                         │
│  activeScopes = ['app', 'confirm-delete']           │
│  页面快捷键（scopes: ['app']）→ ⚠️ 理论上可触发     │
│                                                    │
│  但实际受焦点陷阱限制：                               │
│  - 焦点在弹窗内，页面元素不会获得键盘事件              │
│  - ESC 等键被 ModalOverlay 的 isDismissable 拦截    │
└─────────────────────────────────────────────────────┘
                           ↓ pushModal('edit-rule')
                           ↓ enableScope('edit-rule')
┌─────────────────────────────────────────────────────┐
│  弹窗 B 挂载                                         │
│  activeScopes = ['app', 'confirm-delete', 'edit-rule']
│  三层 scope 都在活跃列表中                           │
│                                                    │
│  实际生效的只有顶层弹窗的焦点范围内的热键             │
└─────────────────────────────────────────────────────┘
                           ↓ popModal()
                           ↓ disableScope('edit-rule')
┌─────────────────────────────────────────────────────┐
│  弹窗 B 卸载                                         │
│  activeScopes = ['app', 'confirm-delete']           │
└─────────────────────────────────────────────────────┘
                           ↓ popModal()
                           ↓ disableScope('confirm-delete')
┌─────────────────────────────────────────────────────┐
│  弹窗 A 卸载                                         │
│  activeScopes = ['app']                             │
│  恢复到初始状态                                      │
└─────────────────────────────────────────────────────┘
```

### 2.7 关键结论

1. **enableScope 不自动禁用其他 scope**（除非 '*' 活跃时被替换）——这一点需要在之前的分析中更正
2. **代码注释的预期行为与库文档描述的实际行为存在差异**
3. **实际效果的保障依赖多层机制**：焦点陷阱 + DOM 遮挡 + 键盘事件拦截，而非单纯依赖 hotkeys scope 的互斥
4. **disableScope 确实会从活跃列表移除**，因此弹窗卸载后其 scope 不再活跃，这一点之前的分析是正确的

---

## 三、完整系统交互：isHidden + Hotkeys + 焦点陷阱

### 3.1 加载状态下的完整状态

当 `start-load` 事件触发（`isHidden = true`）时，系统各层的状态：

```
Redux 层
├── modals.modalStack: [ModalA, ModalB]  ← 不变
└── modals.isHidden: true                 ← 切换为 true
          │
          ▼
Hotkeys 层
├── activeScopes: ['app', 'A', 'B']      ← 不变
└── 各 scope 的启用状态: 全部保留
          │
          ▼
React 组件层
├── ModalA 实例: 保留，state/ref 不变
└── ModalB 实例: 保留，state/ref 不变
          │
          ▼
React Aria 层
├── ModalA ModalOverlay: 未卸载，焦点陷阱保留
├── ModalB ModalOverlay: 未卸载，焦点陷阱保留
└── 触发元素引用: 两个都保留在组件实例中
          │
          ▼
DOM / 视觉层
├── ModalA: opacity: 0（完全透明）
├── ModalB: opacity: 0（完全透明）
└── pointer-events: auto（仍接收事件，但用户看不到）
```

### 3.2 加载完成后的清理

`finish-load` 事件触发：

```typescript
store.dispatch(closeModal());                     // 1. modalStack = []
store.dispatch(setAppState({ loadingText: null })); // 2. isHidden = false
```

**执行顺序的细微差别**：
1. `closeModal()` → 所有 `<Modal>` 组件因 `modalStack` 为空而卸载
2. 卸载触发各 Modal 的 `useEffect` cleanup → `disableScope(name)`
3. `setAppState({ loadingText: null })` → isHidden 切换为 false（但此时已经没有 Modal 组件在读取它了）

所以 `isHidden` 恢复的时机在实际中不会被观察到——因为它恢复之前弹窗已经被清空了。

---

## 四、关键代码索引

| 主题 | 代码位置 |
|------|---------|
| start-load / finish-load 事件监听 | [global-events.ts#L142-L165](file:///d:/fz/0601-1/solo-dogfeeding/code/74-actual/packages/desktop-client/src/global-events.ts#L142-L165) |
| setAppState → isHidden 关联 | [modalsSlice.ts#L749-L755](file:///d:/fz/0601-1/solo-dogfeeding/code/74-actual/packages/desktop-client/src/modals/modalsSlice.ts#L749-L755) |
| ModalsState 类型定义 | [modalsSlice.ts#L691-L699](file:///d:/fz/0601-1/solo-dogfeeding/code/74-actual/packages/desktop-client/src/modals/modalsSlice.ts#L691-L699) |
| useModalState 透传 isHidden | [useModalState.ts#L15-L42](file:///d:/fz/0601-1/solo-dogfeeding/code/74-actual/packages/desktop-client/src/hooks/useModalState.ts#L15-L42) |
| opacity: isHidden ? 0 : 1 | [Modal.tsx#L134](file:///d:/fz/0601-1/solo-dogfeeding/code/74-actual/packages/desktop-client/src/components/common/Modal.tsx#L134) |
| enableScope 调用 + 注释 | [Modal.tsx#L60-L66](file:///d:/fz/0601-1/solo-dogfeeding/code/74-actual/packages/desktop-client/src/components/common/Modal.tsx#L60-L66) |
| HotkeysProvider 初始化 | [App.tsx#L200](file:///d:/fz/0601-1/solo-dogfeeding/code/74-actual/packages/desktop-client/src/components/App.tsx#L200) |
| 页面级快捷键 scopes: ['app'] | [SelectedTransactionsButton.tsx#L227-L229](file:///d:/fz/0601-1/solo-dogfeeding/code/74-actual/packages/desktop-client/src/components/transactions/SelectedTransactionsButton.tsx#L227-L229) |
| react-hotkeys-hook 版本号 | [desktop-client/package.json#L188](file:///d:/fz/0601-1/solo-dogfeeding/code/74-actual/packages/desktop-client/package.json#L188) |
