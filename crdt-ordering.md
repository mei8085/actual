# Actual Budget CRDT 时间戳与冲突合并机制

## 概述

Actual Budget 采用了基于 Hybrid Unique Logical Clock (HULC) 的 CRDT 实现，用于在多设备环境下实现分布式数据同步。这种机制能够处理以下场景：

- 多设备离线编辑后的数据合并
- 时钟漂移问题
- 网络延迟导致的乱序消息
- 最终一致性保证

---

## 1. 时间戳生成机制

### 1.1 HULC 时间戳结构

Actual Budget 使用的时间戳由三部分组成，序列化后为 46 字符的可排序字符串：

```
格式: ISO时间-计数器-节点ID
示例: 2015-04-24T22:23:42.123Z-1000-0123456789ABCDEF
```

**组成部分详解：**

1. **millis (毫秒级时间戳)**：
   - 使用系统时钟的毫秒时间
   - 可序列化和反序列化为 ISO 8601 格式
   - 范围: 1970-01-01T00:00:00.000Z ~ 9999-12-31T23:59:59.999Z

2. **counter (16位计数器)**：
   - 范围: 0000 ~ FFFF (0 ~ 65535)
   - 十六进制表示，左补零至 4 位
   - 当时间戳未增加时，用于区分同一时间点的不同事件
   - 防止在同一毫秒内生成的事件顺序混乱

3. **node (16位十六进制节点ID)**：
   - 由 UUID 后16位生成
   - 确保全局唯一性
   - 用于区分不同设备/实例

### 1.2 时间戳生成算法

#### 发送时生成 (Timestamp.send())

当本地生成消息时，使用 `Timestamp.send()` 方法生成时间戳：

**算法逻辑：**

```typescript
// 1. 获取当前物理时间
const phys = Date.now();

// 2. 获取之前的逻辑时间和计数器
const lOld = clock.timestamp.millis();
const cOld = clock.timestamp.counter();

// 3. 计算新的逻辑时间（取物理时间和旧逻辑时间的最大值）
const lNew = Math.max(lOld, phys);

// 4. 如果逻辑时间未增加，计数器加1；否则重置为0
const cNew = lOld === lNew ? cOld + 1 : 0;
```

**设计要点：**
- **逻辑时间永不倒退**：即使系统时钟回退，逻辑时间也保持单调递增
- **时钟漂移检测**：如果 `lNew - phys > 5分钟`，抛出 `ClockDriftError`
- **计数器溢出保护**：如果计数器超过 65535，抛出 `OverflowError`

#### 接收时同步 (Timestamp.recv())

当接收来自其他节点的消息时，使用 `Timestamp.recv()` 同步本地时钟：

**算法逻辑：**

```typescript
// 1. 获取当前物理时间
const phys = Date.now();

// 2. 获取消息中的时间和计数器
const lMsg = msg.millis();
const cMsg = msg.counter();

// 3. 获取本地旧时间和计数器
const lOld = clock.timestamp.millis();
const cOld = clock.timestamp.counter();

// 4. 新的逻辑时间取三者最大值
const lNew = Math.max(Math.max(lOld, phys), lMsg);

// 5. 计数器计算逻辑：
const cNew =
  lNew === lOld && lNew === lMsg
    ? Math.max(cOld, cMsg) + 1      // 三者时间相同，取最大计数器加1
    : lNew === lOld
      ? cOld + 1                    // 本地时间最大，本地计数器加1
      : lNew === lMsg
        ? cMsg + 1                  // 消息时间最大，消息计数器加1
        : 0;                        // 物理时间最大，重置计数器
```

**设计要点：**
- 确保本地时钟始终比所有已知时钟（本地、消息、物理）更晚
- 保留了逻辑时间的因果关系
- 接收消息会更新本地时钟，用于后续消息的时间戳生成

### 1.3 节点 ID 生成

```typescript
export function makeClientId() {
  return uuidv4().replace(/-/g, '').slice(-16);
}
```

- 生成 128 位 UUID
- 移除连字符
- 取最后 16 位作为节点 ID
- 确保每个实例具有唯一标识符

---

## 2. 冲突合并顺序算法

### 2.1 时间戳比较规则

时间戳的比较遵循**字典序**，优先级从高到低：

1. **millis (毫秒时间)** → 主要排序依据
2. **counter (计数器)** → 同一毫秒内的排序依据
3. **node (节点ID)** → 保证全局唯一排序

**比较关系定义：**
- 时间戳 T1 < 时间戳 T2，当且仅当 T1.toString() < T2.toString()
- 时间戳 T1 > 时间戳 T2，当且仅当 T1.toString() > T2.toString()
- 时间戳 T1 == 时间戳 T2，当且仅当 T1.toString() == T2.toString()

**比较示例：**

```
时间戳A: 2023-01-01T10:00:00.000Z-0001-AAAAAAAAAAAAAAA1
时间戳B: 2023-01-01T10:00:00.000Z-0000-BBBBBBBBBBBBBBB2
时间戳C: 2023-01-01T10:00:00.001Z-0000-CCCCCCCCCCCCCCC3

字符串比较:
  "2023-01-01T10:00:00.000Z-0000-BBBBBBBBBBBBBBB2"
  "2023-01-01T10:00:00.000Z-0001-AAAAAAAAAAAAAAA1"
  "2023-01-01T10:00:00.001Z-0000-CCCCCCCCCCCCCCC3"

排序结果: B < A < C
解析:
  - B < A: 毫秒相同，计数器 0000 < 0001
  - A < C: 毫秒 10:00:00.000 < 10:00:00.001，计数器无需比较
```

**排序方向说明：**
- 较小的时间戳（`T1 < T2`）表示较早的事件
- 较大的时间戳（`T1 > T2`）表示较晚的事件
- 在 Last Write Wins 策略中，时间戳较大的消息获胜

### 2.2 消息应用顺序

在 `applyMessages` 函数中，消息按时间戳**升序**排序后应用：

```typescript
messages = [...messages].sort((m1, m2) => {
  const t1 = m1.timestamp ? m1.timestamp.toString() : '';
  const t2 = m2.timestamp ? m2.timestamp.toString() : '';
  if (t1 < t2) {
    return -1;
  } else if (t1 > t2) {
    return 1;
  }
  return 0;
});
```

**为什么需要升序排序？**
- 较早的消息先应用，较晚的消息后应用
- 较晚的消息会覆盖较早消息对同一字段的修改
- 保证所有设备按相同顺序应用消息
- 确保最终一致性
- 使冲突解决结果可预测

### 2.3 冲突检测与解决策略

Actual Budget 使用的是 **Last Write Wins (LWW)** 策略，结合时间戳实现。

#### 冲突检测 (compareMessages)

在应用消息前，会检查该消息是否已被"后来的"消息覆盖：

```typescript
async function compareMessages(messages: Message[]): Promise<Message[]> {
  const newMessages = [];

  for (let i = 0; i < messages.length; i++) {
    const message = messages[i];
    const { dataset, row, column, timestamp } = message;
    const timestampStr = timestamp.toString();

    // 查询是否存在相同 (dataset, row, column) 且时间戳 >= 当前消息的记录
    const res = db.runQuery(
      'SELECT timestamp FROM messages_crdt WHERE dataset = ? AND row = ? AND column = ? AND timestamp >= ?',
      [dataset, row, column, timestampStr],
      true,
    );

    if (res.length === 0) {
      // 没有更新的消息，这是新消息
      newMessages.push(message);
    } else if (res[0].timestamp !== timestampStr) {
      // 存在更新的消息，标记为旧消息（需要记录到 merkle 树，但不应用）
      newMessages.push({ ...message, old: true });
    }
    // 如果 res[0].timestamp === timestampStr，则是重复消息，忽略
  }

  return newMessages;
}
```

#### 三种消息状态的处理规则

对于同一字段（`dataset + row + column`）的消息，根据其时间戳与数据库中已有记录的时间戳关系，分为以下三种情况：

| 状态 | 时间戳关系 | SQL 查询条件 | 处理方式 |
|------|-----------|-------------|---------|
| **新消息** | 消息时间戳 < 所有已有记录<br>或数据库中无记录 | `res.length === 0` | 1. 添加到 newMessages<br>2. 应用到数据库（执行 INSERT 或 UPDATE）<br>3. 记录到 messages_crdt 表<br>4. 插入到 Merkle 树 |
| **旧消息** | 存在已有记录的时间戳 > 消息时间戳 | `res.length > 0`<br>`&& res[0].timestamp !== timestampStr` | 1. 标记 `old: true`<br>2. 添加到 newMessages<br>3. **不**应用到数据库<br>4. 仍记录到 messages_crdt 表<br>5. 仍插入到 Merkle 树 |
| **重复消息** | 存在已有记录的时间戳 == 消息时间戳 | `res.length > 0`<br>`&& res[0].timestamp === timestampStr` | 1. 忽略，不添加到 newMessages<br>2. **不**应用到数据库<br>3. **不**重复记录到 messages_crdt 表<br>4. Merkle 树中已存在，无需重复插入 |

**SQL 查询逻辑详解：**
```sql
SELECT timestamp 
FROM messages_crdt 
WHERE dataset = ? AND row = ? AND column = ? 
  AND timestamp >= ?  -- 查找时间戳 >= 当前消息的记录
```

- 如果查询结果为空（`res.length === 0`）：说明没有任何记录的时间戳 >= 当前消息，即当前消息是最新的 → **新消息**
- 如果查询结果非空，但第一条的时间戳 != 当前消息（`res[0].timestamp !== timestampStr`）：说明存在记录的时间戳 > 当前消息 → **旧消息**
- 如果查询结果非空，且第一条的时间戳 == 当前消息（`res[0].timestamp === timestampStr`）：说明已存在完全相同的消息 → **重复消息**

#### 冲突解决总结

| 时间戳关系 | 消息状态 | 是否应用到数据库 | 是否记录到 messages_crdt | 是否插入到 Merkle 树 |
|-----------|---------|-----------------|------------------------|---------------------|
| 消息时间戳 < 所有已有记录 | 新消息 | ✅ 是 | ✅ 是 | ✅ 是 |
| 存在记录时间戳 > 消息时间戳 | 旧消息 | ❌ 否 | ✅ 是 | ✅ 是 |
| 存在记录时间戳 == 消息时间戳 | 重复消息 | ❌ 否 | ❌ 否 | ❌ 否（已存在） |

**这意味着：**
- 时间戳较大的消息"获胜"
- 时间戳由 HULC 保证全局单调递增
- 即使消息乱序到达，最终结果也一致
- 旧消息仍需记录到消息日志和 Merkle 树，以保证同步完整性

### 2.4 Merkle 树同步机制

为了高效检测设备间的差异，使用了 Merkle Trie：

#### 结构
- 三进制基数树 (Trinary Radix Trie)
- 键 = 时间戳的分钟级时间（base-3 编码）
- 值 = 该时间段内所有时间戳的异或哈希

#### 差异检测 (diff)

```typescript
export function diff(trie1: TrieNode, trie2: TrieNode): number | null {
  if (trie1.hash === trie2.hash) {
    return null;  // 完全相同
  }

  // 深度优先遍历，找到第一个不同的时间点
  // 返回需要同步的最早时间戳
}
```

**工作流程：**
1. 客户端和服务器交换 Merkle 树根哈希
2. 如果相同，无需同步
3. 如果不同，通过树遍历来找到差异所在的时间段
4. 从该时间点开始重新同步消息

#### 修剪 (prune)

为了控制树的大小，只保留最近的时间段：

```typescript
export function prune(trie: TrieNode, n = 2): TrieNode {
  // 只保留最后 n 个子节点（最近的时间段）
  // 旧的时间段被丢弃，但消息仍保存在数据库中
}
```

---

## 3. 适用场景与优势

### 3.1 适用场景

**1. 离线优先应用**
- 用户可以在无网络时自由编辑
- 恢复网络后自动同步
- 所有设备最终达成一致状态

**2. 多设备同步**
- 桌面应用 (Electron)
- 网页应用
- 移动应用 (通过浏览器)

**3. 个人财务数据特点**
- 以单元格 (dataset, row, column) 为单位
- 最后修改的值应该保留
- 不需要复杂的合并逻辑

### 3.2 相比其他方案的优势

| 特性 | HULC + LWW CRDT | 传统时间戳 | 向量时钟 |
|------|-----------------|------------|----------|
| 抗时钟漂移 | ✅ 逻辑时钟保护 | ❌ 依赖精确时钟 | ⚠️ 需要同步 |
| 空间复杂度 | O(1) 每个消息 | O(1) | O(N) N=节点数 |
| 实现复杂度 | 中等 | 简单 | 复杂 |
| 冲突解决 | 自动 (LWW) | 可能不一致 | 需自定义 |
| 最终一致性 | ✅ 保证 | ⚠️ 可能无法保证 | ✅ 保证 |

### 3.3 局限性

**1. LWW 语义限制**
- 简单的"最后写入获胜"可能不是所有场景的最佳选择
- 例如：并发修改同一交易的不同字段，可能需要更复杂的合并策略

**2. 计数器溢出**
- 16位计数器在极端高频场景下可能溢出（65536/毫秒）
- 实际应用中不太可能发生

**3. 时钟漂移阈值**
- 最大允许 5 分钟的时钟漂移
- 如果设备时钟偏差超过此值，同步会失败

**4. 无法跟踪因果关系**
- HULC 提供的是偏序关系
- 无法精确追踪"事件 A 导致事件 B"的因果链
- 对于个人财务应用通常足够

---

## 4. 实际应用示例

### 场景：双设备并发修改

**设备 A (节点 ID: AAAAAAAA)**
```
时间: 10:00:00.000
操作: 修改 transactions 表中 id='tx1' 的 amount 字段为 $100
生成时间戳: 2023-01-01T10:00:00.000Z-0000-AAAAAAAA
消息键: (dataset='transactions', row='tx1', column='amount')
```

**设备 B (节点 ID: BBBBBBBB)**
```
时间: 10:00:00.001 (比A晚1毫秒)
操作: 修改 transactions 表中 id='tx1' 的 amount 字段为 $200
生成时间戳: 2023-01-01T10:00:00.001Z-0000-BBBBBBBB
消息键: (dataset='transactions', row='tx1', column='amount')
```

**时间戳比较：**
- `2023-01-01T10:00:00.000Z-0000-AAAAAAAA` < `2023-01-01T10:00:00.001Z-0000-BBBBBBBB`

**同步结果（无论消息到达顺序如何）：**

| 到达顺序 | 设备A消息状态 | 设备B消息状态 | 最终值 |
|---------|--------------|--------------|--------|
| A先到，B后到 | 新消息（应用） | 新消息（应用，覆盖A） | $200 |
| B先到，A后到 | 旧消息（不应用） | 新消息（应用） | $200 |

**结论：** 两台设备最终都显示 $200，因为 B 的时间戳 > A 的时间戳。

### 场景：离线后恢复同步

**设备 A 离线操作：**
```
1. 离线状态修改 transactions.id='tx1'.amount 为 $300
   时间戳: 2023-01-01T11:00:00.000Z-0000-AAAAAAAA
   应用成功（本地数据库无更新记录）

2. 恢复网络，收到设备 B 的旧消息
   B消息时间戳: 2023-01-01T10:30:00.000Z-0000-BBBBBBBB
   相同消息键: (transactions, tx1, amount)

3. compareMessages 检测：
   - 查询 messages_crdt 中 timestamp >= 10:30:00 的记录
   - 找到 A 的记录: 11:00:00.000 > 10:30:00.000
   - 且 11:00:00.000 != 10:30:00.000
   
4. 判定结果：
   - B的消息: 旧消息（old: true）
   - 不应用到数据库
   - 但记录到 messages_crdt 和 Merkle 树

最终值保持 $300
```

---

## 5. 代码位置参考

| 功能 | 文件位置 |
|------|----------|
| 时间戳实现 | `packages/crdt/src/crdt/timestamp.ts` |
| Merkle 树实现 | `packages/crdt/src/crdt/merkle.ts` |
| 同步逻辑入口 | `packages/loot-core/src/server/sync/index.ts` |
| 时间戳测试 | `packages/crdt/src/crdt/timestamp.test.ts` |
| 同步测试 | `packages/loot-core/src/server/sync/sync.test.ts` |

---

## 6. 总结

Actual Budget 的 CRDT 实现采用了 **HULC 混合逻辑时钟** + **Last Write Wins** 策略：

1. **HULC 时间戳**：
   - 结合物理时间、逻辑计数器和节点 ID
   - 保证全局唯一且单调递增
   - 能够容忍时钟漂移和网络延迟
   - 字符串序列化后可直接比较，无需额外比较函数

2. **冲突合并**：
   - 按时间戳字典序**升序**排序消息，较早的先应用
   - 时间戳**较大**的消息覆盖旧消息（Last Write Wins）
   - 三种消息状态：新消息（应用）、旧消息（不应用但记录）、重复消息（忽略）
   - 所有设备按相同规则合并，保证最终一致性

3. **适用场景**：
   - 离线优先的个人财务应用
   - 多设备数据同步
   - 需要简单可靠的冲突解决策略

这种设计在实现复杂度和功能完整性之间取得了良好平衡，非常适合 Actual Budget 这类以"最后修改值为准"的应用场景。
