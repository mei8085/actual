# Actual Budget 备份恢复边界校准报告

## 摘要

本文档对备份恢复链路的三个关键边界进行了精准校准，所有结论均附有代码证据和影响评估。

| 验证项 | 之前假设 | 校准后结论 | 风险级别 |
|--------|---------|-----------|---------|
| **backup-load 并发保护** | 依赖 close-budget 收敛门 | ❌ 收敛门仅等待异步方法，不阻止重入；且 loading overlay 不阻止点击 | 🟡 中风险 |
| **界面防重入约束** | backupDisabled 动态控制加载状态 | ❌ backupDisabled 硬编码为 false，无实际防重入效果 | 🟡 中风险 |
| **latest 基线失效时机** | 用户手动删除或回退后删除 | ❌ makeBackup（手动/定时）会无条件删除 latest，即使解压正在进行中 | 🔴 高风险 |

---

## 一、backup-load 与 close-budget 收敛门的真实关系

### 1.1 代码证据链

**前端调用序列**：`packages/desktop-client/src/budgetfiles/budgetfilesSlice.ts:387-397`

```typescript
export const loadBackup = createAppAsyncThunk(
  `${sliceName}/loadBackup`,
  async ({ budgetId, backupId }, { dispatch, getState }) => {
    const prefs = getState().prefs.local;
    
    // 第 1 步：关闭预算（触发收敛门）
    if (prefs && prefs.id) {
      await dispatch(closeBudget());  // await 保证完成才继续
    }
    
    // 第 2 步：执行备份加载
    await send('backup-load', { id: budgetId, backupId });
    
    // 第 3 步：重新加载预算
    await dispatch(loadBudget({ id: budgetId }));
  },
);
```

**close-budget 收敛门实现**：`packages/loot-core/src/server/mutators.ts:63-71`

```typescript
if (name === 'close-budget') {
  await flushRunningMethods();  // 等待所有正在执行的异步方法完成
}

const promise = handler(args);
runningMethods.add(promise);  // 加入追踪集合
void promise.then(() => {
  runningMethods.delete(promise);  // 完成后移除
});
return promise as Promise<ReturnType<T>>;
```

**flushRunningMethods 细节**：`packages/loot-core/src/server/mutators.ts:23-35`

```typescript
async function flushRunningMethods() {
  await wait(200);  // 给客户端 200ms 发起新请求的窗口
  
  while (runningMethods.size > 0) {
    await Promise.all([...runningMethods.values()]);  // 等待所有运行中
    await wait(100);  // 再等 100ms 防止新请求
  }
}
```

**关键发现：不阻止重入**

- `flushRunningMethods` **只等待已在运行的方法完成**，但不阻止新的调用
- 如果用户快速点击两次，第一次 `backup-load` 还在进行中，第二次可以**绕过 close-budget**（因为 id 可能已被清空或不同）
- close-budget 的保护是**弱保护**，只对"关闭之前已经在运行的操作"有效

### 1.2 Loading Overlay 不阻止用户交互

**AppBackground 渲染**：`packages/desktop-client/src/components/AppBackground.tsx:18-54`

```typescript
export function AppBackground({ isLoading }: AppBackgroundProps) {
  const loadingText = useSelector(state => state.app.loadingText);
  const showLoading = isLoading || loadingText !== null;
  
  return (
    <>
      <Background />
      {showLoading &&
        transitions((style, item) => (
          <animated.div key={item} style={style}>
            <View
              className={css({
                position: 'absolute',
                top: 0,
                left: 0,
                right: 0,
                padding: 50,
                paddingTop: 200,
                // ❌ 没有 pointer-events: 'none'！！
                // ❌ 没有 z-index 控制！
              })}
            >
              <Block style={{ marginBottom: 20, fontSize: 18 }}>
                {loadingText}
              </Block>
              <AnimatedLoading width={25} color={theme.pageText} />
            </View>
          </animated.div>
        ))}
    </>
  );
}
```

**关键问题**：
1. ❌ Loading 遮罩层没有 `pointer-events: 'none'`，但也没有覆盖全屏可点击区域
2. ❌ Loading 遮罩层没有设置高 `z-index`，模态框可能显示在它上面
3. ❌ 用户在加载过程中仍然可以点击下面的 UI 元素

### 1.3 修正结论

✅ **close-budget 收敛门确实存在，但保护范围有限**
- ✅ 等待已运行的异步方法完成（200ms + 循环等待）
- ❌ **不阻止新的 backup-load 调用**（用户快速重复点击）
- ❌ Loading 遮罩没有真正阻止用户交互

**并发窗口准确描述**：
- 窗口 1：第一次 backup-load 执行期间，用户点击第二次 → 可绕过 close-budget（如果 id 匹配）
- 窗口 2：Modal 仍然打开，用户可以多次点击备份列表项
- 窗口 3：定时备份服务在解压期间触发（服务端并发）

---

## 二、界面防重入约束（backupDisabled）验证

### 2.1 代码证据链

**LoadBackupModal 接收参数**：`packages/desktop-client/src/components/modals/LoadBackupModal.tsx:52-55`

```typescript
export function LoadBackupModal({
  budgetId,
  watchUpdates,
  backupDisabled,  // ⚠️ 作为 props 传入
}: LoadBackupModalProps) {
```

**Modal 注册处硬编码为 false**：`packages/desktop-client/src/components/Modals.tsx:164-171`

```typescript
case 'load-backup':
  return (
    <LoadBackupModal
      key={key}
      watchUpdates
      {...modal.options}
      backupDisabled={false}  // ⚠️ 硬编码为 false！永远不禁用
    />
  );
```

**backupDisabled 实际使用位置**：`packages/desktop-client/src/components/modals/LoadBackupModal.tsx:141-147`

```typescript
<Button
  variant="primary"
  isDisabled={backupDisabled}  // 永远是 false
  onPress={() => dispatch(makeBackup())}
>
  <Trans>Back up now</Trans>
</Button>
```

**关键发现**：备份列表项没有禁用机制

```typescript
// LoadBackupModal.tsx:156-164
<BackupTable
  backups={previousBackups}
  onSelect={id => {
    if (budgetIdToLoad && id) {
      // ❌ 没有 isDisabled 属性！点击永远有效
      void dispatch(loadBackup({ budgetId: budgetIdToLoad, backupId: id }));
    }
  }}
/>
```

### 2.2 修正结论

❌ **backupDisabled 是死代码，没有实际防重入效果**

1. ❌ `backupDisabled` 在 Modal 注册时硬编码为 `false`，永远不会禁用按钮
2. ❌ 备份列表项（触发 `loadBackup` 的主要入口）**完全没有禁用机制**
3. ❌ 没有 loading 状态追踪，无法根据 thunk pending 状态禁用 UI

**对竞态概率的影响**：
- 普通用户：低概率（通常不会快速重复点击）
- 急躁用户：中概率（点击无反馈就再点一次）
- 自动化测试：高概率（快速连续操作）

---

## 三、latest 回退基线失效时机与恢复手段

### 3.1 代码证据链

**makeBackup 无条件删除 latest**：`packages/loot-core/src/server/budgetfiles/backups.ts:106-114`

```typescript
export async function makeBackup(id: string) {
  const budgetDir = fs.getBudgetDir(id);

  // ⚠️ 当创建备份时，我们不再认为用户在查看备份
  // ⚠️ 如果存在 latest 备份，就删除它，并认为当前是最新版本
  if (await fs.exists(fs.join(budgetDir, LATEST_BACKUP_FILENAME))) {
    // ❌ 无条件删除！不管解压是否正在进行中！
    await fs.removeFile(fs.join(fs.getBudgetDir(id), LATEST_BACKUP_FILENAME));
  }
  
  // ... 后续创建 ZIP 备份逻辑
}
```

**定时自动备份触发**：`packages/loot-core/src/server/budgetfiles/backups.ts:233-246`

```typescript
export function startBackupService(id: string) {
  if (serviceInterval) {
    clearInterval(serviceInterval);
  }

  // 每 15 分钟自动创建一次备份
  serviceInterval = setInterval(
    async () => {
      logger.log('Making backup');
      await makeBackup(id);  // ⚠️ 调用的就是上面会删除 latest 的函数
    },
    1000 * 60 * 15,  // 15 分钟
  );
}
```

**latest 创建时机**：`packages/loot-core/src/server/budgetfiles/backups.ts:164-183`

```typescript
// 只在文件不存在时才创建
if (!(await fs.exists(fs.join(budgetDir, LATEST_BACKUP_FILENAME)))) {
  await fs.copyFile(db.sqlite → db.latest.sqlite);
  await fs.copyFile(metadata.json → metadata.latest.json);
  startBackupService(id);  // ⚠️ 启动定时备份服务！
}
```

### 3.2 失效时序分析

**关键发现：定时备份服务在 latest 创建后立即启动**

```
用户点击加载备份
    ↓
[backup-load 开始执行]
    ├─ 创建 latest 基线 ✓
    ├─ 启动定时备份服务 (15分钟定时器) ⚠️
    ├─ 执行解压 (可能需要几秒到几十秒)
    └─ 如果定时器刚好在解压期间触发...
        └─ makeBackup() → 删除 latest ❌
            └─ 如果解压失败... 没有 latest 可以回退！🔴
```

### 3.3 失效触发点汇总

| 触发方式 | 是否删除 latest | 可否与解压并发 | 风险级别 |
|---------|---------------|---------------|---------|
| **用户手动创建备份** | ✅ 无条件删除 | ✅ 可以并发（Modal 不关闭） | 🟡 中 |
| **15 分钟定时自动备份** | ✅ 无条件删除 | ✅ 可以并发（后台静默执行） | 🔴 高 |
| **回退操作成功后** | ✅ 设计上删除 | ❌ 串行（同一调用链） | 无风险 |
| **连续加载备份第二次** | ❌ 不删除（文件存在跳过） | ✅ 可以并发 | 无影响 |

### 3.4 失效后的恢复手段

**情景 1：latest 被删除但解压成功**
- ✅ 数据已加载，无需恢复
- ⚠️ 但后续无法回退到"加载备份之前"的状态

**情景 2：latest 被删除且解压失败**
- ❌ **db.sqlite 可能已损坏**（部分写入）
- ❌ **db.latest.sqlite 已被删除**
- 🟡 **历史备份 ZIP 仍然存在**（在 backups/ 目录下）
- 🟡 **metadata.latest.json 可能还在**（makeBackup 只删 db，不删 metadata）

**用户可用的恢复手段**：
1. ✅ 从历史备份 ZIP 手动恢复（需要知道文件位置）
2. ✅ 如果有云端同步，从云端重新下载
3. ❌ **一键回退按钮失效**（UI 上"Revert to original version"不可用）

### 3.5 修正结论

🔴 **latest 回退基线保护存在设计缺陷：定时备份服务可能在解压期间删除 latest**

- ❌ makeBackup 无条件删除 latest，不考虑是否正在加载备份
- ❌ startBackupService 在 latest 创建后立即启动，定时器可能在解压期间触发
- ❌ 没有任何机制阻止解压和 makeBackup 并发执行
- ❌ 用户不知道解压期间不要手动创建备份

---

## 四、综合风险评估矩阵

| 风险场景 | 触发概率 | 影响程度 | 综合等级 | 备注 |
|---------|---------|---------|---------|-----|
| 用户快速重复点击备份加载 | 低 | 中 | 🟡 | 可能导致文件写入冲突 |
| 定时备份在解压期间触发并删除 latest | 极低 | 高 | 🟠 | 15 分钟窗口，刚好命中解压的概率低但后果严重 |
| 用户解压期间手动点击 "Back up now" | 低 | 高 | 🟠 | 模态框不关闭，按钮可用 |
| 解压失败但 latest 被删除 | 极低 | 最高 | 🔴 | 双重失败，只能依赖历史 ZIP |
| 云端上传失败但本地解压成功 | 中 | 低 | 🟢 | 不影响本地使用，仅影响同步 |

---

## 五、代码层面修复建议（按优先级排序）

### 🔴 优先级 1：latest 删除时机修复

```typescript
// 修改 makeBackup：只有当没有正在进行的 backup-load 时才删除 latest
export async function makeBackup(id: string) {
  const budgetDir = fs.getBudgetDir(id);
  
  // ✅ 增加：检查当前是否有 backup-load 正在运行（可以通过 lock 文件或状态变量）
  const isLoadInProgress = await fs.exists(fs.join(budgetDir, '.backup-load.lock'));
  
  if (!isLoadInProgress && 
      await fs.exists(fs.join(budgetDir, LATEST_BACKUP_FILENAME))) {
    await fs.removeFile(fs.join(budgetDir, LATEST_BACKUP_FILENAME));
  }
  
  // ... 后续逻辑
}
```

### 🟡 优先级 2：真正的防重入

```typescript
// 在 loadBackup thunk 中增加 pending 状态检查
export const loadBackup = createAppAsyncThunk(
  `${sliceName}/loadBackup`,
  async ({ budgetId, backupId }, { dispatch, getState }) => {
    // ✅ 增加：检查当前是否已有 loadBackup 在运行
    const state = getState();
    const isLoading = state.budgetfiles.isLoadingBackup;
    
    if (isLoading) {
      logger.log('Backup load already in progress, skipping duplicate');
      return;
    }
    
    // ... 后续逻辑
  },
);
```

### 🟡 优先级 3：loading overlay 阻止点击

```typescript
// AppBackground.tsx
<View
  className={css({
    position: 'absolute',
    top: 0,
    left: 0,
    right: 0,
    bottom: 0,  // ✅ 覆盖全屏
    padding: 50,
    paddingTop: 200,
    pointerEvents: 'auto',  // ✅ 拦截所有点击事件
    zIndex: 9999,           // ✅ 确保在最上层
  })}
>
```

---

## 六、总结

本次边界校准揭示了三个关键问题：

1. **close-budget 收敛门**：确实存在但保护有限，不阻止新调用
2. **backupDisabled 防重入**：完全无效，是死代码
3. **latest 基线保护**：存在定时备份并发删除的设计缺陷

**核心洞察**：备份恢复链路在正常情况下工作可靠，但在边缘并发场景下存在多个竞态窗口，尤其是定时备份服务与解压过程的并发，理论上可能导致数据丢失风险（尽管实际概率极低）。
