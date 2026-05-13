# 账单文件导入格式解析与归一化流程分析

## 概述

Actual Budget 支持三种主流账单格式的导入：**CSV**、**OFX/QFX**、**QIF**。三种格式从文件入口到数据落库的流程存在显著差异，主要体现在解析复杂度、归一化程度、用户交互要求等方面。

---

## 一、整体流程架构

### 1.1 流程总览

```
用户选择文件
      ↓
【入口层】 parseFile() 根据扩展名分发
      ↓
【解析层】 格式特定解析器 → 中间格式
      ↓
【归一化层】 结构化转换 → 统一交易对象
      ↓
【客户端处理】 预览、字段映射、金额处理
      ↓
【对账开关】 ← 用户选择是否开启对账
      │
      ├─ 关闭对账 → addTransactions() 直接新增
      │
      └─ 开启对账 → reconcileTransactions()
                        │
                        ├─ isPreview=true  → 仅比对，不落库
                        └─ isPreview=false → 去重后落库
      ↓
【落库层】 batchUpdateTransactions() → SQLite
```

### 1.2 核心入口

**文件位置**: `packages/loot-core/src/server/transactions/import/parse-file.ts`

```typescript
// 入口函数：根据文件扩展名分发
export async function parseFile(
  filepath: string,
  options: ParseFileOptions = {},
): Promise<ParseFileResult> {
  switch (ext.toLowerCase()) {
    case '.qif':           return parseQIF(filepath, options);
    case '.csv': case '.tsv': return parseCSV(filepath, options);
    case '.ofx': case '.qfx': return parseOFX(filepath, options);
    case '.xml':           return parseCAMT(filepath, options);
  }
}
```

### 1.3 对账开关与预览逻辑

**文件位置**: `packages/desktop-client/src/accounts/mutations.ts:280-298`

```typescript
// 客户端调用选择：
if (!reconcile) {
  // 关闭对账：直接调用 addTransactions，不做任何去重
  await send('api/transactions-add', { accountId, transactions });
} else {
  // 开启对账：调用 reconcileTransactions
  //   isPreview=true  → 仅比对，返回 updatedPreview，不落库
  //   isPreview=false → 去重后执行 batchUpdateTransactions 落库
  await send('transactions-import', {
    accountId,
    transactions,
    isPreview: false,  // 预览时传 true
    opts: { reimportDeleted },
  });
}
```

**服务端预览控制**: `packages/loot-core/src/server/accounts/sync.ts:677-680`

```typescript
if (!isPreview) {
  // 只有 isPreview=false 时才真正落库
  await createNewPayees(payeesToCreate, [...added, ...updated]);
  await batchUpdateTransactions({ added, updated });
}
// isPreview=true 时仅返回 updatedPreview 供 UI 展示比对结果
```

---

## 二、三种格式详细对比

### 2.1 对比总表

| 维度 | CSV | OFX/QFX | QIF |
|------|-----|---------|-----|
| **解析方式** | 第三方库 `csv-parse` | XML/SGML 解析 + 自定义 | 逐行字符标记解析 |
| **服务端归一化** | ❌ 几乎不做 | ✅ 完整结构化 | ⚠️ 部分结构化 |
| **字段映射** | ✅ 必须用户配置 | ❌ 无需 | ❌ 无需 |
| **日期解析** | 客户端多格式适配 | 服务端标准化 `YYYYMMDD` | 客户端多格式适配 |
| **金额解析** | 客户端灵活处理 | 服务端严格解析 | 服务端基础解析 |
| **唯一标识 imported_id** | ❌ 无 | ✅ `FITID` | ❌ 无 |
| **用户交互** | 复杂（多选项） | 简单（2个复选框） | 中等（日期格式） |
| **去重能力** | 弱（无 imported_id，仅模糊匹配） | 强（三轮匹配，首推 imported_id） | 弱（无 imported_id，仅模糊匹配） |
| **投资支持** | ❌ 无 | ⚠️ 仅银行式交易子集 | ❌ 无 |

---

### 2.2 CSV 格式流程

#### 特点：最灵活但最复杂

```
┌─────────────────────────────────────────────────────────────┐
│  CSV 流程                                                   │
├─────────────────────────────────────────────────────────────┤
│  服务端 (parse-file.ts:109-156)                             │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ 1. 读取文件内容                                      │    │
│  │ 2. 支持跳过开头/结尾行 (skipStart/skipEnd)           │    │
│  │ 3. csv-parse 解析 → 原始数据 (Record<string,string>) │    │
│  │ 4. ❌ 不做任何字段语义化，直接返回原始键值对          │    │
│  └─────────────────────────────────────────────────────┘    │
│                           ↓                                 │
│  客户端 (ImportTransactionsModal.tsx)                       │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ 1. 自动检测初始字段映射 (getInitialMappings)         │    │
│  │ 2. 用户手动调整字段映射 UI (FieldMappings.tsx)       │    │
│  │ 3. 日期格式选择 (6种格式: yyyy/mm/dd 等)             │    │
│  │ 4. 金额处理:                                         │    │
│  │    - 单金额列 / 收支分列 (splitMode)                 │    │
│  │    - 收支标识列 (inOutMode)                          │    │
│  │    - 金额翻转 (flipAmount)                           │    │
│  │    - 乘数 (multiplier)                               │    │
│  │ 5. applyFieldMappings() → 统一结构                   │    │
│  └─────────────────────────────────────────────────────┘    │
│                           ↓                                 │
│  对账开关选择                                               │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ 关闭对账: api/transactions-add → 直接新增        │    │
│  │ 开启对账: transactions-import → reconcileTransactions │    │
│  │   isPreview=true  → 仅比对返回 updatedPreview       │    │
│  │   isPreview=false → 三轮模糊匹配后落库               │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

#### 关键代码片段

**服务端解析** (`parse-file.ts:135-155`):

```typescript
// 仅做格式解析，不做语义理解
data = csv2json(contents, {
  columns: options?.hasHeaderRow,  // 是否有表头
  delimiter: options?.delimiter || ',',  // 分隔符可配置
  bom: true,
  quote: '"',
  trim: true,
  relax_column_count: true,
  skip_empty_lines: true,
});
return { errors, transactions: data };  // 直接返回原始数据
```

**客户端字段映射** (`utils.ts:153-170`):

```typescript
export function applyFieldMappings(transaction, mappings) {
  const result = {};
  for (const [originalField, target] of Object.entries(mappings)) {
    const field = originalField === 'payee' ? 'payee_name' : originalField;
    result[field] = transaction[target || field];
  }
  return result;
}
```

#### CSV 特有配置项

| 配置项 | 说明 | 存储位置 |
|--------|------|----------|
| `delimiter` | 分隔符: `,` `;` `\|` `\t` `~` | 按账户保存 |
| `hasHeaderRow` | 是否有表头行 | 按账户保存 |
| `skipStartLines` | 跳过开头 N 行 | 按账户保存 |
| `skipEndLines` | 跳过结尾 N 行 | 按账户保存 |
| `fieldMappings` | 字段映射 JSON | 按账户保存 |
| `splitMode` | 收支分列模式 | 临时状态 |
| `inOutMode` | 收支标识列模式 | 按账户保存 |

---

### 2.3 OFX/QFX 格式流程

#### 特点：最标准化，银行直接支持

```
┌─────────────────────────────────────────────────────────────┐
│  OFX 流程                                                   │
├─────────────────────────────────────────────────────────────┤
│  服务端 (ofx2json.ts + parse-file.ts:200-248)               │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ 1. 分离头部 (OFX 头信息)                             │    │
│  │ 2. 双模式解析:                                       │    │
│  │    ├─ 尝试 XML 解析 (标准 OFX 2.0+)                  │    │
│  │    └─ 失败则 SGML → XML 转换 (旧版 OFX 1.x)          │    │
│  │ 3. 多账户类型支持:                                   │    │
│  │    ├─ BANKMSGSRSV1    (银行账户)                     │    │
│  │    ├─ CREDITCARDMSGSRSV1 (信用卡)                    │    │
│  │    └─ INVSTMTMSGSRSV1 (投资账户)                     │    │
│  │       ⚠️ 投资账户仅提取 INVBANKTRAN 中的 STMTTRN        │    │
│  │          (仅银行式交易子集，不含买卖交易本身)                  │    │
│  │ 4. 字段映射 (硬编码):                                │    │
│  │    DTPOSTED → date                                   │    │
│  │    TRNAMT   → amount                                 │    │
│  │    FITID    → imported_id (唯一标识)                 │    │
│  │    NAME     → payee_name                             │    │
│  │    MEMO     → notes                                  │    │
│  │ 5. 日期标准化: YYYYMMDDHHMMSS → YYYY-MM-DD           │    │
│  │ 6. 金额严格解析 (parseOfxAmount)                     │    │
│  └─────────────────────────────────────────────────────┘    │
│                           ↓                                 │
│  客户端                                                     │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ 1. ✅ 无需字段映射 UI                                │    │
│  │ 2. ✅ 无需日期格式选择                               │    │
│  │ 3. 可选配置:                                         │    │
│  │    ├─ "缺少 Payee 时使用 Memo 回退"                  │    │
│  │    └─ "交换 Payee 和 Memo"                          │    │
│  └─────────────────────────────────────────────────────┘    │
│                           ↓                                 │
│  对账开关选择                                               │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ 关闭对账: api/transactions-add → 直接新增        │    │
│  │ 开启对账: transactions-import → reconcileTransactions │    │
│  │   isPreview=true  → 仅比对返回 updatedPreview       │    │
│  │   isPreview=false → 三轮匹配（首推 imported_id）       │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

#### 关键代码片段

**SGML/XML 兼容处理** (`ofx2json.ts:144-151`):

```typescript
// 先尝试 XML 解析，失败则做 SGML 清洗
let dataParsed = null;
try {
  dataParsed = await parseXml(content);
} catch {
  const sanitized = sgml2Xml(content);  // 补全闭合标签等
  dataParsed = await parseXml(sanitized);
}
```

**多账户类型提取** (`ofx2json.ts:48-99`):

```typescript
function getStmtTrn(data) {
  const ofx = data?.['OFX'];
  if (ofx?.['CREDITCARDMSGSRSV1'] != null) {
    return getCcStmtTrn(ofx);      // 信用卡
  } else if (ofx?.['INVSTMTMSGSRSV1'] != null) {
    return getInvStmtTrn(ofx);     // 投资
  } else {
    return getBankStmtTrn(ofx);    // 普通银行
  }
}
```

**投资账户的边界**: `ofx2json.ts:87-99`

```typescript
function getInvStmtTrn(ofx) {
  const msg = ofx?.['INVSTMTMSGSRSV1'];
  const stmtTrnRs = getAsArray(msg?.['INVSTMTTRNRS']);
  const result = stmtTrnRs.flatMap(s => {
    const stmtRs = s?.['INVSTMTRS'];
    const tranList = stmtRs?.['INVTRANLIST'];
    // ⚠️ 仅提取 INVBANKTRAN 中的 STMTTRN（银行式交易子集）
    // 不含 INCOME/EXPENSE/SELLBUY 等投资交易本身
    const stmtTrn = tranList?.['INVBANKTRAN']?.flatMap(t => t?.['STMTTRN']);
    return getAsArray(stmtTrn);
  });
  return result;
}
```

**金额特殊解析** (`parse-file.ts:17-46`):

```typescript
function parseOfxAmount(amount: string): number | null {
  // 处理括号表示负数: "(30.00)" → "-30.00"
  if (cleaned.startsWith('(') && cleaned.endsWith(')')) {
    cleaned = '-' + cleaned.slice(1, -1);
  }
  // 移除货币符号
  cleaned = cleaned.replace(/[^\d.-]/g, '');
  // 处理多个小数点 (某些银行的 bug)
  // ...
}
```

#### OFX 优势

1. **唯一标识**: `FITID (Financial Institution Transaction ID) 允许精确去重
2. **交易类型**: `TRNTYPE` 字段区分 CREDIT/DEBIT/INT/etc.
3. **多账户**: 单文件可包含多个账户交易
4. **投资支持**: 可解析投资账户中的银行式交易 (INVSTMTMSGSRSV1 → INVBANKTRAN → STMTTRN)

#### OFX 投资边界说明

**投资账户场景下，Actual **仅提取** `INVBANKTRAN` 包裹的 `STMTTRN` 记录（银行手续费、利息、费用等银行式交易）。OFX 投资账户中的投资交易（如 `BUYMF、SELLSTK、DIVIDEND、INTERESTSTOCK 等）**不会被解析

---

### 2.4 QIF 格式流程

#### 特点：古老格式，介于 CSV 和 OFX 之间

```
┌─────────────────────────────────────────────────────────────┐
│  QIF 流程                                                   │
├─────────────────────────────────────────────────────────────┤
│  服务端 (qif2json.ts + parse-file.ts:158-198)               │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ 1. 按行解析，首字符为字段标记:                        │    │
│  │    D 日期    T 金额    N 支票号                       │    │
│  │    M 备注    P 收款人  L 分类                         │    │
│  │    A 地址    C 状态    ^ 记录分隔符                   │    │
│  │ 2. 类型头检测: !Type:Bank / !Type:CCard              │    │
│  │ 3. 支持分裂交易 (Splits):                             │    │
│  │    S 子分类  E 子备注  $ 子金额                       │    │
│  │ 4. 服务端归一化:                                      │    │
│  │    date / amount / payee / memo 字段提取             │    │
│  │ 5. looselyParseAmount 金额解析                       │    │
│  └─────────────────────────────────────────────────────┘    │
│                           ↓                                 │
│  客户端                                                     │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ 1. ⚠️ 需要日期格式选择 (6种格式)                     │    │
│  │ 2. 可选: "交换 Payee 和 Memo"                        │    │
│  │ 3. 金额翻转支持                                       │    │
│  └─────────────────────────────────────────────────────┘    │
│                           ↓                                 │
│  对账开关选择                                               │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ 关闭对账: api/transactions-add → 直接新增        │    │
│  │ 开启对账: transactions-import → reconcileTransactions │    │
│  │   isPreview=true  → 仅比对返回 updatedPreview       │    │
│  │   isPreview=false → 三轮模糊匹配后落库               │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

#### 关键代码片段

**逐行标记解析** (`qif2json.ts:44-104):

```typescript
while ((line = lines.shift())) {
  switch (line[0]) {
    case 'D': transaction.date = line.substring(1); break;
    case 'T': transaction.amount = line.substring(1); break;
    case 'P': transaction.payee = line.substring(1); break;
    case 'M': transaction.memo = line.substring(1); break;
    case 'L': transaction.category = line.substring(1); break;
    case '^': transactions.push(transaction); transaction = {}; break;
    // 分裂交易支持
    case 'S': division.category = ...; break;
    case '$': division.amount = parseFloat(...); break;
  }
}
```

**服务端归一化** (`parse-file.ts:178-197`):

```typescript
return {
  errors: [],
  transactions: data.transactions.map(trans => {
    return {
      amount: trans.amount != null ? looselyParseAmount(trans.amount) : null,
      date: trans.date,
      payee_name: swap ? trans.memo : trans.payee,
      notes: options.importNotes ? memoSource : null,
    };
  }).filter(trans => trans.date != null && trans.amount != null),
};
```

#### QIF 特有功能

- **分裂交易 (Splits): 一笔交易可拆分为多个分类
- **内置分类**: `L` 字段自带分类信息
- **支票号**: `N` 字段支持支票号码

---

## 三、归一化输出结构对比

### 3.1 服务端输出

```typescript
// ============ CSV 输出 (原始) ============
type CsvTransaction = Record<string, string> | string[];
// 示例: { "Transaction Date": "2024-01-15", "Amount": "-128.50", ... }

// ============ OFX 输出 (结构化) ============
type StructuredTransaction = {
  amount: number;           // 已解析为数字
  date: string;             // 已标准化 YYYY-MM-DD
  payee_name: string;
  imported_payee: string;
  notes: string;
  imported_id: string;      // FITID 唯一标识 ✅
};

// ============ QIF 输出 (半结构化) ============
type StructuredTransaction = {
  amount: number | null;    // 可能为 null
  date: string;             // 原始格式，需客户端再解析
  payee_name: string | null;
  imported_payee: string | null;
  notes: string | null;
  // 无 imported_id ❌
};
```

### 3.2 客户端最终落库结构

三种格式最终统一为 `TransactionEntity`:

```typescript
type TransactionEntity = {
  id: string;
  account: string;
  date: string;           // YYYY-MM-DD
  amount: number;         // 整数 (分)
  payee: string | null;
  notes: string | null;
  category: string | null;
  cleared: boolean;
  imported_id: string | null;  // 仅 OFX 有值
  imported_payee: string | null;
};
```

---

## 四、对账与去重策略

### 4.1 对账开关逻辑

```
┌─────────────────────────────────────────────────────────────────┐
│  对账开关 (reconcile)                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  关闭对账 (reconcile = false)                               │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ 调用路径:                                         │    │
│  │   useImportTransactionsMutation                   │    │
│  │     → send('api/transactions-add', ...)         │    │
│  │     → addTransactions()                        │    │
│  │     → 直接插入数据库，不做任何去重匹配           │    │
│  │ 结果: 所有交易直接新增                          │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                                 │
│  开启对账 (reconcile = true)                                │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ 调用路径:                                         │    │
│  │   预览: isPreview = true                    │    │
│  │     → reconcileTransactions(isPreview=true)       │    │
│  │     → 执行三轮匹配                          │    │
│  │     → 返回 updatedPreview，不落库                │    │
│  │                                                 │    │
│  │   导入: isPreview = false                   │    │
│  │     → reconcileTransactions(isPreview=false)      │    │
│  │     → 执行三轮匹配                          │    │
│  │     → 新增/更新落库                          │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**关键代码**: `packages/loot-core/src/server/accounts/sync.ts:538-696`

```typescript
export async function reconcileTransactions(
  acctId,
  transactions,
  isBankSyncAccount = false,
  strictIdChecking = true,
  isPreview = false,   // ← 控制是否落库
  defaultCleared = true,
  updateDates = false,
  reimportDeleted?: boolean,
): Promise<ReconcileTransactionsResult> {
  // 1. 先执行三轮匹配 (matchTransactions)
  const { transactionsStep3 } = await matchTransactions(...);

  // 2. 根据匹配结果生成 added/updated/ignored
  for (const { trans, match } of transactionsStep3) {
    if (match && !trans.forceAddTransaction) {
      // 匹配到了 → 更新或忽略
    } else {
      // 未匹配 → 新增
    }
  }

  // 3. 仅当 isPreview=false 时才真正落库
  if (!isPreview) {
    await createNewPayees(payeesToCreate, [...added, ...updated]);
    await batchUpdateTransactions({ added, updated });
  }

  return { added, updated, updatedPreview };
}
```

### 4.2 统一三轮去重匹配逻辑

**所有格式共用同一套 `matchTransactions` 函数，按顺序执行三轮匹配：

**文件位置**: `packages/loot-core/src/server/accounts/sync.ts:698-897`

```
┌─────────────────────────────────────────────────────────────────┐
│  三轮去重匹配流程 (matchTransactions)                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  第一轮: imported_id 精确匹配 (最高保真度)                │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ if (trans.imported_id) {                               │    │
│  │   SELECT * FROM v_transactions                        │    │
│  │   WHERE imported_id = ? AND account = ?                │    │
│  │                                                         │    │
│  │  条件: 仅 OFX 有 imported_id (FITID)            │    │
│  │      CSV/QIF 无 imported_id → 跳过此轮               │    │
│  └─────────────────────────────────────────────────────┘    │
│                           ↓                                 │
│  第二轮: 同金额 + 7 天窗口 + 收款人 ID 匹配                │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ fuzzyDataset = SELECT ...                            │    │
│  │   WHERE date >= [date-7] AND date <= [date+7]       │    │
│  │     AND amount = ? AND account = ?                     │    │
│  │                                                         │    │
│  │ 然后在 fuzzyDataset 中找:                               │    │
│  │   find(row => payee 相同且未被匹配)                │    │
│  └─────────────────────────────────────────────────────┘    │
│                           ↓                                 │
│  第三轮: 同金额 + 7 天窗口 + 第一个未匹配                │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ 直接取 fuzzyDataset 中第一个未被匹配的记录           │    │
│  │ (不校验收款人，仅同金额 + 日期窗口)                   │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**关键代码**: `packages/loot-core/src/server/accounts/sync.ts:743-889

```typescript
// 第一轮: imported_id 精确匹配
if (trans.imported_id) {
  const table = reimportDeleted ? 'v_transactions' : 'v_transactions_internal';
  match = await db.first(
    `SELECT * FROM ${table} WHERE imported_id = ? AND account = ?`,
    [trans.imported_id, acctId]
  );
  if (match) hasMatched.add(match.id);
}

// 若未匹配，准备模糊数据集
if (!match) {
  const sevenDaysBefore = db.toDateRepr(monthUtils.subDays(trans.date, 7));
  const sevenDaysAfter = db.toDateRepr(monthUtils.addDays(trans.date, 7));
  
  fuzzyDataset = await db.all(
    `SELECT ... FROM v_transactions
     WHERE date >= ? AND date <= ? AND amount = ? AND account = ?`,
    [sevenDaysBefore, sevenDaysAfter, trans.amount || 0, acctId]
  );
  // 按日期距离排序
  fuzzyDataset = fuzzyDataset.sort((a, b) => aDistance > bDistance ? 1 : -1);
}

// 第二轮: 收款人 ID 匹配 (高保真模糊匹配)
const transactionsStep2 = transactionsStep1.map(data => {
  if (!data.match && data.fuzzyDataset) {
    const match = data.fuzzyDataset.find(
      row => !hasMatched.has(row.id) && data.trans.payee === row.payee
    );
    if (match) {
      hasMatched.add(match.id);
      return { ...data, match };
    }
  }
  return data;
});

// 第三轮: 第一个未匹配 (低保真)
const transactionsStep3 = transactionsStep2.map(data => {
  if (!data.match && data.fuzzyDataset) {
    const match = data.fuzzyDataset.find(row => !hasMatched.has(row.id));
    if (match) {
      hasMatched.add(match.id);
      return { ...data, match };
    }
  }
  return data;
});
```

### 4.3 三种格式去重能力对比

| 格式 | imported_id 精确匹配 | 两轮模糊匹配 | 整体去重能力 |
|------|---------------------|----------------|---------------|
| **OFX** | ✅ FITID → 最高保真度 | ✅ 作为兜底 | 强 |
| **CSV** | ❌ 无 | ✅ 仅两轮模糊 | 弱 |
| **QIF** | ❌ 无 | ✅ 仅两轮模糊 | 弱 |

### 4.4 匹配结果处理

```
匹配结果 → 三种处理方式:
│
├── 匹配到已对账交易 (match.reconciled = true
│   → ignored=true (跳过更新，保护已对账)
│
├── 匹配到未对账交易
│   → 检查字段是否变化
│       ├── 有变化 → updated (更新)
│       └── 无变化 → ignored=true (跳过)
│
└── 未匹配
    └── added (新增)
```

---

## 五、用户交互复杂度对比

```
┌─────────────────────────────────────────────────────────────────┐
│  用户界面复杂度                                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  CSV (最复杂)                                                   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ ✓ 字段映射选择器 (Date/Payee/Amount/Notes/Category)      │   │
│  │ ✓ 日期格式下拉 (6种)                                     │   │
│  │ ✓ 分隔符选择 (, ; | tab ~)                              │   │
│  │ ✓ 跳行设置 (开头/结尾)                                   │   │
│  │ ✓ 表头开关                                               │   │
│  │ ✓ 收支分列模式开关                                       │   │
│  │ ✓ 收支标识列模式 + 支出值设置                            │   │
│  │ ✓ 金额翻转开关                                           │   │
│  │ ✓ 乘数设置                                               │   │
│  │ ✓ 对账开关 (默认开启)                                   │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  QIF (中等)                                                     │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ ✓ 日期格式下拉 (6种)                                     │   │
│  │ ✓ 交换 Payee/Memo 开关                                   │   │
│  │ ✓ 金额翻转开关                                           │   │
│  │ ✓ 乘数设置                                               │   │
│  │ ✓ 对账开关 (默认开启)                                   │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  OFX (最简单)                                                   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ ✓ Memo 回退 Payee 开关                                   │   │
│  │ ✓ 交换 Payee/Memo 开关                                   │   │
│  │ ✓ 对账开关 (默认开启)                                   │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 六、设计决策分析

### 6.1 为什么 CSV 的归一化放在客户端？

**原因**:
1. **CSV 无标准 schema**: 不同银行的字段名千差万别
   - 日期字段: `Date`, `Transaction Date`, `Posting Date`, `交易日期`...
   - 金额字段: `Amount`, `Amount (USD)`, `交易金额`...
2. **灵活配置记忆**: 用户只需配置一次，后续按账户记忆
3. **预览驱动**: 用户在预览时调整映射，实时看到效果

### 6.2 为什么 OFX 的归一化放在服务端？

**原因**:
1. **有严格标准**: OFX 规范定义了固定字段名
2. **解析复杂**: SGML/XML 解析适合在服务端完成
3. **安全性**: 银行导出的文件可能包含敏感头部信息

### 6.3 为什么预览只做比对不落库？

**设计原因**:
1. **用户确认**: 预览让用户在真正导入前查看哪些交易会新增、哪些会更新、哪些被忽略
2. **幂等性**: 预览可反复执行，不会影响数据库状态
3. **批量确认**: 用户可取消或调整后再确认导入

**关键实现** (`sync.ts:677-680`):

```typescript
if (!isPreview) {
  // 仅在真正导入时执行落库操作
  await createNewPayees(payeesToCreate, [...added, ...updated]);
  await batchUpdateTransactions({ added, updated });
}
// isPreview=true 时仅返回 updatedPreview 供 UI 展示
```

### 6.4 配置持久化策略

| 格式 | 配置项 | 持久化 |
|------|--------|--------|
| CSV | 字段映射、分隔符、跳行、日期格式 | ✅ 按账户保存到 prefs |
| OFX | 回退开关、交换开关 | ✅ 按账户保存到 prefs |
| QIF | 日期格式、交换开关 | ✅ 按账户保存到 prefs |

---

## 七、总结

### 7.1 格式推荐优先级

1. **OFX/QFX** → 首选，银行标准支持，三轮去重能力最强，配置最少
2. **CSV** → 次选，万能格式但需要一次性配置
3. **QIF** → 古老格式，仅当其他格式不可用时使用

### 7.2 核心差异根源

```
格式标准化程度
    ↓
OFX > QIF > CSV
    ↓
服务端归一化程度
    ↓
OFX (高) → QIF (中) → CSV (低)
    ↓
客户端配置复杂度
    ↓
OFX (低) → QIF (中) → CSV (高)
    ↓
去重能力
    ↓
OFX (强) → QIF (弱) → CSV (弱)
```

### 7.3 关键代码分布

| 模块 | 文件路径 | 职责 |
|------|----------|------|
| 服务端入口 | `loot-core/.../import/parse-file.ts` | 格式分发、基础归一化 |
| OFX 解析 | `loot-core/.../import/ofx2json.ts` | SGML/XML 解析 |
| QIF 解析 | `loot-core/.../import/qif2json.ts` | 逐行标记解析 |
| 对账与去重 | `loot-core/.../accounts/sync.ts` | reconcileTransactions, matchTransactions |
| 直接新增 | `loot-core/.../accounts/sync.ts` | addTransactions (关闭对账) |
| 客户端 UI | `desktop-client/.../ImportTransactionsModal.tsx` | 预览、映射、导入 |
| 客户端工具 | `desktop-client/.../ImportTransactionsModal/utils.ts` | 日期/金额/映射逻辑 |
| 字段映射 | `desktop-client/.../ImportTransactionsModal/FieldMappings.tsx` | CSV 字段映射 UI |
| 客户端调用 | `desktop-client/.../accounts/mutations.ts` | useImportTransactionsMutation (reconcile 开关) |

---

## 八、附录：关键文件索引

| 文件 | 行数范围 | 说明 |
|------|----------|------|
| `parse-file.ts` | 77-107 | 入口分发逻辑 |
| `parse-file.ts` | 109-156 | CSV 解析 (原始输出) |
| `parse-file.ts` | 158-198 | QIF 解析 + 部分归一化 |
| `parse-file.ts` | 200-248 | OFX 解析 + 完整归一化 |
| `ofx2json.ts` | 126-157 | OFX 主解析函数 |
| `ofx2json.ts` | 48-99 | 多账户类型提取 (含投资边界) |
| `qif2json.ts` | 44-104 | QIF 逐行标记解析 |
| `sync.ts` | 538-696 | reconcileTransactions (对账 + 预览控制) |
| `sync.ts` | 698-897 | matchTransactions (三轮去重匹配) |
| `sync.ts` | 899-966 | addTransactions (关闭对账直接新增) |
| `ImportTransactionsModal.tsx` | 98-161 | CSV 自动字段映射检测 |
| `utils.ts` | 153-170 | 字段映射应用 |
| `utils.ts` | 190-263 | 金额字段解析 (split/inOut/flip) |
| `mutations.ts` | 268-338 | useImportTransactionsMutation (reconcile 开关逻辑) |
