# Actual Budget 备份恢复机制深度分析

## 一、从前端按钮到后端处理的完整命令入口链路

### 1.1 完整调用链路总览

```
用户交互 (React Component)
      ↓
Redux Action Creator (createAsyncThunk)
      ↓
客户端 send() 调用 (platform/client/connection)
      ↓
IPC/WebWorker 消息传递
      ↓
服务端消息接收 (platform/server/connection)
      ↓
runHandler() 执行 (mutators.ts)
      ↓
业务处理函数 (budgetfiles/app.ts → backups.ts)
```

---

### 1.2 第一层：前端用户交互

**文件位置**：`packages/desktop-client/src/components/modals/LoadBackupModal.tsx:141-148`

```typescript
// 用户点击 "Back up now" 按钮触发
<Button
  variant="primary"
  isDisabled={backupDisabled}
  onPress={() => dispatch(makeBackup())}
>
  Back up now
</Button>
```

**文件位置**：`packages/desktop-client/src/components/modals/LoadBackupModal.tsx:156-166`

```typescript
// 用户选择备份列表项触发恢复
<BackupTable
  backups={previousBackups}
  onSelect={id => {
    if (budgetIdToLoad && id) {
      dispatch(loadBackup({ budgetId: budgetIdToLoad, backupId: id }));
    }
  }}
/>
```

---

### 1.3 第二层：Redux Action 层

**文件位置**：`packages/desktop-client/src/budgetfiles/budgetfilesSlice.ts:386-408`

#### makeBackup - 创建备份 Action
```typescript
export const makeBackup = createAppAsyncThunk(
  `${sliceName}/makeBackup`,
  async (_, { getState }) => {
    const prefs = getState().prefs.local;
    if (prefs && prefs.id) {
      // 关键点：调用客户端 send() 向服务端发送消息
      await send('backup-make', { id: prefs.id });
    }
  },
);
```

#### loadBackup - 加载备份 Action
```typescript
export const loadBackup = createAppAsyncThunk(
  `${sliceName}/loadBackup`,
  async ({ budgetId, backupId }, { dispatch, getState }) => {
    const prefs = getState().prefs.local;
    if (prefs && prefs.id) {
      await dispatch(closeBudget());  // 先关闭当前预算
    }
    // 关键点：发送 backup-load 消息到服务端
    await send('backup-load', { id: budgetId, backupId });
    // 重新加载预算（包含验证流程）
    await dispatch(loadBudget({ id: budgetId }));
  },
);
```

---

### 1.4 第三层：客户端连接层 (send 函数)

**文件位置**：`packages/loot-core/src/platform/client/connection/index.electron.ts:83-109`

```typescript
export const send: T.Send = function (
  ...params: Parameters<T.Send>
): ReturnType<T.Send> {
  const [name, args, { catchErrors = false } = {}] = params;
  return new Promise((resolve, reject) => {
    const id = uuidv4();  // 每个请求生成唯一ID
    replyHandlers.set(id, { resolve, reject });

    if (socketClient) {
      // 通过 IPC socket 发送消息到服务端
      socketClient.emit('message', {
        id,
        name,        // e.g. 'backup-make', 'backup-load'
        args,        // { id: prefs.id } 或 { id, backupId }
        undoTag: undo.snapshot(),
        catchErrors: !!catchErrors,
      });
    } else {
      // 如果连接未建立，加入消息队列稍后发送
      messageQueue.push({ id, name, args, undoTag, catchErrors });
    }
  });
};
```

**关键特性**：
- 异步 Promise-based 调用
- 唯一请求 ID 关联请求/响应
- 连接断开时自动排队重试
- 支持 undo 快照（用于撤销操作）

---

### 1.5 第四层：服务端连接与 Handler 注册

**文件位置**：`packages/loot-core/src/server/budgetfiles/app.ts:74-95`

```typescript
export const app = createApp<Handlers>();

// 注册备份相关的 handler 函数
app.method('backups-get', getBackups);        // 获取备份列表
app.method('backup-load', loadBackup);        // 加载备份
app.method('backup-make', makeBackup);        // 创建备份
```

**文件位置**：`packages/loot-core/src/server/main-app.ts:18-24`

```typescript
// 服务端内部 send 函数（直接调用 handler）
export async function send<K extends keyof Handlers>(
  name: K,
  args?: Parameters<Handlers[K]>[0],
): Promise<Awaited<ReturnType<Handlers[K]>>> {
  return runHandler(app.handlers[name], args, { name }) as Promise<
    Awaited<ReturnType<Handlers[K]>>
  >;
}
```

---

### 1.6 第五层：Handler 执行层 (runHandler)

**文件位置**：`packages/loot-core/src/server/mutators.ts:41-73`

```typescript
export async function runHandler<T extends Handlers[keyof Handlers]>(
  handler: T,
  args?: Parameters<T>[0],
  { undoTag, name }: { undoTag?; name? } = {},
): Promise<ReturnType<T>> {
  // 1. 追踪最近调用的 handler（用于调试）
  _latestHandlerNames.push(name);
  if (_latestHandlerNames.length > 5) {
    _latestHandlerNames = _latestHandlerNames.slice(-5);
  }

  // 2. 如果是 mutator 方法（需要事务保护），走 runMutator 路径
  if (mutatingMethods.has(handler)) {
    return runMutator(() => handler(args), { undoTag }) as Promise<
      ReturnType<T>
    >;
  }

  // 3. 特殊处理：关闭预算前等待所有异步方法完成
  if (name === 'close-budget') {
    await flushRunningMethods();
  }

  // 4. 正常执行 handler
  const promise = handler(args);
  runningMethods.add(promise);
  void promise.then(() => {
    runningMethods.delete(promise);
  });
  return promise as Promise<ReturnType<T>>;
}
```

**runMutator 机制**：`sequential(_runMutator)` 确保所有 mutator 调用串行执行，避免并发冲突。

---

### 1.7 第六层：实际业务处理

**文件位置**：`packages/loot-core/src/server/budgetfiles/app.ts:661-671`

```typescript
// 包装转发到实际的备份实现
async function getBackups({ id }) {
  return getAvailableBackups(id);
}

async function loadBackup({ id, backupId }) {
  await _loadBackup(id, backupId);  // 转发到 backups.ts
}

async function makeBackup({ id }) {
  await _makeBackup(id);            // 转发到 backups.ts
}
```

---

## 二、加载历史备份时云端上传与本地解压的先后顺序及影响

### 2.1 loadBackup 函数执行流程分析

**文件位置**：`packages/loot-core/src/server/budgetfiles/backups.ts:161-230`

```typescript
export async function loadBackup(id: string, backupId: string) {
  const budgetDir = fs.getBudgetDir(id);

  // ┌──────────────────────────────────────────────┐
  // │ 步骤 1: 检查并保存当前版本（首次加载时）       │
  // └──────────────────────────────────────────────┘
  if (!(await fs.exists(fs.join(budgetDir, LATEST_BACKUP_FILENAME)))) {
    // 保存当前数据库文件作为回退点
    await fs.copyFile(
      fs.join(budgetDir, 'db.sqlite'),
      fs.join(budgetDir, LATEST_BACKUP_FILENAME),  // db.latest.sqlite
    );
    // 保存当前元数据文件
    await fs.copyFile(
      fs.join(budgetDir, 'metadata.json'),
      fs.join(budgetDir, 'metadata.latest.json'),   // metadata.latest.json
    );

    stopBackupService();
    startBackupService(id);  // 重启备份计时器
    await prefs.loadPrefs(id);
  }

  // ┌──────────────────────────────────────────────┐
  // │ 步骤 2: 分支处理 - 回退 vs 加载历史备份       │
  // └──────────────────────────────────────────────┘
  if (backupId === LATEST_BACKUP_FILENAME) {
    // ┌────────────────────────────────────────────────┐
    // │ 分支 A: 回退到原始版本 (LATEST_BACKUP_FILENAME) │
    // └────────────────────────────────────────────────┘
    logger.log('Reverting backup');

    // 2A-1: 恢复数据库文件
    await fs.copyFile(
      fs.join(budgetDir, LATEST_BACKUP_FILENAME),
      fs.join(budgetDir, 'db.sqlite'),
    );
    // 2A-2: 恢复元数据文件
    await fs.copyFile(
      fs.join(budgetDir, 'metadata.latest.json'),
      fs.join(budgetDir, 'metadata.json'),
    );
    // 2A-3: 删除临时回退文件
    await fs.removeFile(fs.join(budgetDir, LATEST_BACKUP_FILENAME));
    await fs.removeFile(fs.join(budgetDir, 'metadata.latest.json'));

    // 2A-4: 重新上传到云端（注意：这是**文件恢复之后**才执行！）
    try {
      await cloudStorage.upload();  // ← 上传是异步但有 await
    } catch {}
    prefs.unloadPrefs();
  } else {
    // ┌────────────────────────────────────────────────┐
    // │ 分支 B: 加载历史备份 (ZIP 文件)                 │
    // └────────────────────────────────────────────────┘
    logger.log('Loading backup', backupId);

    // 2B-1: 先加载当前 prefs，重置同步状态
    await prefs.loadPrefs(id);
    await prefs.savePrefs({
      groupId: null,              // 清除同步组 ID
      lastSyncedTimestamp: null,  // 清除最后同步时间戳
      lastUploaded: null,         // 清除最后上传时间
    });

    // 2B-2: 先上传当前状态到云端（注意：解压之前！）
    try {
      await cloudStorage.upload();  // ← 关键：先上传再解压！
    } catch {}

    prefs.unloadPrefs();

    // 2B-3: 最后才解压备份文件覆盖本地
    const zip = new AdmZip(fs.join(budgetDir, 'backups', backupId));
    zip.extractEntryTo('db.sqlite', budgetDir, false, true);
    zip.extractEntryTo('metadata.json', budgetDir, false, true);
  }
}
```

---

### 2.2 关键执行顺序图解

#### 分支 A：回退到原始版本 (LATEST_BACKUP_FILENAME)

```
执行顺序：
  1. copyFile(db.latest.sqlite → db.sqlite)       ✓ 本地文件已覆盖
  2. copyFile(metadata.latest.json → metadata.json) ✓ 元数据已恢复
  3. removeFile(db.latest.sqlite)                  ✓ 临时文件已删除
  4. removeFile(metadata.latest.json)              ✓ 临时文件已删除
  5. await cloudStorage.upload()                  ⟳ 开始云端上传
     ├─ 导出当前 db.sqlite (已恢复的旧版本)
     ├─ 加密（如果启用）
     └─ 发送到同步服务器
```

#### 分支 B：加载历史备份 (ZIP 文件)

```
执行顺序（重点注意！）：
  1. prefs.loadPrefs(id)                          ✓ 加载当前元数据
  2. prefs.savePrefs({ groupId: null, ... })     ✓ 清除同步标记
  3. await cloudStorage.upload()                  ⟳ 先上传当前状态到云端！
     ├─ 导出当前 db.sqlite (加载备份前的状态)
     ├─ 加密（如果启用）
     └─ 发送到同步服务器
  4. prefs.unloadPrefs()                          ✓ 卸载 prefs
  5. zip.extractEntryTo(db.sqlite → budgetDir)   ✓ 最后才解压覆盖本地
  6. zip.extractEntryTo(metadata.json → budgetDir) ✓ 覆盖元数据
```

---

### 2.3 顺序设计的意图与影响分析

#### 设计意图（为什么先上传后解压？）

1. **云端备份保护**：
   - 确保在覆盖本地文件前，当前状态已上传到云端
   - 即使本地解压失败，云端至少保存了一份最新状态

2. **重置同步状态**：
   - `groupId: null` 清除同步组，意味着这个文件将脱离原同步组
   - 上传后，云端保存的是"备份加载前的最后状态"
   - 后续重新同步时，会基于备份内容创建新的同步组

3. **灾难恢复考量**：
   - 用户加载备份 → 发现备份更旧/有问题 → 可以从云端找回最新版本
   - 这形成了：本地备份列表 ↔ 云端最新版本 的双重保护

#### 潜在问题与影响

| 问题场景 | 后果 | 严重程度 |
|---------|------|---------|
| **上传成功但解压失败** | 云端已更新为旧状态标记，但本地解压失败 → 用户可能丢失数据 | ⚠️ 中高 |
| **上传超时但解压成功** | 云端未更新但本地已覆盖 → 本地和云端不一致 | ⚠️ 中 |
| **网络中断** | 上传部分失败但解压继续 → 状态不一致 | ⚠️ 中 |

#### 关键风险点代码证据

```typescript
// packages/loot-core/src/server/budgetfiles/backups.ts:220-223
try {
  await cloudStorage.upload();  // 可能失败，但不影响后续流程
} catch {}  // ← 静默吞掉所有错误！

// 无论上传成功与否，都会继续解压...
prefs.unloadPrefs();
const zip = new AdmZip(...);
zip.extractEntryTo('db.sqlite', budgetDir, false, true);  // 解压会执行
zip.extractEntryTo('metadata.json', budgetDir, false, true);
```

**问题分析**：上传失败被完全静默忽略，没有任何回滚或通知机制。用户以为加载了备份，但实际上云端可能没有更新。

---

## 三、解压或文件写入失败时的中间状态、现有保护和可观察后果

### 3.1 AdmZip 解压的失败场景分析

**关键代码**：`packages/loot-core/src/server/budgetfiles/backups.ts:227-229`

```typescript
const zip = new AdmZip(fs.join(budgetDir, 'backups', backupId));
zip.extractEntryTo('db.sqlite', budgetDir, false, true);
zip.extractEntryTo('metadata.json', budgetDir, false, true);
```

#### extractEntryTo 参数解析
```typescript
/**
 * @param entryName - 要提取的条目名
 * @param targetPath - 目标路径
 * @param maintainEntryPath - 是否保持条目原有路径结构
 * @param overwrite - 是否覆盖已存在文件
 */
extractEntryTo(entryName, targetPath, maintainEntryPath, overwrite)
```

#### AdmZip 库的失败模式（基于库行为分析）

1. **ZIP 文件损坏**：
   - `new AdmZip()` 构造函数可能直接抛出异常
   - 抛出后后续代码不执行 → **db.sqlite 和 metadata.json 都不会被覆盖**

2. **条目不存在**：
   - ZIP 文件中找不到 'db.sqlite' 或 'metadata.json'
   - `extractEntryTo` 会抛出异常
   - **如果第一个成功第二个失败 → 只有部分文件被覆盖！**（不一致状态）

3. **磁盘空间不足**：
   - 解压过程中写盘失败
   - 可能出现**部分写入**的损坏 db.sqlite 文件

4. **文件锁冲突**：
   - 其他进程（如 antivirus、indexer）正在读取 db.sqlite
   - 覆盖写入失败 → 抛出异常

---

### 3.2 Electron/Node.js fs 层的写入保护机制

**文件位置**：`packages/loot-core/src/platform/server/fs/index.electron.ts:129-164`

```typescript
export const writeFile: typeof T.writeFile = async (filepath, contents) => {
  try {
    // 使用 promise-retry 进行重试
    await promiseRetry(
      (retry, attempt) => {
        return new Promise((resolve, reject) => {
          fs.writeFile(filepath, contents, 'utf8', err => {
            if (err) {
              logger.error(
                `Failed to write to ${filepath}. Attempted ${attempt} times. ` +
                'Something is locking the file - potentially a virus scanner or backup software.',
              );
              reject(err);
            } else {
              if (attempt > 1) {
                logger.info(
                  `Successfully recovered from file lock. It took ${attempt} retries`,
                );
              }
              resolve(undefined);
            }
          });
        }).catch(retry);
      },
      {
        retries: 20,          // 最多重试 20 次
        minTimeout: 100,      // 首次重试等待 100ms
        maxTimeout: 500,      // 最长等待 500ms
        factor: 1.5,          // 指数退避因子
      },
    );

    return undefined;
  } catch (err) {
    logger.error(`Unable to recover from file lock on file ${filepath}`);
    throw err;  // 重试全部失败后重新抛出
  }
};
```

**保护机制总结**：
- ✅ **20 次重试**：应对临时文件锁
- ✅ **指数退避**：避免忙等
- ✅ **日志记录**：失败时记录详细信息
- ❌ **无原子写入**：失败后可能留下部分写入的文件

---

### 3.3 copyFile 的失败模式

**文件位置**：`packages/loot-core/src/platform/server/fs/index.electron.ts:94-105`

```typescript
export const copyFile: typeof T.copyFile = (frompath, topath) => {
  return new Promise<boolean>((resolve, reject) => {
    const readStream = fs.createReadStream(frompath);
    const writeStream = fs.createWriteStream(topath);

    readStream.on('error', reject);
    writeStream.on('error', reject);

    writeStream.on('open', () => readStream.pipe(writeStream));
    writeStream.once('close', () => resolve(true));
  });
};
```

**流式复制的失败点**：
1. 源文件不存在/不可读 → `readStream error` → 目标文件可能被创建但为空
2. 目标磁盘满 → `writeStream error` → 目标文件只有部分内容
3. 中途网络断开（如果是网络路径）→ 部分写入

---

### 3.4 失败时的中间状态矩阵

| 失败发生时机 | db.sqlite 状态 | metadata.json 状态 | LATEST_BACKUP 文件 | 对回退可靠性的影响 |
|-------------|----------------|-------------------|-------------------|-------------------|
| **任何写入前**（如 ZIP 构造失败） | 原始 | 原始 | 存在 ✓ | ✅ 安全，可通过 LATEST 回退 |
| **db.sqlite 写入中失败** | ❌ 损坏/部分写入 | 原始 | 存在 ✓ | ⚠️ 需手动恢复：删除 db.sqlite 后从 LATEST 复制 |
| **db.sqlite 成功但 metadata 失败** | 备份版本 | ❌ 原始/损坏 | 存在 ✓ | ⚠️ 不一致但可恢复 - 回退到 LATEST |
| **都成功但上传失败** | 备份版本 | 备份版本 | 存在 ✓ | ✅ 本地可用，但云端未更新 |
| **所有步骤都成功** | 备份版本 | 备份版本 | 存在 ✓ | ✅ 完整，可随时回退 |

---

### 3.5 现有保护机制评估

#### ✅ 已有的保护措施

1. **LATEST_BACKUP 机制**：
   - 首次加载备份时，先保存完整副本
   - 即使后续解压完全失败，至少原始版本还在
   - **代码证据**：`backups.ts:164-183` 复制后才执行后续操作

2. **写入重试机制**：
   - `writeFile` 有 20 次重试应对临时文件锁
   - 日志记录失败次数，便于排查

3. **预算加载验证**：
   - 解压后会重新 `loadBudget`，包含数据库完整性检查
   - 迁移版本验证、数据结构验证

#### ❌ 缺失的保护措施

1. **解压操作无原子性**：
   - 没有先解压到临时文件，验证后再替换
   - **风险**：解压中途失败留下损坏的 db.sqlite
   - **建议修复模式**：
     ```typescript
     // 当前实现（有风险）
     zip.extractEntryTo('db.sqlite', budgetDir, false, true);

     // 建议实现（安全）
     const tempDb = join(budgetDir, `db.backup.tmp.${Date.now()}`);
     zip.extractEntryTo('db.sqlite', tempDb, false, true);
     // 验证 db 文件完整性（SQLite header 检查、表结构检查）
     await verifyDatabase(tempDb);
     // 原子重命名替换
     await fs.rename(tempDb, join(budgetDir, 'db.sqlite'));
     ```

2. **无回滚机制**：
   - 如果第一个文件成功第二个失败，没有自动回滚第一个文件
   - 失败后用户可能处于"半备份"状态

3. **错误静默吞噬**：
   - `cloudStorage.upload()` 的错误被完全忽略
   - 用户不知道上传是否成功
   - **代码证据**：`backups.ts:221-223` 的空 catch

4. **缺少用户可见的错误反馈**：
   - 解压失败会抛出异常，但前端可能没有友好展示
   - 用户只看到"加载失败"但不知道原因和恢复方法

---

### 3.6 可观察后果与用户影响

#### 场景 1：ZIP 损坏无法读取
```
用户体验：
  - 点击备份 → 转圈 → 报错
  - 实际文件：原始 db.sqlite 和 metadata.json 完全未动
  - LATEST_BACKUP：已创建（因为步骤 1 在解压前）

恢复方式：
  - 用户可以正常操作，不受影响
  - 可以选择其他备份文件
```

#### 场景 2：db.sqlite 解压成功但 metadata 失败
```
用户体验：
  - 点击备份 → 转圈 → 报错
  - 实际状态：db.sqlite 已是备份版本，但 metadata 还是旧的
  - 后果：预算 ID、加密密钥、同步状态不匹配

恢复方式：
  - 系统提供的 "Revert to original" 按钮可用
  - 因为 LATEST_BACKUP 文件已保存，点击回退即可完全恢复
```

#### 场景 3：解压成功但数据库损坏（坏备份）
```
用户体验：
  - 备份加载成功 → 但打开预算时报错
  - 可能看到 "Database is corrupted" 或迁移失败

恢复方式：
  - LATEST_BACKUP 存在，可以点击回退
  - 但用户可能不知道可以回退，以为数据丢了
```

#### 场景 4：云端上传失败但本地解压成功
```
用户体验：
  - 备份加载看起来完全成功 ✓
  - 但实际上云端没有保存这个状态

潜在问题：
  - 用户以为数据已安全同步
  - 其他设备没有收到这个备份版本的更新
  - 只有当用户手动触发同步时才会发现问题
```

---

## 四、对回退可靠性的综合影响

### 4.1 现有设计的优点

1. **先保存后操作**的模式很关键：
   - LATEST_BACKUP 文件在任何破坏性操作前就已创建
   - 这是整个系统最可靠的安全网

2. **双重备份层级**：
   - Level 1: LATEST_BACKUP（最近一次加载备份前的状态）
   - Level 2: 历史备份 ZIP 列表（更早期的版本）

3. **数据库级别的验证**：
   - 加载预算时会执行迁移和完整性检查
   - 坏的备份不会静默通过

### 4.2 需要改进的关键点

| 改进点 | 当前风险 | 建议方案 |
|-------|---------|---------|
| **解压原子性** | 部分写入导致损坏 | 先解压到临时文件，验证后重命名替换 |
| **上传错误静默** | 同步状态不一致 | 至少记录 error log，考虑前端提示 |
| **部分失败回滚** | db/metadata 版本不匹配 | 第二个文件失败时，恢复第一个文件 |
| **用户指导** | 用户不知道如何恢复 | 失败时明确提示："可点击 Revert 恢复" |

### 4.3 代码层面的具体改进建议

```typescript
// 修改后的 loadBackup 分支 B 安全版本
} else {
  logger.log('Loading backup', backupId);

  await prefs.loadPrefs(id);
  await prefs.savePrefs({ groupId: null, lastSyncedTimestamp: null, lastUploaded: null });

  // 先上传
  let uploadSuccess = true;
  try {
    await cloudStorage.upload();
  } catch (e) {
    logger.error('Failed to upload before loading backup', e);
    uploadSuccess = false;
    // 不抛出，继续，但记录失败状态
  }

  prefs.unloadPrefs();

  // 安全解压
  const zip = new AdmZip(fs.join(budgetDir, 'backups', backupId));
  const tempDb = fs.join(budgetDir, `db.temp.${Date.now()}`);
  const tempMeta = fs.join(budgetDir, `metadata.temp.${Date.now()}`);

  try {
    // 1. 先解压到临时文件
    zip.extractEntryTo('db.sqlite', tempDb, false, true);
    zip.extractEntryTo('metadata.json', tempMeta, false, true);

    // 2. 验证文件完整性
    await verifyDatabase(tempDb);
    await verifyMetadata(tempMeta);

    // 3. 原子替换
    await fs.rename(tempDb, fs.join(budgetDir, 'db.sqlite'));
    await fs.rename(tempMeta, fs.join(budgetDir, 'metadata.json'));

  } catch (e) {
    // 失败时清理临时文件
    await fs.removeFile(tempDb).catch(() => {});
    await fs.removeFile(tempMeta).catch(() => {});
    throw e;  // 重新抛出，让上层处理
  }

  if (!uploadSuccess) {
    // 通知用户虽然加载成功但同步未更新
    connection.send('backup-warning', { message: '云端同步失败，仅本地可用' });
  }
}
```

---

## 五、总结

### 5.1 命令入口链路关键点

```
前端按钮 → Redux thunk → send() IPC → runHandler() → 业务逻辑
     ✅ 类型安全      ✅ Promise-based    ✅ 串行执行
```

### 5.2 上传/解压顺序的设计权衡

- **设计意图**：确保灾难性场景下云端有保底
- **代价**：增加了"上传成功但解压失败"的不一致窗口
- **整体评价**：合理的权衡，优先保证数据不丢失

### 5.3 失败时的回退可靠性评级

| 失败场景 | 回退可用性 | 数据丢失风险 | 评级 |
|---------|-----------|-------------|-----|
| ZIP 文件完全损坏 | ✅ 可用 | 无 | A |
| db.sqlite 部分写入 | ⚠️ 需手动 | 低 | B |
| 上传失败但解压成功 | ✅ 本地可用 | 同步延迟 | B+ |
| 备份文件本身已损坏 | ✅ 可回退 | 无 | A |

**总体评级**：B+ 级。核心设计可靠，但边缘情况的原子性和错误处理有改进空间。
