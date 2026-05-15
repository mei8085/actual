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
│  ┌─────────────────────────────────────────────────────────────┐           │
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
│  ┌─────────────────────────────────────────────────────────────┐           │
│  │  batchMessages() → batchUpdateTransactions()               │           │
│  │  - HULC 时间戳生成 (Timestamp.send())                       │           │
│  │  - Merkle Tree 增量同步                                     │           │
│  └──────────────────────────┬──────────────────────────────────┘           │
└─────────────────────────────┼───────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                        桌面客户端展示层                                     │
│  ┌─────────────────────────────────────────────────────────────┐           │
│  │  getAccounts() → AccountEntity[]                            │           │
│  │  - 直接读取 accounts.name 字段用于展示                       │           │
│  └─────────────────────────────────────────────────────────────┘           │
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

## 三、冲突解决逻辑

### 3.1 账户名称冲突场景

账户名称在以下场景可能发生冲突：

| 冲突场景 | 触发条件 | 处理策略 |
|---------|---------|---------|
| **银行返回名称变更** | 银行端账户名称发生变化 | 直接更新数据库 `accounts.name` |
| **用户手动修改** | 用户在客户端编辑账户名称 | 直接更新数据库 `accounts.name` |
| **多设备同步冲突** | 不同设备同时修改同一账户名称 | CRDT 时间戳仲裁 |

### 3.2 交易 Payee 名称匹配冲突

在 `matchTransactions()` 函数中处理 payee 匹配：

```typescript
// 第一优先级：通过 imported_id 精确匹配
if (trans.imported_id) {
  match = await db.first<db.DbTransaction>(
    `SELECT * FROM ${table} WHERE imported_id = ? AND account = ?`,
    [trans.imported_id, acctId],
  );
}

// 第二优先级：通过 payee ID 匹配
const match = data.fuzzyDataset.find(
  row => !hasMatched.has(row.id) && data.trans.payee === row.payee,
);

// 第三优先级：模糊匹配（日期 ±7 天，金额相同）
const match = data.fuzzyDataset.find(row => !hasMatched.has(row.id));
```

**冲突解决策略**：
- **高保真匹配**：优先使用 `imported_id`（银行交易唯一标识）
- **中等保真匹配**：使用 payee ID
- **低保真匹配**：日期范围 + 金额匹配

---

## 四、CRDT 机制对标题变更的记录与合并

### 4.1 时间戳生成机制

使用 **HULC (Hybrid Unique Logical Clock)** 时间戳：

```typescript
export class Timestamp {
  _state: { millis: number; counter: number; node: string };

  toString() {
    return [
      new Date(this.millis()).toISOString(),
      ('0000' + this.counter().toString(16).toUpperCase()).slice(-4),
      ('0000000000000000' + this.node()).slice(-16),
    ].join('-');
  }
  // 示例输出: 2015-04-24T22:23:42.123Z-1000-0123456789ABCDEF
}
```

**时间戳结构**：
- **毫秒时间戳**：ISO 8601 格式
- **计数器**：4 位十六进制，处理同一毫秒内的多次操作
- **节点 ID**：16 位十六进制，标识客户端/设备

### 4.2 发送与接收逻辑

```typescript
// 发送时生成新时间戳
static send(): Timestamp | null {
  const phys = Date.now();
  const lOld = clock.timestamp.millis();
  const cOld = clock.timestamp.counter();
  
  const lNew = Math.max(lOld, phys);
  const cNew = lOld === lNew ? cOld + 1 : 0;
  
  // 更新本地时钟
  clock.timestamp.setMillis(lNew);
  clock.timestamp.setCounter(cNew);
  
  return new Timestamp(lNew, cNew, clock.timestamp.node());
}

// 接收时合并远程时间戳
static recv(msg: Timestamp): Timestamp | null {
  const phys = Date.now();
  const lMsg = msg.millis();
  const cMsg = msg.counter();
  const lOld = clock.timestamp.millis();
  const cOld = clock.timestamp.counter();
  
  // 取最大值作为新的逻辑时间
  const lNew = Math.max(Math.max(lOld, phys), lMsg);
  
  // 根据不同情况计算新计数器
  const cNew =
    lNew === lOld && lNew === lMsg
      ? Math.max(cOld, cMsg) + 1
      : lNew === lOld
        ? cOld + 1
        : lNew === lMsg
          ? cMsg + 1
          : 0;
  
  // 更新本地时钟
  clock.timestamp.setMillis(lNew);
  clock.timestamp.setCounter(cNew);
  
  return new Timestamp(lNew, cNew, clock.timestamp.node());
}
```

### 4.3 批量更新与同步

```typescript
export async function batchUpdateTransactions({ added, updated, ... }) {
  // ...
  await batchMessages(async () => {
    // 批量写入变更
  });
}
```

**同步流程**：
1. **捕获变更**：账户名称更新通过 `batchMessages()` 捕获
2. **生成时间戳**：每次变更生成唯一的 HULC 时间戳
3. **增量同步**：通过 Merkle Tree 计算差异，仅同步变更部分
4. **冲突仲裁**：
   - 时间戳较大的变更获胜
   - 相同时间戳时，计数器较大的获胜
   - 计数器也相同时，节点 ID 较大的获胜

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

### 5.2 账户名称更新流程

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
updateAccount() handler
        │
        ▼
db.update('accounts', { id, name })
        │
        ▼
batchMessages() 捕获变更
        │
        ▼
Timestamp.send() 生成时间戳
        │
        ▼
Merkle Tree 更新哈希
        │
        ▼
同步到其他设备时通过 Timestamp.recv() 合并
```

### 5.3 Payee 名称归一化流程

```
银行交易数据
        │
        ▼
normalizeBankSyncTransactions()
        │
        ▼
payeeName = trans[mapping.get('payee')] ?? trans.payeeName
        │
        ▼
title(payeeName) 标题化处理
        │
        ▼
resolvePayee() 解析或创建 payee
        │
        ▼
db.insertPayee() 或返回现有 payee ID
```

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

### 7.2 优化建议

```typescript
// 建议：统一标题化函数位置
// packages/shared/util/title.ts (共享模块)
export { title } from '#shared/util/title';

// 建议：增加账户名称验证
async function updateAccount({ id, name, last_reconciled }) {
  // 名称长度限制
  if (name.length > 100) {
    throw new Error('Account name exceeds maximum length');
  }
  // 空名称检查
  if (!name.trim()) {
    throw new Error('Account name cannot be empty');
  }
  await db.update('accounts', { id, name, ...(last_reconciled && { last_reconciled }) });
}

// 建议：增加冲突检测与提示
async function handleAccountNameConflict(localName, remoteName, timestamp) {
  // 检测到冲突时通知用户
  dispatch(addNotification({
    type: 'warning',
    message: t('Account name conflict detected'),
    actions: [
      { label: t('Keep my change'), action: () => keepLocal() },
      { label: t('Use remote change'), action: () => acceptRemote() },
      { label: t('Edit manually'), action: () => showEditDialog() },
    ],
  }));
}
```

---

## 八、总结

账户标题的归一化逻辑涉及三个主要层级：

1. **Bank Sync 层**：负责将不同银行 API 返回的数据统一格式，组合账户名称
2. **loot-core 层**：负责账户 CRUD 操作和交易 payee 名称的标题化处理
3. **CRDT 层**：负责分布式场景下的变更追踪和冲突解决

核心设计特点：
- **职责清晰**：各层专注于特定功能
- **容错性强**：完善的字段降级策略
- **一致性保障**：HULC 时间戳确保全局唯一和单调递增
- **性能优化**：批量操作和增量同步减少开销

未来可优化方向：统一标题化函数位置、增加名称验证、提供用户友好的冲突解决机制。