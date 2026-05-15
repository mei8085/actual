# 账户标题归一化逻辑分析报告

## 一、数据路径总览

账户标题从银行同步接入到桌面客户端展示的完整路径如下：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          Bank Sync 接入层                                   │
│  ┌─────────────┐  ┌─────────────────┐  ┌─────────────────┐                │
│  │ GoCardless  │  │ EnableBanking   │  │ SimpleFin/Pluggy│                │
│  │ integration │  │   integration   │  │    AI           │                │
│  └─────┬───────┘  └────────┬────────┘  └────────┬────────┘                │
│        │                   │                    │                          │
│        ▼                   ▼                    ▼                          │
│  ┌─────────────────────────────────────────────────────┐                   │
│  │              normalizeAccount() 归一化              │                   │
│  │  - name 字段组合: name/iban/currency/institution   │                   │
│  └──────────────────────────┬──────────────────────────┘                   │
└─────────────────────────────┼───────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          loot-core 层                                      │
│  ┌─────────────────────────────────────────────────────┐           │
│  │  linkGoCardlessAccount / linkSimpleFinAccount / ...        │           │
│  │  - db.insertWithUUID('accounts', { name: account.name })   │           │
│  └──────────────────────────┬──────────────────────────────────┘           │
│                             │                                             │
│                             ▼                                             │
│  ┌─────────────────────────────────────────────────────┐           │
│  │  updateAccount() (用户手动修改)                       │           │
│  │  - db.update('accounts', { id, name, ... })                │           │
│  └──────────────────────────┬──────────────────────────────────┘           │
└─────────────────────────────┼───────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          CRDT 变更追踪                                     │
│  ┌─────────────────────────────────────────────────────┐           │
│  │  db.update() → sendMessages() → applyMessages()    │           │
│  │  - HULC 时间戳生成 (Timestamp.send())               │           │
│  │  - messages_crdt 持久化                              │           │
│  └──────────────────────────┬──────────────────────────────────┘           │
└─────────────────────────────┼───────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                        桌面客户端展示层                                     │
│  ┌─────────────────────────────────────────────────────┐           │
│  │  getAccounts() → AccountEntity[]                    │           │
│  │  - 直接读取 accounts.name 字段用于展示               │           │
│  └─────────────────────────────────────────────────────┘           │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 二、各模块标题字段格式转换处理

### 2.1 Bank Sync 接入层

#### GoCardless 集成 (`packages/sync-server/src/app-gocardless/banks/integration-bank.js`)

```javascript
normalizeAccount(account) {
  return {
    account_id: account.id,
    institution: account.institution,
    mask: (account?.iban || '0000').slice(-4),
    iban: account?.iban || null,
    name: [
      account.name ?? account.displayName ?? account.product,
      printIban(account),
      account.currency,
    ]
      .filter(Boolean)
      .join(' '),
    official_name: account.product ?? `integration-${account.institution_id}`,
    type: 'checking',
  };
}
```

**处理逻辑**：
- **name 字段优先级**：`name` → `displayName` → `product`
- **组合规则**：`[名称] ([IBAN后4位]) [货币]`，如 `"Current Account (XXX 4321) EUR"`
- **降级策略**：当 IBAN 缺失时使用 `"0000"` 作为掩码

#### EnableBanking 集成 (`packages/sync-server/src/app-enablebanking/services/enablebanking-service.ts`)

```typescript
export function normalizeAccount(
  account: EnableBankingSessionAccount,
  aspsp?: { name?: string },
): NormalizedAccount {
  return {
    account_id: account.uid,
    name: account.name || account.account_id?.iban || account.uid,
    institution: aspsp?.name || account.account_servicer?.name || 'Unknown',
    currency: account.currency,
    iban: account.account_id?.iban,
  };
}
```

**处理逻辑**：
- **name 字段优先级**：`name` → `iban` → `uid`
- **机构名称优先级**：`aspsp.name` → `account_servicer.name` → `"Unknown"`

### 2.2 loot-core 层

#### 账户创建（首次链接）(`packages/loot-core/src/server/accounts/app.ts`)

```typescript
// linkGoCardlessAccount 中的账户创建逻辑
id = uuidv4();
await db.insertWithUUID('accounts', {
  id,
  account_id: account.account_id,
  mask: account.mask,
  name: account.name,           // 仅在此处设置名称
  official_name: account.official_name,
  bank: bank.id,
  offbudget: offBudget ? 1 : 0,
  account_sync_source: 'goCardless',
});
```

**关键事实**：账户名称仅在**首次链接银行账户时**写入数据库。

#### 账户更新（用户手动修改）(`packages/loot-core/src/server/accounts/app.ts`)

```typescript
async function updateAccount({
  id,
  name,
  last_reconciled,
}: Pick<AccountEntity, 'id' | 'name'> &
  Partial<Pick<AccountEntity, 'last_reconciled'>>) {
  await db.update('accounts', {
    id,
    name,
    ...(last_reconciled && { last_reconciled }),
  });
  return {};
}
```

---

## 三、银行同步链路分析与冲突判定澄清

### 3.1 银行同步不更新账户名称的事实证据

**核心结论**：当前仓库中，**银行同步链路不会写入 `accounts.name` 字段**。

**代码证据**：`processBankSyncDownload` 函数（`packages/loot-core/src/server/accounts/sync.ts:968-1098`）仅处理：
- 交易协调（`reconcileTransactions`）
- 余额更新（`updateAccountBalance`）

### 3.2 为什么不存在"银行同步改名 vs 用户手动改名"的并发冲突

**根本原因**：银行同步链路根本不修改账户名称，因此不存在冲突场景。

**时间线分析**：

```
T0: 用户首次链接银行账户
        │
        ▼
linkGoCardlessAccount() → db.insertWithUUID('accounts', { name: bankName })

T1: 用户手动修改账户名称
        │
        ▼
updateAccount() → db.update('accounts', { id, name: "我的工资卡" })

T2: 银行同步执行
        │
        ▼
syncAccount() → processBankSyncDownload()
        │
        ▼
accounts.name 保持为 "我的工资卡"（不受影响）
```

---

## 四、CRDT 机制对账户标题变更的记录、传播与回放

### 4.1 同步模式类型定义

```typescript
// packages/loot-core/src/server/sync/index.ts:41-42
let SYNCING_MODE = 'enabled';
type SyncingMode = 'enabled' | 'offline' | 'disabled' | 'import';
```

### 4.2 模式判定函数

```typescript
// packages/loot-core/src/server/sync/index.ts:65-78
export function checkSyncingMode(mode: SyncingMode): boolean {
  switch (mode) {
    case 'enabled':
      return SYNCING_MODE === 'enabled' || SYNCING_MODE === 'offline';
    case 'disabled':
      return SYNCING_MODE === 'disabled' || SYNCING_MODE === 'import';
    case 'offline':
      return SYNCING_MODE === 'offline';
    case 'import':
      return SYNCING_MODE === 'import';
    default:
      throw new Error('checkSyncingMode: invalid mode: ' + mode);
  }
}
```

**判定矩阵**：

| 当前模式 | `checkSyncingMode('enabled')` | `checkSyncingMode('offline')` | `checkSyncingMode('disabled')` | `checkSyncingMode('import')` |
|---------|------------------------------|------------------------------|------------------------------|----------------------------|
| `enabled` | **true** | false | false | false |
| `offline` | **true** | true | false | false |
| `disabled` | false | false | **true** | false |
| `import` | false | false | **true** | true |

### 4.3 `applyMessages` 分支逻辑

```typescript
// packages/loot-core/src/server/sync/index.ts:261-325
export const applyMessages = sequential(async (messages: Message[]) => {
  if (checkSyncingMode('import')) {
    // 分支一：import 模式
    applyMessagesForImport(messages);
    return undefined;
  } else if (checkSyncingMode('enabled')) {
    // 分支二：enabled 或 offline 模式
    messages = await compareMessages(messages);
  }

  messages = [...messages].sort(...);

  let clock;
  let currentMerkle;
  if (checkSyncingMode('enabled')) {
    clock = getClock();
    currentMerkle = clock.merkle;
  }
  // ...
});
```

### 4.4 各同步模式下 CRDT 行为对照表

| CRDT 操作 | `enabled` | `offline` | `disabled` | `import` |
|-----------|-----------|-----------|-----------|----------|
| **compareMessages** (冲突检测) | ✅ 发生 | ✅ 发生 | ❌ 不发生 | ❌ 不发生 |
| **messages_crdt 写入** | ✅ 发生 | ✅ 发生 | ❌ 不发生 | ❌ 不发生 |
| **merkle 更新** | ✅ 发生 | ✅ 发生 | ❌ 不发生 | ❌ 不发生 |
| **业务表更新** | ✅ 发生 | ✅ 发生 | ✅ 发生 | ✅ 发生 |
| **服务器同步** | ✅ 发生 | ❌ 不发生 | ❌ 不发生 | ❌ 不发生 |

### 4.5 现状实现证据链（按同步模式拆解）

#### 4.5.1 `enabled` 模式：完整 CRDT 流程

**场景**：用户手动修改账户名称，且开启了同步服务器

**完整流程**：

```
1. 用户调用 updateAccount()
        │
        ▼
2. db.update('accounts', { id, name }) 
   → sendMessages() 收集消息
        │
        ▼
3. Timestamp.send() 生成 HULC 时间戳
        │
        ▼
4. batchMessages() 批处理
        │
        ▼
5. applyMessages() 处理
   ┌─────────────────────────────────────────────────────────────┐
   │ 5.1 compareMessages() 冲突检测                              │
   │     - 查询 messages_crdt 表中是否有更新的时间戳               │
   │     - 如果 timestamp >= existing: 新消息，应用               │
   │     - 如果 timestamp < existing: 标记 old: true             │
   └─────────────────────────────────────────────────────────────┘
   ┌─────────────────────────────────────────────────────────────┐
   │ 5.2 messages_crdt 写入 (checkSyncingMode('enabled')=true)   │
   │     INSERT INTO messages_crdt (timestamp, dataset, row,     │
   │       column, value) VALUES (?, ?, ?, ?, ?)                 │
   └─────────────────────────────────────────────────────────────┘
   ┌─────────────────────────────────────────────────────────────┐
   │ 5.3 merkle 更新 (checkSyncingMode('enabled')=true)         │
   │     currentMerkle = merkle.insert(currentMerkle, timestamp) │
   │     currentMerkle = merkle.prune(currentMerkle)             │
   │     db.runQuery('INSERT OR REPLACE INTO messages_clock...') │
   └─────────────────────────────────────────────────────────────┘
        │
        ▼
6. scheduleFullSync()
   ┌─────────────────────────────────────────────────────────────┐
   │ checkSyncingMode('enabled') && !checkSyncingMode('offline') │
   │ = true && !false = true                                     │
   │ → 执行 fullSync() 发送到服务器                               │
   └─────────────────────────────────────────────────────────────┘
        │
        ▼
7. fullSync()
   - getMessagesSince(since) 获取本地消息
   - encoder.encode() 编码消息
   - POST /sync 发送到服务器
   - 接收服务器响应并 applyMessages()
```

**关键代码证据**：

```typescript
// packages/loot-core/src/server/sync/index.ts:357-362
if (checkSyncingMode('enabled')) {
  db.runQuery(
    db.cache(`INSERT INTO messages_crdt (timestamp, dataset, row, column, value)
     VALUES (?, ?, ?, ?, ?)`),
    [timestamp.toString(), dataset, row, column, serializeValue(value)],
  );
}

// packages/loot-core/src/server/sync/index.ts:373-383
if (checkSyncingMode('enabled')) {
  currentMerkle = merkle.prune(currentMerkle);
  db.runQuery(
    'INSERT OR REPLACE INTO messages_clock (id, clock) VALUES (1, ?)',
    [serializeClock({ ...clock, merkle: currentMerkle })],
  );
}

// packages/loot-core/src/server/sync/index.ts:555
if (checkSyncingMode('enabled') && !checkSyncingMode('offline')) {
  return fullSync().then(...);
}
```

#### 4.5.2 `offline` 模式：本地 CRDT 但不同步到服务器

**场景**：用户手动修改账户名称，但网络不可用或用户选择离线模式

**与 enabled 模式的区别**：

| 操作 | `enabled` | `offline` |
|-----|----------|-----------|
| compareMessages | ✅ | ✅ |
| messages_crdt 写入 | ✅ | ✅ |
| merkle 更新 | ✅ | ✅ |
| 业务表更新 | ✅ | ✅ |
| 服务器同步 | ✅ | ❌ |

**关键代码证据**：

```typescript
// packages/loot-core/src/server/sync/index.ts:555
if (checkSyncingMode('enabled') && !checkSyncingMode('offline')) {
  // offline 模式下 !checkSyncingMode('offline') = !true = false
  // 条件不满足，不执行 fullSync()
}
```

**离线消息积累**：

```typescript
// packages/loot-core/src/server/sync/index.ts:687-692
export function getMessagesSince(since: string): Message[] {
  if (
    checkSyncingMode('disabled') ||
    checkSyncingMode('offline') ||    // <-- offline 模式返回空数组
    !currentId
  ) {
    return [];
  }
  // ...
}
```

**流程总结**：
```
用户修改账户名称 → CRDT 消息写入本地 messages_crdt
                              ↓
               恢复网络后调用 scheduleFullSync()
                              ↓
               offline → enabled 模式切换
                              ↓
               fullSync() 发送积累的消息到服务器
```

#### 4.5.3 `import` 模式：绕过 CRDT 直接应用

**场景**：导入 QIF/OFX 文件时的处理

**关键代码证据**：

```typescript
// packages/loot-core/src/server/sync/index.ts:262-264
if (checkSyncingMode('import')) {
  applyMessagesForImport(messages);
  return undefined;  // 直接返回，不执行后续 CRDT 逻辑
}

// packages/loot-core/src/server/sync/index.ts:231-250
function applyMessagesForImport(messages: Message[]): void {
  db.transaction(() => {
    for (let i = 0; i < messages.length; i++) {
      const msg = messages[i];
      const { dataset } = msg;

      if (!msg.old) {
        try {
          apply(msg);  // 直接应用，不经过 CRDT
        } catch {
          apply(msg, true);  // UPDATE 模式
        }

        if (dataset === 'prefs') {
          throw new Error('Cannot set prefs while importing');
        }
      }
    }
  });
}
```

**import 模式特点**：

| 特性 | 说明 |
|-----|------|
| messages_crdt 写入 | ❌ 不发生 |
| merkle 更新 | ❌ 不发生 |
| compareMessages | ❌ 不发生 |
| 业务表更新 | ✅ 直接 apply |
| 服务器同步 | ❌ 不发生 |

**注意事项**（代码注释）：

```typescript
// packages/loot-core/src/server/sync/index.ts:225-230
// This is the fast path `apply` function when in "import" mode.
// There's no need to run through the whole sync system when
// importing, but **there is a caveat**: because we don't run sync
// listeners importers should not rely on any functions that use any
// projected state (like rules). We can't fire those because they
// depend on having both old and new data which we don't query here
```

#### 4.5.4 `disabled` 模式：仅本地更新，无任何 CRDT 追踪

**场景**：文件未关联到同步服务器（如使用"不使用服务器"模式）

**关键代码证据**：

```typescript
// packages/loot-core/src/server/sync/index.ts:261-270
export const applyMessages = sequential(async (messages: Message[]) => {
  if (checkSyncingMode('import')) {
    applyMessagesForImport(messages);
    return undefined;
  } else if (checkSyncingMode('enabled')) {
    // disabled 模式下 checkSyncingMode('enabled') = false
    // 不进入此分支，不执行 compareMessages
    messages = await compareMessages(messages);
  }
  // ...
});
```

**disabled 模式特点**：

| 特性 | 说明 |
|-----|------|
| messages_crdt 写入 | ❌ 不发生 |
| merkle 更新 | ❌ 不发生 |
| compareMessages | ❌ 不发生 |
| 业务表更新 | ✅ 发生（但不走 applyMessages） |
| 服务器同步 | ❌ 不发生 |

**实际行为**：`disabled` 模式下，`sendMessages()` 收集的消息会直接通过底层的 `apply()` 函数应用到业务表，**完全不经过** `applyMessages()`，因此没有任何 CRDT 追踪。

### 4.6 冲突检测核心逻辑（`compareMessages`）

```typescript
// packages/loot-core/src/server/sync/index.ts:197-222
async function compareMessages(messages: Message[]): Promise<Message[]> {
  const newMessages = [];

  for (const message of messages) {
    const { dataset, row, column, timestamp } = message;
    const timestampStr = timestamp.toString();

    // 查询 messages_crdt 表中是否存在 >= 此时间戳的记录
    const res = db.runQuery(
      'SELECT timestamp FROM messages_crdt WHERE dataset = ? AND row = ? AND column = ? AND timestamp >= ?',
      [dataset, row, column, timestampStr],
      true,
    );

    if (res.length === 0) {
      // 无冲突记录 → 新消息，需要应用
      newMessages.push(message);
    } else if (res[0].timestamp !== timestampStr) {
      // 存在更新的记录 → 标记为 old
      newMessages.push({ ...message, old: true });
    }
    // 时间戳相等 → 重复消息，丢弃
  }

  return newMessages;
}
```

### 4.7 桌面端最终展示依赖的状态来源

```
桌面客户端展示
        │
        ▼
调用 getAccounts() API
        │
        ▼
loot-core: getAccounts()
        │
        ▼
db.getAccounts() → SELECT * FROM accounts
        │
        ▼
返回 AccountEntity[]
        │
        ▼
客户端直接使用 account.name 字段展示
```

**状态来源**：`accounts.name` 字段（最新应用的变更）

---

## 五、总结

### 同步模式与 CRDT 行为矩阵

| 模式 | compareMessages | messages_crdt 写入 | merkle 更新 | 服务器同步 | 典型场景 |
|-----|----------------|-------------------|------------|-----------|---------|
| `enabled` | ✅ | ✅ | ✅ | ✅ | 正常同步 |
| `offline` | ✅ | ✅ | ✅ | ❌ | 网络断开 |
| `disabled` | ❌ | ❌ | ❌ | ❌ | 不使用服务器 |
| `import` | ❌ | ❌ | ❌ | ❌ | 导入文件 |

### 账户名称变更路径

1. **首次链接**：`linkGoCardlessAccount()` → `db.insertWithUUID()` → 账户名称写入
2. **用户手动修改**：`updateAccount()` → `db.update()` → `sendMessages()` → `applyMessages()`
3. **银行同步**：`processBankSyncDownload()` → **不更新账户名称**

### 冲突场景

- **不存在**：银行同步改名 vs 用户手动改名的并发冲突（因为银行同步不更新名称）
- **存在**：多设备间用户并发修改同一账户名称（通过 CRDT 时间戳仲裁）

### 核心代码位置

| 功能 | 文件位置 |
|-----|---------|
| 同步模式定义 | `packages/loot-core/src/server/sync/index.ts:41-42` |
| 模式判定函数 | `packages/loot-core/src/server/sync/index.ts:65-78` |
| applyMessages 主逻辑 | `packages/loot-core/src/server/sync/index.ts:261-392` |
| compareMessages | `packages/loot-core/src/server/sync/index.ts:197-222` |
| applyMessagesForImport | `packages/loot-core/src/server/sync/index.ts:231-250` |
| 账户更新 | `packages/loot-core/src/server/accounts/app.ts` (updateAccount 函数) |
| 银行同步处理 | `packages/loot-core/src/server/accounts/sync.ts:968-1098` |