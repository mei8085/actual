# 银行同步字段映射 - 交互与数据流报告

## 一、概述

Actual Budget 的银行同步字段映射功能允许用户将外部银行API返回的数据字段与本地预算系统的字段进行对齐配置。该功能支持按交易方向（支出/收入）分别配置映射规则，确保不同银行的数据格式都能正确导入。

## 二、UI 交互流程

### 2.1 入口与组件结构

```
EditSyncAccount Modal (银行同步账户设置)
├── FieldMapping (字段映射表格)
│   ├── 交易方向选择器 (Payment / Deposit)
│   └── 字段映射行 (Date / Payee / Notes)
└── BankSyncCheckboxOptions (其他同步选项)
```

### 2.2 字段映射界面详解

**核心组件**：`FieldMapping.tsx`

界面布局采用三列表格形式：

| Actual 本地字段 | → | 银行字段选择 | = | 示例值 |
|----------------|----|-------------|----|-------|
| Date           | → | [下拉选择]  | = | 2024-01-15 |
| Payee          | → | [下拉选择]  | = | Amazon.com |
| Notes          | → | [下拉选择]  | = | Purchase #12345 |

### 2.3 交互步骤

1. **选择交易方向**：
   - Payment（支出，金额 ≤ 0）
   - Deposit（收入，金额 > 0）

2. **选择映射字段**：
   - 为每个本地字段（Date / Payee / Notes）从下拉列表中选择对应的银行字段
   - 下拉列表根据已同步交易的 `raw_synced_data` 动态生成，只显示实际存在的字段
   - 支持嵌套字段路径（如 `paymentData.payer.name`）

3. **实时预览**：
   - 选择后立即显示该字段在实际交易中的示例值
   - 帮助用户确认选择的字段是否正确

4. **保存配置**：
   - 点击"Save"按钮将所有配置持久化存储

## 三、数据结构定义

### 3.1 映射数据类型

**文件**：`packages/loot-core/src/server/util/custom-sync-mapping.ts`

```typescript
// 映射数据结构
type Mappings = Map<string, Map<string, string>>;
//                    ↑             ↑         ↑
//              交易方向    本地字段    银行字段

// 默认映射配置
const defaultMappings: Mappings = new Map([
  [
    'payment',
    new Map([
      ['date', 'date'],
      ['payee', 'payeeName'],
      ['notes', 'notes'],
    ]),
  ],
  [
    'deposit',
    new Map([
      ['date', 'date'],
      ['payee', 'payeeName'],
      ['notes', 'notes'],
    ]),
  ],
]);
```

### 3.2 可映射字段配置

**文件**：`packages/desktop-client/src/components/banksync/EditSyncAccount.tsx`

```typescript
const mappableFields = [
  {
    actualField: 'date',
    syncFields: [
      'date', 'bookingDate', 'valueDate', 'postedDate',
      'transactedDate', 'booking_date', 'value_date', 'transaction_date',
    ],
  },
  {
    actualField: 'payee',
    syncFields: [
      'payeeName', 'creditorName', 'debtorName',
      'remittanceInformationUnstructured',
      'remittanceInformationStructured',
      'additionalInformation',
      'paymentData.payer.name',
      'paymentData.receiver.name',
      'merchant.name', 'creditor.name', 'debtor.name',
      // ... 更多字段
    ],
  },
  {
    actualField: 'notes',
    syncFields: [
      'notes', 'remittanceInformationUnstructured',
      'remittanceInformationStructured',
      'additionalInformation', 'category',
      'entry_reference', 'transaction_id',
      // ... 更多字段
    ],
  },
];
```

### 3.3 序列化/反序列化

```typescript
// Map 转 JSON 字符串
export const mappingsToString = (mapping: Mappings): string =>
  JSON.stringify(
    Object.fromEntries(
      [...mapping.entries()].map(([key, value]) => [
        key,
        Object.fromEntries(value),
      ]),
    ),
  );

// JSON 字符串转 Map
export const mappingsFromString = (str: string): Mappings => {
  const parsed = JSON.parse(str);
  return new Map(
    Object.entries(parsed).map(([key, value]) => [
      key,
      new Map(Object.entries(value as object)),
    ]),
  );
};
```

## 四、配置保存与回放链路

### 4.1 完整数据流图

```
┌─────────────┐     ┌────────────────┐     ┌──────────────┐
│  用户UI操作  │────▶│  React State   │────▶│  持久化存储   │
└─────────────┘     └────────────────┘     └──────────────┘
       ▲                                            │
       │                                            ▼
┌─────────────┐     ┌────────────────┐     ┌──────────────┐
│  映射应用    │◀────│  配置读取      │◀────│  同步触发     │
└─────────────┘     └────────────────┘     └──────────────┘
```

### 4.2 保存链路详解

**核心 Hook**：`useBankSyncAccountSettings.ts`

#### Step 1: 状态管理

```typescript
// 初始化时从持久化存储读取
const [savedMappings = mappingsToString(defaultMappings), setSavedMappings] =
  useSyncedPref(`custom-sync-mappings-${accountId}`);

// 本地状态缓存
const [mappings, setMappings] = useState<Mappings>(
  mappingsFromString(savedMappings),
);

// 更新单个字段映射
const setMapping = (field: string, value: string) => {
  setMappings(prev => {
    const updated = new Map(prev);
    const directionMap = updated.get(transactionDirection);
    if (directionMap) {
      const newDirectionMap = new Map(directionMap);
      newDirectionMap.set(field, value);  // 更新映射
      updated.set(transactionDirection, newDirectionMap);
    }
    return updated;
  });
};
```

#### Step 2: 持久化保存

```typescript
const saveSettings = () => {
  // 将 Map 序列化为 JSON 字符串
  const mappingsStr = mappingsToString(mappings);
  
  // 保存到 preferences 表
  setSavedMappings(mappingsStr);
  
  // 同时保存其他同步选项
  setSavedImportPending(String(importPending));
  setSavedImportNotes(String(importNotes));
  setSavedReimportDeleted(String(reimportDeleted));
  setSavedImportTransactions(String(importTransactions));
  setSavedUpdateDates(String(updateDates));
};
```

#### Step 3: 存储位置

配置保存在 `preferences` 表中，使用以下 key 格式：

| 配置项 | Key 格式 |
|--------|----------|
| 字段映射 | `custom-sync-mappings-${accountId}` |
| 导入待处理交易 | `sync-import-pending-${accountId}` |
| 导入备注 | `sync-import-notes-${accountId}` |
| 重新导入已删除交易 | `sync-reimport-deleted-${accountId}` |
| 启用交易导入 | `sync-import-transactions-${accountId}` |
| 更新交易日期 | `sync-update-dates-${accountId}` |

存储的 JSON 示例：

```json
{
  "payment": {
    "date": "bookingDate",
    "payee": "remittanceInformationUnstructured",
    "notes": "additionalInformation"
  },
  "deposit": {
    "date": "valueDate",
    "payee": "creditorName",
    "notes": "notes"
  }
}
```

### 4.3 回放链路（同步时应用映射）- 真实实现

**核心函数**：`normalizeBankSyncTransactions` in `sync.ts`

> **重要实现差异说明**：
> - **UI 预览阶段**：使用 `getByPath` 函数支持嵌套字段路径解析
> - **同步入库阶段**：直接按键名读取，**不支持嵌套路径**，但有回退逻辑

#### Step 1: 读取配置

```typescript
async function normalizeBankSyncTransactions(transactions, acctId) {
  // 从数据库读取该账户的映射配置
  const customMappingsRaw = await aqlQuery(
    q('preferences')
      .filter({ id: `custom-sync-mappings-${acctId}` })
      .select('value'),
  ).then(data => data?.data?.[0]?.value);

  // 反序列化，如无配置则使用默认值
  const mappings = customMappingsRaw
    ? mappingsFromString(customMappingsRaw)
    : defaultMappings;
    
  // 同时读取其他同步选项
  const [importPending, importNotes] = await Promise.all([
    aqlQuery(...).then(data => String(data?.data?.[0]?.value ?? 'true') === 'true'),
    aqlQuery(...).then(data => String(data?.data?.[0]?.value ?? 'true') === 'true'),
  ]);
```

#### Step 2: 按交易应用映射 - **真实代码实现**

```typescript
  const normalized = [];
  for (const trans of transactions) {
    trans.cleared = Boolean(trans.booked);

    if (!importPending && !trans.cleared) continue;

    if (!trans.amount) {
      trans.amount = trans.transactionAmount.amount;
    }

    // 根据金额选择映射方向
    const mapping = mappings.get(trans.amount <= 0 ? 'payment' : 'deposit');

    // ⚠️ 关键实现：直接按键名读取，不支持嵌套路径！
    // 例如：如果配置为 'paymentData.payer.name'，会尝试读取 trans['paymentData.payer.name']
    // 这会返回 undefined，因为对象结构是 trans.paymentData.payer.name
    
    const date = trans[mapping.get('date')] ?? trans.date;           // 有回退
    const payeeName = trans[mapping.get('payee')] ?? trans.payeeName; // 有回退
    const notes = trans[mapping.get('notes')];                        // 无回退

    // Validate the date because we do some stuff with it. The db
    // layer does better validation, but this will give nicer errors
    if (date == null) {
      throw new Error('`date` is required when adding a transaction');
    }

    if (payeeName == null) {
      throw new Error('`payeeName` is required when adding a transaction');
    }

    // ... 后续处理
  }
```

#### Step 3: UI 预览阶段的嵌套字段支持（仅用于预览）

```typescript
// 文件: packages/desktop-client/src/components/banksync/EditSyncAccount.tsx

// ✅ UI 预览使用 getByPath 支持嵌套路径
function getByPath(obj: unknown, path: string): unknown {
  if (obj == null) return undefined;

  const keys = path.split('.');
  let current: unknown = obj;

  for (const key of keys) {
    if (current == null || typeof current !== 'object') {
      return undefined;
    }
    current = (current as Record<string, unknown>)[key];
  }

  return current;
}

// UI 显示字段列表时使用 getByPath 解析示例值
export const getFields = (transaction: Record<string, unknown>): MappableFieldWithExample[] =>
  mappableFields.map(field => ({
    actualField: field.actualField,
    syncFields: field.syncFields
      .map(syncField => {
        const value = getByPath(transaction, syncField);  // ✅ 使用路径解析
        return value !== undefined
          ? { field: syncField, example: String(value) }
          : null;
      })
      .filter((item): item is { field: string; example: string } => item !== null),
  }));
```

### 4.4 实现差异对比表

| 阶段 | 字段读取方式 | 嵌套路径支持 | 回退逻辑 |
|------|-------------|------------|---------|
| **UI 预览** | `getByPath(obj, path)` 递归解析 | ✅ 支持 | ❌ 无 |
| **同步入库** | `trans[fieldName]` 直接按键名读取 | ❌ 不支持 | ✅ date/payee 有回退 |

### 4.5 差异对配置生效的影响

#### 问题现象

用户在配置界面选择了嵌套字段（如 `paymentData.payer.name`）：
- ✅ **UI 显示正常**：示例值正确显示（因为使用 `getByPath` 解析）
- ❌ **实际同步失效**：该字段值为 `undefined`，最终使用回退值

#### 受影响的字段配置

如果用户配置了以下嵌套路径字段，同步时将无法正确读取：

| 本地字段 | 常见嵌套路径配置 | 实际行为 |
|---------|----------------|---------|
| payee | `paymentData.payer.name` | 返回 `undefined` → 回退到 `trans.payeeName` |
| payee | `paymentData.receiver.name` | 返回 `undefined` → 回退到 `trans.payeeName` |
| payee | `merchant.name` | 返回 `undefined` → 回退到 `trans.payeeName` |
| notes | `paymentData.payer.accountNumber` | 返回 `undefined` → notes 为 null |

#### 回退逻辑的具体行为

```typescript
// date 字段回退
// 配置为嵌套路径 → trans['paymentData.xxx'] = undefined
// → 回退到 trans.date
const date = trans[mapping.get('date')] ?? trans.date;

// payee 字段回退  
// 配置为嵌套路径 → trans['merchant.name'] = undefined
// → 回退到 trans.payeeName
const payeeName = trans[mapping.get('payee')] ?? trans.payeeName;

// notes 字段无回退
// 配置为嵌套路径 → trans['some.nested.field'] = undefined
// → notes 最终为 null
const notes = trans[mapping.get('notes')];
```

## 五、示例交易数据流程

### 5.1 原始银行数据（含嵌套结构）

```json
{
  "transactionId": "TXN001",
  "date": "2024-01-16",
  "bookingDate": "2024-01-15",
  "payeeName": "Default Payee",
  "paymentData": {
    "payer": {
      "name": "Actual Merchant Name",
      "accountNumber": "****1234"
    }
  },
  "transactionAmount": {
    "amount": "-49.99"
  }
}
```

### 5.2 用户配置

```json
{
  "payment": {
    "date": "bookingDate",
    "payee": "paymentData.payer.name",
    "notes": "paymentData.payer.accountNumber"
  }
}
```

### 5.3 实际映射结果

| 字段 | 预期值 | 实际值 | 原因 |
|-----|-------|-------|------|
| date | "2024-01-15" | "2024-01-15" | ✅ bookingDate 是顶层字段，正常读取 |
| payee | "Actual Merchant Name" | "Default Payee" | ❌ 嵌套路径无法解析，回退到 payeeName |
| notes | "****1234" | null | ❌ 嵌套路径无法解析，notes 无回退 |

### 5.4 映射后的标准化交易

```typescript
{
  date: "2024-01-15",           // ✅ 来自 bookingDate (顶层字段)
  payee_name: "Default Payee",  // ❌ 回退值，paymentData.payer.name 配置未生效
  notes: null,                  // ❌ 无回退，嵌套路径配置未生效
  amount: -4999,
  imported_id: "TXN001",
  // ...
}
```

## 六、关键技术点

### 6.1 交易方向驱动

**实际实现说明**：
- 代码仅根据交易金额符号选择两套独立的映射配置（payment 或 deposit）
- 两套映射配置都包含相同的三个本地字段：date、payee、notes
- 具体映射到哪些银行数据字段，完全由用户配置决定，并非固定绑定付款方/收款方字段

**代码证据**：
```typescript
// sync.ts:462 - 仅按金额符号选择映射配置对象
const mapping = mappings.get(trans.amount <= 0 ? 'payment' : 'deposit');

// sync.ts:464-466 - 从选中的映射配置中读取对应字段
const date = trans[mapping.get('date')] ?? trans.date;
const payeeName = trans[mapping.get('payee')] ?? trans.payeeName;
const notes = trans[mapping.get('notes')];
```

**默认配置示例**（两套配置默认完全相同）：
```typescript
// custom-sync-mapping.ts:31-48
export const defaultMappings: Mappings = new Map([
  [
    'payment',
    new Map([['date', 'date'], ['payee', 'payeeName'], ['notes', 'notes']]),
  ],
  [
    'deposit',
    new Map([['date', 'date'], ['payee', 'payeeName'], ['notes', 'notes']]),
  ],
]);
```

**配置灵活性**：
- 用户可以为支出（payment）和收入（deposit）分别配置不同的银行字段映射
- 例如：支出时 payee 映射到 `debtorName`，收入时 payee 映射到 `creditorName`

### 6.2 动态字段发现

**实际实现说明**：
- 映射选项不是硬编码的，而是从实际交易数据中提取
- 查询满足条件的交易后，取查询结果数组的第一条作为示例交易
- 解析该交易的 `raw_synced_data` 原始数据后，动态生成可用字段候选列表

**代码证据 1：查询条件**
```typescript
// useBankSyncAccountSettings.ts:60-67 - 用户查询未显式指定 orderBy
const transactionQuery = q('transactions')
  .filter({
    account: accountId,
    amount: transactionDirection === 'payment' ? { $lte: 0 } : { $gt: 0 },
    raw_synced_data: { $ne: null },
  })
  .options({ splits: 'none' })
  .select('*');

// useBankSyncAccountSettings.ts:73 - 直接取数组首条
const data = transactions?.[0]?.raw_synced_data;
```

**代码证据 2：AQL 默认排序（schema/index.ts:268-274）**
```javascript
// AQL customizeQuery 自动注入的默认排序逻辑
case 'transactions':
  return [
    { date: 'desc' },           // 1. 日期降序
    'starting_balance_flag',    // 2. 期初余额标志
    { sort_order: 'desc' },     // 3. 排序值降序
    'id',                       // 4. id
  ];
```

**代码证据 3：SQL 视图排序（schema/index.ts:411）**
```sql
-- v_transactions 视图中的 ORDER BY 子句
ORDER BY _.date desc, _.starting_balance_flag, _.sort_order desc, _.id;
```

**关键结论**：
- 示例交易来自**默认排序后的首条记录**
- 默认排序规则：date 降序 → starting_balance_flag → sort_order 降序 → id
- 由于有确定的默认排序，首条记录即为满足条件的最新交易

### 6.3 嵌套字段支持差异

- **UI 层**：使用 `getByPath` 函数支持点分隔的嵌套路径访问
- **业务层**：同步入库时仅支持顶层字段键名访问
- **不一致风险**：UI 预览可能展示实际上无法工作的配置选项

### 6.4 回退机制

- date 和 payee 字段有默认回退值，保证同步不会中断
- notes 字段无回退，配置错误会导致备注为空
- 回退机制掩盖了配置未生效的问题，用户可能不易察觉

### 6.5 跨平台同步

- 使用 `useSyncedPref` hook，配置会通过同步机制在设备间同步
- 与其他预算数据一起保存在本地数据库中

## 七、相关文件索引

| 文件路径 | 功能描述 |
|---------|---------|
| `packages/desktop-client/src/components/banksync/FieldMapping.tsx` | 字段映射UI组件 |
| `packages/desktop-client/src/components/banksync/EditSyncAccount.tsx` | 银行同步账户设置模态框，含 `getByPath` 嵌套路径解析 |
| `packages/desktop-client/src/components/banksync/useBankSyncAccountSettings.ts` | 配置状态管理Hook |
| `packages/loot-core/src/server/util/custom-sync-mapping.ts` | 映射数据类型与序列化 |
| `packages/loot-core/src/server/accounts/sync.ts` | 同步核心逻辑，映射应用（不支持嵌套路径） |
| `packages/loot-core/src/types/models/bank-sync.ts` | 银行同步类型定义 |
