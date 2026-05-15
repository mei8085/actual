# Actual Budget 备份恢复机制关键验证报告

## 验证总结

| 验证项 | 之前假设 | 实际结果 | 对回退可靠性的影响 |
|--------|---------|---------|-------------------|
| **1. mutator 串行保护** | 可能有保护 | ❌ 无保护 | 并发写入风险 |
| **2. 解压重试保护** | 可能有重试 | ❌ 无重试 | 文件锁导致失败不可恢复 |
| **3. latest 基线刷新** | 可能每次刷新 | ❌ 仅首次创建 | 连续加载后无法回退到中间状态 |

---

## 一、backup-load 与 backup-make 的 mutator 串行保护验证

### 1.1 代码证据

**注册位置**：`packages/loot-core/src/server/budgetfiles/app.ts:93-94`

```typescript
// 第 79 行：reset-budget-cache 使用了 mutator 包装
app.method('reset-budget-cache', mutator(resetBudgetCache));

// 第 93-94 行：backup-load 和 backup-make 没有使用 mutator 包装！
app.method('backup-load', loadBackup);  // ❌ 无 mutator 保护
app.method('backup-make', makeBackup);  // ❌ 无 mutator 保护
```

**mutator 注册机制**：`packages/loot-core/src/server/mutators.ts:14-17`

```typescript
export function mutator<T extends HandlerFunctions>(handler: T): T {
  mutatingMethods.set(handler, true);  // 仅注册的 handler 才会进入串行队列
  return handler;
}
```

**runHandler 执行路径判断**：`packages/loot-core/src/server/mutators.ts:53-57`

```typescript
if (mutatingMethods.has(handler)) {
  // ✅ 经过 sequential() 包装，串行执行
  return runMutator(() => handler(args), { undoTag }) as Promise<ReturnType<T>>;
}

// ❌ 直接执行，没有任何串行保护
const promise = handler(args);
```

**sequential() 实现**：`packages/loot-core/src/shared/async.ts:4-60`

```typescript
export function sequential<T extends AnyFunction>(fn: T) {
  const sequenceState = {
    running: null,  // 当前正在执行的 Promise
    queue: [],      // 等待队列
  };

  // 关键：如果正在运行，就放入队列等待
  if (!sequenceState.running) {
    run(args, resolve, reject);  // 立即执行
  } else {
    sequenceState.queue.push({ resolve, reject, args });  // 排队等待
  }
}
```

### 1.2 纠正后的结论

❌ **backup-load 和 backup-make 不具备 mutator 串行保护**

- 这两个 handler 没有被 `mutator()` 函数包装
- 它们不在 `mutatingMethods` WeakMap 中
- `runHandler` 会直接调用它们，**不会经过 sequential 队列**

### 1.3 真实并发边界

| 操作组合 | 并发行为 | 风险 |
|---------|---------|-----|
| **backup-make + backup-make** | 可以并行执行 | 两个进程同时写 ZIP 文件可能损坏 |
| **backup-load + backup-load** | 可以并行执行 | 同时覆盖 db.sqlite 可能导致文件损坏 |
| **backup-make + backup-load** | 可以并行执行 | 一个读 ZIP 一个写 ZIP，极端竞态 |
| **backup-* + 其他 mutator** | 可以并行执行 | backup 操作与正常的 CRUD 操作冲突 |

### 1.4 对回退可靠性的影响

⚠️ **中等风险**

- **竞态条件**：如果用户快速连续点击"加载备份"，可能触发并发解压
- **文件损坏**：同时覆盖 db.sqlite 可能导致文件损坏
- **latest 基线竞态**：虽然 latest 只创建一次，但并发的 copyFile 仍可能冲突
- **实际概率**：前端通常有 UI 禁用机制，实际生产中概率较低，但代码层面存在隐患

---

## 二、历史备份解压路径的重试保护验证

### 2.1 代码证据

**解压调用位置**：`packages/loot-core/src/server/budgetfiles/backups.ts:227-229`

```typescript
const zip = new AdmZip(fs.join(budgetDir, 'backups', backupId));
zip.extractEntryTo('db.sqlite', budgetDir, false, true);       // ❌ 直接调用 AdmZip
zip.extractEntryTo('metadata.json', budgetDir, false, true);    // ❌ 直接调用 AdmZip
```

**AdmZip extractEntryTo 行为**（基于 Node.js 库实现）：
- 直接调用 Node.js 原生 `fs.createWriteStream()`
- **没有**任何重试逻辑
- 遇到 `EBUSY`（文件被占用）或 `EACCES`（权限问题）直接抛出异常

**Actual 带重试的 writeFile（对比参考）**：`packages/loot-core/src/platform/server/fs/index.electron.ts:129-164`

```typescript
export const writeFile = async (filepath, contents) => {
  try {
    await promiseRetry(
      (retry, attempt) => {
        return new Promise((resolve, reject) => {
          fs.writeFile(filepath, contents, 'utf8', err => {
            if (err) {
              logger.error(`Failed to write... Attempt ${attempt}/20`);
              reject(err);  // 重试
            } else {
              resolve(undefined);
            }
          });
        }).catch(retry);  // ✅ 最多重试 20 次
      },
      { retries: 20, minTimeout: 100, maxTimeout: 500, factor: 1.5 },
    );
  } catch (err) {
    logger.error(`Unable to recover after 20 retries`);
    throw err;
  }
};
```

### 2.2 纠正后的结论

❌ **解压路径没有经过带重试的写文件封装**

- `extractEntryTo` 是 AdmZip 库的原生方法，绕过了 Actual 的 FS 抽象层
- 遇到文件锁（如 Antivirus 扫描、索引器）直接失败，**不会重试**
- 与之前的假设一致，但**实际失败模型比预期更严重**

### 2.3 实际失败模型与假设的差异

| 失败场景 | 之前假设 | 实际行为 | 严重程度 |
|---------|---------|---------|---------|
| **文件锁 (EBUSY)** | 重试 20 次后失败 | ❌ 直接失败，零重试 | 🔴 高 |
| **磁盘满 (ENOSPC)** | 重试几次后失败 | ❌ 直接失败 | 🟡 中 |
| **权限错误 (EACCES)** | 无法重试 | ❌ 直接失败 | 🟡 中 |
| **ZIP 条目损坏** | 直接失败 | ❌ 直接失败 | 🟡 中 |

### 2.4 对回退可靠性的影响

⚠️ **中高风险**

- **Windows 平台尤为严重**：AntiVirus、Windows Search 经常会临时锁定 SQLite 文件
- **没有重试意味着**：只要有任何进程持有文件句柄，备份加载就会失败
- **用户体验**：用户看到"加载失败"，但没有任何提示告诉他们重试
- **庆幸点**：latest 基线在解压前就已创建，即使解压失败，回退路径仍然存在

---

## 三、latest 回退基线刷新机制验证

### 3.1 代码证据

**创建条件**：`packages/loot-core/src/server/budgetfiles/backups.ts:164-183`

```typescript
export async function loadBackup(id: string, backupId: string) {
  const budgetDir = fs.getBudgetDir(id);

  // ⚠️ 关键判断：只有文件不存在时才创建！
  if (!(await fs.exists(fs.join(budgetDir, LATEST_BACKUP_FILENAME)))) {
    // 如果这是第一次加载备份，保存当前版本以便回退
    await fs.copyFile(
      fs.join(budgetDir, 'db.sqlite'),
      fs.join(budgetDir, LATEST_BACKUP_FILENAME),  // db.latest.sqlite
    );
    await fs.copyFile(
      fs.join(budgetDir, 'metadata.json'),
      fs.join(budgetDir, 'metadata.latest.json'),   // metadata.latest.json
    );

    stopBackupService();
    startBackupService(id);
    await prefs.loadPrefs(id);
  }

  // 后续解压逻辑...
}
```

**makeBackup 时的删除行为**：`packages/loot-core/src/server/budgetfiles/backups.ts:109-114`

```typescript
export async function makeBackup(id: string) {
  const budgetDir = fs.getBudgetDir(id);

  // ⚠️ 创建手动备份时，会删除 latest 基线！
  if (await fs.exists(fs.join(budgetDir, LATEST_BACKUP_FILENAME))) {
    await fs.removeFile(fs.join(fs.getBudgetDir(id), LATEST_BACKUP_FILENAME));
  }

  // 后续创建 ZIP 逻辑...
}
```

**回退时的删除行为**：`packages/loot-core/src/server/budgetfiles/backups.ts:198-199`

```typescript
if (backupId === LATEST_BACKUP_FILENAME) {
  // 回退到 latest 后，删除临时文件
  await fs.removeFile(fs.join(budgetDir, LATEST_BACKUP_FILENAME));
  await fs.removeFile(fs.join(budgetDir, 'metadata.latest.json'));
}
```

### 3.2 纠正后的结论

❌ **latest 回退基线仅在第一次加载备份时创建，连续加载不会刷新**

- 条件判断 `if (!(await fs.exists(...)))` 确保**文件存在就不会覆盖**
- 连续多次加载备份，latest 始终指向**第一次加载备份之前**的状态
- 第二次及以后的加载，没有任何中间状态的保护

### 3.3 完整的 latest 生命周期状态机

```
状态 1: latest 不存在 (初始状态)
    ↓ 用户加载备份 A (第一次)
状态 2: latest = 加载 A 之前的状态 ✓ (已创建)
    ↓ 用户加载备份 B (第二次)
状态 2: latest 不变 ❌ (仍然是加载 A 之前的状态，不是 A 的状态)
    ↓ 用户点击 "Revert" (回退)
状态 1: latest 被删除 ✓ (回退到了最原始的状态)
    ↓ 用户点击 "Back up now" (手动备份)
状态 1: latest 如果存在会被删除 ✓ (makeBackup 主动清理)
    ↓ 用户再次加载备份 C
状态 2: latest = 加载 C 之前的状态 ✓ (重新创建)
```

### 3.4 回退按钮最终指向的时点

| 用户操作序列 | latest 指向的时点 | 是否符合预期 |
|-------------|-----------------|------------|
| 正常使用 → 加载备份 A → 回退 | 正常使用的状态 | ✅ 符合预期 |
| 正常使用 → 加载备份 A → 加载备份 B → 回退 | **正常使用的状态** ❌ | 不符合：用户以为能回退到 A，但实际回退到了原始状态 |
| 正常使用 → 加载备份 A → 手动备份 → 加载备份 B → 回退 | **手动备份后的状态** ✅ | makeBackup 会删除 latest，下一次加载会重新创建 |
| 正常使用 → 加载备份 A → 回退 → 加载备份 B → 回退 | 每次都是当时的当前状态 | ✅ 符合预期 |

### 3.5 对回退可靠性的影响

🔴 **高风险（设计意图 vs 用户预期不一致）**

- **用户预期**：每次加载备份前都应该保存当前状态，回退能回到上一步
- **实际行为**：只有第一次加载会保存，后续加载没有保护
- **陷阱场景**：用户加载备份 A → 发现不好 → 加载备份 B → 发现更糟 → 想回退到 A
  - ❌ **无法做到**：回退按钮会直接回到最原始的状态，A 这个中间状态丢失了！
- **数据丢失**：如果用户在备份 A 中做了修改，加载 B 后这些修改将无法恢复

### 3.6 设计意图推测（为什么这么设计？）

可能的设计考量：
1. **简化实现**：不需要管理多个回退点
2. **避免磁盘膨胀**：只保留一个 latest 文件
3. **"回退"语义**：设计者认为"回退"应该回到加载任何备份之前的状态

但这个设计与直觉严重不符，大部分用户都会预期"撤销上一步"而不是"撤销所有"。

---

## 四、综合建议

### 4.1 立即修复建议

**建议 1：为 backup-load 和 backup-make 添加 mutator 保护**

```typescript
// packages/loot-core/src/server/budgetfiles/app.ts
app.method('backup-load', mutator(loadBackup));    // +mutator
app.method('backup-make', mutator(makeBackup));    // +mutator
```

**建议 2：解压前先验证 latest 存在性，解压失败给出明确提示**

```typescript
// 在 loadBackup 开头添加
const hasLatest = await fs.exists(fs.join(budgetDir, LATEST_BACKUP_FILENAME));

// 解压失败捕获
try {
  zip.extractEntryTo('db.sqlite', budgetDir, false, true);
} catch (e) {
  if (e.code === 'EBUSY') {
    logger.error('文件被锁定，建议关闭 Antivirus 后重试');
  }
  throw e;
}
```

### 4.2 中长期改进建议

**建议 3：改变 latest 基线刷新策略**

- 选项 A：每次加载备份都刷新 latest（简单，符合直觉）
- 选项 B：维护一个回退栈（最多 3-5 个历史回退点）
- 选项 C：在 UI 上明确告知用户"回退将回到第一次加载备份前的状态"

**建议 4：为解压添加临时文件 + 原子重命名**

```typescript
const tempDb = fs.join(budgetDir, `db.restore.tmp.${Date.now()}`);
zip.extractEntryTo('db.sqlite', tempDb, false, true);
// 验证文件完整性
await fs.rename(tempDb, fs.join(budgetDir, 'db.sqlite'));  // 原子替换
```

---

## 五、最终回退可靠性评分

| 评估维度 | 评分 (1-10) | 说明 |
|---------|------------|-----|
| **latest 基线创建时机** | 6/10 | 至少会创建一次，但不会刷新 |
| **并发保护** | 4/10 | 无 mutator 保护，依赖前端 UI 禁用 |
| **失败原子性** | 5/10 | latest 在解压前创建，但解压本身非原子 |
| **重试机制** | 2/10 | 解压路径无重试，Windows 下容易失败 |
| **用户预期匹配度** | 3/10 | 连续加载后的回退行为违反直觉 |

**总体回退可靠性评分：4/10**

核心问题总结：
1. ✅ latest 基线机制本身是有效的（至少有一个安全网）
2. ❌ 并发保护缺失（理论风险，实际概率较低）
3. ❌ 解压重试缺失（Windows 下实际问题）
4. ❌ 连续加载无中间状态保护（严重的设计与预期不一致）
