# 客户端偏好(Preferences)版本迁移与默认值补齐链路分析

## 一、整体架构概览

Actual Budget 的偏好系统分为 **四层存储架构**，各自承担不同的生命周期和同步语义：

| 层级 | 类型 | 存储位置 | 同步范围 | 典型内容 |
|------|------|----------|----------|----------|
| L1 | `SyncedPrefs` | SQLite `preferences` 表 | 跨设备同步 | 日期格式、货币格式、预算类型、功能开关 |
| L2 | `MetadataPrefs` | 预算目录下 `metadata.json` | 本地文件级 | 预算名称、云文件ID、加密密钥ID、同步时间戳 |
| L3 | `LocalPrefs` | 浏览器 localStorage / Electron 本地存储 | 单设备 | 侧边栏宽度、预算折叠状态、UI 展示偏好 |
| L4 | `GlobalPrefs` | `global-store.json` / asyncStorage | 单设备全局 | 主题、语言、文档目录、同步服务器配置 |

核心类型定义见 [prefs.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/21-actual/packages/loot-core/src/types/prefs.ts)。

---

## 二、启动时偏好读取的完整链路

### 2.1 客户端入口初始化流程

**触发点**：[App.tsx#L49-L131](file:///d:/fz/0601-2/solo-dogfeeding/code/21-actual/packages/desktop-client/src/components/App.tsx#L49-L131) 的 `AppInner` 组件 `useEffect`。

```
App启动
  ├─ 1. initConnection()           // 建立与本地后端的 IPC/Worker 连接
  ├─ 2. dispatch(loadGlobalPrefs())
  │     └─ send('load-global-prefs')
  │           └─ [preferences/app.ts#L131-L191]
  │                 └─ asyncStorage.multiGet(...) → 解析 → 返回 GlobalPrefs
  ├─ 3. send('get-last-opened-backup')
  │     └─ 读取 asyncStorage 'lastBudget' 字段
  └─ 4. 如有 lastBudget → dispatch(loadBudget({ id: budgetId }))
```

### 2.2 GlobalPrefs 读取与默认值补齐

**服务端处理函数**：[preferences/app.ts#L131-L191](file:///d:/fz/0601-2/solo-dogfeeding/code/21-actual/packages/loot-core/src/server/preferences/app.ts#L131-L191)

关键设计：**边读取边做类型校验与默认值回填**，每个字段独立兜底：

| 字段 | 存储原始值类型 | 解析后类型 | 兜底策略 |
|------|--------------|-----------|----------|
| `floatingSidebar` | `"true"` / `"false"` 字符串 | `boolean` | `=== 'true'` 严格判定，缺失 → `false` |
| `categoryExpandedState` | 数字字符串 | `number` | `stringToInteger` 失败 → `0` |
| `maxMonths` | 数字字符串 | `number` | `stringToInteger` 失败 → `1` |
| `documentDir` | 路径字符串 | `string` | 缺失 → `getDefaultDocumentDir()` |
| `theme` | 自由字符串 | 枚举联合类型 | 合法值校验：仅允许 `light/dark/auto/midnight`，否则 → `'auto'` |
| `preferredDarkTheme` | 自由字符串 | 枚举联合类型 | 合法值校验：仅允许 `dark/midnight`，否则 → `'dark'` |
| `notifyWhenUpdateIsAvailable` | `undefined` / 任意 | `boolean` | **特殊兜底**：`undefined` → `true`（默认开启更新通知） |

> **设计洞察**：`GlobalPrefs` 是唯一在读取时就做**枚举值合法性校验**的层级。因为主题/语言等配置如果传入非法值会导致渲染崩溃，必须在边界处净化输入。

### 2.3 预算加载（Budget Load）阶段

**触发点**：[budgetfilesSlice.ts#L55-L96](file:///d:/fz/0601-2/solo-dogfeeding/code/21-actual/packages/desktop-client/src/budgetfiles/budgetfilesSlice.ts#L55-L96) 的 `loadBudget` thunk。

```
dispatch(loadBudget({ id }))
  └─ send('load-budget', { id })
        └─ [budgetfiles/app.ts#L508-L640] _loadBudget(id)
              ├─ 1. prefs.loadPrefs(id)          // 加载 MetadataPrefs
              ├─ 2. db.openDatabase(id)          // 打开 SQLite
              ├─ 3. updateVersion()              // ★ 数据库迁移（含偏好表迁移）
              ├─ 4. db.loadClock()               // 加载 CRDT 时钟
              ├─ 5. 检查 resetClock 标志
              ├─ 6. sheet.loadSpreadsheet()      // 加载电子表格引擎
              ├─ 7. 读取 budgetType 设置 sheet meta
              └─ 8. 触发 'load-budget' 事件
  └─ 成功后 → dispatch(loadPrefs())
```

### 2.4 MetadataPrefs 读取与兜底

**核心函数**：[prefs.ts#L24-L45](file:///d:/fz/0601-2/solo-dogfeeding/code/21-actual/packages/loot-core/src/server/prefs.ts#L24-L45)

```typescript
export async function loadPrefs(id?: string): Promise<MetadataPrefs> {
  // 测试环境直接返回默认值，不走文件系统
  if (process.env.NODE_ENV === 'test' && !id) {
    prefs = getDefaultPrefs('test', 'test_LocalPrefs');
    return prefs;
  }

  const fullpath = fs.join(fs.getBudgetDir(id), 'metadata.json');

  try {
    prefs = JSON.parse(await fs.readFile(fullpath));
  } catch {
    // ★ 核心兜底：JSON 解析失败（用户手改文件、文件损坏等）
    // 仍然允许加载数据库，只降级使用默认元数据
    prefs = { id, budgetName: id };
  }

  // ★ 强制矫正：id 字段永远以传入的目录名为基准
  // 防止用户手动移动/重命名文件夹后内部 id 不一致
  prefs.id = id;
  return prefs;
}
```

**两层保护机制**：
1. **解析异常兜底**：`try/catch` 住整个 JSON 解析，失败不抛错，用 `{ id, budgetName: id }` 最低可用配置继续
2. **ID 强制矫正**：无论文件里存的是什么 id，最终都覆盖为当前传入的参数（目录名即真实 id）

### 2.5 三层 Prefs 聚合加载

预算成功打开后，客户端调用 `loadPrefs` thunk 聚合三层数据：

**实现**：[prefsSlice.ts#L37-L70](file:///d:/fz/0601-2/solo-dogfeeding/code/21-actual/packages/desktop-client/src/prefs/prefsSlice.ts#L37-L70)

```typescript
export const loadPrefs = createAppAsyncThunk(
  `${sliceName}/loadPrefs`,
  async (_, { dispatch, getState }) => {
    // L2: MetadataPrefs (已在 load-budget 时加载到内存，这里只是取出)
    const prefs = await send('load-prefs');

    // L4 + L1: 并行读取 GlobalPrefs 和 SyncedPrefs
    const [globalPrefs, syncedPrefs] = await Promise.all([
      send('load-global-prefs'),
      send('preferences/get'),   // SQLite preferences 表全量读出
    ]);

    // 一次性写入 Redux store
    dispatch(setPrefs({ local: prefs, global: globalPrefs, synced: syncedPrefs }));

    // 副作用：更新外部依赖的全局工具状态
    setNumberFormat(parseNumberFormat({
      format: syncedPrefs.numberFormat,
      hideFraction: syncedPrefs.hideFraction,
    }));

    // 副作用：加载 i18n 翻译资源
    setI18NextLanguage(globalPrefs.language ?? '');

    return prefs;
  },
);
```

**SyncedPrefs 读取实现**：[preferences/app.ts#L55-L64](file:///d:/fz/0601-2/solo-dogfeeding/code/21-actual/packages/loot-core/src/server/preferences/app.ts#L55-L64)

```typescript
async function getSyncedPrefs(): Promise<SyncedPrefs> {
  const prefs = await db.all('SELECT id, value FROM preferences');
  // 行式存储 → 对象映射
  return prefs.reduce<SyncedPrefs>((carry, { value, id }) => {
    carry[id as keyof SyncedPrefs] = value;
    return carry;
  }, {});
}
```

> **注意**：SyncedPrefs 读取时**不做任何默认值补齐**，空表就返回空对象 `{}`。默认值逻辑下沉到**消费侧**（各组件/hooks 读取时用解构默认值）。

---

## 三、数据库版本迁移（含偏好表迁移）

### 3.1 迁移触发时序

每次打开预算时必经的迁移流程：

[_loadBudget → updateVersion()](file:///d:/fz/0601-2/solo-dogfeeding/code/21-actual/packages/loot-core/src/server/budgetfiles/app.ts#L550-L568)
  ↓
[updateVersion()](file:///d:/fz/0601-2/solo-dogfeeding/code/21-actual/packages/loot-core/src/server/update.ts#L34-L37)
  ├─ runMigrations()   // 执行 .sql/.js 迁移脚本
  └─ updateViews()     // 重建数据库视图

### 3.2 迁移执行引擎核心逻辑

[migrations.ts#L178-L192](file:///d:/fz/0601-2/solo-dogfeeding/code/21-actual/packages/loot-core/src/server/migrate/migrations.ts#L178-L192)

```typescript
export async function migrate(db: Database): Promise<string[]> {
  // 步骤1：修补历史上有 Bug 的迁移记录（避免重复执行）
  await patchBadMigrations(db);

  // 步骤2：读取已执行迁移ID列表（从 __migrations__ 表）
  const appliedIds = await getAppliedMigrations(db);

  // 步骤3：读取磁盘上所有可用迁移文件，按 ID 数字排序
  const available = await getMigrationList(MIGRATIONS_DIR);

  // 步骤4：★ 安全性检查：检测"数据库版本超前于代码版本"
  checkDatabaseValidity(appliedIds, available);

  // 步骤5：计算待执行的差集
  const pending = getPending(appliedIds, available);

  // 步骤6：顺序执行所有未执行的迁移
  for (const migration of pending) {
    await applyMigration(db, migration, MIGRATIONS_DIR);
  }
  return pending;
}
```

### 3.3 回退兜底检测（超前版本检测）

[migrations.ts#L146-L176](file:///d:/fz/0601-2/solo-dogfeeding/code/21-actual/packages/loot-core/src/server/migrate/migrations.ts#L146-L176)

```typescript
function checkDatabaseValidity(appliedIds: number[], available: string[]): void {
  // 场景A：数据库已执行的迁移数 > 代码能识别的迁移数
  // → 用户用新版本的 App 创建/打开了预算，又回退到旧版本 App
  if (appliedIds.length > available.length) {
    throw new Error('out-of-sync-migrations');
  }

  // 场景B：迁移序列不是前缀匹配（中间有缺失的迁移ID）
  // → 数据库被手动篡改，或迁移脚本被删除/重命名
  for (let i = 0; i < appliedIds.length; i++) {
    if (appliedIds[i] !== getMigrationId(available[i])) {
      throw new Error('out-of-sync-migrations');
    }
  }
}
```

**错误处理链路（客户端侧）**：

[budgetfilesSlice.ts#L63-L68](file:///d:/fz/0601-2/solo-dogfeeding/code/21-actual/packages/desktop-client/src/budgetfiles/budgetfilesSlice.ts#L63-L68)
```
error === 'out-of-sync-migrations'
  → 弹出 "out-of-sync-migrations" 模态框
  → 提示用户需要升级 App 到最新版本

error === 'out-of-sync-data'
  → 弹出确认框，询问是否加载备份
  → 是：打开 load-backup 模态框
  → 否：仅提示确认 App 版本
```

### 3.4 关键迁移脚本：偏好表创建与数据迁移

**迁移 ID: 1723665565000（历史上的偏好存储格式大迁移）**

文件：[1723665565000_prefs.js](file:///d:/fz/0601-2/solo-dogfeeding/code/21-actual/packages/loot-core/migrations/1723665565000_prefs.js)

**背景**：早期版本的 SyncedPrefs 直接存放在 `metadata.json` 中，与 MetadataPrefs 混用。为了支持跨设备 CRDT 同步，需要把偏好拆到 SQLite 的 `preferences` 表中。

```javascript
// 迁移白名单：只把属于跨设备同步语义的字段迁到新表
const SYNCED_PREF_KEYS = [
  'firstDayOfWeekIdx',
  'dateFormat',
  'numberFormat',
  'hideFraction',
  'isPrivacyEnabled',
  /^show-extra-balances-/,     // 正则匹配动态 key
  /^hide-cleared-/,
  /^parse-date-/,
  /^csv-mappings-/,
  'budgetType',
  /^flags\./,
  // ... 其他键
];

export default async function runMigration(db, { fs, fileId }) {
  // 1. 创建新的 preferences 表
  await db.execQuery(`
    CREATE TABLE preferences
       (id TEXT PRIMARY KEY,
        value TEXT);
  `);

  try {
    // 2. 读取旧的 metadata.json
    const budgetDir = fs.getBudgetDir(fileId);
    const fullpath = fs.join(budgetDir, 'metadata.json');
    const prefs = JSON.parse(await fs.readFile(fullpath));

    if (typeof prefs !== 'object') {
      return;  // 数据异常，静默跳过
    }

    // 3. 遍历旧 prefs，白名单匹配的迁移到新表
    await Promise.all(
      Object.keys(prefs).map(async key => {
        if (!SYNCED_PREF_KEYS.find(keyMatcher =>
          keyMatcher instanceof RegExp
            ? keyMatcher.test(key)
            : keyMatcher === key,
        )) {
          return;  // 不在白名单（如 cloudFileId），留在 metadata.json
        }

        // ★ 注意：全部转为 TEXT 类型存储
        db.runQuery(
          'INSERT INTO preferences (id, value) VALUES (?, ?)',
          [key, String(prefs[key])],
        );
      }),
    );
  } catch {
    // ★ 迁移失败不阻塞整体启动
    // 最坏情况：用户需要重新设置偏好，但预算数据完好
  }
}
```

> **关键设计原则**：迁移脚本必须**最大化容错**。偏好丢失可重设，但预算数据加载失败就是 P0 事故。所以整个迁移包裹在空 catch 中，任何异常都静默。

---

## 四、默认值补齐的分布与策略

Actual 采用 **"消费侧就近默认"** 策略，而不是在集中初始化时注入所有默认值。这样做的好处：新增偏好字段不需要改初始化代码，老版本数据库打开新版本 App 时自动适配。

### 4.1 典型默认值应用位置

| 偏好字段 | 默认值 | 应用位置 | 应用方式 |
|----------|--------|----------|----------|
| `budgetType` | `'envelope'` | [budgetfiles/app.ts#L601-L606](file:///d:/fz/0601-2/solo-dogfeeding/code/21-actual/packages/loot-core/src/server/budgetfiles/app.ts#L601-L606) | SQL 查询解构默认值 |
| `budgetType` | `'envelope'` | [sheet.ts#L206-L209](file:///d:/fz/0601-2/solo-dogfeeding/code/21-actual/packages/loot-core/src/server/sheet.ts#L206-L209) | SQL 查询解构默认值 |
| `budgetType` | `'envelope'` | [forecast/app.ts#L70-L73](file:///d:/fz/0601-2/solo-dogfeeding/code/21-actual/packages/loot-core/src/server/forecast/app.ts#L70-L73) | SQL 查询解构默认值 |
| `budgetType` | `'envelope'` | [base.ts#L15-L18](file:///d:/fz/0601-2/solo-dogfeeding/code/21-actual/packages/loot-core/src/server/budget/base.ts#L15-L18) | `getBudgetType()` 函数兜底 |
| `budgetType` | `'envelope'` | [useOverspentCategories.ts#L28](file:///d:/fz/0601-2/solo-dogfeeding/code/21-actual/packages/desktop-client/src/hooks/useOverspentCategories.ts#L28) | Hook 解构默认值 `[budgetType = 'envelope']` |
| `theme` | `'auto'` | [preferences/app.ts#L170-L176](file:///d:/fz/0601-2/solo-dogfeeding/code/21-actual/packages/loot-core/src/server/preferences/app.ts#L170-L176) | 枚举校验失败回退 |
| `notifyWhenUpdateIsAvailable` | `true` | [preferences/app.ts#L186-L189](file:///d:/fz/0601-2/solo-dogfeeding/code/21-actual/packages/loot-core/src/server/preferences/app.ts#L186-L189) | `undefined` 特殊兜底 |
| `documentDir` | `{ACTUAL_DOCUMENT_DIR}/Actual` | [main.ts#L152-L154](file:///d:/fz/0601-2/solo-dogfeeding/code/21-actual/packages/loot-core/src/server/main.ts#L152-L154) | `getDefaultDocumentDir()` |

### 4.2 三层 Hook 读取模式

消费 SyncedPrefs 的标准范式 —— **读取时解构赋默认值**：

```typescript
// useSyncedPref.ts —— 不提供默认值，交给调用方
const pref = useSelector(state => state.prefs.synced[prefName]);
return [pref, setPref];

// 调用方组件侧
const [budgetType = 'envelope'] = useSyncedPref('budgetType');
const [firstDayOfWeekIdx = '0'] = useSyncedPref('firstDayOfWeekIdx');
const [dateFormat = 'MM/DD/YYYY'] = useSyncedPref('dateFormat');
```

同理，GlobalPrefs 和 MetadataPrefs 也遵循相同模式：

- [useGlobalPref.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/21-actual/packages/desktop-client/src/hooks/useGlobalPref.ts)
- [useMetadataPref.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/21-actual/packages/desktop-client/src/hooks/useMetadataPref.ts)

### 4.3 文档目录的双重兜底

文档目录是偏好中最关键的路径配置，因为它决定了所有预算文件的存储位置：

[main.ts#L156-L182](file:///d:/fz/0601-2/solo-dogfeeding/code/21-actual/packages/loot-core/src/server/main.ts#L156-L182)

```
读取 asyncStorage 'document-dir'
  │
  ├─ 有值？→ 尝试 ensureExists(目录)
  │           ├─ 成功 → 使用该目录
  │           └─ 失败（路径不存在、权限不足）→ 清空 documentDir，走兜底
  │
  └─ 无值/兜底 → 使用 getDefaultDocumentDir()
                  = join(process.env.ACTUAL_DOCUMENT_DIR, 'Actual')
```

---

## 五、跨设备同步时的偏好变更传播

### 5.1 本地修改 → 同步到服务器

**保存 SyncedPrefs**：[preferences/app.ts#L38-L53](file:///d:/fz/0601-2/solo-dogfeeding/code/21-actual/packages/loot-core/src/server/preferences/app.ts#L38-L53)

```typescript
async function saveSyncedPrefs({ id, value }) {
  if (!id) return;
  // upsert 到 preferences 表
  await db.update('preferences', { id, value });
  // mutator 装饰器会自动把变更包装成 CRDT Message 加入同步队列
}
```

**保存 MetadataPrefs（仅 budgetName 参与同步）**：[prefs.ts#L47-L79](file:///d:/fz/0601-2/solo-dogfeeding/code/21-actual/packages/loot-core/src/server/prefs.ts#L47-L79)

```typescript
export async function savePrefs(prefsToSet, { avoidSync = false } = {}) {
  Object.assign(prefs, prefsToSet);

  if (!avoidSync) {
    // ★ 仅 budgetName 字段会被包装成同步消息
    // 其他字段（cloudFileId, lastSyncedTimestamp 等）只存本地
    const messages = Object.keys(prefsToSet)
      .map(key => key === 'budgetName' ? {
        dataset: 'prefs',
        row: key,
        column: 'value',
        value: prefsToSet[key],
        timestamp: Timestamp.send(),
      } : null)
      .filter(x => x);

    if (messages.length > 0) {
      await sendMessages(messages);
    }
  }

  // 写回 metadata.json
  await fs.writeFile(prefsPath, JSON.stringify(prefs));
}
```

### 5.2 接收服务器消息 → 本地应用

**同步消息处理**：[sync/index.ts#L340-L360](file:///d:/fz/0601-2/solo-dogfeeding/code/21-actual/packages/loot-core/src/server/sync/index.ts#L340-L360)

```typescript
// applyMessages 中遍历每条 message
if (dataset === 'prefs') {
  // dataset: 'prefs' 的消息是旧格式的 MetadataPrefs 同步（仅 budgetName）
  prefsToSet[row] = value;  // 暂存，最后统一写回 metadata.json
} else {
  // dataset: 'preferences' 的消息才是 SyncedPrefs（通过 db.update 自动记录）
}
```

### 5.3 同步完成 → 刷新客户端状态

**同步事件监听**：[sync-events.ts#L39-L66](file:///d:/fz/0601-2/solo-dogfeeding/code/21-actual/packages/desktop-client/src/sync-events.ts#L39-L66)

```typescript
const unlistenSuccess = listen('sync-event', event => {
  if (event.type === 'success' || event.type === 'applied') {
    const tables = event.tables;

    // 本次同步涉及了 prefs 表 → 重新从后端拉取完整的 prefs 状态
    // （因为 sync 只传 CRDT 增量，Redux 需要全量覆盖才不会有局部更新 bug）
    if (tables.includes('prefs')) {
      void store.dispatch(loadPrefs());
    }
    // ... 其他表的缓存失效逻辑
  }
});
```

---

## 六、异常场景与降级策略总结

| 异常场景 | 触发条件 | 降级行为 | 代码位置 |
|----------|----------|----------|----------|
| metadata.json 损坏 | JSON 解析失败 | 使用 `{ id, budgetName: id }` 最小配置继续 | [prefs.ts#L32-L39](file:///d:/fz/0601-2/solo-dogfeeding/code/21-actual/packages/loot-core/src/server/prefs.ts#L32-L39) |
| metadata.json 中 id 与目录名不一致 | 用户手动移动文件夹 | 强制覆盖 `prefs.id = id`（以目录名为准） | [prefs.ts#L41-L43](file:///d:/fz/0601-2/solo-dogfeeding/code/21-actual/packages/loot-core/src/server/prefs.ts#L41-L43) |
| 偏好迁移脚本执行失败 | 旧文件格式异常 | catch 吞掉异常，迁移中断但预算继续打开 | [1723665565000_prefs.js#L56-L58](file:///d:/fz/0601-2/solo-dogfeeding/code/21-actual/packages/loot-core/migrations/1723665565000_prefs.js#L56-L58) |
| 数据库迁移超前 | 用户回退 App 版本 | 抛 `out-of-sync-migrations`，UI 提示用户升级 | [migrations.ts#L150-L159](file:///d:/fz/0601-2/solo-dogfeeding/code/21-actual/packages/loot-core/src/server/migrate/migrations.ts#L150-L159) |
| 迁移序列断裂 | __migrations__ 表记录与磁盘脚本不匹配 | 同上抛 `out-of-sync-migrations` | [migrations.ts#L161-L175](file:///d:/fz/0601-2/solo-dogfeeding/code/21-actual/packages/loot-core/src/server/migrate/migrations.ts#L161-L175) |
| theme 配置非法值 | 用户手改 global-store.json | 回退到 `'auto'` 主题 | [preferences/app.ts#L170-L176](file:///d:/fz/0601-2/solo-dogfeeding/code/21-actual/packages/loot-core/src/server/preferences/app.ts#L170-L176) |
| document-dir 路径不可访问 | 磁盘卸载 / 权限变更 | 回退到默认文档目录 | [main.ts#L168-L174](file:///d:/fz/0601-2/solo-dogfeeding/code/21-actual/packages/loot-core/src/server/main.ts#L168-L174) |
| SyncedPrefs 字段缺失 | 旧版本数据库 | 消费侧 Hook / 查询解构赋默认值 | 分散各处，典型见 [budgetfiles/app.ts#L601](file:///d:/fz/0601-2/solo-dogfeeding/code/21-actual/packages/loot-core/src/server/budgetfiles/app.ts#L601) |
| _loadBudget 中途异常 | 任意步骤抛错 | 调用 `closeBudget()` 清理已加载资源，返回错误码 | [budgetfiles/app.ts#L536-L568](file:///d:/fz/0601-2/solo-dogfeeding/code/21-actual/packages/loot-core/src/server/budgetfiles/app.ts#L536-L568) |

---

## 七、关键设计模式总结

1. **分层存储，按同步语义分类**：把偏好按同步需求拆到 4 个层级，避免了"一刀切"全同步或全本地的窘境
2. **消费侧就近默认值**：不在加载时集中注入，改为每个读点自行兜底——新增偏好字段零侵入
3. **迁移 = 最低保障原则**：迁移失败绝不阻塞主流程，偏好是"可重设"的，预算数据才是核心
4. **ID 真实性双来源**：目录名是 id 的唯一真相源（source of truth），metadata.json 的 id 字段仅作缓存，加载时强制矫正
5. **边界净化模式**：GlobalPrefs 读取时做枚举值合法性校验，把"脏数据"拦截在服务端边界，避免下游组件因非法值崩溃
6. **同步完成后全量重拉**：CRDT 只传增量，Redux 却做全量刷新——用一次全量查询的代价换来状态一致性的简单性
