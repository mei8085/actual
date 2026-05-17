# CLI 锁机制对账版 - gate 与 readers 先后关系说明 v4

> **本版唯一聚焦**：gate 锁与 readers 目录的先后协作关系、四阶段时序对齐、等待原因拆解、最小化竞态时序图

---

## 1. 核心抽象

```
┌─────────────────────────────────────────────────────────────────┐
│                  锁元数据目录：{dataDir}/.actual-cli/{syncId}/          │
├─────────────────────────────────────────────────────────────────┤
│                                                           │
│  ┌─────────────┐          ┌──────────────────────────┐   │
│  │  lock 文件   │          │  readers/ 目录          │   │
│  │  (gate 锁)  │          │  ├─ {pid}-{rand}    │   │
│  │             │          │  ├─ {pid}-{rand}    │   │
│  │  proper-     │          │  └─ ...                 │   │
│  │  lockfile  │          │                         │   │
│  └─────────────┘          └──────────────────────────┘   │
│                                                           │
│  ✅ 互斥锁，保护       ✅ 每个共享锁持有者的标记        │
│     锁元数据操作           目录                                 │
│                                                           │
│  ✅ 任何锁获取前         ✅ 独占锁获取前必须               │
│     必须先持有            等待此目录为空                       │
│                                                           │
└─────────────────────────────────────────────────────────────────┘
```

**核心原则**：**先抢 gate，再操作 readers。gate 是保护 readers 目录操作的互斥锁。

---

## 2. 共享锁 (Shared) 四阶段时序对账

### 2.1 四阶段定义

| 阶段 | 行为 | gate 锁状态 | readers 目录状态 |
|------|------|------------|------------------|
| **1. 请求** | 调用 `acquireShared()` | 未持有 | 未知 |
| **2. 等待** | 等待获取 gate 锁 | 等待中 | - |
| **3. 进入临界区** | 1. 获取 gate 锁<br>2. 创建 reader 标记<br>3. **释放 gate 锁** | 先持有后释放 | 增加一个标记 |
| **4. 释放** | 执行业务逻辑后删除 reader 标记 | 未持有 | 减少一个标记 |

### 2.2 逐行代码时序

```
行号  代码                                          gate 状态    readers 操作
───  ───────────────────────────────────────────  ─────────  ──────────────
129  acquireShared(dir, { timeoutMs })            未持有
133  ├─ gate = await acquireGate(...)        ── 等待 → 持有
136  ├─ readers = readersDir(dir)            持有
137  ├─ ensureDir(readers)                    持有
138  ├─ markerName = `${pid}-${rand}`         持有
139  ├─ markerPath = join(readers, markerName)  持有
140  ├─ writeFileSync(markerPath, '')        ── 持有      创建标记 ✓
145  └─ await gate()                          ── 持有 → 释放
146  return async () => {                        释放
147    rmSync(markerPath, { force: true })    ─ 释放      删除标记 ✓
148  }
```

**关键洞察**：共享锁持有者在**业务逻辑执行期间不持有 gate 锁**，仅通过 reader 标记文件表示存在。

---

## 3. 独占锁 (Exclusive) 四阶段时序对账

### 3.1 四阶段定义

| 阶段 | 行为 | gate 锁状态 | readers 目录状态 |
|------|------|------------|------------------|
| **1. 请求** | 调用 `acquireExclusive()` | 未持有 | 未知 |
| **2. 等待** | 1. 等待获取 gate 锁<br>2. 等待 readers 清空 | 先等待 gate<br>后持有 gate | 扫描并等待清空 |
| **3. 进入临界区** | 持 gate 锁执行业务逻辑 | 持续持有 | 为空 |
| **4. 释放** | 释放 gate 锁 | 持有 → 释放 | 保持为空 |

### 3.2 逐行代码时序

```
行号  代码                                          gate 状态    readers 操作
───  ───────────────────────────────────────────  ─────────  ──────────────
113  acquireExclusive(dir, { timeoutMs })        未持有
118  ├─ release = await acquireGate(...)      ── 等待 → 持有
121  └─ await waitForReadersEmpty(dir, ...)    持有
        89    ├─ sweepStaleReaders(dir)          持有      清理死亡进程标记
        90    ├─ if (readers.length === 0)  持有
        91    └─ sleep(100ms) ... 循环       持有      等待...
126  return () => release()                        持有
      执行业务逻辑...                             持有
      调用 release()                              ── 持有 → 释放
```

**关键洞察**：独占锁持有者在**整个业务逻辑执行期间持续持有 gate 锁**。

---

## 4. 等待原因拆解

所有等待分为**两类本质不同的等待**：

### 等待类型 1：等 gate 锁

**触发前提**：
- gate 锁当前被其他进程持有
- 可能的持有者：
  - 另一个独占锁（在临界区中）
  - 另一个共享锁（在创建 reader 标记的短暂窗口中）

**发生位置**：
- 共享锁：第 133 行 `acquireGate()`
- 独占锁：第 118 行 `acquireGate()`

**超时行为**：
- proper-lockfile 内部指数退避重试
- 最大重试次数 = `max(1, floor(timeoutMs / 200))`
- 重试间隔：100ms ~ 500ms

**错误信息**：
```
Error: Another CLI process is holding the budget (waited Xs).
```

---

### 等待类型 2：等 readers 清空

**触发前提**：
- 已持有 gate 锁
- 但 readers 目录中存在其他共享锁持有者的标记文件
- 且这些标记对应的进程仍存活

**发生位置**：
- 仅独占锁：第 121 行 `waitForReadersEmpty()`
- 共享锁**永远不会**等 readers 清空

**超时行为**：
- 每 100ms 扫描一次 readers 目录
- 每次扫描前清理 stale 标记（进程已死亡的）
- 超过剩余超时时间后抛出错误

**错误信息**：
```
Error: Another CLI process is holding the budget (waited Xs).
```

---

## 5. 等待原因快速判断表

| 锁类型 | 等待阶段 | 可能的等待类型 | 判断方法 |
|---------|---------|-------------|---------|
| 共享锁 | acquireGate | 等 gate | 第 133 行前阻塞 |
| 共享锁 | 业务逻辑中 | 无等待 | 第 145 行后不持有任何锁 |
| 独占锁 | acquireGate | 等 gate | 第 118 行前阻塞 |
| 独占锁 | waitForReadersEmpty | 等 readers 清空 | 第 121 行阻塞（已持有 gate） |

---

## 6. 最小化竞态时序图

### 场景 1：共享锁并行（无等待）

```
进程 A (只读)                进程 B (只读)
    │                            │
    ├─ acquireGate()           │
    │  获取 gate ✓                │
    │  创建 reader-A             │
    │  释放 gate                 │
    │  进入业务逻辑              │
    │                            ├─ acquireGate()
    │                            │  获取 gate ✓
    │                            │  创建 reader-B
    │                            │  释放 gate
    │                            │  进入业务逻辑
    │  执行业务                  │  执行业务
    │  删除 reader-A             │  删除 reader-B
    ▼                            ▼
```

**解读**：两个共享锁持有者并行执行，无等待。

---

### 场景 2：共享锁持有期间，独占锁请求（等 readers 清空）

```
进程 A (只读)                    进程 B (写入)
    │                                │
    ├─ acquireShared()             │
    │  获取 gate ✓                  │
    │  创建 reader-A               │
    │  释放 gate                   │
    │  进入业务逻辑                │
    │                                ├─ acquireExclusive()
    │                                │  获取 gate ✓  (A 已释放 gate
    │                                │  扫描 readers → 发现 reader-A
    │                                │  sleep(100ms)
    │  执行业务                    │  扫描 readers → 发现 reader-A
    │                                │  sleep(100ms)
    │  删除 reader-A                 │  ...
    │                                │  扫描 readers → 为空！
    │                                │  进入业务逻辑
    ▼                                │  执行业务
                                     ▼
```

**解读**：B 在第 2 阶段等待，属于**等 readers 清空**。

---

### 场景 3：独占锁持有期间，共享锁请求（等 gate）

```
进程 A (写入)                    进程 B (只读)
    │                                │
    ├─ acquireExclusive()             │
    │  获取 gate ✓                  │
    │  等待 readers 清空             │
    │  进入业务逻辑                │
    │  持 gate 执行业务            │
    │                                ├─ acquireShared()
    │                                │  尝试获取 gate → 被持有
    │                                │  等待...
    │  释放 gate                     │  获取 gate ✓
    │                                │  创建 reader-B
    │                                │  释放 gate
    │                                │  进入业务逻辑
    ▼                                │  执行业务
                                     ▼
```

**解读**：B 在第 2 阶段等待，属于**等 gate**。

---

### 场景 4：独占锁持有期间，另一个独占锁请求（等 gate）

```
进程 A (写入)                    进程 B (写入)
    │                                │
    ├─ acquireExclusive()             │
    │  获取 gate ✓                  │
    │  进入业务逻辑                │
    │                                ├─ acquireExclusive()
    │                                │  尝试获取 gate → 被持有
    │  执行业务                      │  等待...
    │  释放 gate                     │  获取 gate ✓
    │                                │  等待 readers 清空
    │                                │  进入业务逻辑
    ▼                                ▼
```

**解读**：B 先等 gate，再等 readers（通常为空）。

---

### 场景 5：多个共享锁同时请求（串行获取 gate）

```
进程 A (只读)  进程 B (只读)  进程 C (只读)
    │              │              │
    ├─ acquireGate │              │
    │  获取 gate ✓│              │
    │  创建标记   │              │
    │  释放 gate │              │
    │              ├─ acquireGate │
    │              │  获取 gate ✓│
    │              │  创建标记   │
    │              │  释放 gate │
    │              │              ├─ acquireGate
    │              │              │  获取 gate ✓
    │              │              │  创建标记
    │              │              │  释放 gate
    ▼              ▼              ▼
    并行执行业务逻辑
```

**解读**：gate 获取是串行的，但创建标记后立即释放，所以三个进程几乎同时进入业务逻辑。

---

## 7. 阻塞类型速查表

看到阻塞时，用下表快速判断：

| 现象 | 等待类型 | 解锁条件 |
|------|---------|---------|
| 共享锁在 acquireGate 卡住 | 等 gate | 其他进程释放 gate |
| 独占锁在 acquireGate 卡住 | 等 gate | 其他进程释放 gate |
| 独占锁在 waitForReadersEmpty 卡住 | 等 readers 清空 | 所有共享锁持有者释放 |
| 共享锁在业务逻辑中不卡住 | 无等待 | - |
| 独占锁在业务逻辑中不卡住 | 无等待 | - |

---

## 8. 代码位置索引

| 功能 | 文件 | 行号 |
|------|------|------|
| acquireShared 完整实现 | `packages/cli/src/lock.ts` | 129-149 |
| acquireExclusive 完整实现 | `packages/cli/src/lock.ts` | 113-127 |
| acquireGate (gate 锁获取) | `packages/cli/src/lock.ts` | 96-111 |
| waitForReadersEmpty (等待 readers) | `packages/cli/src/lock.ts` | 85-94 |
| sweepStaleReaders (清理 stale 标记) | `packages/cli/src/lock.ts` | 75-83 |
| 锁测试用例 | `packages/cli/src/lock.test.ts` | 1-159 |
