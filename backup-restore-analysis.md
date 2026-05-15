# Actual Budget 备份与恢复链路分析

## 一、概述

Actual Budget 的备份与恢复系统由三个核心层协作完成：
- **本地存储层**：负责备份文件的创建、存储和管理
- **同步层（CRDT/Merkle）**：保证多设备数据一致性
- **命令入口层**：提供用户交互和系统触发接口

---

## 二、用户触发备份的入口

### 2.1 用户界面入口

**文件位置**：`packages/desktop-client/src/components/modals/LoadBackupModal.tsx`

#### 手动触发备份
```typescript
// 用户点击 "Back up now" 按钮触发
<Button variant="primary" onPress={() => dispatch(makeBackup())}>
  Back up now
</Button>
```

#### 加载历史备份
```typescript
// 用户选择备份列表中的某一项
<BackupTable
  backups={previousBackups}
  onSelect={id => {
    dispatch(loadBackup({ budgetId: budgetIdToLoad, backupId: id }));
  }}
/>
```

#### 回退到原始版本
```typescript
// 用户处于备份查看模式时，可以回退到加载备份前的版本
<Button variant="primary" onPress={() => {
  dispatch(loadBackup({ budgetId: budgetIdToLoad, backupId: LATEST_BACKUP_FILENAME }));
}}>
  Revert to original version
</Button>
```

### 2.2 Redux Action 层

**文件位置**：`packages/desktop-client/src/budgetfiles/budgetfilesSlice.ts`

#### `makeBackup` - 创建备份
```typescript
export const makeBackup = createAppAsyncThunk(
  `${sliceName}/makeBackup`,
  async (_, { getState }) => {
    const prefs = getState().prefs.local;
    if (prefs && prefs.id) {
      await send('backup-make', { id: prefs.id });
    }
  },
);
```

#### `loadBackup` - 加载备份
```typescript
export const loadBackup = createAppAsyncThunk(
  `${sliceName}/loadBackup`,
  async ({ budgetId, backupId }, { dispatch, getState }) => {
    const prefs = getState().prefs.local;
    if (prefs && prefs.id) {
      await dispatch(closeBudget());
    }
    await send('backup-load', { id: budgetId, backupId });
    await dispatch(loadBudget({ id: budgetId }));
  },
);
```

### 2.3 自动备份服务

**文件位置**：`packages/loot-core/src/server/budgetfiles/backups.ts`

```typescript
export function startBackupService(id: string) {
  // 每 15 分钟自动创建一次备份
  serviceInterval = setInterval(
    async () => {
      logger.log('Making backup');
      await makeBackup(id);
    },
    1000 * 60 * 15,  // 15 分钟
  );
}
```

**触发时机**：预算加载时自动启动
```typescript
// packages/loot-core/src/server/budgetfiles/app.ts:588-590
if (!Platform.isBrowser && process.env.NODE_ENV !== 'test') {
  startBackupService(id);
}
```

---

## 三、备份内容的组装范围

### 3.1 备份创建流程

**文件位置**：`packages/loot-core/src/server/budgetfiles/backups.ts`

```typescript
export async function makeBackup(id: string) {
  const budgetDir = fs.getBudgetDir(id);
  
  // 1. 删除之前的 "latest" 备份（如果存在）
  if (await fs.exists(fs.join(budgetDir, LATEST_BACKUP_FILENAME))) {
    await fs.removeFile(fs.join(budgetDir, LATEST_BACKUP_FILENAME));
  }

  // 2. 生成备份文件名：YYYY-MM-DD_HH-mm-ss.zip
  const backupId = `${dateFns.format(new Date(), 'yyyy-MM-dd_HH-mm-ss')}.zip`;
  const backupPath = fs.join(budgetDir, 'backups', backupId);

  // 3. 创建 backups 目录（如果不存在）
  if (!(await fs.exists(fs.join(budgetDir, 'backups')))) {
    await fs.mkdir(fs.join(budgetDir, 'backups'));
  }

  // 4. 临时复制数据库文件以便清理 CRDT 消息
  const tempDbPath = fs.join(budgetDir, 'backups', `db.${Date.now()}.sqlite.tmp`);
  await fs.copyFile(fs.join(budgetDir, 'db.sqlite'), tempDbPath);

  let db: Database | undefined;
  try {
    // 5. 从备份中清除同步消息（减少体积，保证一致性）
    db = await sqlite.openDatabase(tempDbPath);
    sqlite.runQuery(db, 'DELETE FROM messages_crdt');
    sqlite.runQuery(db, 'DELETE FROM messages_clock');

    // 6. 打包成 ZIP 文件
    const zip = new AdmZip();
    zip.addLocalFile(tempDbPath, '', 'db.sqlite');       // 清理后的数据库
    zip.addLocalFile(fs.join(budgetDir, 'metadata.json')); // 元数据文件
    zip.writeZip(backupPath);
  } finally {
    // 7. 清理临时文件
    if (db) sqlite.closeDatabase(db);
    if (await fs.exists(tempDbPath)) await fs.removeFile(tempDbPath);
  }

  // 8. 备份保留策略：只保留最新的 10 个备份
  const toRemove = await updateBackups(await getBackups(id));
  for (const id of toRemove) {
    await fs.removeFile(fs.join(budgetDir, 'backups', id));
  }

  // 9. 通知前端备份列表更新
  connection.send('backups-updated', await getAvailableBackups(id));
}
```

### 3.2 备份文件内容

备份是一个 ZIP 压缩包，包含以下两个文件：

| 文件名 | 内容说明 | 处理方式 |
|--------|----------|----------|
| `db.sqlite` | SQLite 数据库文件 | **已清理**：删除 `messages_crdt` 和 `messages_clock` 表 |
| `metadata.json` | 预算元数据 | **原样保留** |

### 3.3 备份保留策略

**文件位置**：`packages/loot-core/src/server/budgetfiles/backups.ts:81-104`

```typescript
export async function updateBackups(backups) {
  const byDay = backups.reduce((groups, backup) => {
    const day = dateFns.format(backup.date, 'yyyy-MM-dd');
    groups[day] = groups[day] || [];
    groups[day].push(backup);
    return groups;
  }, {});

  const removed = [];
  for (const day of Object.keys(byDay)) {
    const dayBackups = byDay[day];
    const isToday = day === monthUtils.currentDay();
    // 当天保留 3 个备份，其他天保留 1 个
    for (const backup of dayBackups.slice(isToday ? 3 : 1)) {
      removed.push(backup.id);
    }
  }

  // 最终只保留最新的 10 个备份
  const currentBackups = backups.filter(backup => !removed.includes(backup.id));
  return removed.concat(currentBackups.slice(10).map(backup => backup.id));
}
```

**保留策略总结**：
- 当天备份：最多保留 3 个
- 历史备份：每天保留 1 个
- 总数限制：最多保留 10 个备份

---

## 四、恢复时的验证和回填逻辑

### 4.1 加载备份流程

**文件位置**：`packages/loot-core/src/server/budgetfiles/backups.ts:161-231`

```typescript
export async function loadBackup(id: string, backupId: string) {
  const budgetDir = fs.getBudgetDir(id);

  // 1. 首次加载备份时：保存当前版本作为回退点
  if (!(await fs.exists(fs.join(budgetDir, LATEST_BACKUP_FILENAME)))) {
    // 保存当前数据库
    await fs.copyFile(
      fs.join(budgetDir, 'db.sqlite'),
      fs.join(budgetDir, LATEST_BACKUP_FILENAME),
    );
    // 保存当前元数据
    await fs.copyFile(
      fs.join(budgetDir, 'metadata.json'),
      fs.join(budgetDir, 'metadata.latest.json'),
    );

    // 重启备份服务计时器
    stopBackupService();
    startBackupService(id);
    await prefs.loadPrefs(id);
  }

  // 2. 判断是回退还是加载历史备份
  if (backupId === LATEST_BACKUP_FILENAME) {
    // 2a. 回退到原始版本
    logger.log('Reverting backup');
    
    // 恢复数据库文件
    await fs.copyFile(
      fs.join(budgetDir, LATEST_BACKUP_FILENAME),
      fs.join(budgetDir, 'db.sqlite'),
    );
    // 恢复元数据文件
    await fs.copyFile(
      fs.join(budgetDir, 'metadata.latest.json'),
      fs.join(budgetDir, 'metadata.json'),
    );
    
    // 删除临时回退文件
    await fs.removeFile(fs.join(budgetDir, LATEST_BACKUP_FILENAME));
    await fs.removeFile(fs.join(budgetDir, 'metadata.latest.json'));

    // 重新上传到云端
    try {
      await cloudStorage.upload();
    } catch {}
    prefs.unloadPrefs();
  } else {
    // 2b. 加载历史备份
    logger.log('Loading backup', backupId);

    // 清除同步状态（重要！）
    await prefs.loadPrefs(id);
    await prefs.savePrefs({
      groupId: null,              // 清除同步组 ID
      lastSyncedTimestamp: null,  // 清除最后同步时间戳
      lastUploaded: null,         // 清除最后上传时间
    });

    // 重新上传到云端
    try {
      await cloudStorage.upload();
    } catch {}

    prefs.unloadPrefs();

    // 解压备份文件覆盖当前文件
    const zip = new AdmZip(fs.join(budgetDir, 'backups', backupId));
    zip.extractEntryTo('db.sqlite', budgetDir, false, true);
    zip.extractEntryTo('metadata.json', budgetDir, false, true);
  }
}
```

### 4.2 恢复后的预算加载流程

**文件位置**：`packages/loot-core/src/server/budgetfiles/app.ts:508-640`

恢复备份后，系统会重新加载预算，执行以下验证步骤：

```typescript
async function _loadBudget(id: Budget['id']) {
  // 1. 验证目录存在性
  const dir = fs.getBudgetDir(id);
  if (!(await fs.exists(dir))) {
    return { error: 'budget-not-found' };
  }

  // 2. 加载元数据和数据库
  await prefs.loadPrefs(id);
  await db.openDatabase(id);

  // 3. 执行数据版本迁移验证
  try {
    await updateVersion();
  } catch (e) {
    if (e.message.includes('out-of-sync-migrations')) {
      return { error: 'out-of-sync-migrations' };
    } else if (e.message.includes('out-of-sync-data')) {
      return { error: 'out-of-sync-data' };
    }
    return { error: 'loading-budget' };
  }

  // 4. 加载 CRDT 时钟
  await db.loadClock();

  // 5. 如果需要重置时钟（从云端下载的文件）
  if (prefs.getPrefs().resetClock) {
    // 生成新的客户端 ID
    CRDT.getClock().timestamp.setNode(CRDT.makeClientId());
    db.runQuery(
      'INSERT OR REPLACE INTO messages_clock (id, clock) VALUES (1, ?)',
      [CRDT.serializeClock(CRDT.getClock())],
    );
    await prefs.savePrefs({ resetClock: false });
  }

  // 6. 启动备份服务
  if (!Platform.isBrowser && process.env.NODE_ENV !== 'test') {
    startBackupService(id);
  }

  // 7. 加载电子表格和业务逻辑
  await sheet.loadSpreadsheet(db, onSheetChange);
  await budget.createAllBudgets();
  await mappings.loadMappings();
  await rules.loadRules();

  // 8. 设置同步模式
  if (id === DEMO_BUDGET_ID) {
    setSyncingMode('disabled');
  } else {
    setSyncingMode(getServer() ? 'enabled' : 'disabled');
  }

  // 9. 可能触发云端上传
  await cloudStorage.possiblyUpload();
}
```

---

## 五、元数据、文件指纹与增量片段详解

### 5.1 元数据结构

**文件位置**：`packages/loot-core/src/server/prefs.ts`

```typescript
// metadata.json 包含以下关键字段
type MetadataPrefs = {
  id: string;                    // 预算唯一 ID
  budgetName: string;            // 预算名称
  cloudFileId?: string;          // 云端文件 ID
  groupId?: string;              // 同步组 ID（多设备同步）
  lastUploaded?: string;         // 最后上传日期（YYYY-MM-DD）
  lastSyncedTimestamp?: string;  // 最后同步时间戳（CRDT 格式）
  encryptKeyId?: string;         // 加密密钥 ID
  resetClock?: boolean;          // 是否需要重置时钟标记
  userId?: string;               // 用户 ID
  budgetType?: BudgetType;       // 预算类型：envelope | tracking
};
```

### 5.2 Merkle 树（文件指纹）

**文件位置**：`packages/crdt/src/crdt/merkle.ts`

Merkle 树用于高效检测多设备间的数据差异：

#### 核心结构
```typescript
export type TrieNode = {
  '0'?: TrieNode;
  '1'?: TrieNode;
  '2'?: TrieNode;
  hash?: number;  // 节点哈希值（整个子树的指纹）
};
```

#### 插入消息到 Merkle 树
```typescript
export function insert(trie: TrieNode, timestamp: Timestamp) {
  const hash = timestamp.hash();
  // 按分钟为单位，将时间戳转换为基数 3 的键
  const key = Number(Math.floor(timestamp.millis() / 1000 / 60)).toString(3);
  
  trie = Object.assign({}, trie, { hash: (trie.hash || 0) ^ hash });
  return insertKey(trie, key, hash);
}
```

#### 差异检测（diff）
```typescript
export function diff(trie1: TrieNode, trie2: TrieNode): number | null {
  // 如果根哈希相同，说明数据完全一致
  if (trie1.hash === trie2.hash) {
    return null;
  }

  // 否则逐层向下查找差异开始的时间点
  let node1 = trie1;
  let node2 = trie2;
  let k = '';

  while (true) {
    const keyset = new Set([...getKeys(node1), ...getKeys(node2)]);
    const keys = [...keyset.values()];
    keys.sort((a, b) => a.localeCompare(b));

    let diffkey: null | '0' | '1' | '2' = null;

    // 遍历查找第一个哈希不同的子节点
    for (let i = 0; i < keys.length; i++) {
      const key = keys[i];
      const next1 = node1[key];
      const next2 = node2[key];

      if (!next1 || !next2) {
        break;  // 某侧节点不存在，说明差异从此处开始
      }

      if (next1.hash !== next2.hash) {
        diffkey = key;
        break;
      }
    }

    if (!diffkey) {
      // 转换时间键为实际时间戳（毫秒）
      return keyToTimestamp(k);
    }

    k += diffkey;
    node1 = node1[diffkey] || emptyTrie();
    node2 = node2[diffkey] || emptyTrie();
  }
}
```

#### Merkle 修剪（减少内存占用）
```typescript
export function prune(trie: TrieNode, n = 2): TrieNode {
  // 只保留最近 n 个时间窗口，防止树无限增长
  const keys = getKeys(trie);
  keys.sort((a, b) => a.localeCompare(b));

  const next: TrieNode = { hash: trie.hash };
  for (const k of keys.slice(-n)) {
    const node = trie[k];
    next[k] = prune(node, n);
  }

  return next;
}
```

### 5.3 增量同步片段

**文件位置**：`packages/loot-core/src/server/sync/index.ts`

#### 增量同步流程
```typescript
async function _fullSync(sinceTimestamp: string, count: number, prevDiffTime: number) {
  // 1. 获取当前时钟状态
  const currentTime = getClock().timestamp.toString();
  
  // 2. 确定同步起始点
  const since = sinceTimestamp || lastSyncedTimestamp || 
    new Timestamp(Date.now() - 5 * 60 * 1000, 0, '0').toString();

  // 3. 获取本地增量消息
  const messages = getMessagesSince(since);

  // 4. 编码并发送到服务器
  const buffer = await encoder.encode(groupId, cloudFileId, since, messages);
  const resBuffer = await postBinary(getServer().SYNC_SERVER + '/sync', buffer, headers);

  // 5. 解码服务器返回的消息
  const res = await encoder.decode(resBuffer);

  // 6. 应用接收到的远程消息
  if (res.messages.length > 0) {
    receivedMessages = await receiveMessages(
      res.messages.map(msg => ({
        ...msg,
        value: deserializeValue(msg.value as string),
      })),
    );
  }

  // 7. Merkle 树差异检测
  const diffTime = merkle.diff(res.merkle, getClock().merkle);

  if (diffTime !== null) {
    // 存在差异，递归同步差异点之后的消息
    // 最多重试 10 次防止无限循环
    if ((count >= 10 && diffTime === prevDiffTime) || count >= 100) {
      // 重试失败，重建 Merkle 树
      const rebuiltMerkle = rebuildMerkleHash();
      if (rebuiltMerkle.trie.hash === res.merkle.hash) {
        // 重建成功，但数据库与时钟不一致
      }
      throw new SyncError('out-of-sync');
    }

    // 继续同步，从差异点开始
    receivedMessages = receivedMessages.concat(
      await _fullSync(
        new Timestamp(diffTime, 0, '0').toString(),
        localTimeChanged ? 0 : count + 1,
        diffTime,
      ),
    );
  } else {
    // 同步完成，保存当前同步点
    await prefs.savePrefs({
      lastSyncedTimestamp: getClock().timestamp.toString(),
    });
  }

  return receivedMessages;
}
```

#### 获取增量消息
```typescript
export function getMessagesSince(since: string): Message[] {
  return db.runQuery(
    'SELECT timestamp, dataset, row, column, value FROM messages_crdt WHERE timestamp > ?',
    [since],
    true,
  );
}
```

---

## 六、失败场景下的状态回退路径

### 6.1 回退机制概览

Actual Budget 采用**双重保险**的回退策略：

| 层级 | 机制 | 触发时机 | 存储位置 |
|------|------|----------|----------|
| Level 1 | Latest Backup | 首次加载任何备份时 | `db.latest.sqlite` + `metadata.latest.json` |
| Level 2 | 历史备份列表 | 用户手动选择 | `backups/*.zip` |

### 6.2 详细回退流程

#### 场景 1：加载备份后发现数据有误

**回退路径**：`Latest Backup → 原始版本`

```
用户操作流程：
1. 加载备份 B (backupId = "2024-01-15_14-30-00.zip")
   ├─ 系统自动保存当前状态为 "latest"
   └─ 应用备份 B 的数据

2. 用户发现数据有问题，点击 "Revert to original version"
   ├─ 从 db.latest.sqlite 恢复数据库文件
   ├─ 从 metadata.latest.json 恢复元数据
   ├─ 删除临时的 latest 备份文件
   └─ 触发云端重新上传
```

**核心代码**：
```typescript
// 恢复逻辑
await fs.copyFile(
  fs.join(budgetDir, LATEST_BACKUP_FILENAME),
  fs.join(budgetDir, 'db.sqlite'),
);
await fs.copyFile(
  fs.join(budgetDir, 'metadata.latest.json'),
  fs.join(budgetDir, 'metadata.json'),
);
await fs.removeFile(fs.join(budgetDir, LATEST_BACKUP_FILENAME));
await fs.removeFile(fs.join(budgetDir, 'metadata.latest.json'));
```

#### 场景 2：预算加载失败（迁移或数据不一致）

**回退路径**：`错误提示 → 备份选择界面`

```typescript
// packages/desktop-client/src/budgetfiles/budgetfilesSlice.ts:63-88
if (error) {
  const message = getSyncError(error, id);
  if (error === 'out-of-sync-migrations') {
    // 迁移版本不一致 - 提示用户升级
    dispatch(pushModal({ modal: { name: 'out-of-sync-migrations' } }));
  } else if (error === 'out-of-sync-data') {
    // 数据版本不一致 - 提供备份恢复选项
    const showBackups = window.confirm(
      message + ' Make sure the app is up-to-date. Do you want to load a backup?'
    );
    if (showBackups) {
      dispatch(pushModal({ modal: { name: 'load-backup', options: {} } }));
    }
  } else {
    alert(message);
  }
}
```

#### 场景 3：同步失败（Merkle 树不一致）

**回退路径**：`重试同步 → 重建 Merkle 树 → 抛出 out-of-sync 错误`

```typescript
// packages/loot-core/src/server/sync/index.ts:754-816
if (diffTime !== null) {
  if ((count >= 10 && diffTime === prevDiffTime) || count >= 100) {
    // 1. 尝试重建本地 Merkle 树
    const rebuiltMerkle = rebuildMerkleHash();
    
    if (rebuiltMerkle.trie.hash === res.merkle.hash) {
      // 重建成功，说明本地时钟数据有问题
      const clocks = await db.all('SELECT * FROM messages_clock');
      // 日志记录调试信息
    }
    
    // 2. 抛出同步失败，前端可以提示用户使用备份
    throw new SyncError('out-of-sync');
  }
}
```

#### 场景 4：云端文件下载或解密失败

**回退路径**：`用户确认 → 重试或取消`

```typescript
// packages/loot-core/src/server/cloud-storage.ts:456-464
if (fileData.encryptMeta) {
  try {
    buffer = await encryption.decrypt(buffer, fileData.encryptMeta);
  } catch (e) {
    throw FileDownloadError('decrypt-failure', {
      isMissingKey: e.message === 'missing-key',
    });
  }
}
```

**前端处理**：
```typescript
// packages/desktop-client/src/budgetfiles/budgetfilesSlice.ts:319-335
if (error.reason === 'decrypt-failure') {
  const opts = {
    hasExistingKey: Boolean(error.meta?.isMissingKey),
    cloudFileId,
    onSuccess: () => {
      dispatch(downloadBudget({ cloudFileId, replace: true }));
    },
  };
  dispatch(pushModal({ modal: { name: 'fix-encryption-key', options: opts } }));
}
```

### 6.3 失败原子性保证

#### 数据库事务保证
```typescript
// packages/loot-core/src/server/sync/index.ts:338-385
db.transaction(() => {
  const added = new Set();

  for (const msg of messages) {
    if (!msg.old) {
      apply(msg, getIn(oldData, [dataset, row]) || added.has(dataset + row));
      // ... 更新内存状态
    }

    if (checkSyncingMode('enabled')) {
      // 插入 CRDT 消息
      db.runQuery(INSERT_CRDT_MESSAGE, params);
      // 更新 Merkle 树
      currentMerkle = merkle.insert(currentMerkle, timestamp);
    }
  }

  if (checkSyncingMode('enabled')) {
    currentMerkle = merkle.prune(currentMerkle);
    // 保存时钟到数据库
    db.runQuery(INSERT_OR_REPLACE_CLOCK, [serializeClock({ ...clock, merkle: currentMerkle })]);
  }
});
```

**关键保证**：
- 所有消息应用和状态更新在**单个数据库事务**中完成
- 事务失败则**全部回滚**，不会出现部分应用的状态
- 内存状态（Merkle 树）只在事务成功后才更新

#### 文件操作的原子性
```typescript
// 备份创建时的安全流程：
1. 先复制到临时文件：db.{timestamp}.sqlite.tmp
2. 在临时文件上执行清理操作
3. 将临时文件打包进 ZIP
4. 只有 ZIP 完全写入磁盘后才删除旧备份
5. 整个过程中原始 db.sqlite 不受影响
```

---

## 七、架构总结图

```
┌─────────────────────────────────────────────────────────────────────┐
│                        用户界面层 (Frontend)                         │
├─────────────────────────────────────────────────────────────────────┤
│  LoadBackupModal.tsx  │  Backups.tsx  │  budgetfilesSlice.tsx        │
│  ───────────────────  │  ───────────  │  ─────────────────────        │
│  - 手动创建备份       │  - 显示信息   │  - Redux Actions             │
│  - 选择历史备份       │               │  - makeBackup()              │
│  - 回退到原始版本     │               │  - loadBackup()              │
└──────────────────────────┬──────────────────────────────────────────┘
                           │ IPC / send()
┌──────────────────────────▼──────────────────────────────────────────┐
│                        服务层 (Backend)                              │
├─────────────────────────────────────────────────────────────────────┤
│  backups.ts                     │  cloud-storage.ts                  │
│  ─────────────                  │  ─────────────────                 │
│  - makeBackup()                 │  - upload()                        │
│    ├─ 清理 CRDT 消息            │  - download()                      │
│    ├─ 打包 db.sqlite            │  - encrypt/decrypt                 │
│    └─ 打包 metadata.json        │  - possiblyUpload()                │
│  - loadBackup()                 │                                     │
│    ├─ 保存 latest 备份          │  app.ts (budgetfiles)              │
│    └─ 解压覆盖当前文件          │  ─────────────────                  │
│  - startBackupService()         │  - _loadBudget()                   │
│    └─ 每15分钟自动备份         │    ├─ 版本迁移验证                  │
│                                 │    ├─ 加载 CRDT 时钟               │
│                                 │    └─ 启动同步服务                 │
└──────────────────────────┬──────────────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────────────┐
│                        同步层 (CRDT Sync)                            │
├─────────────────────────────────────────────────────────────────────┤
│  sync/index.ts                │  merkle.ts                           │
│  ─────────────                │  ─────────                           │
│  - fullSync()                 │  - TrieNode 结构                     │
│    ├─ getMessagesSince()      │  - insert() 插入时间戳               │
│    ├─ encoder.encode()        │  - diff() 检测差异                   │
│    ├─ POST /sync              │  - prune() 修剪树                    │
│    ├─ receiveMessages()       │                                     │
│    └─ merkle.diff() 验证      │  timestamp.ts                        │
│                               │  ─────────────                        │
│  applyMessages()              │  - Lamport 时钟                      │
│    ├─ 数据库事务              │  - 节点 ID                           │
│    ├─ 更新 Merkle 树          │  - 哈希计算                          │
│    └─ 触发业务逻辑            │                                     │
└──────────────────────────┬──────────────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────────────┐
│                        存储层 (Storage)                              │
├─────────────────────────────────────────────────────────────────────┤
│  SQLite Database: db.sqlite                                          │
│  ─────────────────────────────                                      │
│  - messages_crdt: CRDT 消息表                                       │
│  - messages_clock: 时钟 + Merkle 树                                 │
│  - 业务数据表 (accounts, transactions, categories...)               │
│                                                                      │
│  元数据文件: metadata.json                                           │
│  ─────────────────────────                                           │
│  - budgetId, budgetName                                              │
│  - cloudFileId, groupId                                              │
│  - lastSyncedTimestamp, lastUploaded                                 │
│                                                                      │
│  备份目录: backups/*.zip                                             │
│  ───────────────────────                                             │
│  - YYYY-MM-DD_HH-mm-ss.zip                                          │
│    ├─ db.sqlite (已清理 CRDT)                                        │
│    └─ metadata.json                                                  │
│                                                                      │
│  临时回退文件: db.latest.sqlite, metadata.latest.json               │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 八、关键设计决策总结

| 决策点 | 设计选择 | 理由 |
|--------|----------|------|
| **备份清理 CRDT** | 备份中删除 messages_crdt 和 messages_clock | 减少备份体积，恢复后重新同步即可 |
| **Latest Backup 机制** | 首次加载备份时自动保存当前版本 | 提供快速回退路径，防止用户误操作 |
| **Merkle 树修剪** | 只保留最近 2 个时间窗口 | 控制内存占用，平衡检测精度和性能 |
| **15 分钟自动备份** | 定时服务 + 最多保留 10 个 | 平衡安全性和存储空间 |
| **同步重试限制** | 最多 100 次，相同 diffTime 最多 10 次 | 防止无限循环，失败时提供备份选项 |
| **数据库事务** | 所有消息应用在单个事务中 | 保证原子性，避免部分应用的不一致 |
