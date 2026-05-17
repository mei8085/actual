# CLI 缓存与锁协作机制 - 深度分析报告 v2

> **本版聚焦**：缓存与锁的协作链路、只读并发同步场景、写命令双同步澄清、并发竞争的回退与风险边界

---

## 1. 锁的实际保证边界

### 1.1 锁类型与保证矩阵

| 锁类型 | 获取时机 | 保证的互斥性 | 不保证的互斥性 |
|--------|---------|-------------|---------------|
| **共享锁 (Shared)** | 只读命令 (`mutates: false`) | ✅ 与独占锁互斥 | ❌ 与其他共享锁不互斥 |
| **独占锁 (Exclusive)** | 写入命令 (`mutates: true`) | ✅ 与所有锁互斥 | - |

**核心结论**：锁只保证「读写互斥」和「写写互斥」，但**不保证「读读互斥」**。多个只读进程可以同时持有共享锁，并行执行任何操作。

### 1.2 锁的临界区范围

```
withConnection 调用
    ├─ api.init()
    ├─ 🔒 获取锁 (共享/独占)
    │   ├─ 读取缓存状态
    │   ├─ 同步决策
    │   ├─ 下载/加载/同步预算
    │   ├─ 执行业务回调
    │   └─ [写命令] 再次同步
    └─ 🔓 释放锁
    └─ api.shutdown()
```

**关键观察**：锁覆盖了从预算加载到业务执行完毕的全过程，但不包含 `api.init()` 和 `api.shutdown()`。

---

## 2. 只读并发仍会触发同步的场景

只读命令 (`mutates: false`) 获取**共享锁**，但在以下 5 种场景下仍会执行网络同步：

### 场景 1：TTL 过期
```typescript
// cache.ts:103-106
const age = now - state.lastSyncedAt;
if (age < ttlMs) return { action: 'skip', state };  // TTL 内，跳过
return { action: 'sync', state };                     // TTL 过期，同步
```
- 默认 TTL = 60 秒
- 并发影响：N 个只读进程同时发现 TTL 过期 → N 个进程同时执行 `api.sync()`

### 场景 2：强制刷新 (`--refresh` / `--no-cache`)
```typescript
// cache.ts:100
if (mutates || refresh || ttlMs === 0 || encrypted) {
  return { action: 'sync', state };
}
```
- 用户显式要求刷新缓存
- 并发影响：所有带 `--refresh` 的只读进程同时同步

### 场景 3：加密预算
```typescript
// cache.ts:100
if (mutates || refresh || ttlMs === 0 || encrypted) {
  return { action: 'sync', state };
}
```
- 设计原因：加密预算的密钥可能轮换，无法信任本地缓存
- 并发影响：加密预算的所有只读命令**永远**会同步，即使 TTL 内

### 场景 4：TTL 设为 0
```typescript
// cache.ts:100
if (mutates || refresh || ttlMs === 0 || encrypted) {
  return { action: 'sync', state };
}
```
- 用户通过 `--cache-ttl 0` 禁用缓存
- 并发影响：所有只读命令都同步

### 场景 5：时钟回拨
```typescript
// cache.ts:104
if (age < 0) return { action: 'sync', state };
```
- 系统时间回退导致 `now < state.lastSyncedAt`
- 保守策略：视为缓存过期，强制同步

### 只读同步的并发影响

```
时间轴：
T0: 进程 A、B、C 几乎同时启动 (都是 accounts list)
T1: 三个进程都获取共享锁 (因为共享锁允许多持有者)
T2: 三个进程都读取 state.json，发现 TTL 过期
T3: 三个进程同时调用 api.sync()
T4: 三个进程各自执行业务查询
T5: 三个进程释放锁
```

**后果**：
- ✅ **正确性**：CRDT 同步协议保证数据一致，无损坏风险
- ❌ **效率**：重复的网络请求和服务器负载
- ❌ **资源浪费**：N 倍带宽和计算资源消耗

---

## 3. 写命令路径拆解与双同步澄清

### 3.1 写命令执行总览

所有写命令 (`mutates: true`) 都获取**独占锁**，保证串行执行。无论缓存状态如何，写命令路径上**始终存在两次同步**。

### 分支 A：首次下载（无缓存或缓存失效）

**触发条件**：
- `state === null`（首次运行）
- `state.syncId !== config.syncId`（syncId 变更）
- `state.serverUrl !== config.serverUrl`（服务器变更）

**执行路径**：

```
withConnection(mutates: true)
    ├─ 🔒 获取独占锁
    ├─ readCacheState() → null
    ├─ decideSyncAction() → 'download'
    ├─ api.downloadBudget(syncId)
    │   └─ loot-core 内部执行:
    │       ├─ cloudStorage.download()   ← 从服务器拉取完整预算文件
    │       ├─ loadBudget(id)            ← 加载本地数据库
    │       └─ syncBudget()              ← 第 1 次同步 (增量拉取最新变更)
    ├─ writeCacheState()                 ← 写入缓存状态
    ├─ 执行业务回调 (如 createAccount)
    ├─ api.sync()                        ← 第 2 次同步 (推送本地修改)
    ├─ writeCacheState()                 ← 更新 lastSyncedAt
    └─ 🔓 释放锁
```

**同步次数：2 次**
- 第 1 次：`downloadBudget` 内部的 `syncBudget()`
- 第 2 次：写操作后的显式 `api.sync()`

### 分支 B：已有缓存（sync 动作）

**触发条件**：
- 缓存存在且有效，但 `mutates: true` 强制同步

**执行路径**：

```
withConnection(mutates: true)
    ├─ 🔒 获取独占锁
    ├─ readCacheState() → 有效状态
    ├─ decideSyncAction() → 'sync' (因为 mutates: true)
    ├─ api.loadBudget(budgetId)           ← 加载本地数据库
    ├─ api.sync()                         ← 第 1 次同步 (拉取服务器最新变更)
    ├─ writeCacheState()                  ← 更新 lastSyncedAt
    ├─ 执行业务回调 (如 updateTransaction)
    ├─ api.sync()                         ← 第 2 次同步 (推送本地修改)
    ├─ writeCacheState()                  ← 更新 lastSyncedAt
    └─ 🔓 释放锁
```

**同步次数：2 次**
- 第 1 次：写前同步，确保基于最新数据修改
- 第 2 次：写后同步，推送本地修改到服务器

### 3.2 双同步的设计意图

| 同步时机 | 目的 | 必要性 |
|---------|------|--------|
| **写前同步** | 确保本地数据是最新的，避免基于过期状态做出错误修改 | 高（如转账时余额计算必须准确） |
| **写后同步** | 将本地修改立即推送到服务器，对其他客户端可见 | 高（CLI 退出后其他客户端应看到修改） |

### 3.3 双同步的优化空间

**理论上可以优化的场景**：
1. 如果写前同步后没有其他客户端修改，写后同步是冗余的
2. 某些幂等操作（如设置分类名称）即使基于过期数据执行，最终结果也一致

**当前不优化的原因**：
- 简单可靠：不需要追踪"是否真的有修改"
- CRDT 安全：重复同步不会造成数据损坏
- 性能影响可接受：同步是增量的，只有变更数据传输

---

## 4. 并发竞争下的回退与风险边界

### 4.1 风险矩阵

| 风险场景 | 概率 | 影响 | 回退机制 |
|---------|------|------|---------|
| 多只读进程同时同步 | 中 | 资源浪费 | 无自动回退；延长 TTL 可减少 |
| 缓存写入竞争 (last-write-wins) | 高 | 可能丢失较早的 lastSyncedAt | 设计使然，不影响正确性 |
| 崩溃进程锁残留 | 低 | 30 秒内新写进程超时 | stale 机制自动清理；`--no-lock` 绕过 |
| 锁获取超时 | 中 | 命令失败 | 用户重试；或延长 `--lock-timeout` |
| 共享锁下的过期读 | 中 | 读到 TTL 内但已被修改的数据 | 最终一致性权衡；`--refresh` 强制同步 |

### 4.2 各风险详细分析

#### 风险 1：多只读进程同时同步

**场景复现**：
```bash
# 同时执行 3 个只读命令，缓存刚好过期
actual accounts list &
actual categories list &
actual query run --table transactions &
```

**发生条件**：
- 缓存 TTL 刚过期
- 多个只读命令在短时间内（< 网络往返时间）启动

**回退与缓解**：
- ✅ 无正确性问题：CRDT 同步是幂等的
- ⚠️ 无自动去重：3 个进程都会完成完整同步流程
- 🛠️ 缓解手段：
  - 延长 TTL：`--cache-ttl 3600`（1 小时）
  - 主动预热：`actual sync` 然后执行批量操作
  - 脚本串行化：避免在脚本中并行调用 CLI

#### 风险 2：缓存写入竞争 (Last-Write-Wins)

**场景**：
- 进程 A 和 B 同时完成同步
- 都调用 `writeCacheState(meta, state)`

**原子写入保证**：
```typescript
// cache.ts:64-66
const tmp = `${target}.${process.pid}-${randomBytes(4).toString('hex')}.tmp`;
writeFileSync(tmp, JSON.stringify(state));
renameSync(tmp, target);  // 原子操作
```

**竞争结果**：
- ✅ 文件不会损坏（原子重命名保证）
- ❌ 最后完成写入的进程覆盖之前的 `lastSyncedAt`
- ❌ 较早的 `lastSyncedAt` 丢失，可能导致下次命令认为缓存更新

**示例**：
```
T0: 进程 A 同步完成，lastSyncedAt = 1000
T1: 进程 B 同步完成，lastSyncedAt = 1002
T2: 进程 A 写入 state.json → lastSyncedAt = 1000
T3: 进程 B 写入 state.json → lastSyncedAt = 1002  (覆盖 A)
    结果：正常，最新时间戳保留

反向时序：
T0: 进程 A 同步完成，lastSyncedAt = 1002
T1: 进程 B 同步完成，lastSyncedAt = 1000
T2: 进程 B 写入 state.json → lastSyncedAt = 1000
T3: 进程 A 写入 state.json → lastSyncedAt = 1002  (覆盖 B)
    结果：正常

最坏情况：
T0: 进程 A 同步完成，lastSyncedAt = 1002
T1: 进程 B 同步完成，lastSyncedAt = 1000
T2: 进程 A 写入 state.json → lastSyncedAt = 1002
T3: 进程 B 写入 state.json → lastSyncedAt = 1000  (覆盖 A)
    结果：缓存时间戳倒退 2ms，下次命令可能提前同步
```

**回退**：
- 这是设计上的"最后写入者胜"策略
- 影响极小：最多提前几毫秒触发下次同步
- 不影响数据正确性

#### 风险 3：崩溃进程的锁残留

**场景**：持有锁的进程被强制终止（如 `kill -9`、断电）

**自动恢复机制**：

| 锁组件 | 恢复机制 | 恢复时间 |
|--------|---------|---------|
| Gate 锁 (proper-lockfile) | `stale: 30_000` 选项，30 秒后视为失效 | 30 秒 |
| Reader 标记文件 | 下次 `acquireExclusive()` 时扫描清理 | 下一个写命令启动时 |

**回退手段**：
- 等待 30 秒自动恢复
- 使用 `--no-lock` 绕过（单进程环境）
- 手动删除 `{dataDir}/.actual-cli/{syncId}/lock` 和 `readers/` 目录

#### 风险 4：锁获取超时

**错误信息**：
```
Error: Another CLI process is holding the budget (waited 10s).
Retry, or use a different --data-dir.
```

**常见原因**：
- 另一个 CLI 进程正在执行慢操作（如大额数据导入）
- 崩溃进程的锁残留（见风险 3）

**回退手段**：
- 重试命令
- 延长超时：`--lock-timeout 60`
- 检查是否有僵死进程：`ps aux | grep actual`
- 确认不是桌面端在操作（桌面端不参与 CLI 锁，但可能占用服务器资源）

#### 风险 5：共享锁下的过期读

**场景**：
```
T0: 进程 A (accounts list, 只读) 获取共享锁，检查 TTL 有效
T1: 进程 A 加载本地预算 (此时数据是 T-60s 的)
T2: 进程 B (transactions add, 写入) 获取独占锁等待...
T3: 进程 A 执行查询，返回 T-60s 的数据
T4: 进程 A 释放锁
T5: 进程 B 获取独占锁，同步、修改、再次同步
T6: 进程 B 释放锁
```

**结果**：进程 A 读到了过期数据（最多 TTL 秒）

**这是设计使然**：
- 最终一致性系统的正常权衡
- TTL 窗口内的数据不一致是可接受的
- 需要强一致性时使用 `--refresh` 标志

---

## 5. 锁与缓存协作的时序漏洞

### 5.1 检查-使用-时间差 (TOCTOU)

```
T0: 进程 A 获取共享锁
T1: 进程 A readCacheState() → TTL 有效，决定 skip
T2: 进程 A api.loadBudget()  → 加载本地数据
    ┌───────────────────────────────────────┐
    │  此时进程 B (写入) 修改了数据并同步   │
    └───────────────────────────────────────┘
T3: 进程 A 执行业务查询 → 返回过期数据
T4: 进程 A 释放锁
```

**漏洞本质**：锁保护了临界区，但临界区内的决策（TTL 检查）与实际使用（加载数据）之间没有强制一致性保证。

**为什么不修复**：
- 修复需要在锁内完成同步，违背了缓存的设计目的
- TTL 窗口内的不一致是可接受的权衡
- 用户可通过 `--refresh` 显式要求强一致

### 5.2 下载与读取的竞态

**场景**：
- 进程 A（只读，无缓存）获取共享锁，执行 `downloadBudget`
- 进程 B（只读，无缓存）同时启动，也获取共享锁

**可能的时序**：
```
T0: A 获取共享锁，开始 downloadBudget
T1: B 获取共享锁 (因为是共享锁！)，也开始 downloadBudget
T2: A 的 downloadBudget 完成，写入缓存
T3: B 的 downloadBudget 完成，覆盖缓存
T4: A 和 B 各自执行业务查询
```

**问题**：两个只读进程同时下载同一个预算，浪费带宽。

**为什么允许**：
- 共享锁的设计就是允许多个读者
- 下载是幂等的，不会损坏数据
- 这种场景在实际中很少发生（通常只有首次运行会下载）

---

## 6. 关键设计决策的权衡

| 决策 | 收益 | 代价 |
|------|------|------|
| 读者-写者锁 (允许多读者) | 读密集场景下高并发 | 多读者可能同时同步，浪费资源 |
| 写命令双同步 | 确保修改基于最新数据，修改后立即可见 | 额外的网络往返 |
| 原子重命名写缓存 | 避免并发写入文件损坏 | 最后写入者胜，可能丢失时间戳 |
| 最佳努力缓存持久化 | 缓存写入失败不阻塞业务 | 可能重复下载 |
| 30 秒 stale 锁 | 防止崩溃进程永久持有锁 | 崩溃后 30 秒内新写进程等待 |
| 加密预算强制同步 | 应对密钥轮换等安全场景 | 加密预算的读操作永远不缓存 |

---

## 7. 代码位置索引

| 功能 | 文件 | 行号 |
|------|------|------|
| 同步决策逻辑 | `packages/cli/src/cache.ts` | 88-106 |
| 共享锁实现 | `packages/cli/src/lock.ts` | 129-149 |
| 独占锁实现 | `packages/cli/src/lock.ts` | 113-127 |
| 主连接流程 | `packages/cli/src/connection.ts` | 40-154 |
| downloadBudget 内部同步 | `packages/loot-core/src/server/budgetfiles/app.ts` | 182-216 |
| API sync handler | `packages/loot-core/src/server/api.ts` | 253-259 |
| 原子缓存写入 | `packages/cli/src/cache.ts` | 56-71 |
