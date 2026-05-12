# 账户间转账配对识别机制

## 概述

Actual Budget 通过 **收款人（Payee）关联机制** 和 **transaction.transfer_id 双向绑定** 实现账户间转账的自动配对。系统支持两种配对方式：

1. **自动配对**：用户选择"转账收款人"时自动创建配对
2. **手动配对**：用户手动选择两笔交易进行配对

---

## 核心数据结构

### 1. 转账收款人（Transfer Payee）

每个账户有一个对应的"转账收款人"，通过 `payees.transfer_acct` 字段标识：

```
账户 A → payees 表中有一条记录，其中 transfer_acct = A
账户 B → payees 表中有一条记录，其中 transfer_acct = B
```

- 当交易的 `payee` 指向一个 `transfer_acct` 非空的收款人时，系统认为这是一笔转账
- `transfer_acct` 的值即为**目标账户 ID**

### 2. 交易配对字段

- `transaction.transfer_id`：指向配对交易的 ID（双向引用）
- 两笔配对交易互相指向对方的 ID

---

## 识别条件

### 自动配对条件

在 `packages/loot-core/src/shared/transfer.ts:3-16` 中定义：

```typescript
export function validForTransfer(
  fromTransaction: TransactionEntity,
  toTransaction: TransactionEntity,
) {
  if (
    // 1. 两笔交易都尚未是转账
    [fromTransaction, toTransaction].every(tran => tran.transfer_id == null) &&
    // 2. 属于不同账户
    fromTransaction.account !== toTransaction.account &&
    // 3. 金额相加为零（一正一负）
    fromTransaction.amount + toTransaction.amount === 0
  ) {
    return true;
  }
  return false;
}
```

三个条件必须同时满足：
1. **非转账状态**：两笔交易的 `transfer_id` 都为 null
2. **跨账户**：两笔交易属于不同账户
3. **金额对称**：`amount + amount = 0`

---

## 触发时机

转账配对的逻辑集中在 `packages/loot-core/src/server/transactions/transfer.ts`。

### 1. 事务批量更新入口

在 `packages/loot-core/src/server/transactions/index.ts:40-151` 中，`batchUpdateTransactions` 函数在执行完数据库操作后，会触发转账处理：

```typescript
export async function batchUpdateTransactions({
  added,
  deleted,
  updated,
  runTransfers = true,  // 默认自动处理转账
  ...
}) {
  // ... 先执行数据库增删改 ...
  
  if (runTransfers) {
    await batchMessages(async () => {
      // 新增交易 → 触发 onInsert
      await Promise.all(allAdded.map(t => transfer.onInsert(t)));
      
      // 更新交易 → 触发 onUpdate
      transfersUpdated = (
        await Promise.all(allUpdated.map(t => transfer.onUpdate(t)))
      ).filter(Boolean);
      
      // 删除交易 → 触发 onDelete
      await Promise.all(allDeleted.map(t => transfer.onDelete(t)));
    });
  }
}
```

### 2. 插入时触发：`onInsert`

位置：`packages/loot-core/src/server/transactions/transfer.ts:141-147`

```typescript
export async function onInsert(transaction) {
  // 检查收款人是否指向转账账户
  const transferredAccount = await getTransferredAccount(transaction);

  if (transferredAccount) {
    // 是转账 → 创建配对交易
    return addTransfer(transaction, transferredAccount);
  }
}
```

**触发条件**：
- 新插入的交易，其 `payee` 对应的收款人有 `transfer_acct`
- 即：用户选择了"转账到某个账户"的收款人

### 3. 更新时触发：`onUpdate`

位置：`packages/loot-core/src/server/transactions/transfer.ts:155-173`

```typescript
export async function onUpdate(transaction) {
  const transferredAccount = await getTransferredAccount(transaction);

  // 情况 A：父交易（拆分交易的父）→ 移除转账配对
  if (transaction.is_parent) {
    return removeTransfer(transaction);
  }

  // 情况 B：之前不是转账，现在变成转账 → 创建配对
  if (transferredAccount && !transaction.transfer_id) {
    return addTransfer(transaction, transferredAccount);
  }

  // 情况 C：之前是转账，现在不是 → 移除配对
  if (!transferredAccount && transaction.transfer_id) {
    return removeTransfer(transaction);
  }

  // 情况 D：一直是转账，可能改变了目标账户 → 更新配对
  if (transferredAccount && transaction.transfer_id) {
    return updateTransfer(transaction, transferredAccount);
  }
}
```

**四种更新场景**：

| 场景 | 之前状态 | 现在状态 | 操作 |
|------|---------|---------|------|
| A | 任何 | 变成父交易 | 移除配对 |
| B | 非转账 | 转账 | 创建配对 |
| C | 转账 | 非转账 | 移除配对 |
| D | 转账（账户X） | 转账（账户Y） | 更新配对 |

### 4. 删除时触发：`onDelete`

位置：`packages/loot-core/src/server/transactions/transfer.ts:149-153`

```typescript
export async function onDelete(transaction) {
  if (transaction.transfer_id) {
    // 被删除的交易有配对 → 也需要处理配对方
    await removeTransfer(transaction);
  }
}
```

---

## 配对创建流程

### `addTransfer` 函数

位置：`packages/loot-core/src/server/transactions/transfer.ts:47-93`

```
输入：原始交易 + 目标账户
↓
1. 检查是否是父交易（is_parent=true）→ 跳过
2. 获取源账户的转账收款人
3. 创建配对交易：
   - account: 目标账户
   - amount: -原始金额（取反）
   - payee: 源账户的转账收款人
   - date: 相同日期
   - transfer_id: 原始交易 ID
   - notes: 相同备注
   - schedule: 相同计划
   - cleared: false
4. 运行交易规则
5. 插入配对交易到数据库
6. 更新原始交易的 transfer_id = 配对交易 ID
7. 清除分类（如果需要）
8. 返回更新信息
```

**双向绑定完成**：
- 交易 A 的 `transfer_id` = 交易 B 的 ID
- 交易 B 的 `transfer_id` = 交易 A 的 ID

### 分类清除规则

位置：`packages/loot-core/src/server/transactions/transfer.ts:24-45`

```typescript
async function clearCategory(transaction, transferAcct) {
  // 检查两个账户的预算状态
  const { offbudget: fromOffBudget } = await db.first(...);
  const { offbudget: toOffBudget } = await db.first(...);

  // 情况1：两个都是预算内账户 → 清除分类
  // 情况2：两个都是预算外账户 → 清除分类
  if (fromOffBudget === toOffBudget) {
    await db.updateTransaction({ id: transaction.id, category: null });
    if (transaction.transfer_id) {
      await db.updateTransaction({ id: transaction.transfer_id, category: null });
    }
    return true;
  }
  // 情况3：一个预算内、一个预算外 → 保留分类
  return false;
}
```

**设计意图**：同类型账户间的转账只是资金转移，不涉及消费/收入分类。

---

## 撤销情形

### `removeTransfer` 函数

位置：`packages/loot-core/src/server/transactions/transfer.ts:95-118`

```typescript
export async function removeTransfer(transaction) {
  // 获取配对交易
  const transferTrans = await db.getTransaction(transaction.transfer_id);

  if (transferTrans) {
    if (transferTrans.is_child) {
      // 配对交易是拆分交易的子交易：
      // → 不能删除（会破坏整个拆分交易）
      // → 只能解除配对关系，变成普通交易
      await db.updateTransaction({
        id: transaction.transfer_id,
        transfer_id: null,
        payee: null,
      });
    } else {
      // 配对交易是普通交易：直接删除
      await db.deleteTransaction({ id: transaction.transfer_id });
    }
  }
  // 清除当前交易的配对关系
  await db.updateTransaction({ id: transaction.id, transfer_id: null });
  return { id: transaction.id, transfer_id: null };
}
```

### 触发撤销的场景

#### 场景 1：用户将转账收款人改为普通收款人

- 位置：`onUpdate` 情况 C
- 条件：`!transferredAccount && transaction.transfer_id`
- 操作：调用 `removeTransfer`

#### 场景 2：用户删除转账中的任意一笔

- 位置：`onDelete`
- 条件：`transaction.transfer_id` 非空
- 操作：调用 `removeTransfer`

#### 场景 3：普通交易变为拆分交易的父交易

- 位置：`onUpdate` 情况 A
- 条件：`transaction.is_parent`
- 操作：调用 `removeTransfer`
- 原因：拆分交易的转账应在**子交易**级别处理

#### 场景 4：用户更新时改变了转账目标账户

- 位置：`onUpdate` 情况 D → `updateTransfer`
- 操作：`updateTransfer` 直接更新配对交易的账户信息
- 注意：这不算"撤销"，而是"更新"

### 撤销时的两种处理策略

| 配对交易类型 | 处理方式 | 原因 |
|------------|---------|------|
| **普通交易** | 删除配对交易 | 它是系统自动创建的，用户无需保留 |
| **子交易（拆分）** | 解除配对，变为普通交易 | 删除子交易会破坏整个拆分交易 |

---

## 手动配对机制

位置：`packages/desktop-client/src/hooks/useTransactionBatchActions.ts:519-567`

### 流程

```
1. 用户选择两笔交易
2. 点击"设为转账"
3. 系统调用 validForTransfer 验证
4. 验证通过后，直接更新两笔交易：
   - 互相设置 transfer_id
   - 清除 category
   - 设置对方账户的转账 payee
5. 发送 batch-update，设置 runTransfers: false
   （避免再次触发自动处理）
```

### 代码逻辑

```typescript
if (transactions.length === 2 && validForTransfer(fromTrans, toTrans)) {
  // 找到两个账户对应的转账收款人
  const fromPayee = payees.find(p => p.transfer_acct === fromTrans.account);
  const toPayee = payees.find(p => p.transfer_acct === toTrans.account);

  const changes = {
    updated: [
      {
        ...fromTrans,
        category: null,      // 清除分类
        payee: toPayee?.id,  // 设置为目标账户的转账收款人
        transfer_id: toTrans.id,  // 双向绑定
      },
      {
        ...toTrans,
        category: null,
        payee: fromPayee?.id,
        transfer_id: fromTrans.id,
      },
    ],
    runTransfers: false,  // 关键：跳过自动转账处理
  };

  await send('transactions-batch-update', changes);
}
```

**为什么设置 `runTransfers: false`？**
- 手动配对已经设置好了 `transfer_id`
- 如果再次触发自动处理，`onUpdate` 会检测到：
  - `transferredAccount` 非空（因为 payee 是转账收款人）
  - `transfer_id` 非空
  - 进入情况 D（更新转账）
- 这是冗余的，甚至可能导致问题

---

## 拆分交易的特殊处理

### 父交易（is_parent = true）

位置：`transfer.ts:48-54` 和 `transfer.ts:158-160`

```typescript
// addTransfer 中
if (transaction.is_parent) {
  // 父交易不能自动创建转账
  // 应该用子交易来创建转账
  return null;
}

// onUpdate 中
if (transaction.is_parent) {
  // 如果变成父交易，移除之前的转账配对
  return removeTransfer(transaction);
}
```

### 子交易（is_child = true）

位置：`transfer.ts:103-111`

```typescript
if (transferTrans.is_child) {
  // 撤销配对时，不能删除子交易
  // 只能解除配对关系
  await db.updateTransaction({
    id: transaction.transfer_id,
    transfer_id: null,
    payee: null,
  });
}
```

**设计意图**：
- 拆分交易的父交易是"汇总"，金额是子交易的总和
- 转账应该发生在子交易级别，每个子交易可以转账到不同账户
- 删除子交易会破坏整个拆分交易的完整性

---

## 完整流程图

```
用户操作
    │
    ▼
┌─────────────────────────────────────────┐
│  batchUpdateTransactions                │
│  (默认 runTransfers = true)             │
└────────────────┬────────────────────────┘
                 │
    ┌────────────┼────────────┐
    │            │            │
    ▼            ▼            ▼
  新增         更新         删除
    │            │            │
    ▼            ▼            ▼
┌────────┐  ┌────────┐  ┌────────┐
│onInsert│  │onUpdate│  │onDelete│
└───┬────┘  └───┬────┘  └───┬────┘
    │            │            │
    │    ┌───────┼───────┐    │
    │    │       │       │    │
    ▼    ▼       ▼       ▼    ▼
┌─────────┐ ┌──────────┐ ┌──────────┐
│addTransfer│ │updateTransfer│ │removeTransfer│
└────┬────┘ └─────┬────┘ └─────┬────┘
     │            │            │
     ▼            ▼            ▼
  创建配对    更新配对信息   解除/删除配对
  双向绑定    (账户、金额等)  清理 transfer_id
```

---

## 关键文件索引

| 文件路径 | 职责 |
|---------|------|
| `packages/loot-core/src/shared/transfer.ts` | 手动配对验证逻辑 `validForTransfer` |
| `packages/loot-core/src/server/transactions/transfer.ts` | 自动配对核心逻辑：`onInsert/onUpdate/onDelete/addTransfer/updateTransfer/removeTransfer` |
| `packages/loot-core/src/server/transactions/index.ts` | 事务入口：`batchUpdateTransactions`，触发转账处理 |
| `packages/loot-core/src/server/transactions/transfer.test.ts` | 完整测试用例 |
| `packages/desktop-client/src/hooks/useTransactionBatchActions.ts` | UI 层手动配对逻辑 `onSetTransfer` |
