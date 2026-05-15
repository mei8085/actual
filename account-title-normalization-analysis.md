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

**处理逻辑**：
- 用户手动修改账户名称时调用
- 直接将 name 字段写入数据库

---

## 三、银行同步链路分析与冲突判定澄清

### 3.1 银行同步不更新账户名称的事实证据

**核心结论**：当前仓库中，**银行同步链路不会写入 `accounts.name` 字段**。

**代码证据**：查看 `processBankSyncDownload` 函数（`packages/loot-core/src/server/accounts/sync.ts:968-1098`）：

```typescript
async function processBankSyncDownload(
  download,
  id,
  acctRow,
  initialSync = false,
  customStartingBalance?: number,
  customStartingDate?: string,
) {
  // ... 初始化逻辑 ...

  const {
    transactions: originalTransactions,
    startingBalance: currentBalance,
  } = download;

  if (initialSync) {
    // 首次同步：创建初始余额交易 + 协调交易
    return runMutator(async () => {
      const initialId = await db.insertTransaction({
        account: id,
        amount: balanceToUse,
        // ... 交易字段 ...
      });

      const result = await reconcileTransactions(id, transactions, ...);
      return { ...result, added: [initialId, ...result.added] };
    });
  }

  // 非首次同步：仅协调交易和更新余额
  return runMutator(async () => {
    const result = await reconcileTransactions(id, transactions, ...);

    if (currentBalance != null) {
      await updateAccountBalance(id, currentBalance);  // 仅更新余额，不更新名称
    }

    return result;
  });
}
```

**函数职责分析**：

| 函数 | 职责 | 是否更新 accounts.name |
|-----|------|----------------------|
| `processBankSyncDownload` | 处理银行同步下载 | **否** |
| `reconcileTransactions` | 协调交易（匹配/新增） | **否** |
| `updateAccountBalance` | 更新账户余额 | **否** |
| `linkGoCardlessAccount` | 首次链接账户 | **是（仅首次）** |
| `updateAccount` | 用户手动修改账户 | **是** |

### 3.2 为什么不存在"银行同步改名 vs 用户手动改名"的并发冲突

**根本原因**：银行同步链路根本不修改账户名称，因此不存在冲突场景。

**时间线分析**：

```
T0: 用户首次链接银行账户
        │
        ▼
linkGoCardlessAccount() → db.insertWithUUID('accounts', { name: bankName })
        │
        ▼
accounts.name = "Checking Account" (银行返回的名称)

T1: 用户手动修改账户名称
        │
        ▼
updateAccount() → db.update('accounts', { id, name: "我的工资卡" })
        │
        ▼
accounts.name = "我的工资卡" (用户自定义名称)

T2: 银行同步执行
        │
        ▼
syncAccount() → processBankSyncDownload()
        │
        ▼
reconcileTransactions() + updateAccountBalance()
        │
        ▼
accounts.name 保持为 "我的工资卡"（不受影响）
```

### 3.3 真正存在的冲突场景

虽然银行同步不更新账户名称，但**多设备间的用户手动修改仍可能产生冲突**：

| 场景 | 触发条件 | 处理策略 |
|-----|---------|---------|
| **多设备并发修改** | 不同设备同时修改同一账户名称 | CRDT 时间戳仲裁 |
| **重复消息** | 同一修改被多次同步 | 时间戳去重 |

#### 冲突判定核心逻辑

```typescript
// packages/loot-core/src/server/sync/index.ts:197-222
async function compareMessages(messages: Message[]): Promise<Message[]> {
  const newMessages = [];

  for (const message of messages) {
    const { dataset, row, column, timestamp } = message;
    const timestampStr = timestamp.toString();

    // 查询数据库中是否存在更大或相等的时间戳
    const res = db.runQuery(
      'SELECT timestamp FROM messages_crdt WHERE dataset = ? AND row = ? AND column = ? AND timestamp >= ?',
      [dataset, row, column, timestampStr],
      true,
    );

    if (res.length === 0) {
      // 无冲突，应用变更
      newMessages.push(message);
    } else if (res[0].timestamp !== timestampStr) {
      // 存在更新的变更，标记为旧消息
      newMessages.push({ ...message, old: true });
    }
    // 时间戳相等：重复消息，丢弃
  }

  return newMessages;
}
```

#### 胜出规则

| 优先级 | 判定条件 | 结果 |
|-------|---------|------|
| 1 | 时间戳更大（`timestamp > existing`） | **获胜**，应用变更 |
| 2 | 时间戳相等（`timestamp === existing`） | **丢弃**，重复消息 |
| 3 | 时间戳更小（`timestamp < existing`） | **标记 old**，不应用但记录到 merkle |

---

## 四、CRDT 机制对账户标题变更的记录、传播与回放

### 4.1 变更记录

**触发点**：用户调用 `updateAccount()` 修改账户名称

```typescript
// packages/loot-core/src/server/db/index.ts:205-223
export async function update(table, params) {
  const fields = Object.keys(params).filter(k => k !== 'id');

  if (params.id == null) {
    throw new Error('update: id is required');
  }

  await sendMessages(
    fields.map(k => ({
      dataset: table,           // 'accounts'
      row: params.id,           // 账户 UUID
      column: k,                // 'name'
      value: params[k],         // 新账户名称
      timestamp: Timestamp.send(), // HULC 时间戳
    })),
  );
}
```

**时间戳生成**：

```typescript
// packages/crdt/src/crdt/timestamp.ts:195-230
static send(): Timestamp | null {
  const phys = Date.now();
  const lOld = clock.timestamp.millis();
  const cOld = clock.timestamp.counter();
  
  const lNew = Math.max(lOld, phys);           // 确保单调递增
  const cNew = lOld === lNew ? cOld + 1 : 0;   // 同一毫秒内递增计数器
  
  clock.timestamp.setMillis(lNew);
  clock.timestamp.setCounter(cNew);
  
  return new Timestamp(lNew, cNew, clock.timestamp.node());
}
```

### 4.2 消息应用与持久化

```typescript
// packages/loot-core/src/server/sync/index.ts:261-445
export const applyMessages = sequential(async (messages: Message[]) => {
  // 1. 冲突检测
  messages = await compareMessages(messages);

  // 2. 按时间戳排序
  messages = [...messages].sort((m1, m2) => {
    const t1 = m1.timestamp?.toString() || '';
    const t2 = m2.timestamp?.toString() || '';
    return t1 < t2 ? -1 : t1 > t2 ? 1 : 0;
  });

  // 3. 原子事务应用
  db.transaction(() => {
    for (const msg of messages) {
      const { dataset, row, column, timestamp, value } = msg;

      if (!msg.old) {
        // 应用到业务表 accounts
        apply(msg, getIn(oldData, [dataset, row]) || added.has(dataset + row));
      }

      // 记录到 CRDT 表
      db.runQuery(
        `INSERT INTO messages_crdt (timestamp, dataset, row, column, value)
         VALUES (?, ?, ?, ?, ?)`,
        [timestamp.toString(), dataset, row, column, serializeValue(value)],
      );

      // 更新 Merkle Tree
      currentMerkle = merkle.insert(currentMerkle, timestamp);
    }

    // 保存时钟状态
    db.runQuery(
      'INSERT OR REPLACE INTO messages_clock (id, clock) VALUES (1, ?)',
      [serializeClock({ ...clock, merkle: currentMerkle })],
    );
  });
});
```

### 4.3 传播到服务器

```typescript
// packages/loot-core/src/server/sync/index.ts:597-671
export const fullSync = once(async function () {
  // 获取需要同步的消息
  const messages = getMessagesSince(since);

  // 编码并发送到同步服务器
  const buffer = await encoder.encode(groupId, cloudFileId, since, messages);
  const resBuffer = await postBinary(getServer().SYNC_SERVER + '/sync', buffer);

  // 解码服务器响应
  const res = await encoder.decode(resBuffer);

  // 应用服务器返回的消息
  if (res.messages.length > 0) {
    receivedMessages = await receiveMessages(res.messages);
  }

  // 检查 Merkle Tree 差异
  const diffTime = merkle.diff(res.merkle, getClock().merkle);
  if (diffTime !== null) {
    return _fullSync(new Timestamp(diffTime, 0, '0').toString(), ...);
  }
});
```

### 4.4 其他客户端接收与回放

```typescript
// packages/loot-core/src/server/sync/index.ts:447-460
export function receiveMessages(messages: Message[]): Promise<Message[]> {
  // 合并远程时间戳到本地时钟
  try {
    messages.forEach(msg => {
      Timestamp.recv(msg.timestamp);
    });
  } catch (e) {
    if (e instanceof Timestamp.ClockDriftError) {
      throw new SyncError('clock-drift');
    }
    throw e;
  }

  // 应用消息（包括账户名称变更）
  return runMutator(() => applyMessages(messages));
}
```

### 4.5 桌面端最终展示依赖的状态来源

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

**状态来源优先级**：
1. **本地数据库**：`accounts.name` 字段（最新应用的变更）
2. **CRDT 消息**：`messages_crdt` 表记录变更历史
3. **同步服务器**：作为消息中继，不存储最终状态

---

## 五、非现状假设：未来支持银行同步改名

> **注意**：以下内容为假设性设计，**当前未实现**。

### 5.1 需求场景

当银行端账户名称发生变更时（如用户在银行 APP 中修改账户昵称），期望同步到 Actual。

### 5.2 设计方案

#### 方案一：自动覆盖（简单但有风险）

```typescript
async function processBankSyncDownload(download, id, acctRow, ...) {
  // ... 现有逻辑 ...

  // 新增：更新账户名称
  if (download.accountName && download.accountName !== acctRow.name) {
    await db.update('accounts', { id, name: download.accountName });
  }

  // ... 现有逻辑 ...
}
```

**问题**：可能覆盖用户自定义名称。

#### 方案二：用户确认机制（推荐）

```typescript
async function processBankSyncDownload(download, id, acctRow, ...) {
  // ... 现有逻辑 ...

  // 检测名称变更
  if (download.accountName && download.accountName !== acctRow.name) {
    // 检查是否为用户自定义名称
    const isCustomName = acctRow.official_name && 
      acctRow.name !== acctRow.official_name;

    if (isCustomName) {
      // 记录冲突，通知用户选择
      await db.insert('account_name_conflicts', {
        account_id: id,
        bank_name: download.accountName,
        user_name: acctRow.name,
        timestamp: Date.now(),
      });

      // 触发客户端通知
      connection.send('sync-event', {
        type: 'account-name-conflict',
        accountId: id,
        bankName: download.accountName,
        userName: acctRow.name,
      });
    } else {
      // 用户未自定义，自动更新
      await db.update('accounts', { id, name: download.accountName });
    }
  }

  // ... 现有逻辑 ...
}
```

#### 方案三：名称锁定机制

```typescript
// 在账户表中添加锁定标记
// ALTER TABLE accounts ADD COLUMN name_locked BOOLEAN DEFAULT FALSE;

async function processBankSyncDownload(download, id, acctRow, ...) {
  // ... 现有逻辑 ...

  if (download.accountName && download.accountName !== acctRow.name) {
    if (!acctRow.name_locked) {
      await db.update('accounts', { id, name: download.accountName });
    }
    // 已锁定：跳过更新
  }
}

// 用户手动修改时自动锁定
async function updateAccount({ id, name, last_reconciled }) {
  await db.update('accounts', { 
    id, 
    name, 
    name_locked: true,  // 锁定名称
    ...(last_reconciled && { last_reconciled }) 
  });
}
```

### 5.3 冲突解决策略对比

| 方案 | 优点 | 缺点 | 适用场景 |
|-----|------|------|---------|
| 自动覆盖 | 简单、无用户干预 | 可能覆盖用户自定义名称 | 银行名称权威性高的场景 |
| 用户确认 | 用户可控、体验友好 | 需要额外 UI 支持 | 注重用户体验的场景 |
| 名称锁定 | 一次锁定永久生效 | 用户可能忘记解锁 | 银行名称变更不频繁的场景 |

---

## 六、设计特点与架构考量

### 6.1 归一化职责分离

| 层级 | 职责 | 设计考量 |
|-----|------|---------|
| **Bank Sync** | 外部数据源适配 | 处理不同银行 API 的字段差异 |
| **loot-core** | 业务逻辑处理 | 保持标题化逻辑统一 |
| **CRDT** | 分布式一致性 | 确保多设备同步正确性 |

### 6.2 当前实现的隐式保护机制

- **账户名称保护**：银行同步不更新 `accounts.name`，用户自定义名称不会被覆盖
- **自动冲突解决**：CRDT 时间戳仲裁，无需用户干预
- **原子性保障**：所有变更在数据库事务中原子提交

### 6.3 性能优化

- **批量操作**：通过 `batchMessages()` 减少数据库往返
- **增量同步**：Merkle Tree 只同步变更部分

---

## 七、潜在问题与优化建议

### 7.1 当前问题

1. **标题化逻辑重复**：`title()` 函数在 `sync-server` 和 `loot-core` 中各有一份实现
2. **账户名称缺乏验证**：`updateAccount()` 直接接受 name 参数，无长度或格式校验

### 7.2 优化建议

#### 建议一：统一标题化函数位置

```typescript
// packages/shared/util/title.ts (共享模块)
export { title } from '#shared/util/title';
```

#### 建议二：增加账户名称验证

```typescript
async function updateAccount({ id, name, last_reconciled }) {
  if (name.length > 100) {
    throw new Error('Account name exceeds maximum length');
  }
  if (!name.trim()) {
    throw new Error('Account name cannot be empty');
  }
  await db.update('accounts', { id, name, ...(last_reconciled && { last_reconciled }) });
}
```

---

## 八、总结

### 核心事实

1. **账户名称设置时机**：仅在首次链接银行账户时设置，后续银行同步**不更新**账户名称
2. **用户修改路径**：通过 `updateAccount()` 手动修改，触发 CRDT 消息同步
3. **冲突场景范围**：仅存在于多设备间的用户并发修改，不存在"银行同步改名 vs 用户手动改名"的冲突

### CRDT 同步机制要点

| 阶段 | 机制 | 关键组件 |
|-----|------|---------|
| **记录** | `db.update()` → `sendMessages()` | HULC 时间戳 |
| **应用** | `applyMessages()` → `compareMessages()` | 冲突检测 |
| **传播** | `fullSync()` → 编码发送 | Merkle Tree |
| **回放** | `receiveMessages()` → `Timestamp.recv()` | 远程时钟合并 |

### 当前实现特点

- **隐式名称保护**：银行同步不会覆盖用户自定义名称
- **自动冲突解决**：完全依赖时间戳仲裁
- **无用户感知**：冲突解决过程对用户透明

### 未来优化方向

1. 统一 `title()` 函数位置
2. 增加账户名称验证
3. （可选）实现银行名称同步的用户确认机制