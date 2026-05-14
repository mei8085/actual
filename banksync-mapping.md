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

### 4.3 回放链路（同步时应用映射）

**核心函数**：`normalizeBankSyncTransactions` in `sync.ts`

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

#### Step 2: 按交易应用映射

```typescript
  const normalized = [];
  for (const trans of transactions) {
    // 1. 根据金额判断交易方向
    const mapping = mappings.get(trans.amount <= 0 ? 'payment' : 'deposit');

    // 2. 应用字段映射
    const date = trans[mapping.get('date')] ?? trans.date;
    const payeeName = trans[mapping.get('payee')] ?? trans.payeeName;
    const notes = trans[mapping.get('notes')];

    // 3. 支持嵌套字段路径解析 (通过 getByPath 函数)
    // 例: paymentData.payer.name → trans.paymentData.payer.name

    // 4. 构建标准化交易对象
    normalized.push({
      payee_name: payeeName,
      trans: {
        amount: amountToInteger(trans.amount),
        payee: trans.payee,
        account: trans.account,
        date,
        notes: importNotes && notes ? notes.trim() : null,
        // ... 其他字段
      },
    });
  }

  return { normalized, payeesToCreate };
}
```

#### Step 3: 嵌套字段路径解析

```typescript
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
```

## 五、示例交易数据流程

### 5.1 原始银行数据 (GoCardless 示例)

```json
{
  "transactionId": "TXN001",
  "bookingDate": "2024-01-15",
  "valueDate": "2024-01-16",
  "remittanceInformationUnstructured": "AMAZON PURCHASE #12345",
  "additionalInformation": "Online Shopping",
  "transactionAmount": {
    "amount": "-49.99",
    "currency": "EUR"
  },
  "creditorName": null,
  "debtorName": "John Doe"
}
```

### 5.2 用户配置的映射

```json
{
  "payment": {
    "date": "bookingDate",
    "payee": "remittanceInformationUnstructured",
    "notes": "additionalInformation"
  }
}
```

### 5.3 映射后的标准化交易

```typescript
{
  date: "2024-01-15",                    // 来自 bookingDate
  payee_name: "AMAZON PURCHASE #12345",  // 来自 remittanceInformationUnstructured
  notes: "Online Shopping",              // 来自 additionalInformation
  amount: -4999,                         // 金额转换为整数（分）
  imported_id: "TXN001",
  // ...
}
```

## 六、关键技术点

### 6.1 交易方向驱动

- 支出（Payment）：金额 ≤ 0，使用付款方相关字段
- 收入（Deposit）：金额 > 0，使用收款方相关字段
- 两种方向可以独立配置不同的映射规则

### 6.2 动态字段发现

- 映射选项不是硬编码的，而是从实际交易数据中提取
- 查询该账户最近的一笔同步交易（`raw_synced_data`）
- 解析原始数据后动态生成可用字段列表

### 6.3 嵌套字段支持

- 使用点分隔符路径访问嵌套对象属性
- 示例：`paymentData.payer.name` → `trans.paymentData.payer.name`

### 6.4 回退机制

- 如果映射的字段不存在或为空，使用默认字段作为后备
- 示例：`trans[mapping.get('date')] ?? trans.date`

### 6.5 跨平台同步

- 使用 `useSyncedPref` hook，配置会通过同步机制在设备间同步
- 与其他预算数据一起保存在本地数据库中

## 七、相关文件索引

| 文件路径 | 功能描述 |
|---------|---------|
| `packages/desktop-client/src/components/banksync/FieldMapping.tsx` | 字段映射UI组件 |
| `packages/desktop-client/src/components/banksync/EditSyncAccount.tsx` | 银行同步账户设置模态框 |
| `packages/desktop-client/src/components/banksync/useBankSyncAccountSettings.ts` | 配置状态管理Hook |
| `packages/loot-core/src/server/util/custom-sync-mapping.ts` | 映射数据类型与序列化 |
| `packages/loot-core/src/server/accounts/sync.ts` | 同步核心逻辑，映射应用 |
| `packages/loot-core/src/types/models/bank-sync.ts` | 银行同步类型定义 |
