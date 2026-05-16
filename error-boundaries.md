# Actual Budget 错误边界分层设计文档

## 概述

Actual Budget 采用三层错误处理机制，通过 `react-error-boundary` 库实现 React 组件树的错误捕获和优雅降级，确保应用在遇到异常时能够提供良好的用户体验。

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
- 模态框与主应用隔离，避免模态框内错误导致整个应用崩溃
- 即使模态框渲染失败，用户仍可操作主界面
- 复用 FatalError 作为降级 UI，但作用域仅限于模态堆栈

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

4. **通用内部错误** (`addGenericErrorNotification`)
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

A: 不会。ErrorBoundary 仅捕获渲染阶段、生命周期函数和构造函数中的同步异常。异步错误需要单独 try/catch 处理，通过通知系统反馈给用户。

---

*文档生成时间: 2025-06-16*
*代码版本: 基于 packages/desktop-client v0.1.x*
