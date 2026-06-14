# 弹窗叠加深层机制分析报告

> 本文档深入分析 Actual Budget 弹窗叠加系统的四个核心深层机制：
> 1. react-hotkeys-hook 作用域管理
> 2. 弹窗栈与 React Aria 焦点陷阱的关联
> 3. noAnimation 路径对 display 的特殊处理
> 4. 多个 ReactAriaModalOverlay 叠加时的焦点陷阱优先级

---

## 一、react-hotkeys-hook 作用域管理

### 1.1 整体架构

**HotkeysProvider 初始化**（[App.tsx#L200](file:///d:/fz/0601-1/solo-dogfeeding/code/74-actual/packages/desktop-client/src/components/App.tsx#L200)）：
```tsx
<HotkeysProvider initiallyActiveScopes={['app']}>
  {/* 整个应用 */}
</HotkeysProvider>
```

**核心机制**：
- 应用启动时，默认激活 `'app'` 作用域
- 所有页面级快捷键都绑定到 `'app'` scope
- 弹窗打开时，激活弹窗自己的 scope，与 `'app'` scope 互斥

### 1.2 弹窗作用域的激活与禁用

在 [Modal.tsx#L60-L66](file:///d:/fz/0601-1/solo-dogfeeding/code/74-actual/packages/desktop-client/src/components/common/Modal.tsx#L60-L66) 中：

```typescript
const { enableScope, disableScope } = useHotkeysContext();

// This deactivates any key handlers in the "app" scope
useEffect(() => {
  enableScope(name);           // 激活当前弹窗的 scope
  return () => disableScope(name);  // 卸载时禁用
}, [enableScope, disableScope, name]);
```

**关键行为**：
- `enableScope(name)` 会**自动禁用其他所有 scope**（包括 `'app'`）
- 每个弹窗使用自己的 `name` 作为 scope 名称
- 弹窗卸载时调用 `disableScope(name)`，此时 `'app'` scope 会被重新激活

### 1.3 多层弹窗叠加时的作用域链

假设场景：页面 → 弹窗 A（`name='confirm-delete'`）→ 弹窗 B（`name='edit-rule'`）

```
┌─────────────────────────────────────────────────────┐
│  初始状态：activeScopes = ['app']                   │
│  页面快捷键（ctrl+a、f、u 等）全部可用                │
└─────────────────────────────────────────────────────┘
                           ↓ pushModal('confirm-delete')
┌─────────────────────────────────────────────────────┐
│  弹窗 A 挂载：                                       │
│  enableScope('confirm-delete')                      │
│  → activeScopes = ['confirm-delete']                │
│  → 'app' scope 被禁用，页面快捷键失效                 │
│  → 只有 scope='confirm-delete' 的快捷键可用           │
└─────────────────────────────────────────────────────┘
                           ↓ pushModal('edit-rule')
┌─────────────────────────────────────────────────────┐
│  弹窗 B 挂载：                                       │
│  enableScope('edit-rule')                           │
│  → activeScopes = ['edit-rule']                     │
│  → 'confirm-delete' scope 也被禁用                   │
│  → 只有 scope='edit-rule' 的快捷键可用               │
└─────────────────────────────────────────────────────┘
                           ↓ popModal() 关闭弹窗 B
┌─────────────────────────────────────────────────────┐
│  弹窗 B 卸载：                                       │
│  disableScope('edit-rule')                          │
│  → activeScopes 恢复为 ['confirm-delete']           │
│  → 弹窗 A 的快捷键恢复可用                           │
└─────────────────────────────────────────────────────┘
                           ↓ popModal() 关闭弹窗 A
┌─────────────────────────────────────────────────────┐
│  弹窗 A 卸载：                                       │
│  disableScope('confirm-delete')                     │
│  → activeScopes 恢复为 ['app']                      │
│  → 页面快捷键恢复可用                                │
└─────────────────────────────────────────────────────┘
```

### 1.4 页面级快捷键的 scope 绑定

在 [SelectedTransactionsButton.tsx#L227-L275](file:///d:/fz/0601-1/solo-dogfeeding/code/74-actual/packages/desktop-client/src/components/transactions/SelectedTransactionsButton.tsx#L227-L275) 中：
```typescript
const hotKeyOptions = {
  preventDefault: true,
  enableOnFormTags: ['INPUT', 'SELECT', 'TEXTAREA'],
  scopes: ['app'],  // ← 绑定到 'app' scope
};

useHotkeys('f', () => onShow(selectedIds), hotKeyOptions, [...]);
useHotkeys('u', () => onDuplicate(selectedIds), hotKeyOptions, [...]);
useHotkeys('d', () => onDelete(selectedIds), hotKeyOptions, [...]);
// ... 更多快捷键
```

**设计目的**：
- 弹窗打开时，页面级快捷键（如 `f` 表示查找、`d` 表示删除）不会意外触发
- 每个弹窗可以定义自己的快捷键，不会与页面或其他弹窗冲突
- 完全由 `react-hotkeys-hook` 库管理，无需手动维护

---

## 二、弹窗栈与 React Aria 焦点陷阱的关联

### 2.1 React Aria Modal 的焦点陷阱机制

`react-aria-components` 的 `ModalOverlay` 组件内置了完整的焦点陷阱（Focus Trap）实现，符合 WAI-ARIA 模态对话框规范。

**ModalOverlay 的三层结构**（[Modal.tsx#L77-L175](file:///d:/fz/0601-1/solo-dogfeeding/code/74-actual/packages/desktop-client/src/components/common/Modal.tsx#L77-L175)）：
```tsx
<ReactAriaModalOverlay      // ← 第1层：遮罩层 + 焦点陷阱
  isDismissable
  defaultOpen
  onOpenChange={...}
  style={{ position: 'fixed', inset: 0, zIndex: 3000 }}
>
  <View>                    // ← 第2层：居中容器
    <ReactAriaModal>        // ← 第3层：内容容器 + 焦点管理
      {modalProps => (
        <Dialog aria-label="Modal dialog">
          <ModalContentContainer>
            {/* 弹窗内容 */}
          </ModalContentContainer>
        </Dialog>
      )}
    </ReactAriaModal>
  </View>
</ReactAriaModalOverlay>
```

**ModalOverlay 自动处理的焦点逻辑**：
1. **打开时**：
   - 保存当前 `document.activeElement`（触发元素）
   - 将焦点移动到弹窗内的第一个可聚焦元素（或 `InitialFocus` 指定的元素）
   - 安装焦点陷阱，Tab 键在弹窗内循环

2. **关闭时**：
   - 自动将焦点恢复到之前保存的触发元素
   - 卸载焦点陷阱

3. **按 ESC 时**：
   - 触发 `onOpenChange(false)`，关闭弹窗

### 2.2 弹窗栈与焦点陷阱的一一对应关系

**渲染流程**（[Modals.tsx#L90-L180](file:///d:/fz/0601-1/solo-dogfeeding/code/74-actual/packages/desktop-client/src/components/Modals.tsx#L90-L180)）：
```typescript
const modals = modalStack
  .map((modal, idx) => {
    const { name } = modal;
    const key = `${name}-${idx}`;
    switch (name) {
      case 'confirm-delete':
        return <ConfirmDeleteModal key={key} {...modal.options} />;
      case 'edit-rule':
        return <EditRuleModal key={key} {...modal.options} />;
      // ... 约 70 种弹窗类型
    }
  });

return <>{modals}</>;
```

**对应关系**：
- `modalStack` 数组中的每个元素 → 一个 `<Modal>` 组件
- 每个 `<Modal>` 组件 → 一个 `<ReactAriaModalOverlay>` → 一个独立的焦点陷阱
- 因此，**弹窗栈的长度 = 焦点陷阱的数量**

### 2.3 isActive 状态与可访问性

在 [useModalState.ts#L25-L34](file:///d:/fz/0601-1/solo-dogfeeding/code/74-actual/packages/desktop-client/src/hooks/useModalState.ts#L25-L34) 中：
```typescript
const lastModal = modalStack[modalStack.length - 1];
const isActive = useCallback((name: string) => {
  return !lastModal || name === lastModal.name;
}, [lastModal]);
```

**isActive 的作用**：
- 只有栈顶弹窗的 `isActive = true`
- 下层弹窗的 `isActive = false`

在 [ModalContentContainer#L207-L214](file:///d:/fz/0601-1/solo-dogfeeding/code/74-actual/packages/desktop-client/src/components/common/Modal.tsx#L207-L214) 中：
```typescript
if (isActive) {
  contentRef.current.style.transform = 'none';
  contentRef.current.style.pointerEvents = 'auto';  // 可交互
} else {
  contentRef.current.style.transform = 'translateY(-40px) scale(.95) rotate(...)';
  contentRef.current.style.pointerEvents = 'none';  // 不可交互
}
```

**关键设计**：
- 下层弹窗虽然 DOM 还在，但 `pointer-events: none` 阻止了鼠标交互
- 但这**不能阻止 Tab 键聚焦**到下层弹窗的元素
- 这就是为什么需要 `noAnimation` 路径的 `display: none` 处理（见第三节）

---

## 三、noAnimation 路径对 display 的特殊处理

### 3.1 noAnimation 的使用场景

`noAnimation` 主要用于**自动完成类弹窗**，在桌面端（非窄屏）禁用动画：

| 弹窗 | 代码位置 |
|------|---------|
| PayeeAutocompleteModal | [PayeeAutocompleteModal.tsx#L43](file:///d:/fz/0601-1/solo-dogfeeding/code/74-actual/packages/desktop-client/src/components/modals/PayeeAutocompleteModal.tsx#L43) |
| CategoryAutocompleteModal | [CategoryAutocompleteModal.tsx#L47](file:///d:/fz/0601-1/solo-dogfeeding/code/74-actual/packages/desktop-client/src/components/modals/CategoryAutocompleteModal.tsx#L47) |
| AccountAutocompleteModal | [AccountAutocompleteModal.tsx#L38](file:///d:/fz/0601-1/solo-dogfeeding/code/74-actual/packages/desktop-client/src/components/modals/AccountAutocompleteModal.tsx#L38) |
| EditFieldModal | [EditFieldModal.tsx#L263](file:///d:/fz/0601-1/solo-dogfeeding/code/74-actual/packages/desktop-client/src/components/modals/EditFieldModal.tsx#L263) |
| CategoryGroupAutocompleteModal | [CategoryGroupAutocompleteModal.tsx#L45](file:///d:/fz/0601-1/solo-dogfeeding/code/74-actual/packages/desktop-client/src/components/modals/CategoryGroupAutocompleteModal.tsx#L45) |

**使用方式**：
```tsx
<Modal
  name="payee-autocomplete"
  noAnimation={!isNarrowWidth}  // 桌面端禁用动画
  onClose={onClose}
>
```

### 3.2 两种路径的对比

**初始化逻辑**（[ModalContentContainer#L217-L244](file:///d:/fz/0601-1/solo-dogfeeding/code/74-actual/packages/desktop-client/src/components/common/Modal.tsx#L217-L244)）：

| 维度 | 有动画路径（`noAnimation=false`） | 无动画路径（`noAnimation=true`） |
|------|--------------------------------|--------------------------------|
| 初始 opacity | `opacity: '0'`（透明） | `opacity: '1'`（不透明） |
| 初始 transform | `translateY(10px) scale(1)`（下移） | `translateY(0px) scale(1)`（正常） |
| 后续变化 | setTimeout 后淡入上移 | 直接显示 |
| transition | 设置 transition 属性 | setTimeout 后才设置 transition |
| 非活动时 display | 不设置（保持 block） | **`display: 'none'`** |

**关键差异**（[ModalContentContainer#L250-L253](file:///d:/fz/0601-1/solo-dogfeeding/code/74-actual/packages/desktop-client/src/components/common/Modal.tsx#L250-L253)）：
```typescript
style={{
  ...style,
  ...(noAnimation && !isActive && { display: 'none' }),  // ← 关键！
}}
```

### 3.3 为什么需要 display: none？

**问题场景**：
1. 用户打开弹窗 A（confirm-delete），`noAnimation=false`
2. 用户在弹窗 A 中点击按钮，打开弹窗 B（payee-autocomplete），`noAnimation=true`
3. 此时 `modalStack = [confirm-delete, payee-autocomplete]`
4. 弹窗 A 的 `isActive = false`，弹窗 B 的 `isActive = true`

**如果没有 display: none**：
- 弹窗 A 的 DOM 仍然存在，只是 `pointer-events: none`
- 用户按 Tab 键时，焦点可能会跳到弹窗 A 的按钮上
- 这破坏了焦点陷阱，用户可以与非活动弹窗交互

**有了 display: none 后**：
- 非活动的 noAnimation 弹窗会被完全从渲染树中移除
- Tab 键无法聚焦到 `display: none` 的元素
- 焦点被严格限制在顶层弹窗内

### 3.4 行为对比表

| 情况 | 有动画弹窗（下层） | 无动画弹窗（下层） |
|------|-------------------|-------------------|
| DOM 存在性 | 存在 | **不存在**（display: none） |
| pointer-events | none | none |
| 可否被 Tab 聚焦 | **可以**（有风险） | **不可以** |
| 视觉效果 | 缩小上移，半透明 | 完全不可见 |
| 适用场景 | 确认对话框、表单等 | 自动完成、快速选择等 |

**设计权衡**：
- 有动画弹窗保留 DOM 是为了实现平滑的过渡动画
- 无动画弹窗不需要过渡，所以可以安全地设置 `display: none`
- 这解释了为什么只有自动完成类弹窗使用 `noAnimation`——它们打开关闭频繁，不需要动画，且需要严格的焦点控制

---

## 四、多个 ReactAriaModalOverlay 叠加时的焦点陷阱优先级

### 4.1 焦点陷阱的 DOM 结构

当有两个弹窗叠加时，DOM 结构大致如下：
```html
<!-- 页面内容 -->
<div id="root">
  <button id="trigger-btn">打开弹窗</button>
</div>

<!-- 弹窗 A（下层）的 ModalOverlay -->
<div data-testid="confirm-delete-modal" 
     style="position: fixed; inset: 0; z-index: 3000;">
  <div style="display: flex; align-items: center; justify-content: center;">
    <div role="dialog" aria-modal="true">
      <button id="btn-a">确认</button>
      <button id="btn-b">取消</button>
    </div>
  </div>
</div>

<!-- 弹窗 B（顶层）的 ModalOverlay -->
<div data-testid="payee-autocomplete-modal" 
     style="position: fixed; inset: 0; z-index: 3000;">
  <div style="display: flex; align-items: center; justify-content: center;">
    <div role="dialog" aria-modal="true">
      <input id="input-search" />
      <button id="btn-c">选择</button>
    </div>
  </div>
</div>
```

**关键点**：
- 两个 ModalOverlay 都是 `position: fixed; inset: 0; z-index: 3000`
- 弹窗 B 的 DOM 在弹窗 A 之后，所以自然覆盖在上面
- 每个 ModalOverlay 都有自己的焦点陷阱

### 4.2 焦点陷阱的协调机制

React Aria 的焦点陷阱通过以下机制协调：

#### 1. document.activeElement 追踪
- 每个焦点陷阱监听全局的 `focus` 事件
- 当焦点移动到陷阱外时，检查是否移动到了另一个模态对话框内
- 如果是，则不抢回焦点，让另一个陷阱接管

#### 2. aria-modal 属性
- 每个弹窗的 `<Dialog>` 都设置了 `aria-modal="true"`
- 这告诉辅助技术：这是一个模态对话框，应该忽略背后的内容

#### 3. pointer-events: none（下层弹窗）
- 如第三节所述，下层弹窗的内容容器设置了 `pointer-events: none`
- 这阻止了鼠标点击下层弹窗的元素

#### 4. display: none（noAnimation 下层弹窗）
- 对于 noAnimation 弹窗，非活动时完全移除
- 从根本上消除了被聚焦的可能

### 4.3 焦点陷阱的优先级规则

**优先级从高到低**：

| 层级 | 机制 | 效果 |
|------|------|------|
| 1. 最顶层弹窗 | 最后渲染的 DOM + aria-modal=true | 获得用户输入 |
| 2. 下层 noAnimation 弹窗 | display: none | 完全不可交互 |
| 3. 下层有动画弹窗 | pointer-events: none | 阻止鼠标交互，但 Tab 仍可能聚焦 |
| 4. 页面内容 | 被 ModalOverlay 覆盖 | 阻止鼠标交互，焦点陷阱阻止 Tab |

**Tab 键行为**：
```
用户在顶层弹窗按 Tab 键
        ↓
焦点在顶层弹窗的可聚焦元素间循环
（输入框 → 按钮1 → 按钮2 → 输入框 → ...）
        ↓
焦点不会跳到下层弹窗，也不会跳到页面
（React Aria 的焦点陷阱确保）
```

### 4.4 特殊情况：Popover 与弹窗的交互

项目中大量使用 `Popover` 组件（右键菜单、下拉菜单等），它们也使用 React Aria，但**不会**进入 modalStack。

**Popover 的两种模式**（[Popover.tsx#L27-L34](file:///d:/fz/0601-1/solo-dogfeeding/code/74-actual/packages/component-library/src/Popover.tsx#L27-L34)）：

```typescript
useEffect(() => {
  if (!props.isNonModal) return;  // ← isNonModal = true 时才执行
  if (props.isOpen) {
    ref.current?.addEventListener('focusout', handleFocus);
  } else {
    ref.current?.removeEventListener('focusout', handleFocus);
  }
}, [handleFocus, props.isNonModal, props.isOpen]);
```

| 模式 | isNonModal 值 | 焦点行为 | 使用场景 |
|------|--------------|----------|---------|
| 模态模式 | `false`（默认） | React Aria 管理焦点陷阱，类似弹窗 | 单元格编辑器（[TransactionsTable.tsx#L1370](file:///d:/fz/0601-1/solo-dogfeeding/code/74-actual/packages/desktop-client/src/components/transactions/TransactionsTable.tsx#L1370)） |
| 非模态模式 | `true` | 自定义 focusout 处理，焦点移出时关闭 | 右键菜单、下拉菜单（绝大多数场景） |

**非模态 Popover 的焦点处理**（[Popover.tsx#L18-L25](file:///d:/fz/0601-1/solo-dogfeeding/code/74-actual/packages/component-library/src/Popover.tsx#L18-L25)）：
```typescript
const handleFocus = useCallback(
  (e: FocusEvent) => {
    if (!ref.current?.contains(e.relatedTarget as Node)) {
      props.onOpenChange?.(false);  // 焦点移出时关闭
    }
  },
  [props],
);
```

**弹窗内打开 Popover 的场景**：
1. 用户在弹窗内右键点击，打开上下文菜单（Popover，isNonModal=true）
2. Popover 不会干扰弹窗的焦点陷阱
3. 按 ESC 会先关闭 Popover，再按 ESC 才会关闭弹窗
4. 这是因为 Popover 自己监听了键盘事件，优先级更高

---

## 五、完整系统架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                     用户交互层                                    │
│  键盘输入 → react-hotkeys-hook → 按 scope 分发快捷键             │
│  鼠标输入 → 浏览器事件 → 按 pointer-events 过滤                  │
└──────────────────────┬──────────────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────────────┐
│                     弹窗栈状态层 (Redux)                         │
│  modalStack: [ModalA, ModalB, ModalC]                            │
│  - pushModal: 追加到数组末尾                                     │
│  - popModal: 移除数组最后一个                                    │
│  - closeModal: 清空数组                                          │
│  - replaceModal: 替换整个数组                                    │
│  - collapseModals: 截断数组到指定索引（不含）                    │
└──────────────────────┬──────────────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────────────┐
│                     渲染层 (Modals.tsx)                         │
│  map(modalStack) → 为每个弹窗渲染 <Modal> 组件                   │
│  数组索引 → DOM 顺序 → 自然层叠顺序                              │
└──────────────────────┬──────────────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────────────┐
│                  单个弹窗组件层 (<Modal>)                        │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  1. Hotkeys Scope 管理                                    │  │
│  │  - enableScope(name) → 禁用 'app' 和其他弹窗 scope        │  │
│  │  - disableScope(name) → 卸载时恢复                        │  │
│  └──────────────────────────┬───────────────────────────────┘  │
│                             │                                   │
│  ┌──────────────────────────▼───────────────────────────────┐  │
│  │  2. ReactAriaModalOverlay                               │  │
│  │  - 焦点陷阱：Tab 键在弹窗内循环                          │  │
│  │  - 保存/恢复触发元素的焦点                               │  │
│  │  - ESC 关闭、点击遮罩关闭                                │  │
│  └──────────────────────────┬───────────────────────────────┘  │
│                             │                                   │
│  ┌──────────────────────────▼───────────────────────────────┐  │
│  │  3. ModalContentContainer                               │  │
│  │  - isActive ? 正常显示 : 缩小上移 + pointer-events:none │  │
│  │  - noAnimation && !isActive ? display: none             │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────────────┐
│                     DOM 层                                        │
│  - 多个 ModalOverlay 叠加，z-index 相同，靠 DOM 顺序层叠          │
│  - 顶层弹窗 aria-modal=true                                      │
│  - 下层弹窗 pointer-events=none 或 display=none                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 六、关键代码索引

| 机制 | 核心代码位置 |
|------|-------------|
| HotkeysProvider 初始化 | [App.tsx#L200](file:///d:/fz/0601-1/solo-dogfeeding/code/74-actual/packages/desktop-client/src/components/App.tsx#L200) |
| 弹窗作用域管理 | [Modal.tsx#L60-L66](file:///d:/fz/0601-1/solo-dogfeeding/code/74-actual/packages/desktop-client/src/components/common/Modal.tsx#L60-L66) |
| 页面快捷键 scope 绑定 | [SelectedTransactionsButton.tsx#L227-L229](file:///d:/fz/0601-1/solo-dogfeeding/code/74-actual/packages/desktop-client/src/components/transactions/SelectedTransactionsButton.tsx#L227-L229) |
| ModalOverlay 三层结构 | [Modal.tsx#L77-L175](file:///d:/fz/0601-1/solo-dogfeeding/code/74-actual/packages/desktop-client/src/components/common/Modal.tsx#L77-L175) |
| 弹窗栈渲染 | [Modals.tsx#L90-L180](file:///d:/fz/0601-1/solo-dogfeeding/code/74-actual/packages/desktop-client/src/components/Modals.tsx#L90-L180) |
| isActive 判断 | [useModalState.ts#L25-L34](file:///d:/fz/0601-1/solo-dogfeeding/code/74-actual/packages/desktop-client/src/hooks/useModalState.ts#L25-L34) |
| isActive 视觉效果 | [Modal.tsx#L207-L214](file:///d:/fz/0601-1/solo-dogfeeding/code/74-actual/packages/desktop-client/src/components/common/Modal.tsx#L207-L214) |
| noAnimation 初始化逻辑 | [Modal.tsx#L217-L244](file:///d:/fz/0601-1/solo-dogfeeding/code/74-actual/packages/desktop-client/src/components/common/Modal.tsx#L217-L244) |
| display: none 关键行 | [Modal.tsx#L250-L253](file:///d:/fz/0601-1/solo-dogfeeding/code/74-actual/packages/desktop-client/src/components/common/Modal.tsx#L250-L253) |
| Popover isNonModal 处理 | [Popover.tsx#L27-L34](file:///d:/fz/0601-1/solo-dogfeeding/code/74-actual/packages/component-library/src/Popover.tsx#L27-L34) |
| Popover 非模态焦点处理 | [Popover.tsx#L18-L25](file:///d:/fz/0601-1/solo-dogfeeding/code/74-actual/packages/component-library/src/Popover.tsx#L18-L25) |

---

## 七、设计亮点总结

| 设计决策 | 解决的问题 |
|---------|-----------|
| 用 `react-hotkeys-hook` 的 scope 管理快捷键 | 避免弹窗打开时触发页面快捷键，无需手动维护 |
| 每个弹窗独立的 ReactAriaModalOverlay | 每个弹窗独立维护焦点陷阱和触发元素，天然形成焦点栈 |
| `pointer-events: none` 处理下层弹窗 | 阻止鼠标点击下层弹窗，同时保留动画效果 |
| `noAnimation` 路径的 `display: none` | 对于不需要动画的弹窗，彻底阻止 Tab 键聚焦到下层 |
| 相同 z-index 靠 DOM 顺序层叠 | 无需动态计算 z-index，避免 z-index 竞争问题 |
| `isNonModal` Popover 自定义焦点处理 | 右键菜单等非模态 UI 不会干扰弹窗的焦点陷阱 |
