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
│  ┌─────────────────────────────────────────────┐                   │
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
│  │  db.update() → sendMessages() → _sendMessages()    │           │
│  │              → applyMessages()                      │           │
│  │  - HULC 时间戳生成 (Timestamp.send())               │           │
│  │  - messages_crdt 持久化 (仅 enabled/offline)        │           │
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

### 4.3 消息发送调用链

```typescript
// packages/loot-core/src/server/sync/index.ts:526-532
export async function sendMessages(messages: Message[]) {
  if (IS_BATCHING) {
    _BATCHED = _BATCHED.concat(messages);
  } else {
    return _sendMessages(messages);
  }
}

// packages/loot-core/src/server/sync/index.ts:488-494
async function _sendMessages(messages: Message[]): Promise<void> {
  try {
    await applyMessages(messages);
  } catch (e) {
    void errorHandler(e);
    throw e;
  }
}
```

**调用链**：`sendMessages()` → `_sendMessages()` → `applyMessages()`

### 4.4 `applyMessages` 分支逻辑

```typescript
// packages/loot-core/src/server/sync/index.ts:261-271
export const applyMessages = sequential(async (messages: Message[]) => {
  if (checkSyncingMode('import')) {
    // 分支一：import 模式
    applyMessagesForImport(messages);
    return undefined;
  } else if (checkSyncingMode('enabled')) {
    // 分支二：enabled 或 offline 模式
    messages = await compareMessages(messages);
  }
  // 分支三：disabled 模式（不进入上述任何分支）
  // ...
});
```

### 4.5 各同步模式下 CRDT 行为对照表（源码可印证）

| CRDT 操作 | `enabled` | `offline` | `disabled` | `import` | 源码位置 |
|-----------|-----------|-----------|-----------|----------|---------|
| **compareMessages** | ✅ | ✅ | ❌ | ❌ | `packages/loot-core/src/server/sync/index.ts:265` |
| **messages_crdt 写入** | ✅ | ✅ | ❌ | ❌ | `packages/loot-core/src/server/sync/index.ts:357` |
| **merkle 更新** | ✅ | ✅ | ❌ | ❌ | `packages/loot-core/src/server/sync/index.ts:364,373` |
| **业务表更新 (apply)** | ✅ | ✅ | ✅ | ✅ | `packages/loot-core/src/server/sync/index.ts:344` |
| **服务器同步** | ✅ | ❌ | ❌ | ❌ | `packages/loot-core/src/server/sync/index.ts:555` |

### 4.6 现状实现证据链（按同步模式拆解）

#### 4.6.1 `enabled` 模式：完整 CRDT 流程

**代码证据**：

```typescript
// packages/loot-core/src/server/sync/index.ts:265-270
} else if (checkSyncingMode('enabled')) {
  messages = await compareMessages(messages);
}

// packages/loot-core/src/server/sync/index.ts:357-364
if (checkSyncingMode('enabled')) {
  db.runQuery(
    db.cache(`INSERT INTO messages_crdt (timestamp, dataset, row, column, value)
     VALUES (?, ?, ?, ?, ?)`),
    [timestamp.toString(), dataset, row, column, serializeValue(value)],
  );
  currentMerkle = merkle.insert(currentMerkle, timestamp);
}

// packages/loot-core/src/server/sync/index.ts:373-383
if (checkSyncingMode('enabled')) {
  currentMerkle = merkle.prune(currentMerkle);
  db.runQuery(
    db.cache('INSERT OR REPLACE INTO messages_clock (id, clock) VALUES (1, ?)'),
    [serializeClock({ ...clock, merkle: currentMerkle })],
  );
}

// packages/loot-core/src/server/sync/index.ts:555
if (checkSyncingMode('enabled') && !checkSyncingMode('offline')) {
  return fullSync().then(...);
}
```

**流程**：
1. `checkSyncingMode('enabled') = true` → 执行 `compareMessages`
2. `checkSyncingMode('enabled') = true` → 写入 `messages_crdt`
3. `checkSyncingMode('enabled') = true` → 更新 merkle 和 `messages_clock`
4. `checkSyncingMode('enabled') && !checkSyncingMode('offline') = true && !false = true` → 执行 `fullSync`

#### 4.6.2 `offline` 模式：本地 CRDT 但不同步到服务器

**代码证据**：

```typescript
// packages/loot-core/src/server/sync/index.ts:67-68
case 'enabled':
  return SYNCING_MODE === 'enabled' || SYNCING_MODE === 'offline';  // offline 返回 true

// packages/loot-core/src/server/sync/index.ts:555
if (checkSyncingMode('enabled') && !checkSyncingMode('offline')) {
  // offline 模式下 !checkSyncingMode('offline') = !true = false
  // 条件不满足，不执行 fullSync()
}
```

**流程**：
1. `checkSyncingMode('enabled') = true` → 执行 `compareMessages`
2. `checkSyncingMode('enabled') = true` → 写入 `messages_crdt`
3. `checkSyncingMode('enabled') = true` → 更新 merkle 和 `messages_clock`
4. `checkSyncingMode('enabled') && !checkSyncingMode('offline') = true && !true = false` → 不执行 `fullSync`

#### 4.6.3 `import` 模式：快速路径，跳过 CRDT

**代码证据**：

```typescript
// packages/loot-core/src/server/sync/index.ts:262-264
if (checkSyncingMode('import')) {
  applyMessagesForImport(messages);
  return undefined;  // 直接返回，不执行后续逻辑
}

// packages/loot-core/src/server/sync/index.ts:231-250
function applyMessagesForImport(messages: Message[]): void {
  db.transaction(() => {
    for (let i = 0; i < messages.length; i++) {
      const msg = messages[i];
      if (!msg.old) {
        try {
          apply(msg);
        } catch {
          apply(msg, true);
        }
      }
    }
  });
}
```

**流程**：
1. `checkSyncingMode('import') = true` → 调用 `applyMessagesForImport`
2. 直接返回，不执行 `compareMessages`、`messages_crdt` 写入、`merkle` 更新

#### 4.6.4 `disabled` 模式：进入 applyMessages 但跳过 CRDT 追踪

**代码证据**：

```typescript
// packages/loot-core/src/server/sync/index.ts:261-271
export const applyMessages = sequential(async (messages: Message[]) => {
  if (checkSyncingMode('import')) {
    // disabled 模式: checkSyncingMode('import') = false → 不进入
    applyMessagesForImport(messages);
    return undefined;
  } else if (checkSyncingMode('enabled')) {
    // disabled 模式: checkSyncingMode('enabled') = false → 不进入
    messages = await compareMessages(messages);
  }
  // disabled 模式: 继续执行到这里
  messages = [...messages].sort(...);
  // ...
});

// packages/loot-core/src/server/sync/index.ts:344
if (!msg.old) {
  apply(msg, getIn(oldData, [dataset, row]) || added.has(dataset + row));
}

// packages/loot-core/src/server/sync/index.ts:357
if (checkSyncingMode('enabled')) {
  // disabled 模式: checkSyncingMode('enabled') = false → 跳过
  db.runQuery(...);  // 不执行
}
```

**流程**：
1. `checkSyncingMode('import') = false` → 不进入 import 分支
2. `checkSyncingMode('enabled') = false` → 不执行 `compareMessages`
3. 消息排序继续执行
4. `!msg.old = true` → 执行 `apply()` 更新业务表
5. `checkSyncingMode('enabled') = false` → 跳过 `messages_crdt` 写入
6. `checkSyncingMode('enabled') = false` → 跳过 `merkle` 更新

### 4.7 冲突检测核心逻辑（`compareMessages`）

```typescript
// packages/loot-core/src/server/sync/index.ts:197-222
async function compareMessages(messages: Message[]): Promise<Message[]> {
  const newMessages = [];

  for (const message of messages) {
    const { dataset, row, column, timestamp } = message;
    const timestampStr = timestamp.toString();

    const res = db.runQuery(
      'SELECT timestamp FROM messages_crdt WHERE dataset = ? AND row = ? AND column = ? AND timestamp >= ?',
      [dataset, row, column, timestampStr],
      true,
    );

    if (res.length === 0) {
      newMessages.push(message);
    } else if (res[0].timestamp !== timestampStr) {
      newMessages.push({ ...message, old: true });
    }
    // 时间戳相等 → 重复消息，丢弃
  }

  return newMessages;
}
```

### 4.8 桌面端最终展示依赖的状态来源

**代码证据**：客户端调用 `getAccounts()` → `db.getAccounts()` → `SELECT * FROM accounts` → 返回 `AccountEntity[]` → 使用 `account.name` 字段展示。

---

## 五、总结

### 同步模式与 CRDT 行为矩阵（源码可直接印证）

| 模式 | compareMessages | messages_crdt 写入 | merkle 更新 | 业务表更新 | 服务器同步 |
|-----|----------------|-------------------|------------|-----------|-----------|
| `enabled` | ✅ (L265) | ✅ (L357) | ✅ (L364,373) | ✅ (L344) | ✅ (L555) |
| `offline` | ✅ (L265) | ✅ (L357) | ✅ (L364,373) | ✅ (L344) | ❌ (L555) |
| `disabled` | ❌ (L265) | ❌ (L357) | ❌ (L364,373) | ✅ (L344) | ❌ (L555) |
| `import` | ❌ (L262) | ❌ (L262) | ❌ (L262) | ✅ (L239) | ❌ (L555) |

**说明**：括号内数字为 `packages/loot-core/src/server/sync/index.ts` 中的行号。

### 核心调用链

```
db.update() → sendMessages() → _sendMessages() → applyMessages()
                                                    │
                      ┌──────────────────────────────┼──────────────────────────────┐
                      ▼                              ▼                              ▼
            import 模式                        enabled/offline 模式              disabled 模式
            applyMessagesForImport()           compareMessages()                 跳过 compareMessages
            直接 apply()                        messages_crdt 写入               messages_crdt 不写入
            返回                                merkle 更新                       merkle 不更新
                                               业务表更新                        业务表更新
                                               服务器同步(仅 enabled)            服务器不同步
```

### 冲突场景

- **不存在**：银行同步改名 vs 用户手动改名（银行同步不更新名称）
- **存在**：多设备间用户并发修改（通过 CRDT 时间戳仲裁）

### 核心代码位置

| 功能 | 文件位置 |
|-----|---------|
| 同步模式定义 | `packages/loot-core/src/server/sync/index.ts:41-42` |
| 模式判定函数 | `packages/loot-core/src/server/sync/index.ts:65-78` |
| sendMessages | `packages/loot-core/src/server/sync/index.ts:526-532` |
| _sendMessages | `packages/loot-core/src/server/sync/index.ts:488-494` |
| applyMessages | `packages/loot-core/src/server/sync/index.ts:261-392` |
| compareMessages | `packages/loot-core/src/server/sync/index.ts:197-222` |
| applyMessagesForImport | `packages/loot-core/src/server/sync/index.ts:231-250` |
| 账户更新 | `packages/loot-core/src/server/accounts/app.ts` (updateAccount) |
| 银行同步处理 | `packages/loot-core/src/server/accounts/sync.ts:968-1098` |