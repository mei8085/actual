# CLI 缓存与同步机制分析报告

## 1. 概述

Actual Budget CLI 通过**本地缓存**、**文件锁**和**同步决策引擎**三者协同，实现了高效、安全的命令行预算操作。本报告深入分析这些机制的设计与实现。

核心代码位置：
- `packages/cli/src/cache.ts` - 缓存管理
- `packages/cli/src/lock.ts` - 文件锁机制
- `packages/cli/src/connection.ts` - 连接与同步流程
- `packages/cli/src/commands/sync.ts` - 同步命令

---

## 2. 本地缓存生命周期

### 2.1 缓存目录结构

```
{dataDir}/
└── .actual-cli/              # CLI 专属元数据根目录
    └── {syncId}/             # 按预算 syncId 隔离
        ├── state.json        # 缓存状态文件
        ├── lock              # 锁文件（proper-lockfile 管理）
        └── readers/          # 共享锁持有者标记目录
            ├── {pid}-{rand}  # 每个共享锁持有者的标记文件
            └── ...
```

默认 `dataDir` 为 `~/.actual-cli/data`，可通过 `--data-dir` 或 `ACTUAL_DATA_DIR` 覆盖。

### 2.2 缓存状态 (CacheState)

```typescript
type CacheState = {
  version: 1;                    // 缓存格式版本
  syncId: string;                // 预算同步 ID
  budgetId: string;              // 本地预算文件 ID
  serverUrl: string;             // 服务器 URL
  lastSyncedAt: number;          // 上次同步时间戳 (ms)
  lastDownloadedAt: number;      // 上次完整下载时间戳 (ms)
};
```

### 2.3 缓存生命周期阶段

| 阶段 | 触发条件 | 行为 |
|------|---------|------|
| **创建** | 首次运行命令、syncId 变更、serverUrl 变更 | 调用 `api.downloadBudget()` 从服务器完整下载，写入 `state.json` |
| **命中** | 读命令 + 缓存存在 + TTL 内 + 非加密预算 | 直接加载本地预算 `api.loadBudget(budgetId)`，跳过网络同步 |
| **更新** | 写命令、`--refresh`、TTL 过期、加密预算 | 加载本地预算后调用 `api.sync()` 增量同步，更新 `lastSyncedAt` |
| **失效** | `actual sync --clear`、手动删除 state.json | 下次命令触发重新下载 |

### 2.4 同步决策引擎 (decideSyncAction)

[cache.ts:88-106](file:///d:/fz/0508-2/solo-dogfeeding/code/4-actual/packages/cli/src/cache.ts#L88-L106)

决策优先级（从高到低）：
1. `state === null` → **download**（无缓存）
2. `state.syncId !== config.syncId` → **download**（syncId 不匹配）
3. `state.serverUrl !== config.serverUrl` → **download**（服务器变更）
4. `mutates || refresh || ttlMs === 0 || encrypted` → **sync**（写操作/强制刷新/加密预算）
5. `age < 0` → **sync**（时钟回拨视为过期）
6. `age < ttlMs` → **skip**（TTL 内，使用缓存）
7. 否则 → **sync**（TTL 过期）

### 2.5 原子写入机制

[cache.ts:56-71](file:///d:/fz/0508-2/solo-dogfeeding/code/4-actual/packages/cli/src/cache.ts#L56-L71)

为避免并发写入导致文件损坏，采用**临时文件 + 原子重命名**模式：

```typescript
const tmp = `${target}.${process.pid}-${randomBytes(4).toString('hex')}.tmp`;
writeFileSync(tmp, JSON.stringify(state));
renameSync(tmp, target);  // 原子操作
```

关键点：
- 每个写入进程使用唯一的临时文件名（含 PID 和随机数）
- `renameSync` 在 POSIX 和 Windows 上都是原子操作
- 写入失败静默忽略（best-effort 策略），不阻塞 CLI 命令执行

---

## 3. 文件锁机制：避免重复同步

### 3.1 锁的类型与用途

CLI 实现了**读者-写者锁**（Reader-Writer Lock）模式：

| 锁类型 | 获取函数 | 使用场景 | 并发特性 |
|--------|---------|---------|---------|
| **共享锁 (Shared)** | `acquireShared()` | 只读命令（list、query、balance 等） | 多进程可同时持有 |
| **独占锁 (Exclusive)** | `acquireExclusive()` | 写入命令（add、update、delete 等） | 仅单个进程可持有 |

### 3.2 锁实现原理

基于 `proper-lockfile` 库实现，核心组件：

1. **Gate 锁** (`lock` 文件)
   - 保护锁元数据的互斥锁
   - 任何进程获取共享/独占锁前必须先持有 gate 锁
   - 30 秒 stale 检测（持有进程崩溃后自动释放）

2. **Readers 目录** (`readers/`)
   - 每个共享锁持有者创建一个 `{pid}-{rand}` 标记文件
   - 独占锁获取前需等待该目录为空

### 3.3 共享锁获取流程

[lock.ts:129-149](file:///d:/fz/0508-2/solo-dogfeeding/code/4-actual/packages/cli/src/lock.ts#L129-L149)

```
1. 获取 gate 锁（互斥）
2. 在 readers/ 目录创建自身标记文件
3. 释放 gate 锁
4. 返回释放函数（删除标记文件）
```

### 3.4 独占锁获取流程

[lock.ts:113-127](file:///d:/fz/0508-2/solo-dogfeeding/code/4-actual/packages/cli/src/lock.ts#L113-L127)

```
1. 获取 gate 锁（互斥）
2. 扫描并清理 stale reader 标记（进程已死亡的）
3. 轮询等待 readers/ 目录为空（每 100ms 检查一次）
4. 保持 gate 锁持有
5. 返回释放函数（释放 gate 锁）
```

### 3.5 死进程清理

[lock.ts:75-83](file:///d:/fz/0508-2/solo-dogfeeding/code/4-actual/packages/cli/src/lock.ts#L75-L83)

独占锁获取时自动清理 stale 标记：
- 从标记文件名中提取 PID
- 通过 `process.kill(pid, 0)` 检测进程是否存活
- 进程不存在则删除标记文件
- 处理异常 PID（如负数、非数字）

### 3.6 锁超时与重试

[lock.ts:29-36](file:///d:/fz/0508-2/solo-dogfeeding/code/4-actual/packages/cli/src/lock.ts#L29-L36)

```typescript
retries: Math.max(1, Math.floor(timeoutMs / 200)),
minTimeout: 100,
maxTimeout: 500,
factor: 1.5,
```

- 默认锁超时 10 秒（`--lock-timeout` 或 `ACTUAL_LOCK_TIMEOUT`）
- 指数退避重试，最大间隔 500ms
- 超时后抛出用户友好错误：`Another CLI process is holding the budget...`

---

## 4. 同步指令行为详解

### 4.1 `actual sync` - 主动同步

[sync.ts:98-116](file:///d:/fz/0508-2/solo-dogfeeding/code/4-actual/packages/cli/src/commands/sync.ts#L98-L116)

- 强制 `mutates: true`，获取**独占锁**
- 执行完整同步流程（见 5.2 节）
- 输出同步时间、syncId、budgetId

### 4.2 `actual sync --status` - 缓存状态

[sync.ts:46-78](file:///d:/fz/0508-2/solo-dogfeeding/code/4-actual/packages/cli/src/commands/sync.ts#L46-L78)

- **不获取锁**，不执行同步
- 读取 `state.json` 并输出：
  - `neverSynced` - 是否已同步过
  - `syncedAt` / `lastDownloadedAt` - 时间戳
  - `ageSeconds` - 缓存年龄
  - `stale` - 是否超过 TTL
  - `ttlSeconds` - 配置的 TTL

### 4.3 `actual sync --clear` - 清除缓存

[sync.ts:80-96](file:///d:/fz/0508-2/solo-dogfeeding/code/4-actual/packages/cli/src/commands/sync.ts#L80-L96)

- 获取**独占锁**后删除 `state.json`
- 不删除本地预算数据库文件（由 loot-core 管理）
- 下次命令触发重新下载

---

## 5. 完整同步流程 (withConnection)

[connection.ts:40-154](file:///d:/fz/0508-2/solo-dogfeeding/code/4-actual/packages/cli/src/connection.ts#L40-L154)

### 5.1 执行流程

```
开始
  ↓
解析配置 (resolveConfig)
  ↓
API 初始化 (api.init)
  ↓
根据 mutates 获取锁 (acquireShared / acquireExclusive)
  ↓
读取缓存状态 (readCacheState)
  ↓
同步决策 (decideSyncAction) ──┬── download → 下载预算 → 写缓存
  │                            ├── skip     → 加载本地预算
  │                            └── sync     → 加载本地预算 → api.sync() → 更新缓存
  ↓
执行业务回调 (fn)
  ↓
mutates? ── 是 → api.sync() → 更新缓存
  │        否 → 跳过
  ↓
释放锁
  ↓
API 关闭 (api.shutdown)
  ↓
结束
```

### 5.2 写命令的双同步机制

对于 `mutates: true` 的命令，同步执行**两次**：

1. **写前同步**（第 133 行）：确保本地数据最新，避免基于过期数据修改
2. **写后同步**（第 142 行）：将本地修改推送到服务器

这保证了即使多个 CLI 进程交错执行，最终服务器状态也是一致的（基于 CRDT 的冲突解决）。

---

## 6. CLI 与桌面端共享预算的协作方式

### 6.1 数据隔离模型

CLI 和桌面端**不直接共享**本地预算文件，采用"各自缓存、云端同步"模型：

```
┌─────────────────┐     ┌─────────────────┐
│   Desktop App   │     │   CLI Process   │
│  (Electron)     │     │  (Node.js)      │
└─────────┬───────┘     └───────┬─────────┘
          │                     │
          ▼                     ▼
{userData}/Actual/       ~/.actual-cli/data/
  本地预算数据库           本地预算数据库
          │                     │
          └──────────┬──────────┘
                     ▼
            ┌─────────────────┐
            │  Sync Server    │
            │  (CRDT 同步)    │
            └─────────────────┘
```

### 6.2 协作保证

1. **独立数据目录**：CLI 的 `dataDir` 与桌面端的用户数据目录完全分离
2. **CRDT 同步**：所有修改通过 CRDT 协议合并，无锁冲突
3. **最终一致性**：两端独立与服务器同步，最终收敛到相同状态
4. **无互斥机制**：CLI 的文件锁仅保护 CLI 进程间的竞争，不涉及桌面端

### 6.3 并发修改场景

桌面端和 CLI 同时修改同一预算是安全的：
- 两者都通过同步服务器交换 CRDT 变更
- CRDT 算法自动解决并发冲突
- 最后写入者胜（LWW）策略适用于大多数字段
- 预算余额等通过数学合并而非覆盖

---

## 7. 并发竞争与回退路径

### 7.1 竞争场景分析

| 场景 | 行为 | 回退路径 |
|------|------|---------|
| 多进程同时读 | 全部获取共享锁，并行执行 | 无冲突，正常执行 |
| 多读 + 一写 | 写进程等待所有读进程释放 | 写进程超时后报错，用户重试 |
| 多写并发 | 后续写进程等待第一个释放 | 排队执行，超时报错 |
| 持有锁的进程崩溃 | gate 锁 30 秒后自动释放；reader 标记下次被清理 | 新进程等待超时后正常获取锁 |
| 缓存文件并发写入 | 原子重命名保证最终一致性 | 后写入者覆盖先写入者（last-write-wins） |

### 7.2 错误恢复机制

1. **锁获取失败**
   - 输出：`Another CLI process is holding the budget (waited Xs)`
   - 建议：重试或使用 `--no-lock`（单进程环境）

2. **缓存写入失败**
   - 静默忽略，不抛出异常
   - 影响：下次命令可能重新下载预算

3. **同步失败**
   - 网络异常直接抛出，终止命令执行
   - 锁确保已释放，API 已关闭

4. **命令执行中途崩溃**
   - Node.js 进程退出时，操作系统自动关闭文件描述符
   - `proper-lockfile` 的 stale 机制（30 秒）确保锁最终释放
   - reader 标记可能残留，但下次独占锁获取时会被清理

### 7.3 绕过锁的风险

`--no-lock` 标志禁用文件锁，可能导致：
- 并发写入时缓存状态损坏（但原子重命名减轻此风险）
- 多个写进程同时修改，CRDT 仍能保证服务器端一致性
- 仅建议在可信的单进程脚本环境中使用

---

## 8. 关键设计决策总结

| 决策 | 原因 | 权衡 |
|------|------|------|
| 读者-写者锁 | 读密集场景下允许多进程并行 | 实现复杂度增加 |
| 每 syncId 独立锁 | 多预算场景下互不阻塞 | 管理多个锁目录 |
| 原子重命名写缓存 | 避免并发写入文件损坏 | 极端情况丢失最近一次缓存更新 |
| 最佳努力缓存持久化 | 缓存失败不阻塞业务 | 可能重复下载 |
| 30 秒 stale 锁 | 防止崩溃进程永久持有锁 | 崩溃后 30 秒内新进程等待 |
| 写命令双同步 | 确保修改基于最新数据 | 额外的网络往返 |
| CLI 与桌面端数据隔离 | 避免互相干扰 | 磁盘占用翻倍 |

---

## 9. 代码位置索引

| 功能模块 | 文件 | 关键行 |
|---------|------|--------|
| 缓存状态定义 | `packages/cli/src/cache.ts` | 11-18 |
| 同步决策逻辑 | `packages/cli/src/cache.ts` | 88-106 |
| 原子写入 | `packages/cli/src/cache.ts` | 56-71 |
| 共享锁实现 | `packages/cli/src/lock.ts` | 129-149 |
| 独占锁实现 | `packages/cli/src/lock.ts` | 113-127 |
| 死进程清理 | `packages/cli/src/lock.ts` | 75-83 |
| 主连接流程 | `packages/cli/src/connection.ts` | 40-154 |
| 同步命令 | `packages/cli/src/commands/sync.ts` | 31-117 |
