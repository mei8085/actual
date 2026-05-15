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
│  ┌─────────────────────────────────────────────────────────────┐           │
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
│  ┌─────────────────────────────────────────────────────────────┐           │
│  │  updateAccount()                                           │           │
│  │  - db.update('accounts', { id, name, ... })                │           │
│  └──────────────────────────┬──────────────────────────────────┘           │
│                             │                                             │
│                             ▼                                             │
│  ┌─────────────────────────────────────────────────────────────┐           │
│  │  normalizeTransactions() / normalizeBankSyncTransactions() │           │
│  │  - payee_name 通过 title() 函数进行标题化                   │           │
│  └──────────────────────────┬──────────────────────────────────┘           │
└─────────────────────────────┼───────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          CRDT 变更追踪                                     │
│  ┌─────────────────────────────────────────────────────┐           │
│  │  batchMessages() → batchUpdateTransactions()               │           │
│  │  - HULC 时间戳生成 (Timestamp.send())                       │           │
│  │  - Merkle Tree 增量同步                                     │           │
│  └──────────────────────────┬──────────────────────────────────┘           │
└─────────────────────────────┼───────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                        桌面客户端展示层                                     │
│  ┌─────────────────────────────────────────────────────┐           │
│  │  getAccounts() → AccountEntity[]                            │           │
│  │  - 直接读取 accounts.name 字段用于展示                       │           │
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

#### 账户创建与更新 (`packages/loot-core/src/server/accounts/app.ts`)

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
- 直接将 name 字段写入数据库，**不进行额外的格式转换**
- 信任上游 Bank Sync 层的归一化结果

#### Payee 名称标题化 (`packages/loot-core/src/server/accounts/sync.ts`)

```typescript
async function normalizeTransactions(
  transactions,
  acctId,
  { rawPayeeName = false } = {},
) {
  // ...
  let payee_name = originalPayeeName;
  if (payee_name) {
    const trimmed = payee_name.trim();
    if (trimmed === '') {
      payee_name = null;
    } else {
      payee_name = rawPayeeName ? trimmed : title(trimmed);
    }
  }
  // ...
}
```

**处理逻辑**：
- 对 payee_name 进行 `trim()` 处理
- 空字符串转换为 `null`
- 通过 `title()` 函数进行标题化（首字母大写）

#### Title 标题化函数 (`packages/loot-core/src/server/accounts/title/index.ts`)

```typescript
export function title(str, options = { special: undefined }) {
  str = str
    .toLowerCase()
    .replace(regex, (m, lead = '', forced, lower, rest) => {
      const parsedMatch = parseMatch(m);
      if (!parsedMatch) {
        return m;
      }
      if (!forced) {
        const fullLower = lower + rest;
        if (lowerCaseSet.has(fullLower)) {
          return parsedMatch;
        }
      }
      return lead + (lower || forced).toUpperCase() + rest;
    });

  const customSpecials = options.special || [];
  const replace = [...specials, ...customSpecials];
  const replaceRegExp = convertToRegExp(replace);

  replaceRegExp.forEach(([pattern, s]) => {
    str = str.replace(pattern, s);
  });

  return str;
}
```

**处理逻辑**：
1. **小写转换**：先将整个字符串转为小写
2. **首字母大写**：匹配单词边界，将首字母转为大写
3. **特殊词保护**：通过 `lowerCaseSet` 保持小写词（如 `and`, `or`, `the`）
4. **特殊词替换**：通过 `specials` 数组恢复特定词汇的大小写（如 `USA`, `EU`）

---

## 三、账户名称冲突判定路径（银行同步 vs 用户手动改名）

### 3.1 写库字段

账户名称存储在 `accounts` 表的 `name` 字段：

| 字段名 | 类型 | 说明 |
|-------|------|------|
| `id` | TEXT | 账户唯一标识（UUID） |
| `name` | TEXT | 账户名称 |
| `account_id` | TEXT | 银行端账户 ID |
| `bank` | TEXT | 关联的银行 ID |
| `account_sync_source` | TEXT | 同步来源（goCardless/simpleFin/pluggyai/enableBanking） |

### 3.2 同步消息结构

当 `db.update('accounts', { id, name })` 被调用时，自动生成 CRDT 消息：

```typescript
// packages/loot-core/src/server/db/index.ts:205-223
export async function update(table, params) {
  const fields = Object.keys(params).filter(k => k !== 'id');

  if (params.id == null) {
    throw new Error('update: id is required');
  }

  await sendMessages(
    fields.map(k => {
      return {
        dataset: table,           // 'accounts'
        row: params.id,           // 账户 UUID
        column: k,                // 'name'
        value: params[k],         // 新账户名称
        timestamp: Timestamp.send(), // HULC 时间戳
      };
    }),
  );
}
```

**消息结构**：
```typescript
type Message = {
  column: string;              // 'name'
  dataset: string;             // 'accounts'
  row: string;                 // 账户 ID
  timestamp: Timestamp;        // HULC 时间戳
  value: string | number | null; // 新名称
};
```

### 3.3 冲突判定与胜出规则

#### 场景一：银行同步更新账户名称

```
银行 API 返回新名称
        │
        ▼
linkGoCardlessAccount() / 同步更新逻辑
        │
        ▼
db.update('accounts', { id, name: newBankName })
        │
        ▼
sendMessages([{ dataset: 'accounts', row: id, column: 'name', value: newBankName, timestamp: T1 }])
        │
        ▼
applyMessages() → compareMessages() → 写入 messages_crdt
```

#### 场景二：用户手动修改账户名称

```
用户在客户端编辑账户名称
        │
        ▼
dispatch updateAccount action
        │
        ▼
send('account-update', { id, name: newUserName })
        │
        ▼
updateAccount() handler
        │
        ▼
db.update('accounts', { id, name: newUserName })
        │
        ▼
sendMessages([{ dataset: 'accounts', row: id, column: 'name', value: newUserName, timestamp: T2 }])
        │
        ▼
applyMessages() → compareMessages() → 写入 messages_crdt
```

#### 冲突判定核心逻辑

```typescript
// packages/loot-core/src/server/sync/index.ts:197-222
async function compareMessages(messages: Message[]): Promise<Message[]> {
  const newMessages = [];

  for (let i = 0; i < messages.length; i++) {
    const message = messages[i];
    const { dataset, row, column, timestamp } = message;
    const timestampStr = timestamp.toString();

    // 查询数据库中是否存在更大或相等的时间戳
    const res = db.runQuery<Pick<db.DbCrdtMessage, 'timestamp'>>(
      db.cache(
        'SELECT timestamp FROM messages_crdt WHERE dataset = ? AND row = ? AND column = ? AND timestamp >= ?',
      ),
      [dataset, row, column, timestampStr],
      true,
    );

    // 如果没有结果 → 这是新消息，需要应用
    if (res.length === 0) {
      newMessages.push(message);
    } 
    // 如果有结果但时间戳不同 → 这是旧消息，标记为 old
    else if (res[0].timestamp !== timestampStr) {
      newMessages.push({ ...message, old: true });
    }
    // 如果有结果且时间戳相同 → 重复消息，丢弃
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

#### 时间戳比较细节

```typescript
// HULC 时间戳格式: 2024-01-15T10:30:00.123Z-1000-0123456789ABCDEF
//                              ↑毫秒      ↑计数器   ↑节点ID

// 比较逻辑：字符串字典序比较
// T1: 2024-01-15T10:30:00.123Z-0001-A1B2C3D4E5F67890
// T2: 2024-01-15T10:30:00.124Z-0000-B1C2D3E4F567890A
// T2 > T1 → T2 获胜
```

---

## 四、CRDT 机制对账户标题变更的记录、传播与回放

### 4.1 现状实现证据链

#### 阶段一：变更记录

**触发点**：`db.update()` 调用

```typescript
// packages/loot-core/src/server/db/index.ts:212-222
await sendMessages(
  fields.map(k => ({
    dataset: table,
    row: params.id,
    column: k,
    value: params[k],
    timestamp: Timestamp.send(),  // 生成 HULC 时间戳
  })),
);
```

**时间戳生成**：

```typescript
// packages/crdt/src/crdt/timestamp.ts:195-230
static send(): Timestamp | null {
  const phys = Date.now();           // 物理时间
  const lOld = clock.timestamp.millis();
  const cOld = clock.timestamp.counter();
  
  const lNew = Math.max(lOld, phys);           // 取最大值确保单调递增
  const cNew = lOld === lNew ? cOld + 1 : 0;   // 同一毫秒内递增计数器
  
  // 更新本地时钟状态
  clock.timestamp.setMillis(lNew);
  clock.timestamp.setCounter(cNew);
  
  return new Timestamp(lNew, cNew, clock.timestamp.node());
}
```

#### 阶段二：消息批处理

```typescript
// packages/loot-core/src/server/sync/index.ts:501-524
let IS_BATCHING = false;
let _BATCHED: Message[] = [];

export async function batchMessages(func: () => Promise<void>): Promise<void> {
  if (IS_BATCHING) {
    await func();
    return;
  }

  IS_BATCHING = true;
  let batched: Message[] = [];

  try {
    await func();  // 执行数据库操作，消息被收集到 _BATCHED
  } catch (e) {
    void errorHandler(e);
    throw e;
  } finally {
    IS_BATCHING = false;
    batched = _BATCHED;
    _BATCHED = [];
  }

  if (batched.length > 0) {
    await _sendMessages(batched);  // 批量发送
  }
}
```

#### 阶段三：消息应用与持久化

```typescript
// packages/loot-core/src/server/sync/index.ts:261-445
export const applyMessages = sequential(async (messages: Message[]) => {
  // 1. 冲突检测
  if (checkSyncingMode('enabled')) {
    messages = await compareMessages(messages);
  }

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
        // 应用到业务表
        apply(msg, getIn(oldData, [dataset, row]) || added.has(dataset + row));
      }

      // 记录到 CRDT 表
      if (checkSyncingMode('enabled')) {
        db.runQuery(
          db.cache(`INSERT INTO messages_crdt (timestamp, dataset, row, column, value)
           VALUES (?, ?, ?, ?, ?)`),
          [timestamp.toString(), dataset, row, column, serializeValue(value)],
        );

        // 更新 Merkle Tree
        currentMerkle = merkle.insert(currentMerkle, timestamp);
      }
    }

    // 保存时钟状态
    db.runQuery(
      db.cache('INSERT OR REPLACE INTO messages_clock (id, clock) VALUES (1, ?)'),
      [serializeClock({ ...clock, merkle: currentMerkle })],
    );
  });

  // ... 触发事件通知
});
```

**数据库表结构**（`packages/loot-core/src/server/sql/init.sql`）：

```sql
CREATE TABLE messages_crdt (
  timestamp TEXT PRIMARY KEY,
  dataset TEXT,
  row TEXT,
  column TEXT,
  value TEXT
);

CREATE TABLE messages_clock (
  id INTEGER PRIMARY KEY,
  clock TEXT
);
```

#### 阶段四：传播到服务器

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

  // 检查 Merkle Tree 差异，确保完全同步
  const diffTime = merkle.diff(res.merkle, getClock().merkle);
  if (diffTime !== null) {
    // 递归同步直到一致
    return _fullSync(new Timestamp(diffTime, 0, '0').toString(), ...);
  }
});
```

#### 阶段五：其他客户端接收与回放

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

  // 应用消息
  return runMutator(() => applyMessages(messages));
}
```

### 4.2 未实现缺口

#### 缺口一：账户名称变更无特殊冲突处理

**现状**：账户名称变更与其他字段（如 `balance_current`）处理方式完全相同

**问题**：
- 银行同步可能覆盖用户自定义的账户名称
- 用户无法知道名称被自动修改
- 没有版本历史或回滚机制

#### 缺口二：缺乏名称锁定机制

**场景**：
```
用户将 "Checking Account" 改名为 "我的工资卡"
        │
银行同步返回名称 "Checking Account"（时间戳更新）
        │
结果：用户自定义名称被覆盖，无任何提示
```

**期望行为**：
- 检测到用户已修改名称后，银行同步不应自动覆盖
- 提供用户选择：保持自定义名称 / 使用银行名称 / 合并两者

#### 缺口三：无冲突可视化提示

**现状**：冲突完全由时间戳自动仲裁，用户无感知

**期望行为**：
- 在 UI 中标记发生冲突的账户
- 显示冲突的两个版本
- 提供手动解决冲突的选项

#### 缺口四：缺乏变更审计日志

**现状**：`messages_crdt` 表仅用于同步，不保留完整历史

**期望行为**：
- 记录每次名称变更的时间、来源（用户/银行）、变更前值
- 支持查看历史变更记录

### 4.3 桌面端最终展示依赖的状态来源

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

## 五、数据流详细分析

### 5.1 账户创建流程

```
银行 API 返回账户数据
        │
        ▼
normalizeAccount() (GoCardless/EnableBanking)
        │
        ▼
组合 name = [name/displayName/product] + [IBAN后4位] + [currency]
        │
        ▼
linkGoCardlessAccount() / linkSimpleFinAccount()
        │
        ▼
db.insertWithUUID('accounts', { name: normalizedName, ... })
        │
        ▼
accounts 表存储原始名称（无额外转换）
        │
        ▼
getAccounts() 返回给客户端展示
```

### 5.2 账户名称更新流程（用户手动修改）

```
用户在客户端编辑账户名称
        │
        ▼
dispatch updateAccount action
        │
        ▼
send('account-update', { id, name })
        │
        ▼
updateAccount() handler → mutator(undoable(updateAccount))
        │
        ▼
db.update('accounts', { id, name })
        │
        ▼
sendMessages([{ dataset: 'accounts', row: id, column: 'name', value, timestamp }])
        │
        ▼
applyMessages() → compareMessages() → 写入 messages_crdt
        │
        ▼
scheduleFullSync() → 发送到服务器
        │
        ▼
其他客户端 receiveMessages() → applyMessages()
        │
        ▼
所有客户端 accounts.name 字段更新
```

### 5.3 账户名称更新流程（银行同步）

```
银行同步触发
        │
        ▼
syncAccount() → processBankSyncDownload()
        │
        ▼
可能更新账户信息（但当前实现不更新名称）
        │
        ▼
仅更新 transactions 和 balance_current
        │
        ▼
用户名称保持不变
```

> **注意**：当前银行同步流程（`processBankSyncDownload`）仅更新交易和余额，**不更新账户名称**。账户名称仅在首次链接时设置，后续银行同步不会覆盖用户修改。

---

## 六、设计特点与架构考量

### 6.1 归一化职责分离

| 层级 | 职责 | 设计考量 |
|-----|------|---------|
| **Bank Sync** | 外部数据源适配 | 处理不同银行 API 的字段差异 |
| **loot-core** | 业务逻辑处理 | 保持标题化逻辑统一 |
| **CRDT** | 分布式一致性 | 确保多设备同步正确性 |

### 6.2 容错设计

- **字段降级**：当 name 缺失时，依次使用 iban、uid 作为备选
- **空值处理**：空字符串转换为 null，避免脏数据
- **时钟漂移容忍**：允许 5 分钟的时钟偏差 (`maxDrift: 5 * 60 * 1000`)

### 6.3 性能优化

- **批量操作**：通过 `batchMessages()` 减少数据库往返
- **增量同步**：Merkle Tree 只同步变更部分
- **缓存机制**：`payeesToCreate` Map 避免重复数据库查询

---

## 七、潜在问题与优化建议

### 7.1 当前问题

1. **标题化逻辑重复**：`title()` 函数在 `sync-server` 和 `loot-core` 中各有一份实现
2. **账户名称缺乏验证**：`updateAccount()` 直接接受 name 参数，无长度或格式校验
3. **冲突处理简单**：仅通过时间戳仲裁，无用户友好的冲突提示
4. **银行同步不更新名称**：可能导致银行端名称变更无法同步到客户端

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

#### 建议三：增加名称变更冲突检测

```typescript
// 在银行同步时检测用户是否已修改名称
async function syncAccountWithNameProtection(accountId, bankAccountData) {
  const existingAccount = await db.select('accounts', accountId);
  
  // 检测用户是否已自定义名称（与银行原始名称不同）
  const hasCustomName = existingAccount.official_name && 
    existingAccount.name !== existingAccount.official_name;
  
  if (hasCustomName && bankAccountData.name !== existingAccount.name) {
    // 记录冲突，但不覆盖用户自定义名称
    logger.log(`Name conflict detected for account ${accountId}: 
      user name="${existingAccount.name}", bank name="${bankAccountData.name}"`);
    
    // 可选：通知用户
    dispatch(addNotification({
      type: 'info',
      message: t('Bank returned a different name for {{accountName}}', { 
        accountName: existingAccount.name 
      }),
      actions: [
        { label: t('Use bank name'), action: () => acceptBankName(accountId, bankAccountData.name) },
        { label: t('Keep my name'), action: () => {} },
      ],
    }));
    
    return; // 不更新名称
  }
  
  // 无冲突或用户未自定义，正常更新
  await db.update('accounts', { id: accountId, name: bankAccountData.name });
}
```

#### 建议四：增加变更审计日志

```sql
-- 创建变更日志表
CREATE TABLE account_name_changes (
  id TEXT PRIMARY KEY,
  account_id TEXT,
  old_name TEXT,
  new_name TEXT,
  source TEXT,        -- 'user' | 'bank' | 'system'
  timestamp TEXT,
  FOREIGN KEY (account_id) REFERENCES accounts(id)
);
```

---

## 八、总结

账户标题的归一化逻辑涉及三个主要层级：

1. **Bank Sync 层**：负责将不同银行 API 返回的数据统一格式，组合账户名称
2. **loot-core 层**：负责账户 CRUD 操作和交易 payee 名称的标题化处理
3. **CRDT 层**：负责分布式场景下的变更追踪和冲突解决

### CRDT 同步机制核心要点

| 阶段 | 机制 | 关键组件 |
|-----|------|---------|
| **记录** | `db.update()` → `sendMessages()` → `applyMessages()` | HULC 时间戳、messages_crdt 表 |
| **传播** | `fullSync()` → 编码发送 → 服务器中继 | Merkle Tree、sync-server |
| **回放** | `receiveMessages()` → `applyMessages()` | 时间戳排序、冲突检测 |
| **胜出** | 时间戳字典序比较 | 毫秒 > 计数器 > 节点 ID |

### 当前实现特点

- **账户名称保护**：银行同步不会覆盖用户自定义名称（当前实现）
- **自动冲突解决**：完全依赖时间戳仲裁，无需用户干预
- **原子性保障**：所有变更在数据库事务中原子提交

### 未来优化方向

1. 统一 `title()` 函数位置
2. 增加账户名称验证和冲突检测
3. 提供用户友好的冲突解决界面
4. 添加变更审计日志