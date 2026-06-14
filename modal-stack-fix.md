# 弹窗栈与焦点恢复机制 - 代码更正分析报告

> 本文档是对《弹窗栈与焦点恢复机制分析.md》的更正与补充，重点修正了三处与代码实际情况不符的分析，并补充了 `collapseModals` 和 `replaceModal` 的调用场景。

---

## 一、三处关键更正

### 1.1 InitialFocus 中 setTimeout 的真实原因

**原分析错误**：
> 使用 `setTimeout(..., 0)` 确保 DOM 完全渲染后执行

**代码实际情况**（[InitialFocus.ts#L35-L38](file:///d:/fz/0601-1/solo-dogfeeding/code/74-actual/packages/component-library/src/InitialFocus.ts#L35-L38)）：
```typescript
// This is needed to avoid a strange interaction with
// `ScopeTab`, which doesn't allow it to be focused at first for
// some reason. Need to look into it.
setTimeout(() => {
  if (ref.current) {
    ref.current.focus();
  }
}, 0);
```

**正确分析**：
- `setTimeout(0)` 的真正目的是**规避与 `ScopeTab` 库的交互问题**
- 注释明确说明：ScopeTab 在初始阶段不允许元素获得焦点，原因未知，需要进一步调查
- 这不是为了等待 DOM 渲染，而是为了绕过一个库兼容性 bug
- 推迟到下一个事件循环（setTimeout 0）后，ScopeTab 的初始化已完成，此时焦点设置才能生效

**ScopeTab 背景**：
- 从代码上下文看，ScopeTab 应该是一个用于管理 Tab 键焦点循环的工具（类似 react-aria 的 FocusScope）
- 可能在弹窗初始化阶段，ScopeTab 暂时拦截或屏蔽了焦点设置
- 这是一个已知的技术债务（注释中有 "Need to look into it"）

---

### 1.2 pushModal 的完整特殊豁免条件

**原分析遗漏**：
> 已有弹窗打开时，不显示快捷键帮助

**代码实际情况**（[modalsSlice.ts#L720-L727](file:///d:/fz/0601-1/solo-dogfeeding/code/74-actual/packages/desktop-client/src/modals/modalsSlice.ts#L720-L727)）：
```typescript
if (
  modal.name.endsWith('keyboard-shortcuts') &&
  (state.modalStack.length > 0 ||
    window.document.querySelector(
      'div[data-testid="filters-menu-tooltip"]',
    ) !== null)
) {
  return state;
}
```

**完整条件梳理**：
当满足以下**全部**条件时，拒绝推入新弹窗：

| 条件 | 说明 | 代码位置 |
|------|------|----------|
| 1. 弹窗类型是 keyboard-shortcuts | `modal.name.endsWith('keyboard-shortcuts')` | [L721](file:///d:/fz/0601-1/solo-dogfeeding/code/74-actual/packages/desktop-client/src/modals/modalsSlice.ts#L721) |
| 2. 满足以下**任一**子条件 | `(A \|\| B)` | [L722-L725](file:///d:/fz/0601-1/solo-dogfeeding/code/74-actual/packages/desktop-client/src/modals/modalsSlice.ts#L722-L725) |
| &nbsp;&nbsp;A. 弹窗栈非空 | `state.modalStack.length > 0` | [L722](file:///d:/fz/0601-1/solo-dogfeeding/code/74-actual/packages/desktop-client/src/modals/modalsSlice.ts#L722) |
| &nbsp;&nbsp;B. 过滤器菜单 tooltip 正在显示 | `querySelector('div[data-testid="filters-menu-tooltip"]') !== null` | [L723-L725](file:///d:/fz/0601-1/solo-dogfeeding/code/74-actual/packages/desktop-client/src/modals/modalsSlice.ts#L723-L725) |

**filters-menu-tooltip 说明**：
- 这个 tooltip 出现在过滤器编辑界面，z-index 为 2500（[FiltersMenu.tsx#L790](file:///d:/fz/0601-1/solo-dogfeeding/code/74-actual/packages/desktop-client/src/components/filters/FiltersMenu.tsx#L790), [FilterExpression.tsx#L148](file:///d:/fz/0601-1/solo-dogfeeding/code/74-actual/packages/desktop-client/src/components/filters/FilterExpression.tsx#L148)）
- 低于弹窗的 z-index 3000，但仍被视为"模态"UI 元素，需要阻止快捷键弹窗打断
- 这是一个**特殊的硬编码例外**，反映了过滤器菜单的实现没有走标准的 modalStack 流程

---

### 1.3 collapseModals 的实际行为

**原分析错误**：
> 从栈顶移除直到找到目标弹窗（保留目标弹窗）

**代码实际情况**（[modalsSlice.ts#L741-L747](file:///d:/fz/0601-1/solo-dogfeeding/code/74-actual/packages/desktop-client/src/modals/modalsSlice.ts#L741-L747)）：
```typescript
collapseModals(state, action: PayloadAction<CollapseModalPayload>) {
  const idx = state.modalStack.findIndex(
    m => m.name === action.payload.rootModalName,
  );
  state.modalStack =
    idx < 0 ? state.modalStack : state.modalStack.slice(0, idx);
}
```

**正确分析**：
- `slice(0, idx)` 表示保留索引 `[0, idx)` 的元素，**不包含 idx 位置的元素**
- 这意味着：**目标弹窗本身也会被移除**
- 参数名 `rootModalName` 准确表达了语义：这是要关闭的**根**弹窗，所有在它之上（包括它自己）的弹窗都会被关闭

**行为示例**：
```
初始栈: [envelope-budget-summary, transfer, confirm-delete]
              索引 0          索引 1        索引 2

调用: collapseModals({ rootModalName: 'transfer' })

找到 'transfer' 的索引 idx = 1
执行 slice(0, 1) → 保留 [0, 1) → 只保留索引 0 的元素

结果栈: [envelope-budget-summary]
         ↑  transfer 和 confirm-delete 都被移除了
```

**参数定义验证**（[modalsSlice.ts#L709-L711](file:///d:/fz/0601-1/solo-dogfeeding/code/74-actual/packages/desktop-client/src/modals/modalsSlice.ts#L709-L711)）：
```typescript
type CollapseModalPayload = {
  rootModalName: Modal['name'];  // "root" = 要关闭的根节点
};
```

---

## 二、collapseModals 调用场景梳理

`collapseModals` 主要用于**完成某个操作后，关闭一系列相关弹窗，回到某个基础页面**。

### 2.1 桌面端：预算汇总弹窗中的操作

在 [EnvelopeBudgetSummaryModal.tsx](file:///d:/fz/0601-1/solo-dogfeeding/code/74-actual/packages/desktop-client/src/components/modals/EnvelopeBudgetSummaryModal.tsx) 中，有三处使用：

| 操作 | 调用代码 | 说明 |
|------|----------|------|
| 转账完成 | `collapseModals({ rootModalName: 'transfer' })` | [L69](file:///d:/fz/0601-1/solo-dogfeeding/code/74-actual/packages/desktop-client/src/components/modals/EnvelopeBudgetSummaryModal.tsx#L69) |
| 覆盖超支完成 | `collapseModals({ rootModalName: 'cover' })` | [L99](file:///d:/fz/0601-1/solo-dogfeeding/code/74-actual/packages/desktop-client/src/components/modals/EnvelopeBudgetSummaryModal.tsx#L99) |
| 设置缓冲完成 | `collapseModals({ rootModalName: 'hold-buffer' })` | [L121](file:///d:/fz/0601-1/solo-dogfeeding/code/74-actual/packages/desktop-client/src/components/modals/EnvelopeBudgetSummaryModal.tsx#L121) |

**典型调用链**（以转账为例）：
```
1. 用户在预算汇总弹窗中点击"转账"按钮
   → pushModal({ name: 'transfer' })
   → 栈: [envelope-budget-summary, transfer]

2. 用户在转账弹窗中输入金额并确认
   → 执行转账逻辑
   → collapseModals({ rootModalName: 'transfer' })
   → 找到 transfer 索引为 1，slice(0, 1)
   → 栈: [envelope-budget-summary]
   → 回到预算汇总弹窗，显示 undo 通知
```

### 2.2 移动端：分类列表项中的余额菜单操作

在 [IncomeCategoryListItem.tsx](file:///d:/fz/0601-1/solo-dogfeeding/code/74-actual/packages/desktop-client/src/components/mobile/budget/IncomeCategoryListItem.tsx) 和 [ExpenseCategoryListItem.tsx](file:///d:/fz/0601-1/solo-dogfeeding/code/74-actual/packages/desktop-client/src/components/mobile/budget/ExpenseCategoryListItem.tsx) 中：

```typescript
// IncomeCategoryListItem.tsx L207
dispatch(collapseModals({ rootModalName: balanceMenuModalName }));
```

**场景**：
- 用户在移动端分类列表中点击余额，打开余额菜单弹窗
- 用户选择"结转"选项并确认
- 操作完成后，关闭余额菜单及可能叠加在其上的所有弹窗
- 回到分类列表页面

---

## 三、replaceModal 调用场景梳理

`replaceModal` 用于**清空当前弹窗栈，用一个新弹窗取而代之**。这通常用于不关心当前上下文，直接跳转到某个特定弹窗的场景。

### 3.1 撤销操作时恢复弹窗状态

在 [global-events.ts#L102-L107](file:///d:/fz/0601-1/solo-dogfeeding/code/74-actual/packages/desktop-client/src/global-events.ts#L102-L107) 中：

```typescript
if (tagged.openModal) {
  const openModal = tagged.openModal as Modal;
  const { modalStack } = store.getState().modals;

  if (
    modalStack.length === 0 ||
    modalStack[modalStack.length - 1].name !== openModal.name
  ) {
    store.dispatch(replaceModal({ modal: openModal }));
  }
}
```

**场景**：
- 用户执行了某个操作（如修改预算），该操作关联了一个弹窗
- 用户触发撤销（Undo）
- 系统检查 undo tag 中保存的 `openModal`
- 如果当前没有弹窗，或者栈顶弹窗不是之前的那个，就用 `replaceModal` 恢复到之前的弹窗状态
- 这样可以确保撤销操作后，UI 状态与操作前完全一致

### 3.2 侧边栏添加账户按钮

在 [Sidebar.tsx#L63-L65](file:///d:/fz/0601-1/solo-dogfeeding/code/74-actual/packages/desktop-client/src/components/sidebar/Sidebar.tsx#L63-L65) 中：

```typescript
const onAddAccount = () => {
  dispatch(replaceModal({ modal: { name: 'add-account', options: {} } }));
};
```

**场景**：
- 用户在侧边栏点击"添加账户"按钮
- 不管当前有什么弹窗（可能用户正在做其他操作），直接清空栈并打开添加账户弹窗
- 这是一个"全局导航"式的操作，优先级高于当前上下文

### 3.3 合并收款人后跳转到规则编辑

在 [MergeUnusedPayeesModal.tsx#L90-L92](file:///d:/fz/0601-1/solo-dogfeeding/code/74-actual/packages/desktop-client/src/components/modals/MergeUnusedPayeesModal.tsx#L90-L92) 中：

```typescript
dispatch(
  replaceModal({ modal: { name: 'edit-rule', options: { rule } } }),
);
```

**场景**：
- 用户在"合并未使用收款人"弹窗中完成合并
- 系统自动创建了一个分类规则
- 用 `replaceModal` 清空当前栈，直接打开规则编辑弹窗
- 用户可以立即查看和调整新创建的规则

---

## 四、操作对比总结

| 操作 | 对栈的影响 | 典型使用场景 |
|------|------------|--------------|
| `pushModal` | `[...stack, newModal]` 追加 | 正常打开新弹窗 |
| `popModal` | `stack.slice(0, -1)` 移除栈顶 | 按 ESC 或关闭按钮 |
| `closeModal` | `[]` 清空全部 | 路由切换、完成加载 |
| `replaceModal` | `[newModal]` 替换全部 | 全局导航、撤销恢复、流程跳转 |
| `collapseModals(name)` | `stack.slice(0, idx)` 截断到 idx（不包含 idx） | 完成操作后回到基础页面 |

---

## 五、关键代码索引

| 更正点 | 代码位置 |
|--------|----------|
| InitialFocus ScopeTab 注释 | [InitialFocus.ts#L35-L38](file:///d:/fz/0601-1/solo-dogfeeding/code/74-actual/packages/component-library/src/InitialFocus.ts#L35-L38) |
| pushModal 完整豁免条件 | [modalsSlice.ts#L720-L727](file:///d:/fz/0601-1/solo-dogfeeding/code/74-actual/packages/desktop-client/src/modals/modalsSlice.ts#L720-L727) |
| filters-menu-tooltip 定义1 | [FiltersMenu.tsx#L792](file:///d:/fz/0601-1/solo-dogfeeding/code/74-actual/packages/desktop-client/src/components/filters/FiltersMenu.tsx#L792) |
| filters-menu-tooltip 定义2 | [FilterExpression.tsx#L150](file:///d:/fz/0601-1/solo-dogfeeding/code/74-actual/packages/desktop-client/src/components/filters/FilterExpression.tsx#L150) |
| collapseModals 实现 | [modalsSlice.ts#L741-L747](file:///d:/fz/0601-1/solo-dogfeeding/code/74-actual/packages/desktop-client/src/modals/modalsSlice.ts#L741-L747) |
| collapseModals 转账场景 | [EnvelopeBudgetSummaryModal.tsx#L69](file:///d:/fz/0601-1/solo-dogfeeding/code/74-actual/packages/desktop-client/src/components/modals/EnvelopeBudgetSummaryModal.tsx#L69) |
| collapseModals 覆盖场景 | [EnvelopeBudgetSummaryModal.tsx#L99](file:///d:/fz/0601-1/solo-dogfeeding/code/74-actual/packages/desktop-client/src/components/modals/EnvelopeBudgetSummaryModal.tsx#L99) |
| collapseModals 缓冲场景 | [EnvelopeBudgetSummaryModal.tsx#L121](file:///d:/fz/0601-1/solo-dogfeeding/code/74-actual/packages/desktop-client/src/components/modals/EnvelopeBudgetSummaryModal.tsx#L121) |
| replaceModal 撤销场景 | [global-events.ts#L102-L107](file:///d:/fz/0601-1/solo-dogfeeding/code/74-actual/packages/desktop-client/src/global-events.ts#L102-L107) |
| replaceModal 添加账户 | [Sidebar.tsx#L63-L65](file:///d:/fz/0601-1/solo-dogfeeding/code/74-actual/packages/desktop-client/src/components/sidebar/Sidebar.tsx#L63-L65) |
| replaceModal 合并跳转 | [MergeUnusedPayeesModal.tsx#L90-L92](file:///d:/fz/0601-1/solo-dogfeeding/code/74-actual/packages/desktop-client/src/components/modals/MergeUnusedPayeesModal.tsx#L90-L92) |
