# Actual Budget 错误边界分层设计文档

## 概述

Actual Budget 采用三层错误处理机制，通过 `react-error-boundary` 库实现 React 组件树的错误捕获和优雅降级，结合 Redux 通知系统实现异步错误反馈，确保应用在遇到异常时能够提供良好的用户体验。

---

## ErrorBoundary 异常捕获最终判定表

| 类别 | 场景 | 捕获方式 | 备注 |
|-----|-----|---------|------|
| **自动捕获** | 组件 render 函数 | ✅ ErrorBoundary 自动捕获 | JSX 渲染过程中抛出的异常 |
| **自动捕获** | 函数组件体执行 | ✅ ErrorBoundary 自动捕获 | 函数组件主执行流程 |
| **自动捕获** | useEffect 回调 | ✅ ErrorBoundary 自动捕获 | 副作用执行过程 |
| **自动捕获** | useLayoutEffect 回调 | ✅ ErrorBoundary 自动捕获 | 布局副作用执行过程 |
| **自动捕获** | 类组件生命周期 | ✅ ErrorBoundary 自动捕获 | componentDidMount 等 |
| **自动捕获** | 类构造函数 constructor | ✅ ErrorBoundary 自动捕获 | 组件实例化阶段 |
| **必须手动上报** | async/await 异步函数 | ❌ 必须手动 catch + showErrorBoundary | ErrorBoundary 无法捕获异步异常 |
| **必须手动上报** | Promise 链 .catch() | ❌ 必须手动 catch + showErrorBoundary | Promise rejection 不会冒泡到渲染层 |
| **必须手动上报** | 事件处理器 onClick/onChange | ❌ 必须手动 catch + showErrorBoundary | 事件回调独立于 React 渲染周期 |
| **必须手动上报** | setTimeout/setInterval | ❌ 必须手动 catch + showErrorBoundary | 定时器回调在宏任务队列执行 |
| **必须手动上报** | 第三方库回调函数 | ❌ 必须手动 catch + showErrorBoundary | 非 React 控制的代码执行 |
| **必须手动上报** | Redux thunk 异步 action | ❌ 使用 addNotification 通知 | 通过通知系统反馈，不触发 ErrorBoundary |

**代码示例：**
```tsx
// ✅ 自动捕获：渲染异常直接被 ErrorBoundary 捕获
function BadComponent() {
  const data = undefined;
  return <div>{data.name}</div>; // 自动捕获
}

// ❌ 必须手动上报：异步异常
function AsyncComponent() {
  const { showBoundary } = useErrorBoundary();
  
  useEffect(() => {
    // 必须手动 catch
    fetchData().catch(showBoundary);
  }, [showBoundary]);
}
```

---

## 错误边界分层架构

### 第一层：应用级致命错误边界 (FatalError)

**位置**: `packages/desktop-client/src/components/App.tsx:221-226`

```tsx
<ErrorBoundary FallbackComponent={ErrorFallback}>
  <AppInner />
</ErrorBoundary>
```

**覆盖范围**: 整个应用主体，包括预算页面、账户、报告等所有核心功能模块

#### 触发条件

1. **应用初始化失败** (`type === 'app-init-failure'`)
   - IndexedDB 数据库打开失败（IDBFailure）
   - SharedArrayBuffer 不可用（SharedArrayBufferMissing）
   - 后端 Worker 初始化失败（BackendInitFailure）

2. **代码分割懒加载失败** (`LazyLoadFailedError`)
   - 动态 import() 的 chunk 加载失败
   - 网络问题或 CDN 资源不可用

3. **UI 渲染过程中未捕获的异常**
   - 组件渲染函数抛出异常
   - useEffect 等副作用中的异常
   - 任何穿透到 App 根组件的错误

#### FatalError 组件行为

**位置**: `packages/desktop-client/src/components/FatalError.tsx`

| 错误类型 | 显示内容 | 恢复入口 |
|---------|---------|---------|
| IndexedDB 失败 | 浏览器环境不支持提示，建议退出隐私模式 | - |
| SharedArrayBuffer 缺失 | COOP/COEP 头配置说明，链接到排障文档 | "高级选项" → 勾选确认风险 → 强制启用（localStorage 标记） |
| 后端初始化失败 | 建议刷新或硬刷新清除缓存 | - |
| 懒加载失败 | 网络/服务器问题提示，建议刷新 | "Restart app" 按钮 |
| 通用 UI 错误 | 抱歉提示 + 联系支持链接 | "Restart app" 按钮 |

**技术要点**:
- 错误信息默认折叠，点击 "Show Error" 显示完整 stack trace
- 模态框不可关闭（isDismissable={false}），强制用户处理
- 重启调用 `window.Actual.relaunch()`

---

### 第二层：模态框独立错误边界

**位置**: `packages/desktop-client/src/components/App.tsx:229-231`

```tsx
<ErrorBoundary FallbackComponent={FatalError}>
  <Modals />
</ErrorBoundary>
```

**设计意图**:
- 模态框与主应用隔离，避免模态框内错误扩散到整个应用
- 模态框组件自身还包裹了 FeatureErrorBoundary 实现双层防护

#### ⚠️ 模态层错误接管后的交互状态（最终结论）

**统一结论：FatalError 模态框接管后**完全阻断主界面交互**

```tsx
// packages/desktop-client/src/components/common/Modal.tsx:77-96
<ReactAriaModalOverlay
  style={{
    position: 'fixed',  // 固定定位
    inset: 0,           // 覆盖整个视口（上右下左全为0）
    zIndex: 3000,       // 最高层级（MODAL_Z_INDEX = 3000）
    backgroundColor: 'rgba(0, 0, 0, 0.4)',  // 黑色半透明遮罩
    backdropFilter: 'blur(1px) brightness(0.9)',  // 毛玻璃效果
  }}
>
```

**交互阻断四要素**:
1. **视觉遮罩**: `inset: 0` + `position: fixed` 覆盖 100% 视口区域，主界面内容不可见
2. **层级压制**: `zIndex: 3000` 确保在所有 UI 元素之上（Notifications zIndex = 2999）
3. **事件捕获**: React Aria ModalOverlay 内部拦截所有指针事件，点击主界面区域无效
4. **不可关闭**: `isDismissable={false}` 禁用点击遮罩关闭和 ESC 键关闭

**用户可操作范围**: 仅限 FatalError 模态框内部的按钮（Restart app、Show Error）

---

### 第三层：功能级错误边界 (FeatureErrorFallback)

**位置**: `packages/desktop-client/src/components/FeatureErrorFallback.tsx`

**应用场景**: 包裹具体功能模块，实现局部错误隔离

#### 典型使用位置

| 组件/页面 | 文件位置 | 说明 |
|---------|---------|------|
| 交易列表 | TransactionList.tsx:727 | 包裹整个交易表格 |
| 规则页面 | FinancesApp.tsx:294-299 | /routes 路由级包裹 |
| 规则编辑 | FinancesApp.tsx:305-311 | /rules/:id 路由级 |
| 侧边栏 | Sidebar.tsx | 导航菜单隔离 |
| 报告卡片 | Overview.tsx, GetCardData.tsx | 单个报告组件 |
| 预算表格 | DynamicBudgetTable.tsx | 预算电子表格 |
| 账户详情 | Account.tsx | 单账户页面 |
| 计划任务 | schedules/index.tsx | 计划任务列表 |
| Modal 内容 | Modal.tsx:76,144 | 模态框内容双重包裹 |

#### 路由级使用模式

```tsx
<Route
  path="/rules"
  element={
    <ErrorBoundary
      FallbackComponent={FeatureErrorFallback}
      resetKeys={[location.pathname]}
    >
      <NarrowAlternate name="Rules" />
    </ErrorBoundary>
  }
/>
```

**resetKeys 机制**: 当路由路径变化时自动重置错误边界状态，用户切换页面即可从错误中恢复。

#### FeatureErrorFallback UI 组成

1. **错误提示文案**: "Something went wrong loading this section."
2. **错误详情**: 显示 error.message（等宽字体）
3. **恢复按钮**: "Try again" → 调用 `resetErrorBoundary()` 重试渲染

---

## 通知系统与错误上报

### Redux 通知切片 (notificationsSlice)

**位置**: `packages/desktop-client/src/notifications/notificationsSlice.ts`

### 通知类型

```typescript
type Notification = {
  type?: 'message' | 'error' | 'warning';
  title?: string;
  message: string;
  sticky?: boolean;       // 是否常驻（不自动消失）
  timeout?: number;       // 自动关闭超时
  button?: {             // 操作按钮
    title: string;
    action: () => void | Promise<void>;
  };
  messageActions?: Record<string, () => void>;  // 消息内链接动作
  onClose?: () => void;  // 关闭回调
};
```

### 错误通知触发场景

1. **登录过期** (`App.tsx:134-151`)
   - type: 'error'
   - sticky: true
   - button: "Go to login" → 调用 signOut()

2. **应用更新可用** (`FinancesApp.tsx:153-180`)
   - type: 'message'
   - button: "Open changelog" → 打开外部链接
   - onClose: 记录已确认版本

3. **更新下载完成** (`FinancesApp.tsx:117-134`)
   - type: 'message'
   - sticky: true
   - button: "Update now" → 调用 applyAppUpdate()

4. **同步失败** (`sync-events.ts:379-382`)
   - type: 'error'
   - 非致命，不阻断主界面操作

5. **通用内部错误** (`addGenericErrorNotification`)
   - 建议用户重启应用 + 报告 GitHub issue

### Notifications 组件特性

**位置**: `packages/desktop-client/src/components/Notifications.tsx`

- **堆叠显示**: 最多同时显示 3 条通知，新通知在前
- **层级效果**: 背景通知缩放 + 透明度渐变 + 垂直偏移
- **交互方式**: 
  - 仅最前一条可交互（isInteractive）
  - 左右滑动删除
  - 右上角关闭按钮
- **超时自动关闭**: 默认 6.5 秒（sticky 除外）
- **消息内动作**: 支持 Markdown 风格链接 `[text](#actionName)`

---

## 通知触发到恢复动作执行检查清单

### 📋 可执行检查清单

| 序号 | 检查项 | 执行标准 | 验证方式 |
|-----|--------|---------|---------|
| 1 | **异常分类判定** | 正确区分同步渲染异常（走 ErrorBoundary）和异步/业务异常（走通知） | 对照「异常捕获最终判定表」 |
| 2 | **异步异常捕获** | async/await 必须包裹 try/catch，Promise 必须有 .catch() | 静态代码审查 |
| 3 | **dispatch addNotification** | 调用时传入正确的 type、message、id（如需要去重） | Redux DevTools 检查 action |
| 4 | **通知状态更新** | Redux state.notifications 数组新增元素 | Redux DevTools 检查 state |
| 5 | **Notification 组件渲染** | 通知出现在屏幕右下角，z-index = 2999 | 视觉检查 + 元素审查 |
| 6 | **sticky 判定** | sticky=true 常驻不消失；sticky=false 6.5秒后调用 removeNotification | 计时观察 |
| 7 | **按钮动作注入** | button.action 函数正确绑定恢复逻辑（如 signOut、sync 等） | 断点调试 |
| 8 | **用户点击操作按钮** | 按钮进入 loading 状态（setLoading=true），禁用重复点击 | 视觉检查 |
| 9 | **恢复动作执行** | button.action() 异步函数正确执行，无内部异常 | 断点调试 + 日志检查 |
| 10 | **通知移除** | 调用 onRemove() → dispatch(removeNotification)，通知从屏幕消失 | Redux DevTools + 视觉检查 |
| 11 | **按钮状态恢复** | setLoading(false)，按钮恢复可点击状态（如未自动移除） | 视觉检查 |
| 12 | **onClose 回调** | 如有 onClose，必须在通知移除后触发 | 断点验证 |

### 检查清单使用说明

**开发阶段自查**:
- 新增错误处理代码时，逐条对照清单 1-12 项
- 重点检查清单 1-2 项（异常分类和捕获）和 7 项（动作注入）

**Code Review 阶段**:
- Reviewer 对照清单验证 PR 中错误处理逻辑的完整性
- 清单 3-12 项可通过代码走查完成

**Bug 复现排查**:
- 按清单顺序逐项排查，快速定位问题环节
- 例如：通知不显示 → 检查 3-5 项；按钮没反应 → 检查 7、9 项

---

## 通知 vs ErrorBoundary 选择策略

| 错误类型 | 推荐方案 | 原因 |
|---------|---------|------|
| 渲染崩溃 | ErrorBoundary | 需要隔离渲染异常 |
| 应用初始化失败 | FatalError | 必须重启才能恢复 |
| 网络请求失败 | 通知系统 | 用户可继续操作其他功能 |
| 权限不足 | 通知系统 | 提示 + 跳转动作 |
| 数据校验失败 | 表单内联提示 + 通知 | 不阻断全局 |
| 第三方集成失败 | 通知系统 | 可重试或忽略 |

---

## 错误边界复位机制

### 1. resetErrorBoundary()

由 `react-error-boundary` 提供，调用后：
- 清除错误状态
- 重新渲染包裹的组件树
- FeatureErrorFallback 中 "Try again" 按钮直接调用

### 2. resetKeys 路由驱动复位

```tsx
<ErrorBoundary
  FallbackComponent={FeatureErrorFallback}
  resetKeys={[location.pathname]}
>
```

当 `location.pathname` 变化时自动触发复位，用户切换页面即恢复。

### 3. 应用级重启

```tsx
window.Actual.relaunch()
```

FatalError 中的最终恢复手段，完全重新加载应用。

---

## 分层边界触发优先级

```
                  ┌─────────────────────────┐
                  │  功能级错误边界         │
                  │  (FeatureErrorFallback) │
                  │  - 交易列表             │
                  │  - 报告卡片             │
                  │  - 路由页面             │
                  └───────────┬─────────────┘
                              │
                        错误未捕获 ↓
                  ┌─────────────────────────┐
                  │  模态框错误边界         │
                  │  (FatalError)           │
                  │  - 仅 Modals 组件       │
                  └───────────┬─────────────┘
                              │
                        错误未捕获 ↓
                  ┌─────────────────────────┐
                  │  应用级错误边界         │
                  │  (FatalError)           │
                  │  - 整个 AppInner        │
                  └─────────────────────────┘
```

**设计原则**: 错误尽可能在最靠近源头的层级被捕获，避免向上扩散影响更多功能。

---

## 核心技术依赖

### react-error-boundary

- 提供 `<ErrorBoundary>` 组件
- `FallbackComponent` 属性接收错误信息和复位函数
- `resetKeys` 数组驱动自动复位
- `useErrorBoundary()` Hook 手动触发错误边界

### 错误类型定义

```typescript
// packages/desktop-client/src/components/FatalError.tsx
type AppError = Error & {
  type?: string;
  IDBFailure?: boolean;
  SharedArrayBufferMissing?: boolean;
  BackendInitFailure?: boolean;
};
```

---

## 新增错误边界最佳实践

1. **优先使用 FeatureErrorFallback**: 功能模块独立包裹，影响范围最小
2. **合理设置 resetKeys**: 如路由路径、依赖数据 ID 等变化时自动复位
3. **避免过度包裹**: 不要为每个小组件都加边界，按功能模块合理划分
4. **错误日志**: Fallback 组件中必须 `console.error(error)` 便于调试
5. **通知配合**: 非渲染异常优先使用通知系统，不一定要用错误边界
6. **异步必须手动 catch**: 所有 async/await 和 Promise 链必须捕获异常
7. **区分错误严重度**: 致命崩溃用 ErrorBoundary，可恢复错误用通知系统
8. **对照判定表**: 新增错误处理时先对照「异常捕获最终判定表」确定方案
9. **走查检查清单**: 通知类错误处理完成后，对照「检查清单」走查一遍

---

## 常见问题排查

### Q: 为什么某个功能白屏但没有错误提示？

A: 检查该组件是否被 ErrorBoundary 包裹。如果是渲染异常且未被捕获，会被上层 App 级边界捕获并显示 FatalError 模态框。

### Q: 错误边界捕获后如何调试？

A: 
- FeatureErrorFallback: 查看控制台 error 输出
- FatalError: 点击 "Show Error" 展开完整 stack trace

### Q: 如何让错误边界在特定条件下自动重置？

A: 使用 `resetKeys` 属性，传入依赖数组，任意元素变化即触发重置。

### Q: 异步错误（如 API 请求失败）会被错误边界捕获吗？

A: **不会**。ErrorBoundary 仅捕获渲染阶段、生命周期函数和构造函数中的同步异常。异步错误需要单独 try/catch 处理，通过通知系统反馈给用户。请对照「异常捕获最终判定表」确认。

### Q: FatalError 弹出后还能操作主界面吗？

A: **不能**。ModalOverlay 使用 `position: fixed; inset: 0; zIndex: 3000` 完全覆盖视口，拦截所有指针事件。用户只能操作 FatalError 内部的 Restart app 按钮。

### Q: 通知不显示如何排查？

A: 按「通知触发到恢复动作执行检查清单」逐项排查：先检查 dispatch action，再检查 Redux state 更新，最后检查组件渲染。

---

*文档版本: 2.0（一致性校对版）*
*最后更新: 2025-06-16*
*代码版本: 基于 packages/desktop-client v0.1.x*
