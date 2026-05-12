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

## 撤销边界场景深度分析

### 1. 配对交易同时删除的边界场景

#### 问题场景

转账配对通过 `transfer_id` 双向绑定：
- 交易 A 的 `transfer_id` = 交易 B 的 ID
- 交易 B 的 `transfer_id` = 交易 A 的 ID

当用户**同时选中两笔配对交易并批量删除**时，会发生什么？

#### 执行时序

位置：`packages/loot-core/src/server/transactions/index.ts:94-149`

```
Step 1: 数据库删除阶段（先于转账处理）
  ├── 调用 deleteTransaction(A)
  │   └── 实际上设置 tombstone = 1（软删除）
  └── 调用 deleteTransaction(B)
      └── 实际上设置 tombstone = 1

Step 2: 转账后处理阶段
  ├── 调用 onDelete(A)
  │   └── 检查 A.transfer_id 非空 → 调用 removeTransfer(A)
  │       ├── getTransaction(B)
  │       │   └── 查询 v_transactions 视图
  │       │       └── 视图过滤条件: tombstone = 0
  │       │           └── B 已被设置 tombstone=1 → 返回 undefined
  │       └── transferTrans = undefined
  │           └── 跳过配对交易处理块
  └── 调用 onDelete(B)
      └── 检查 B.transfer_id 非空 → 调用 removeTransfer(B)
          ├── getTransaction(A)
          │   └── A 已被设置 tombstone=1 → 返回 undefined
          └── transferTrans = undefined
              └── 跳过配对交易处理块
```

#### 关键代码：软删除机制

位置：`packages/loot-core/src/server/db/index.ts:258-268`

```typescript
export async function delete_(table, id) {
  await sendMessages([
    {
      dataset: table,
      row: id,
      column: 'tombstone',
      value: 1,           // 关键：不是物理删除，而是打 tombstone 标记
      timestamp: Timestamp.send(),
    },
  ]);
}
```

#### 关键代码：getTransaction 的视图过滤

位置：`packages/loot-core/src/server/db/index.ts:781-788`

```typescript
export async function getTransaction(id: DbViewTransaction['id']) {
  const rows = await selectWithSchema(
    'transactions',
    'SELECT * FROM v_transactions WHERE id = ?',  // v_transactions 视图
    [id],
  );
  return rows[0];  // 找不到时返回 undefined
}
```

#### 兜底处理方案

`removeTransfer` 函数已经内置了兜底逻辑：

位置：`packages/loot-core/src/server/transactions/transfer.ts:95-118`

```typescript
export async function removeTransfer(transaction) {
  const transferTrans = await db.getTransaction(transaction.transfer_id);

  if (transferTrans) {
    // 配对交易存在 → 正常处理（删除或解除配对）
    if (transferTrans.is_child) {
      await db.updateTransaction({
        id: transaction.transfer_id,
        transfer_id: null,
        payee: null,
      });
    } else {
      await db.deleteTransaction({ id: transaction.transfer_id });
    }
  }
  // ⭐ 兜底逻辑：无论配对交易是否找到，都要清理当前交易的 transfer_id
  await db.updateTransaction({ id: transaction.id, transfer_id: null });
  return { id: transaction.id, transfer_id: null };
}
```

#### 为什么这是安全的？

1. **数据库删除先于转账处理**：两笔交易已经被设置 `tombstone = 1`，从业务视角已经"删除"
2. **`getTransaction` 返回 undefined 不是错误**：代码用 `if (transferTrans)` 显式检查，这是预期的正常场景
3. **兜底更新保证幂等性**：即使配对交易找不到，当前交易的 `transfer_id` 也会被设为 null，避免留下"悬挂引用"

### 2. 先后快速删除的边界场景

#### 问题场景

用户在短时间内连续删除两笔配对交易（例如在 UI 上快速点击两次）。

#### 时序分析

```
时刻 t1: 删除交易 A
  ├── 数据库: A.tombstone = 1
  └── 转账处理:
      ├── onDelete(A) → removeTransfer(A)
      │   ├── getTransaction(B) → B 还存在
      │   ├── deleteTransaction(B) → B.tombstone = 1
      │   └── updateTransaction(A) → A.transfer_id = null
      └── 完成

时刻 t2: 删除交易 B（用户第二次点击）
  ├── 数据库: B.tombstone 已是 1（重复设置无影响）
  └── 转账处理:
      ├── onDelete(B) → removeTransfer(B)
      │   ├── getTransaction(A) → A.tombstone=1 → undefined
      │   └── 兜底逻辑: updateTransaction(B) → B.transfer_id = null
      └── 完成（无副作用）
```

#### 为什么不会出问题？

1. **tombstone 是幂等操作**：重复设置 `tombstone = 1` 不会产生副作用
2. **`removeTransfer` 是幂等函数**：
   - 如果配对交易已被删除 → 跳过处理块
   - 无论如何都会清理当前交易的 `transfer_id`
3. **并发安全**：即使两次删除几乎同时发生，最多只会：
   - 第一笔删除时正常删除配对交易
   - 第二笔删除时发现配对交易已不存在，执行兜底逻辑

### 3. 兜底时哪些字段会被清理？

#### 场景 A：配对交易存在（正常撤销）

| 字段 | 处理方式 |
|-----|---------|
| `transfer_id` | 设为 null（双向） |
| `payee` | 仅当是子交易时设为 null |
| 其他字段 | 保持不变 |

代码路径：
```typescript
if (transferTrans) {
  if (transferTrans.is_child) {
    // 子交易：只清除配对标识
    await db.updateTransaction({
      id: transaction.transfer_id,
      transfer_id: null,  // 清除配对引用
      payee: null,        // 清除转账收款人
    });
  } else {
    // 普通交易：直接删除
    await db.deleteTransaction({ id: transaction.transfer_id });
  }
}
// 当前交易的 transfer_id 总是被清理
await db.updateTransaction({ id: transaction.id, transfer_id: null });
```

#### 场景 B：配对交易不存在（兜底场景）

| 字段 | 处理方式 |
|-----|---------|
| `transfer_id` | 设为 null（仅当前交易） |
| `payee` | **不处理**（配对交易不存在，无需处理） |
| 其他字段 | 保持不变 |

代码路径：
```typescript
const transferTrans = await db.getTransaction(transaction.transfer_id);
// transferTrans = undefined → 跳过 if 块

// 只执行这一行兜底逻辑
await db.updateTransaction({ id: transaction.id, transfer_id: null });
```

### 4. 为什么不会破坏拆分交易一致性？

#### 拆分交易数据结构

```
父交易 (is_parent = true)
  ├── 子交易 1 (is_child = true, parent_id = 父ID)
  ├── 子交易 2 (is_child = true, parent_id = 父ID)
  └── 子交易 3 (is_child = true, parent_id = 父ID)
```

#### 保护机制 1：父交易不能自动创建转账

位置：`transfer.ts:48-54`

```typescript
export async function addTransfer(transaction, transferredAccount) {
  if (transaction.is_parent) {
    // 父交易不能创建转账
    // 应该用子交易来创建转账
    return null;
  }
  // ... 后续逻辑
}
```

**原因**：父交易的金额是子交易的总和，如果父交易创建转账，会导致金额重复计算。

#### 保护机制 2：父交易变为拆分时自动解除转账

位置：`transfer.ts:158-160`

```typescript
if (transaction.is_parent) {
  // 如果普通交易变成了父交易
  // 必须移除它的转账配对
  return removeTransfer(transaction);
}
```

**原因**：防止出现"既是父交易又是转账"的不一致状态。

#### 保护机制 3：子交易作为配对时，撤销不删除

位置：`transfer.ts:103-111`

```typescript
if (transferTrans.is_child) {
  // ⭐ 关键保护：子交易不能删除
  // 只能解除配对关系，变成普通交易
  await db.updateTransaction({
    id: transaction.transfer_id,
    transfer_id: null,  // 解除配对
    payee: null,        // 清除转账收款人
  });
} else {
  // 普通配对交易：可以删除
  await db.deleteTransaction({ id: transaction.transfer_id });
}
```

#### 为什么不能删除子交易？

**数据一致性风险**：

```
拆分交易结构：
父交易 P (is_parent=true, amount=100)
  ├── 子交易 C1 (is_child=true, amount=60, parent_id=P)
  └── 子交易 C2 (is_child=true, amount=40, parent_id=P)

如果 C2 是转账配对交易，被删除时：

❌ 错误做法（直接删除）：
  ├── C2 被物理删除
  └── 父交易 P 的 amount 仍为 100
      └── 但子交易总和 = 60 ≠ 100 → 数据不一致！

✅ 正确做法（解除配对）：
  ├── C2 的 transfer_id = null
  ├── C2 的 payee = null
  ├── C2 仍然存在
  └── 父交易 P 的 amount = 100，子交易总和 = 60 + 40 = 100 → 一致！
```

#### 设计原则

| 交易类型 | 撤销时处理 | 原因 |
|---------|-----------|------|
| **普通配对交易** | 删除 | 它是系统自动创建的，用户没手动维护它 |
| **子交易（配对）** | 解除配对，变为普通交易 | 子交易是拆分交易结构的一部分，删除会破坏一致性 |
| **父交易** | 不能作为转账（创建时直接跳过） | 父交易是汇总，转账应该在子交易级别 |

### 5. 边界场景测试用例

位置：`packages/loot-core/src/server/transactions/transfer.test.ts:52-220`

测试文件中覆盖了以下边界场景：

```typescript
// 场景 1：转账被正确插入/更新/删除（第 53 行测试）
// 验证：创建 → 更新 → 取消转账 → 重新转账 → 删除

// 场景 2：转账被正确取消分类（第 134 行测试）
// 验证：同类型账户间转账清除分类，不同类型保留分类

// 场景 3：拆分交易的子交易转账被保留（第 174 行测试）
// 验证：
//   - 普通交易转账
//   - 变成父交易 → 转账被移除
//   - 添加子交易 → 子交易可以有自己的转账
//   - 子交易的 transfer_id ≠ 父交易原有的 transfer_id
```

---

## 测试证据核查

### 6.1 现有测试覆盖分析

#### 测试文件列表

| 文件路径 | 测试范围 |
|---------|---------|
| `packages/loot-core/src/shared/transfer.test.ts` | `validForTransfer` 函数的边界条件验证 |
| `packages/loot-core/src/server/transactions/transfer.test.ts` | 转账插入/更新/删除/分类清除/拆分交易 |

#### 逐条核对目标场景

##### 场景 A：同时删除两笔互配转账

**核查结论**：❌ **无直接测试**

**现有证据分析**：

1. **代码注释证据**（`transfer.ts:98-101`）
   ```typescript
   // Perform operations on the transfer transaction only
   // if it is found. For example: when users delete both
   // (in & out) transfer transactions at the same time -
   // transfer transaction will not be found.
   ```
   开发者**明确意识到**这个边界场景，并在注释中提到了"用户同时删除两笔转账交易"的情况。

2. **代码防御逻辑**（`transfer.ts:102`）
   ```typescript
   if (transferTrans) {
     // 只有在配对交易存在时才处理它
   }
   // ⭐ 无论配对交易是否存在，都执行兜底逻辑
   await db.updateTransaction({ id: transaction.id, transfer_id: null });
   ```
   代码结构本身就是防御性的：`if (transferTrans)` 条件判断 + `if` 块外的兜底更新。

3. **测试缺失**：
   - 没有测试用例调用 `batchUpdateTransactions` 并传入 `deleted: [txnA, txnB]`
   - 没有测试验证两笔交易同时被标记 `tombstone=1` 后，`removeTransfer` 的行为

**结论来源**：**代码推导**（基于代码结构和开发者注释）

---

##### 场景 B：先后快速删除两笔互配转账

**核查结论**：❌ **无直接测试**

**现有证据分析**：

1. **间接证据 1：单删测试**（`transfer.test.ts:129-131`）
   ```typescript
   await db.deleteTransaction(transaction);
   await transfer.onDelete(transaction);
   differ.expectToMatchDiff(await getAllTransactions());
   ```
   这个测试验证了**删除一笔转账会自动删除其配对方**。
   - 从快照 `transfer.test.ts.snap:592-655` 可以看到：两笔配对交易都消失了
   - 但这是**正常路径**测试，不是边界场景

2. **间接证据 2：幂等性设计**
   - `delete_` 函数只是设置 `tombstone = 1`，重复设置无副作用
   - `removeTransfer` 函数无论配对交易是否存在，都会清理当前交易的 `transfer_id`

3. **测试缺失**：
   - 没有测试模拟 "第一笔删除时连带删除第二笔，然后第二笔的删除请求再到来" 的时序
   - 没有并发/竞态条件测试

**结论来源**：**代码推导 + 间接测试证据**

---

### 6.2 已有测试覆盖的场景

| 场景 | 测试状态 | 证据位置 |
|-----|---------|---------|
| ✅ 单删转账（自动删除配对方） | 有直接测试 | `transfer.test.ts:129-131` + 快照 7 |
| ✅ 转账创建/更新 | 有直接测试 | `transfer.test.ts:53-132` + 快照 1-6 |
| ✅ 分类清除规则 | 有直接测试 | `transfer.test.ts:134-172` |
| ✅ 拆分交易子交易转账保留 | 有直接测试 | `transfer.test.ts:174-220` |
| ✅ `validForTransfer` 边界条件 | 有直接测试 | `shared/transfer.test.ts:28-74` |
| ❌ 同时删除两笔互配转账 | **无直接测试** | 只有代码注释 |
| ❌ 先后快速删除两笔互配转账 | **无直接测试** | 只有间接证据 |

---

### 6.3 缺失的验证证据

#### 缺失 1：同时删除两笔互配转账的集成测试

**建议新增测试用例**：

```typescript
test('deleting both paired transfers simultaneously works correctly', async () => {
  await prepareDatabase();
  
  // 1. 创建转账配对
  const transferTwo = await db.first(
    "SELECT * FROM payees WHERE transfer_acct = 'two'"
  );
  let txnA = {
    account: 'one',
    amount: 5000,
    payee: transferTwo.id,
    date: '2017-01-01',
  };
  txnA.id = await db.insertTransaction(txnA);
  await transfer.onInsert(txnA);
  
  txnA = await db.getTransaction(txnA.id);
  const txnB = await db.getTransaction(txnA.transfer_id);
  
  // 2. 验证配对建立
  expect(txnA.transfer_id).toBe(txnB.id);
  expect(txnB.transfer_id).toBe(txnA.id);
  
  // 3. 模拟 batchUpdateTransactions 的执行顺序：
  //    先数据库删除，再转账后处理
  //    这是关键：onDelete 执行时，两笔交易都已 tombstone=1
  await db.deleteTransaction(txnA);
  await db.deleteTransaction(txnB);
  
  // 4. 此时 getTransaction 应该返回 undefined
  const txnAAfterDelete = await db.getTransaction(txnA.id);
  const txnBAfterDelete = await db.getTransaction(txnB.id);
  expect(txnAAfterDelete).toBeUndefined();
  expect(txnBAfterDelete).toBeUndefined();
  
  // 5. 调用 onDelete（模拟转账后处理）
  //    这是核心验证：removeTransfer 在配对交易找不到时不应抛错
  await expect(transfer.onDelete(txnA)).resolves.not.toThrow();
  await expect(transfer.onDelete(txnB)).resolves.not.toThrow();
  
  // 6. 验证：即使配对交易不存在，函数也正常完成
  //    （虽然因为 tombstone=1，我们无法直接查询验证 transfer_id）
});
```

**为什么需要这个测试？**
- 验证 `getTransaction` 返回 `undefined` 时，`if (transferTrans)` 分支正确跳过
- 验证兜底的 `updateTransaction` 调用不会出错（即使交易已 tombstone）

---

#### 缺失 2：先后快速删除的时序测试

**建议新增测试用例**：

```typescript
test('deleting transfers in quick succession is idempotent', async () => {
  await prepareDatabase();
  
  // 1. 创建转账配对
  const transferTwo = await db.first(
    "SELECT * FROM payees WHERE transfer_acct = 'two'"
  );
  let txnA = {
    account: 'one',
    amount: 5000,
    payee: transferTwo.id,
    date: '2017-01-01',
  };
  txnA.id = await db.insertTransaction(txnA);
  await transfer.onInsert(txnA);
  
  txnA = await db.getTransaction(txnA.id);
  const txnB = await db.getTransaction(txnA.transfer_id);
  
  // 2. 第一次删除：正常路径，会连带删除 txnB
  await db.deleteTransaction(txnA);
  await transfer.onDelete(txnA);
  
  // 3. 验证 txnB 也被删除了（tombstone=1）
  const txnBAfterFirstDelete = await db.getTransaction(txnB.id);
  expect(txnBAfterFirstDelete).toBeUndefined();
  
  // 4. 模拟用户快速点击删除 txnB（竞态场景）
  //    此时 txnB 已被标记 tombstone=1
  await db.deleteTransaction(txnB);  // 幂等：重复设置 tombstone=1
  await expect(transfer.onDelete(txnB)).resolves.not.toThrow();  // 不应抛错
  
  // 5. 验证：没有副作用，系统状态一致
  const allTxns = await getAllTransactions();
  // 两笔交易都不应出现在 v_transactions 视图中
  const txnAInView = allTxns.find(t => t.id === txnA.id);
  const txnBInView = allTxns.find(t => t.id === txnB.id);
  expect(txnAInView).toBeUndefined();
  expect(txnBInView).toBeUndefined();
});
```

**为什么需要这个测试？**
- 验证 `removeTransfer` 的幂等性
- 验证第二次删除时，`getTransaction` 返回 `undefined` 后的行为
- 验证系统最终状态的一致性

---

#### 缺失 3：通过 batchUpdateTransactions 的端到端测试

**建议新增测试用例**：

```typescript
import { batchUpdateTransactions } from './index';

test('batch deleting both paired transfers via batchUpdateTransactions', async () => {
  await prepareDatabase();
  
  // 1. 创建转账配对
  // ...（同上）
  
  txnA = await db.getTransaction(txnA.id);
  const txnB = await db.getTransaction(txnA.transfer_id);
  
  // 2. 通过 batchUpdateTransactions 同时删除
  //    这才是真实的用户操作路径
  const result = await batchUpdateTransactions({
    deleted: [{ id: txnA.id }, { id: txnB.id }],
  });
  
  // 3. 验证结果
  expect(result.deleted).toHaveLength(2);
  
  // 4. 验证最终状态
  const allTxns = await getAllTransactions();
  expect(allTxns.some(t => t.id === txnA.id)).toBe(false);
  expect(allTxns.some(t => t.id === txnB.id)).toBe(false);
});
```

**为什么需要这个测试？**
- 现有测试都只是直接调用 `transfer.onDelete`
- 真实场景是用户通过 `batchUpdateTransactions` 触发
- 需要验证整个链路：数据库删除 → `getTransactionsByIds` → `onDelete`

---

### 6.4 证据等级总结

| 结论 | 证据类型 | 置信度 |
|-----|---------|-------|
| 单删转账自动删除配对方 | 直接测试 | ⭐⭐⭐⭐⭐ |
| 同时删除两笔时系统安全 | 代码结构 + 开发者注释 | ⭐⭐⭐ |
| 先后快速删除时系统安全 | 代码结构 + 幂等性设计 | ⭐⭐⭐ |
| 子交易作为配对时不被删除 | 直接测试 | ⭐⭐⭐⭐⭐ |
| 父交易不能作为转账 | 直接测试 | ⭐⭐⭐⭐⭐ |

### 6.5 测试差距评估

**高优先级缺失**：
1. 同时删除两笔互配转账的集成测试
2. 通过 `batchUpdateTransactions` 的端到端删除测试

**中优先级缺失**：
3. 先后快速删除的时序测试
4. 并发删除的竞态条件测试（虽然在单线程 Node.js 环境中可能不是高风险）

**低优先级**：
5. 性能测试（大量转账同时删除的处理效率）

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
