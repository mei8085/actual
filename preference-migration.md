# 客户端偏好(Preferences)版本迁移与默认值补齐链路分析

---

## 一、整体架构概览

Actual Budget 的偏好系统分为 **四层存储架构**，各自承担不同的生命周期和同步语义：

| 层级 | 类型 | 存储位置 | 同步范围 | 典型内容 |
|------|------|----------|----------|----------|
| L1 | `SyncedPrefs` | SQLite `preferences` 表 | 跨设备同步 | 日期格式、货币格式、预算类型、功能开关 |
| L2 | `MetadataPrefs` | 预算目录下 `metadata.json` | 本地文件级 | 预算名称、云文件ID、加密密钥ID、同步时间戳 |
| L3 | `LocalPrefs` | 浏览器 localStorage / Electron 本地存储 | 单设备 | 侧边栏宽度、预算折叠状态、UI 展示偏好 |
| L4 | `GlobalPrefs` | `global-store.json` / asyncStorage | 单设备全局 | 主题、语言、文档目录、同步服务器配置 |

核心类型定义见 [prefs.ts](packages/loot-core/src/types/prefs.ts#L1-L169)。

---

## 二、启动时偏好读取的完整链路

### 2.1 客户端入口初始化流程

**触发点**：[App.tsx](packages/desktop-client/src/components/App.tsx#L49-L131) 的 `AppInner` 组件 `useEffect`。

```
App启动
  ├─ 1. initConnection()           // 建立与本地后端的 IPC/Worker 连接
  ├─ 2. dispatch(loadGlobalPrefs())
  │     └─ send('load-global-prefs')
  │           └─ asyncStorage.multiGet(...) → 解析 → 返回 GlobalPrefs
  ├─ 3. send('get-last-opened-backup')
  │     └─ 读取 asyncStorage 'lastBudget' 字段
  └─ 4. 如有 lastBudget → dispatch(loadBudget({ id: budgetId }))
```

### 2.2 GlobalPrefs 读取与默认值补齐

**服务端处理函数**：[preferences/app.ts](packages/loot-core/src/server/preferences/app.ts#L131-L191)

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

**触发点**：[budgetfilesSlice.ts](packages/desktop-client/src/budgetfiles/budgetfilesSlice.ts#L55-L96) 的 `loadBudget` thunk。

```
dispatch(loadBudget({ id }))
  └─ send('load-budget', { id })
        └─ _loadBudget(id)
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

**核心函数**：[prefs.ts](packages/loot-core/src/server/prefs.ts#L24-L45)

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

**实现**：[prefsSlice.ts](packages/desktop-client/src/prefs/prefsSlice.ts#L37-L70)

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

**SyncedPrefs 读取实现**：[preferences/app.ts](packages/loot-core/src/server/preferences/app.ts#L55-L64)

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

```
_loadBudget
  ↓
updateVersion()
  ├─ runMigrations()   // 执行 .sql/.js 迁移脚本
  └─ updateViews()     // 重建数据库视图
```

### 3.2 迁移执行引擎核心逻辑

[migrations.ts](packages/loot-core/src/server/migrate/migrations.ts#L178-L192)

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

[migrations.ts](packages/loot-core/src/server/migrate/migrations.ts#L146-L176)

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

[budgetfilesSlice.ts](packages/desktop-client/src/budgetfiles/budgetfilesSlice.ts#L63-L68)
```
error === 'out-of-sync-migrations'
  → 弹出 "out-of-sync-migrations" 模态框
  → 提示用户需要升级 App 到最新版本

error === 'out-of-sync-data'
  → 弹出确认框，询问是否加载备份
  → 是：打开 load-backup 模态框
  → 否：仅提示确认 App 版本
```

### 3.4 迁移脚本 #1：偏好表创建与数据迁移（ID: 1723665565000）

**文件**：[1723665565000_prefs.js](packages/loot-core/migrations/1723665565000_prefs.js)

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

### 3.5 迁移脚本 #2：budgetType 旧值规整（ID: 1745425408000）

**文件**：[1745425408000_update_budgetType_pref.sql](packages/loot-core/migrations/1745425408000_update_budgetType_pref.sql)

**背景**：历史上 `budgetType` 字段存在三个合法值：`'envelope'`（信封预算）、`'report'`（报告式预算）、以及后续新增的 `'tracking'`（跟踪式预算）。在某次重构中，`'report'` 被 `'tracking'` 完全替代，但旧数据库里仍残留 `'report'` 值。

```sql
BEGIN TRANSACTION;

UPDATE preferences
SET value = CASE
    WHEN id = 'budgetType' AND value = 'report'
        THEN 'tracking'
    ELSE 'envelope'
END
WHERE id = 'budgetType';

COMMIT;
```

**规整规则拆解**（CASE WHEN 表达式）：

| 原始 value 值 | 匹配条件 | 规整后 value | 说明 |
|--------------|----------|-------------|------|
| `'report'` | `id='budgetType' AND value='report'` | `'tracking'` | 旧术语 `report` 重命名为新术语 `tracking` |
| 任意其他值（含 `'envelope'`、`'tracking'`、NULL、空字符串、乱码等） | `ELSE` 分支 | `'envelope'` | 统一设为默认值 |

> **⚠️ 关键注意**：CASE WHEN 只有 `WHEN value = 'report'` 这一个显式分支，ELSE 把**所有其他值**（包括合法的 `'tracking'`）都覆写为 `'envelope'`。这并非"把非法值规整到 envelope"，而是"只保留 report→tracking 的语义映射，其余一律归零到 envelope"。迁移执行时 `preferences` 表中 `id='budgetType'` 的行一定存在（1723665565000 迁移已插入），所以该行一定会被 UPDATE 命中。

**为什么这个迁移在生产环境是安全的？**
- 该迁移发布时，`'tracking'` 这个新值还没有在用户数据库中广泛出现——因为 `'tracking'` 就是本迁移引入的术语
- 迁移前用户数据库中只存在 `'envelope'` 和 `'report'` 两种值
- `report` → `tracking` 是语义映射，`envelope` → `envelope` 是幂等写入，两者都正确
- 即使用户在迁移发布前已经手动存了 `'tracking'`，被改回 `'envelope'` 也不会导致数据损坏，只是需要用户在设置中切回

### 3.6 迁移脚本 #3：CSV 跳过行偏好 key 改名（ID: 1762178745667）

**文件**：[1762178745667_rename_csv_skip_lines_pref.sql](packages/loot-core/migrations/1762178745667_rename_csv_skip_lines_pref.sql)

**背景**：CSV 导入功能从只能跳过"文件开头几行"扩展为支持"开头跳过"和"结尾跳过"两种。为了语义更清晰，旧 key `csv-skip-lines-{accountId}` 被拆分为 `csv-skip-start-lines-{accountId}` 和 `csv-skip-end-lines-{accountId}`。

```sql
BEGIN TRANSACTION;

-- Rename csv-skip-lines-* preferences to csv-skip-start-lines-*
UPDATE preferences
SET id = REPLACE(id, 'csv-skip-lines-', 'csv-skip-start-lines-')
WHERE id LIKE 'csv-skip-lines-%';

COMMIT;
```

**改名规则**：

| 旧 key 格式 | 新 key 格式 | 说明 |
|------------|------------|------|
| `csv-skip-lines-{accountId}` | `csv-skip-start-lines-{accountId}` | 原有的"跳过行数"语义平移为"跳过开头行数" |
| （不存在） | `csv-skip-end-lines-{accountId}` | 新增的"跳过结尾行数"，无历史数据需迁移 |

**SQL 操作细节**：
- `WHERE id LIKE 'csv-skip-lines-%'` 精准匹配所有旧格式的行
- `REPLACE(id, 'csv-skip-lines-', 'csv-skip-start-lines-')` 只替换 key 中的固定前缀，保留动态的 `{accountId}` 部分
- 由于 SQLite PRIMARY KEY 可以在 UPDATE 时直接修改（只要新值不冲突），整个操作一条 UPDATE 完成，不需要临时表

### 3.7 三次偏好相关迁移的执行时序与协作

```
用户打开旧版本预算（从未执行过 172/174/176 迁移）
  │
  ├─ 1. updateVersion() 执行迁移引擎
  │
  ├─ 2. 执行 [1723665565000_prefs.js]
  │     ├─ 创建 preferences 表
  │     └─ 从 metadata.json 把 budgetType、csv-skip-lines-* 等字段迁入
  │           → budgetType 可能是 'report' 或 'envelope'
  │           → csv key 仍然是旧格式 'csv-skip-lines-xxx'
  │
  ├─ 3. 执行 [1745425408000_update_budgetType_pref.sql]
  │     └─ budgetType: 'report' → 'tracking'，其余值（含 'envelope'）→ 'envelope'
  │           迁移后 budgetType 为 'tracking' 或 'envelope'
  │
  ├─ 4. 执行 [1762178745667_rename_csv_skip_lines_pref.sql]
  │     └─ 'csv-skip-lines-xxx' → 'csv-skip-start-lines-xxx'
  │
  └─ 5. 迁移完成 → 后续消费侧默认值兜底（见第四章）
```

> **设计洞察**：三次迁移的依赖关系非常清晰——172 建表是基础，174 和 176 都操作 172 建好的表；174 和 176 互不依赖，顺序不影响结果。

---

## 四、默认值补齐的分布与策略

Actual 采用 **"消费侧就近默认"** 策略，而不是在集中初始化时注入所有默认值。这样做的好处：新增偏好字段不需要改初始化代码，老版本数据库打开新版本 App 时自动适配。

### 4.1 budgetType 的四层防护：各自的覆盖范围不同

`budgetType` 是偏好系统中默认值兜底最密集的字段，因为它决定了核心预算计算的分支逻辑（envelope 预算 vs tracking 预算）。代码中存在**四种不同的防护机制**，它们的作用时机和覆盖范围各不相同，不能笼统地说"运行时一定收敛"。

#### 第一层：一次性迁移规整

[1745425408000_update_budgetType_pref.sql](packages/loot-core/migrations/1745425408000_update_budgetType_pref.sql#L3-L5) 在版本升级时执行一次：`report` → `tracking`，其余值（含 `envelope`、`tracking`、NULL、乱码等）→ `envelope`。

- **作用时机**：仅版本升级时执行一次
- **覆盖范围**：数据库中 `preferences` 表的 `budgetType` 行
- **无法覆盖的场景**：迁移已执行过的数据库不会再跑；用户事后手动篡改数据库写入非法值；迁移脚本自身被 catch 吞掉（1723665565000 的异常兜底）导致 preferences 表不存在
- **副作用**：ELSE 分支会把合法的 `'tracking'` 也覆写为 `'envelope'`（见 3.5 节分析）

#### 第二层：服务端 SQL 查询解构默认值

| 代码位置 | 代码模式 | 兜底值 |
|----------|----------|--------|
| [budgetfiles/app.ts](packages/loot-core/src/server/budgetfiles/app.ts#L601-L606) | `const { value: budgetType = 'envelope' } = db.first(...) ?? {}` | `'envelope'` |
| [sheet.ts](packages/loot-core/src/server/sheet.ts#L206-L209) | `const { value: budgetType = 'envelope' } = db.first(...) ?? {}` | `'envelope'` |
| [forecast/app.ts](packages/loot-core/src/server/forecast/app.ts#L70-L73) | `const { value: budgetType = 'envelope' } = db.first(...) ?? {}` | `'envelope'` |
| [base.ts](packages/loot-core/src/server/budget/base.ts#L15-L18) | `return meta.budgetType \|\| 'envelope'` | `'envelope'` |
| [spreadsheet.ts](packages/loot-core/src/server/spreadsheet/spreadsheet.ts#L55-L57) | `this._meta = { budgetType: 'envelope' }` | `'envelope'` |

典型实现：
```typescript
const { value: budgetType = 'envelope' } =
  (await db.first<Pick<DbPreference, 'value'>>(
    'SELECT value from preferences WHERE id = ?',
    ['budgetType'],
  )) ?? {};
```

- **作用时机**：每次打开预算或 spreadsheet 加载时
- **覆盖范围**：仅当 `preferences` 表中不存在 `id='budgetType'` 的行（`db.first` 返回 `null`），或该行的 `value` 列为 `NULL`（解构赋值到 `{ value: null }`，`null ?? 'envelope'` 仍不触发默认值，但 `{ value: undefined }` 或 `{}` 会触发）
- **无法覆盖的场景**：行存在且 `value` 为非空非法字符串（如 `'report'`、`'foobar'`），解构默认值不会生效，非法值会被原样透传
- **关键限制**：这是**缺失兜底**而非**非法值校验**，它只管"没读到值"的情况

#### 第三层：客户端显式校验（三目运算白名单）

```typescript
// [Spending.tsx]、[SpendingCard.tsx]、[BalanceForecast.tsx]、[BalanceForecastCard.tsx]
const [budgetTypePref] = useSyncedPref('budgetType');
const budgetType: 'envelope' | 'tracking' =
  budgetTypePref === 'tracking' ? 'tracking' : 'envelope';
```

- **作用时机**：组件渲染时，持续生效
- **覆盖范围**：**全覆盖**——无论数据库存了什么值（`undefined`、`'report'`、`'foobar'`、甚至空字符串），只要不是显式等于 `'tracking'`，一律收敛到 `'envelope'`
- **覆盖不了的场景**：服务端代码路径（预算计算、forecast 等）不经过客户端组件，显式校验对服务端无效
- **本质**：这是唯一能做到"运行时对非法值收敛"的层级，但**仅限客户端渲染路径**

#### 第四层：客户端普通解构默认值

```typescript
// [useOverspentCategories.ts]
const [budgetType = 'envelope'] = useSyncedPref('budgetType');

// [BudgetTypeSettings.tsx]
const [budgetType = 'envelope', setBudgetType] = useSyncedPref('budgetType');

// [CustomReport.tsx]、[GetCardData.tsx]
const [budgetType = 'envelope'] = useSyncedPref('budgetType');
```

- **作用时机**：组件渲染时，持续生效
- **覆盖范围**：仅当 Redux store 中 `prefs.synced.budgetType === undefined`（即该 key 从未被设置过）
- **无法覆盖的场景**：`budgetType` 为非 `undefined` 的非法值时，解构默认值不触发，非法值被原样透传
- **与第三层的区别**：第三层是显式校验（只认 `'tracking'`），第四层是缺失兜底（只防 `undefined`）

#### 四层防护的覆盖范围对比

| 非法值场景 | 迁移规整 | 服务端默认值 | 客户端显式校验 | 客户端解构默认值 |
|-----------|---------|------------|--------------|----------------|
| `budgetType` 行不存在 | ❌（表不存在时迁移也可能失败） | ✅ `db.first` 返回 null | ✅ `undefined !== 'tracking'` → `envelope` | ✅ `undefined` 触发默认值 |
| `budgetType` 值为 `NULL` | ❌（迁移只执行一次） | ⚠️ 取决于解构写法（`null ?? 'envelope'` 不生效） | ✅ `null !== 'tracking'` → `envelope` | ❌ `null` 不触发默认值 |
| `budgetType` 值为 `'report'` | ✅（迁移执行时转为 `tracking`） | ❌ 原样透传 | ✅ `'report' !== 'tracking'` → `envelope` | ❌ `'report'` 不触发默认值 |
| `budgetType` 值为乱码 | ❌（迁移已执行过） | ❌ 原样透传 | ✅ 乱码 `!== 'tracking'` → `envelope` | ❌ 乱码不触发默认值 |
| `budgetType` 值为 `tracking` | ❌（ELSE 分支覆写为 `envelope`） | ✅ 原样透传 | ✅ `'tracking' === 'tracking'` → `tracking` | ✅ `'tracking'` 原样使用 |

> **核心结论**：只有客户端显式校验（第三层）能做到"运行时对非法值收敛"。服务端代码路径（预算计算、spreadsheet 等）没有等价的显式校验——它们依赖迁移规整和缺失兜底，对"行存在但值为非法字符串"的场景**不会自动收敛**。因此迁移脚本的一次性规整对服务端路径至关重要，如果迁移被跳过或数据库被事后篡改，服务端可能拿到非法值。

### 4.2 CSV 导入偏好默认值：改名迁移 + 消费侧类型转换兜底

CSV 导入相关偏好全部采用 `csv-{功能}-{accountId}` 的命名格式，存储值均为 TEXT 类型的字符串。

**相关迁移**：[1762178745667_rename_csv_skip_lines_pref.sql](packages/loot-core/migrations/1762178745667_rename_csv_skip_lines_pref.sql#L3-L6)
- 旧 key `csv-skip-lines-{accountId}` → 新 key `csv-skip-start-lines-{accountId}`
- 新增 key `csv-skip-end-lines-{accountId}` 无历史数据，完全依赖消费侧默认值

**消费侧默认值实现**：[ImportTransactionsModal.tsx](packages/desktop-client/src/components/modals/ImportTransactionsModal/ImportTransactionsModal.tsx#L241-L274)

| 偏好 key | 存储值类型 | 消费侧兜底逻辑 | 默认值 |
|----------|-----------|--------------|--------|
| `csv-delimiter-{accountId}` | 字符串（`','` / `'\t'` 等） | `\|\|` 短路或 | 按文件后缀：`.tsv` → `'\t'`，其他 → `','` |
| `csv-skip-start-lines-{accountId}` | 数字字符串 | `parseInt(..., 10) \|\| 0` | `0`（不跳过开头行） |
| `csv-skip-end-lines-{accountId}` | 数字字符串 | `parseInt(..., 10) \|\| 0` | `0`（不跳过结尾行） |
| `csv-in-out-mode-{accountId}` | `'true'` / `'false'` 字符串 | `String(...) === 'true'` | `false`（默认不启用收支分列模式） |
| `csv-out-value-{accountId}` | 自由字符串 | `?? ''` | `''`（空字符串，表示不替换支出标记） |
| `csv-has-header-{accountId}` | `'true'` / `'false'` 字符串 | `String(...) !== 'false'` | `true`（默认假设 CSV 有表头行） |
| `ofx-fallback-missing-payee-{accountId}` | `'true'` / `'false'` 字符串 | `String(...) !== 'false'` | `true`（默认启用缺失收款人降级到 memo） |
| `ofx-swap-payee-memo-{accountId}` | `'true'` / `'false'` 字符串 | `String(...) === 'true'` | `false`（默认不交换收款人和 memo） |
| `qif-swap-payee-memo-{accountId}` | `'true'` / `'false'` 字符串 | `String(...) === 'true'` | `false`（默认不交换） |
| `camt-swap-payee-memo-{accountId}` | `'true'` / `'false'` 字符串 | `String(...) === 'true'` | `false`（默认不交换） |
| `import-reimport-deleted-{accountId}` | `'true'` / `'false'` 字符串 | `String(... \|\| 'true') === 'true'` | `true`（默认允许重新导入已删除交易） |

**类型转换兜底模式解析**：

1. **数字类偏好**：`parseInt(prefs['csv-skip-start-lines-xxx'], 10) || 0`
   - `parseInt` 失败（空字符串、非数字）返回 `NaN`
   - `NaN || 0` → `0`
   - 保证永远得到合法的数字

2. **布尔类偏好（正向判断）**：`String(prefs['csv-in-out-mode-xxx']) === 'true'`
   - `String(undefined)` → `'undefined'`，不等于 `'true'` → `false`
   - 适用于"默认关闭"的功能开关

3. **布尔类偏好（反向判断）**：`String(prefs['csv-has-header-xxx']) !== 'false'`
   - `String(undefined)` → `'undefined'`，不等于 `'false'` → `true`
   - 适用于"默认开启"的功能开关
   - 注意：只有显式存了 `'false'` 字符串才会关闭

4. **布尔类偏好（带默认值短路）**：`String(prefs['xxx'] || 'true') === 'true'`
   - 先 `|| 'true'` 保证有默认值，再做字符串比较
   - 比反向判断更直观：明确声明默认值

### 4.3 其他典型默认值应用位置

| 偏好字段 | 默认值 | 应用位置 | 应用方式 |
|----------|--------|----------|----------|
| `theme` | `'auto'` | [preferences/app.ts](packages/loot-core/src/server/preferences/app.ts#L170-L176) | 枚举校验失败回退 |
| `notifyWhenUpdateIsAvailable` | `true` | [preferences/app.ts](packages/loot-core/src/server/preferences/app.ts#L186-L189) | `undefined` 特殊兜底 |
| `documentDir` | `{ACTUAL_DOCUMENT_DIR}/Actual` | [main.ts](packages/loot-core/src/server/main.ts#L152-L154) | `getDefaultDocumentDir()` |

### 4.4 三层 Hook 读取模式

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

- [useGlobalPref.ts](packages/desktop-client/src/hooks/useGlobalPref.ts)
- [useMetadataPref.ts](packages/desktop-client/src/hooks/useMetadataPref.ts)

### 4.5 文档目录的双重兜底

文档目录是偏好中最关键的路径配置，因为它决定了所有预算文件的存储位置：

[main.ts](packages/loot-core/src/server/main.ts#L156-L182)

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

### 5.0 核心概念：两种偏好变更的本质区别

⚠️ **这是理解整个刷新链路的关键**：Actual 中存在**两种完全独立**的偏好同步机制，它们的 `dataset` 名称、存储位置、刷新逻辑都不同。

| 维度 | `prefs` Dataset 事件 | `preferences` 表变更 |
|------|---------------------|---------------------|
| **dataset 名称** | `'prefs'`（单数） | `'preferences'`（复数） |
| **对应偏好类型** | MetadataPrefs（仅 `budgetName`） | SyncedPrefs（所有跨设备偏好） |
| **存储位置** | `metadata.json` 文件 | SQLite `preferences` 表 |
| **是否写入数据库** | ❌ 从不写入 | ✅ 正常 INSERT/UPDATE |
| **表名检测结果** | `tables.includes('prefs')` → `true` | `tables.includes('preferences')` → `true` |
| **sync-events 自动刷新** | ✅ 触发 `loadPrefs()` | ❌ **不触发**（只检查 `'prefs'`） |
| **本地修改刷新** | 通过 `savePrefs()` → Redux `mergeLocalPrefs` | 通过 `saveSyncedPrefs()` → Redux `mergeSyncedPrefs` |
| **远程修改刷新** | 同步事件 → `loadPrefs()` 全量重拉 | 依赖用户下次 `loadPrefs()` 触发（如重开预算） |
| **独立事件** | 发送 `'prefs-updated'` 自定义事件 | 无独立事件 |

---

### 5.1 本地修改 → 同步到服务器

#### 5.1.1 保存 SyncedPrefs（`preferences` 表）

**代码**：[preferences/app.ts](packages/loot-core/src/server/preferences/app.ts#L38-L53)

```typescript
async function saveSyncedPrefs({ id, value }) {
  if (!id) return;
  // upsert 到 preferences 表
  await db.update('preferences', { id, value });
  // mutator 装饰器会自动把变更包装成 CRDT Message 加入同步队列
}
```

**CRDT 消息格式**（`dataset: 'preferences'`）：
```typescript
{
  dataset: 'preferences',    // ★ 注意是复数
  row: 'budgetType',
  column: 'value',
  value: 'tracking',
  timestamp: Timestamp.send(),
}
```

#### 5.1.2 保存 MetadataPrefs（仅 `budgetName` 参与同步）

**代码**：[prefs.ts](packages/loot-core/src/server/prefs.ts#L47-L79)

```typescript
export async function savePrefs(prefsToSet, { avoidSync = false } = {}) {
  Object.assign(prefs, prefsToSet);

  if (!avoidSync) {
    // ★ 仅 budgetName 字段会被包装成同步消息
    // 其他字段（cloudFileId, lastSyncedTimestamp 等）只存本地
    const messages = Object.keys(prefsToSet)
      .map(key => key === 'budgetName' ? {
        dataset: 'prefs',       // ★ 注意是单数！
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

**CRDT 消息格式**（`dataset: 'prefs'`）：
```typescript
{
  dataset: 'prefs',          // ★ 注意是单数
  row: 'budgetName',
  column: 'value',
  value: 'My New Budget',
  timestamp: Timestamp.send(),
}
```

---

### 5.2 接收服务器消息 → 本地应用

**核心处理函数**：[sync/index.ts](packages/loot-core/src/server/sync/index.ts#L338-L445)

#### 5.2.1 `apply()` 函数中的差异化处理

[sync/index.ts](packages/loot-core/src/server/sync/index.ts#L80-L108)

```typescript
function apply(msg: Message, prev?: boolean) {
  const { dataset, row, column, value } = msg;

  if (dataset === 'prefs') {
    // ★ 'prefs' 不写入数据库，这是 MetadataPrefs 的同步消息
    // Do nothing, it doesn't exist in the db
  } else {
    // ★ 其他所有 dataset（包括 'preferences'）正常写入数据库
    let query;
    try {
      if (prev) {
        query = {
          sql: `UPDATE ${dataset} SET ${column} = ? WHERE id = ?`,
          params: [value, row],
        };
      } else {
        query = {
          sql: `INSERT INTO ${dataset} (id, ${column}) VALUES (?, ?)`,
          params: [row, value],
        };
      }
      db.runQuery(db.cache(query.sql), query.params);
    } catch (error) {
      throw new SyncError('invalid-schema', { ... });
    }
  }
}
```

#### 5.2.2 `applyMessages()` 中的完整处理流程

[sync/index.ts](packages/loot-core/src/server/sync/index.ts#L311-L397)

```typescript
// 暂存 MetadataPrefs 变更，最后统一写回文件
const prefsToSet: MetadataPrefs = {};

// ... 省略 fetchData、undo 记录等前置步骤 ...

db.transaction(() => {
  const added = new Set();

  for (const msg of messages) {
    const { dataset, row, column, timestamp, value } = msg;

    if (!msg.old) {
      apply(msg, getIn(oldData, [dataset, row]) || added.has(dataset + row));

      if (dataset === 'prefs') {
        // ★ 'prefs' 消息：收集到 prefsToSet，稍后写 metadata.json
        prefsToSet[row] = value;
      } else {
        added.add(dataset + row);
      }
    }

    // ... 省略 CRDT 时钟、merkle 树更新 ...

    // ★ SyncedPrefs 特殊处理：budgetType 变更实时更新 spreadsheet
    if (dataset === 'preferences' && row === 'budgetType') {
      void setBudgetType(value);
    }
  }

  // ... 省略时钟保存 ...
});

// ★ 事务成功后，批量写回 MetadataPrefs
if (Object.keys(prefsToSet).length > 0) {
  void prefs.savePrefs(prefsToSet, { avoidSync: true });  // avoidSync: 避免循环同步
  connection.send('prefs-updated');  // ★ 发送独立事件通知
}
```

#### 5.2.3 表名提取逻辑

[sync/index.ts](packages/loot-core/src/server/sync/index.ts#L569-L579)

```typescript
function getTablesFromMessages(messages: Message[]): string[] {
  return messages.reduce((acc, message) => {
    const dataset =
      message.dataset === 'schedules_next_date' ? 'schedules' : message.dataset;

    if (!acc.includes(dataset)) {
      acc.push(dataset);
    }
    return acc;
  }, []);
}
```

> **关键**：这里直接把 `message.dataset` 作为表名。所以：
> - `dataset: 'prefs'` → `tables = ['prefs']`
> - `dataset: 'preferences'` → `tables = ['preferences']`
> - 两种消息的表名**完全不同**

---

### 5.3 同步完成 → 刷新客户端状态

#### 5.3.1 同步事件的发出位置

**本地变更/远程消息应用成功**：[sync/index.ts](packages/loot-core/src/server/sync/index.ts#L436-L442)
```typescript
const tables = getTablesFromMessages(messages.filter(msg => !msg.old));
app.events.emit('sync', {
  type: 'applied',
  tables,      // 可能包含 'prefs' 或 'preferences'
  data: newData,
  prevData: oldData,
});
```

**全量同步成功**：[sync/index.ts](packages/loot-core/src/server/sync/index.ts#L663-L669)
```typescript
const tables = getTablesFromMessages(messages);
app.events.emit('sync', {
  type: 'success',
  tables,      // 可能包含 'prefs' 或 'preferences'
  syncDisabled: checkSyncingMode('disabled'),
});
```

#### 5.3.2 客户端同步事件监听

**代码**：[sync-events.ts](packages/desktop-client/src/sync-events.ts#L39-L66)

```typescript
const unlistenSuccess = listen('sync-event', event => {
  const prefs = store.getState().prefs.local;
  if (!prefs || !prefs.id) return;  // 预算未加载时不处理

  if (event.type === 'success' || event.type === 'applied') {
    const tables = event.tables;

    // ⚠️ 重点：这里只检查 'prefs'（单数），不检查 'preferences'（复数）
    // → MetadataPrefs 变更（budgetName）会触发刷新
    // → SyncedPrefs 变更（budgetType、dateFormat 等）不会触发刷新！
    if (tables.includes('prefs')) {
      void store.dispatch(loadPrefs());  // 全量重拉三层 prefs
    }

    // ... 其他表（categories、accounts 等）的缓存失效逻辑
  }
  // ... 错误处理省略
});
```

#### 5.3.3 `prefs-updated` 独立事件监听

**代码**：[settings/index.tsx](packages/desktop-client/src/components/settings/index.tsx#L181-L185)

```typescript
useEffect(() => {
  // 监听 MetadataPrefs 更新事件（仅 budgetName 变更时触发）
  const unlisten = listen('prefs-updated', () => {
    void dispatch(loadPrefs());
  });
  return unlisten;
}, [dispatch]);
```

---

### 5.4 完整同步刷新时序对比

#### 场景 A：修改预算名称（MetadataPrefs，`budgetName`）

```
本地调用 savePrefs({ budgetName: '新名称' })
  │
  ├─ 1. 更新内存中的 prefs 对象
  ├─ 2. 生成 CRDT 消息 { dataset: 'prefs', row: 'budgetName', ... }
  ├─ 3. sendMessages(messages) 加入同步队列
  ├─ 4. 写回 metadata.json 文件
  │
  └─ [Redux 侧] dispatch(mergeLocalPrefs({ budgetName: '新名称' }))
     → 立即更新本地 Redux 状态

───────────────────────────────────────────────
其他设备同步接收：
  │
  ├─ 1. 收到 { dataset: 'prefs', row: 'budgetName', ... } 消息
  ├─ 2. apply() 中遇到 'prefs' → 跳过数据库操作
  ├─ 3. 收集到 prefsToSet['budgetName'] = '新名称'
  ├─ 4. 事务成功后，savePrefs(prefsToSet, { avoidSync: true })
  ├─ 5. 发送 'prefs-updated' 事件
  ├─ 6. emit('sync', { type: 'applied', tables: ['prefs'] })
  │
  ├─ 7. sync-events.ts: tables.includes('prefs') → true
  │   → dispatch(loadPrefs()) → 全量刷新三层 prefs
  │
  └─ 8. settings/index.tsx: 监听到 'prefs-updated' 事件
      → dispatch(loadPrefs()) → 再次刷新（重复但幂等）
```

#### 场景 B：修改预算类型（SyncedPrefs，`budgetType`）

```
本地调用 saveSyncedPrefs({ budgetType: 'tracking' })
  │
  ├─ 1. db.update('preferences', { id: 'budgetType', value: 'tracking' })
  ├─ 2. mutator 自动生成 CRDT 消息 { dataset: 'preferences', row: 'budgetType', ... }
  ├─ 3. sendMessages(messages) 加入同步队列
  │
  └─ [Redux 侧] dispatch(mergeSyncedPrefs({ budgetType: 'tracking' }))
     → 立即更新本地 Redux 状态

───────────────────────────────────────────────
其他设备同步接收：
  │
  ├─ 1. 收到 { dataset: 'preferences', row: 'budgetType', ... } 消息
  ├─ 2. apply() 中遇到 'preferences' → 正常 UPDATE preferences 表
  ├─ 3. ★ 特殊处理：dataset === 'preferences' && row === 'budgetType'
  │   → 调用 setBudgetType('tracking') → 更新 spreadsheet meta
  ├─ 4. emit('sync', { type: 'applied', tables: ['preferences'] })
  │
  ├─ 5. sync-events.ts: tables.includes('prefs') → false ❌
  │   （注意是 'preferences' 复数，不是 'prefs' 单数）
  │   → 不会自动触发 loadPrefs()！
  │
  └─ 6. 客户端 Redux 中的 syncedPrefs 仍然是旧值
     → 用户需要切换预算或重新打开预算才会刷新
     → 但 spreadsheet 已经通过 setBudgetType() 实时更新了
```

> **设计洞察**：SyncedPrefs 的远程变更之所以不自动触发全量刷新，是因为：
> 1. 大部分 SyncedPrefs（如 `dateFormat`、`numberFormat`）不影响实时计算，只影响展示
> 2. `budgetType` 这种关键值有单独的特殊处理路径（`setBudgetType`）
> 3. 用"下次打开预算时全量重拉"换取同步路径的简洁性

---

## 六、异常场景与降级策略总结

| 异常场景 | 触发条件 | 降级行为 | 代码位置 |
|----------|----------|----------|----------|
| metadata.json 损坏 | JSON 解析失败 | 使用 `{ id, budgetName: id }` 最小配置继续 | [prefs.ts](packages/loot-core/src/server/prefs.ts#L32-L39) |
| metadata.json 中 id 与目录名不一致 | 用户手动移动文件夹 | 强制覆盖 `prefs.id = id`（以目录名为准） | [prefs.ts](packages/loot-core/src/server/prefs.ts#L41-L43) |
| 偏好迁移脚本执行失败 | 旧文件格式异常 | catch 吞掉异常，迁移中断但预算继续打开 | [1723665565000_prefs.js](packages/loot-core/migrations/1723665565000_prefs.js#L56-L58) |
| budgetType 旧值残留 | 迁移前数据库存了 `'report'` | 1745425408000 迁移规整为 `'tracking'` 或 `'envelope'` | [1745425408000_update_budgetType_pref.sql](packages/loot-core/migrations/1745425408000_update_budgetType_pref.sql#L3-L5) |
| budgetType 非法值绕过迁移 | 用户手动篡改 preferences 表 | 消费侧 `budgetTypePref === 'tracking' ? 'tracking' : 'envelope'` 收敛 | 典型见 [Spending.tsx](packages/desktop-client/src/components/reports/reports/Spending.tsx#L80-L82) |
| CSV 旧 key 残留 | 旧版本存了 `csv-skip-lines-*` | 1762178745667 迁移改名为 `csv-skip-start-lines-*` | [1762178745667_rename_csv_skip_lines_pref.sql](packages/loot-core/migrations/1762178745667_rename_csv_skip_lines_pref.sql#L3-L6) |
| CSV 偏好值为非数字 | 用户手改数据库存了乱码 | 消费侧 `parseInt(..., 10) \|\| 0` 收敛为 0 | 典型见 [ImportTransactionsModal.tsx](packages/desktop-client/src/components/modals/ImportTransactionsModal/ImportTransactionsModal.tsx#L245-L249) |
| CSV 布尔偏好值非法 | 用户手改数据库存了非 `'true'/'false'` | 消费侧 `String(...) === 'true'` 收敛为 false | 典型见 [ImportTransactionsModal.tsx](packages/desktop-client/src/components/modals/ImportTransactionsModal/ImportTransactionsModal.tsx#L251-L259) |
| 数据库迁移超前 | 用户回退 App 版本 | 抛 `out-of-sync-migrations`，UI 提示用户升级 | [migrations.ts](packages/loot-core/src/server/migrate/migrations.ts#L150-L159) |
| 迁移序列断裂 | __migrations__ 表记录与磁盘脚本不匹配 | 同上抛 `out-of-sync-migrations` | [migrations.ts](packages/loot-core/src/server/migrate/migrations.ts#L161-L175) |
| theme 配置非法值 | 用户手改 global-store.json | 回退到 `'auto'` 主题 | [preferences/app.ts](packages/loot-core/src/server/preferences/app.ts#L170-L176) |
| document-dir 路径不可访问 | 磁盘卸载 / 权限变更 | 回退到默认文档目录 | [main.ts](packages/loot-core/src/server/main.ts#L168-L174) |
| SyncedPrefs 字段缺失 | 旧版本数据库 | 消费侧 Hook / 查询解构赋默认值 | 分散各处，典型见 [budgetfiles/app.ts](packages/loot-core/src/server/budgetfiles/app.ts#L601) |
| _loadBudget 中途异常 | 任意步骤抛错 | 调用 `closeBudget()` 清理已加载资源，返回错误码 | [budgetfiles/app.ts](packages/loot-core/src/server/budgetfiles/app.ts#L536-L568) |
| SyncedPrefs 远程变更未刷新 | 其他设备修改了 SyncedPrefs | spreadsheet 实时更新 budgetType，其他值依赖下次 loadPrefs | [sync/index.ts](packages/loot-core/src/server/sync/index.ts#L367-L370) |

---

## 七、关键设计模式总结

1. **分层存储，按同步语义分类**：把偏好按同步需求拆到 4 个层级，避免了"一刀切"全同步或全本地的窘境

2. **消费侧就近默认值**：不在加载时集中注入，改为每个读点自行兜底——新增偏好字段零侵入

3. **迁移 = 最低保障原则**：迁移失败绝不阻塞主流程，偏好是"可重设"的，预算数据才是核心

4. **ID 真实性双来源**：目录名是 id 的唯一真相源（source of truth），metadata.json 的 id 字段仅作缓存，加载时强制矫正

5. **边界净化模式**：GlobalPrefs 读取时做枚举值合法性校验，把"脏数据"拦截在服务端边界，避免下游组件因非法值崩溃

6. **同步完成后全量重拉**：MetadataPrefs 变更时，CRDT 只传增量，Redux 却做全量刷新——用一次全量查询的代价换来状态一致性的简单性

7. **关键偏好特殊处理**：`budgetType` 这种影响核心计算的 SyncedPrefs 有独立的实时更新路径（`setBudgetType`），不依赖全量刷新

8. **Dataset 命名双轨制**：`'prefs'` vs `'preferences'` 的命名差异是历史演进的结果，前者承载 MetadataPrefs 的旧同步格式，后者承载 SyncedPrefs 的新同步格式，两者通过 `apply()` 函数中的条件分支实现完全隔离的处理逻辑

9. **迁移规整 + 消费侧兜底的双层防护**：`budgetType` 通过"SQL 迁移强制规整合法值 + 消费侧三目运算再次收敛"的双层机制，确保无论数据库处于什么状态，运行时都只会看到 `'envelope'` 或 `'tracking'`

10. **类型转换兜底模式**：CSV 偏好等字符串存储的数字/布尔值，通过 `parseInt(..., 10) || 0`、`String(...) === 'true'`、`String(...) !== 'false'` 等模式实现安全的类型转换，保证存储值异常时不会崩溃
