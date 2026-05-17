# 银行连接渠道在桌面端的整体路径分析

## 概述

Actual Budget 支持多种银行同步渠道（GoCardless、SimpleFIN、Pluggy.ai、Enable Banking），采用统一的架构模式进行管理。本文档详细梳理从用户发起绑定到写入本地预算的完整流程，重点说明多渠道并存时的职责划分。

---

## 一、整体架构

### 1.1 四层架构模型

```
┌─────────────────────────────────────────────────────────────┐
│                  桌面客户端 (desktop-client)                │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  UI 层: Provider 列表、配置模态框、账户选择模态框      │  │
│  ├───────────────────────────────────────────────────────┤  │
│  │  Hook 层: useBuiltInBankSyncProviders、状态管理       │  │
│  ├───────────────────────────────────────────────────────┤  │
│  │  API 层: send() 调用后端方法                          │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│              本地预算核心 (loot-core/server)                │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  账户处理: link.ts、app.ts (linkGoCardlessAccount)    │  │
│  ├───────────────────────────────────────────────────────┤  │
│  │  同步核心: sync.ts (syncAccount、交易对账)            │  │
│  ├───────────────────────────────────────────────────────┤  │
│  │  数据存储: SQLite 数据库 (accounts、banks 表)         │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                  同步服务 (sync-server)                     │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  Provider 子应用: app-gocardless、app-simplefin 等    │  │
│  ├───────────────────────────────────────────────────────┤  │
│  │  凭据服务: secrets-service (存储 API 密钥)            │  │
│  ├───────────────────────────────────────────────────────┤  │
│  │  外部 API: GoCardless Bank Account Data API           │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 核心设计原则

1. **本地优先**：所有账户数据和交易记录存储在本地 SQLite 数据库
2. **Provider 无关性**：统一的账户模型，通过 `account_sync_source` 字段区分来源
3. **职责分离**：UI 交互、本地数据处理、第三方 API 调用严格分层
4. **凭据隔离**：第三方 API 密钥存储在 sync-server，不进入本地预算文件

---

## 二、用户绑定完整流程

### 2.1 流程概览

```
用户点击"添加银行同步"
        │
        ▼
┌─────────────────────────┐
│ Provider 列表加载       │
│ useBuiltInBankSyncProviders │
└─────────────────────────┘
        │
        ▼
用户选择 Provider (如 GoCardless)
        │
        ▼
┌─────────────────────────┐
│ Provider 配置检查       │
│ 检查密钥是否已配置      │
└─────────────────────────┘
        │
        ├─ 未配置 → 弹出配置模态框 (GoCardlessInitialiseModal)
        │        │
        │        ▼
        │  用户输入 Secret ID / Secret Key
        │        │
        │        ▼
        │  调用 secret-set 存储到 sync-server
        │        │
        └────────┘
        │
        ▼
┌─────────────────────────┐
│ 银行列表加载            │
│ gocardless-get-banks    │
└─────────────────────────┘
        │
        ▼
用户选择国家和银行
        │
        ▼
┌─────────────────────────┐
│ 创建授权请求            │
│ gocardless-create-web-token │
└─────────────────────────┘
        │
        ▼
跳转到银行网站进行 OAuth 授权
        │
        ▼
┌─────────────────────────┐
│ 轮询授权结果            │
│ gocardless-poll-web-token │
└─────────────────────────┘
        │
        ▼
┌─────────────────────────┐
│ 账户选择模态框          │
│ SelectLinkedAccountsModal │
└─────────────────────────┘
        │
        ▼
用户选择要链接的本地账户
        │
        ▼
┌─────────────────────────┐
│ 写入本地预算文件        │
│ gocardless-accounts-link│
└─────────────────────────┘
        │
        ▼
┌─────────────────────────┐
│ 首次同步交易            │
│ syncAccount → reconcileTransactions │
└─────────────────────────┘
```

### 2.2 关键环节详解

#### 2.2.1 Provider 列表加载

**文件**：`packages/desktop-client/src/components/banksync/useBuiltInBankSyncProviders.ts`

```typescript
// 内置 Provider 列表 (bankSyncUtils.ts:11-13)
export const BUILT_IN_BANK_SYNC_PROVIDERS = [
  'goCardless',
  'simpleFin',
  'pluggyai',
] as const;

// 每个 Provider 的状态包括：
type BuiltInBankSyncProviderState = {
  id: BankSyncProviders;
  displayName: string;
  description: string;
  isConfigured: boolean;      // 是否已配置密钥
  canConfigure: boolean;       // 当前用户是否有权限配置
  isLoading?: boolean;
  onConfigure: ProviderAction; // 配置操作
  onLink: ProviderAction;      // 链接操作
  onReset: ProviderAction;     // 重置操作
};
```

**职责**：
- 统一管理所有支持的银行同步 Provider
- 检查每个 Provider 的配置状态
- 提供一致的配置、链接、重置操作接口
- 权限控制（多用户模式下只有管理员可配置）

#### 2.2.2 Provider 配置与凭据存储

**配置模态框**：`GoCardlessInitialiseModal.tsx`

用户输入 `Secret ID` 和 `Secret Key` 后，通过 `secret-set` 方法发送到 sync-server：

```typescript
// app.ts:623-658
async function setSecret({ name, value }) {
  // 调用 sync-server 的 /secret 接口存储
  return await post(serverConfig.BASE_SERVER + '/secret', {
    name,
    value,
  }, {
    'X-ACTUAL-TOKEN': userToken,
  });
}
```

**sync-server 端存储**：`secrets-service.js`

```javascript
// 存储在 sync-server 的 SQLite 数据库的 secrets 表
class SecretsDb {
  set(name, value) {
    this.db.mutate(
      `INSERT OR REPLACE INTO secrets (name, value) VALUES (?,?)`,
      [name, value]
    );
  }
}
```

> **关键点**：第三方 API 密钥**不存储**在用户的本地预算文件中，而是存储在 sync-server 的独立数据库中。这意味着：
> - 预算文件本身不包含敏感凭据
> - 切换 sync-server 需要重新配置 Provider
> - 多用户共享同一 sync-server 时，Provider 配置共享

#### 2.2.3 银行列表加载

**GoCardlessExternalMsgModal.tsx:29-66** 中的 `useAvailableBanks` hook：

```typescript
function useAvailableBanks(country: string) {
  const [banks, setBanks] = useState<GoCardlessInstitution[]>([]);
  
  useEffect(() => {
    async function fetch() {
      // 调用后端方法获取银行列表
      const { data } = await sendCatch('gocardless-get-banks', country);
      setBanks(data);
    }
    void fetch();
  }, [country]);
  
  return { data: banks, isLoading, isError };
}
```

**后端处理**：`app.ts:1055-1074`

```typescript
async function getGoCardlessBanks(country: string) {
  // 转发到 sync-server 的 /gocardless/get-banks 接口
  return post(serverConfig.GOCARDLESS_SERVER + '/get-banks', {
    country,
    showDemo: isNonProductionEnvironment(),
  });
}
```

**sync-server 处理**：`app-gocardless.js:118-140`

```javascript
app.post('/get-banks', async (req, res) => {
  const { country, showDemo = false } = req.body;
  await goCardlessService.setToken();  // 确保 access token 有效
  const data = await goCardlessService.getInstitutions(country);
  res.send({ status: 'ok', data });
});
```

#### 2.2.4 OAuth 授权流程

1. **创建授权会话** (`gocardless-create-web-token`)

   ```typescript
   // gocardless.ts:26-29
   const resp = await send('gocardless-create-web-token', {
     institutionId,
     accessValidForDays: 90,
   });
   ```

   sync-server 端创建 requisition（授权请求）：
   ```typescript
   // gocardless-service.ts:276-326
   createRequisition({ institutionId, host }) {
     const body = {
       redirectUrl: host + '/gocardless/link',  // 授权完成回调地址
       institutionId,
       referenceId: uuidv4(),
       accessValidForDays: institution.max_access_valid_for_days,
       // ...
     };
     return client.initSession(body);
   }
   ```

2. **浏览器跳转授权**

   ```typescript
   // gocardless.ts:33
   window.Actual.openURLInBrowser(link);
   ```

3. **轮询授权结果** (`gocardless-poll-web-token`)

   ```typescript
   // app.ts:683-756
   async function pollGoCardlessWebToken({ requisitionId }) {
     // 最长轮询 10 分钟，每 3 秒轮询一次
     const startTime = Date.now();
     stopPolling = false;
     
     async function getData(cb) {
       if (Date.now() - startTime >= 1000 * 60 * 10) {
         cb({ status: 'timeout' });
         return;
       }
       
       const data = await post(
         serverConfig.GOCARDLESS_SERVER + '/get-accounts',
         { requisitionId }
       );
       
       if (data && !data.error_code) {
         cb({ status: 'success', data });
       } else {
         setTimeout(() => getData(cb), 3000);
       }
     }
     
     return new Promise(resolve => {
       void getData(data => resolve(data));
     });
   }
   ```

#### 2.2.5 账户选择与映射

**SelectLinkedAccountsModal.tsx** 提供账户映射界面：

```typescript
// 显示外部银行账户列表，用户选择映射到本地账户
Object.entries(chosenAccounts).forEach(
  ([externalAccountId, localAccountId]) => {
    // 根据 syncSource 调用不同的链接方法
    if (syncSource === 'goCardless') {
      linkAccount.mutate({
        requisitionId,
        account: externalAccount,
        upgradingId: localAccountId,  // 现有本地账户 ID（如适用）
        offBudget,
        startingDate,
        startingBalance,
      });
    }
    // ... 其他 Provider 类似
  }
);
```

#### 2.2.6 写入本地预算文件

**核心方法**：`linkGoCardlessAccount` (app.ts:159-226)

```typescript
async function linkGoCardlessAccount({
  requisitionId,
  account,
  upgradingId,
  offBudget = false,
  startingDate,
  startingBalance,
}) {
  // 1. 查找或创建银行记录
  const bank = await link.findOrCreateBank(account.institution, requisitionId);
  
  let id;
  if (upgradingId) {
    // 2a. 更新现有账户
    id = upgradingId;
    await db.update('accounts', {
      id,
      account_id: account.account_id,        // 外部账户 ID
      bank: bank.id,                         // 关联银行记录
      account_sync_source: 'goCardless',     // 标记同步来源
    });
  } else {
    // 2b. 创建新账户
    id = uuidv4();
    await db.insertWithUUID('accounts', {
      id,
      account_id: account.account_id,
      mask: account.mask,
      name: account.name,
      official_name: account.official_name,
      bank: bank.id,
      offbudget: offBudget ? 1 : 0,
      account_sync_source: 'goCardless',
    });
    // 创建对应的转账收款人
    await db.insertPayee({ name: '', transfer_acct: id });
  }
  
  // 3. 触发首次同步
  const syncRes = await bankSync.syncAccount(
    undefined, undefined, id,
    account.account_id, bank.bank_id,
    startingDate, startingBalance
  );
  
  await handleSyncResponse(syncRes, id);
  
  return 'ok';
}
```

**银行记录管理**：`link.ts:6-24`

```typescript
export async function findOrCreateBank(institution, requisitionId) {
  // 按 bank_id (requisitionId) 查找现有银行记录
  const bank = await db.first(
    'SELECT id, bank_id FROM banks WHERE bank_id = ?',
    [requisitionId]
  );
  
  if (bank) return bank;
  
  // 创建新银行记录
  const bankData = {
    id: uuidv4(),
    bank_id: requisitionId,  // GoCardless: requisitionId
                             // SimpleFIN: orgDomain
                             // Pluggy.ai: orgId
                             // Enable Banking: account_id
    name: institution.name,
  };
  
  await db.insertWithUUID('banks', bankData);
  return bankData;
}
```

> **关键点**：不同 Provider 使用不同的 `bank_id` 策略：
> - **GoCardless**：使用 `requisitionId`（授权会话级，多个账户共享同一 requisition）
> - **SimpleFIN / Pluggy.ai**：使用 `orgDomain` 或 `orgId`（机构级）
> - **Enable Banking**：使用 `account_id`（账户级，每个账户独立会话）

---

## 三、同步流程与交易对账

### 3.1 同步核心逻辑

**sync.ts:1100-1146** 中的 `syncAccount` 方法：

```typescript
export async function syncAccount(
  userId, userKey,
  id, acctId, bankId,
  customStartingDate?, customStartingBalance?
) {
  const acctRow = await db.select('accounts', id);
  
  // 确定同步起始日期（默认最多回溯 90 天）
  const syncStartDate = customStartingDate ?? (await getAccountSyncStartDate(id));
  const oldestTransaction = await getAccountOldestTransaction(id);
  const newAccount = oldestTransaction == null;
  
  // 根据 syncSource 选择不同的下载方法
  let download;
  if (acctRow.account_sync_source === 'simpleFin') {
    download = await downloadSimpleFinTransactions(acctId, syncStartDate);
  } else if (acctRow.account_sync_source === 'pluggyai') {
    download = await downloadPluggyAiTransactions(acctId, syncStartDate);
  } else if (acctRow.account_sync_source === 'goCardless') {
    download = await downloadGoCardlessTransactions(
      userId, userKey, acctId, bankId, syncStartDate, newAccount
    );
  } else if (acctRow.account_sync_source === 'enableBanking') {
    download = await downloadEnableBankingTransactions(acctId, syncStartDate);
  }
  
  return processBankSyncDownload(
    download, id, acctRow, newAccount,
    customStartingBalance, customStartingDate
  );
}
```

### 3.2 交易对账 (Reconciliation)

**reconcileTransactions** (sync.ts:538-696) 是核心算法：

```
下载的交易记录
      │
      ▼
┌─────────────────────────┐
│ 标准化处理              │
│ normalizeBankSyncTransactions │
└─────────────────────────┘
      │
      ▼
┌─────────────────────────┐
│ 精确匹配 (imported_id)  │
│ 按外部交易 ID 匹配      │
└─────────────────────────┘
      │
      ├─ 匹配 → 更新现有交易
      │
      ▼
┌─────────────────────────┐
│ 模糊匹配                │
│ 条件：日期 ±7 天 + 金额相同 │
│   1. 按收款人匹配       │
│   2. 按最早未匹配匹配   │
└─────────────────────────┘
      │
      ├─ 匹配 → 更新现有交易
      │
      ▼
┌─────────────────────────┐
│ 新增交易                │
│ 创建新交易记录          │
└─────────────────────────┘
```

---

## 四、多渠道职责划分

### 4.1 按层次划分

| 层次 | 职责 | 关键文件 |
|------|------|----------|
| **UI 层** | 统一的 Provider 选择界面、配置模态框、账户映射界面 | `useBuiltInBankSyncProviders.ts`、`SelectLinkedAccountsModal.tsx` |
| **前端 API 层** | 封装各 Provider 的调用方法，暴露统一接口 | `gocardless.ts`、`enablebanking.ts`、`mutations.ts` |
| **本地核心层** | 账户管理、交易对账、本地数据持久化 | `app.ts`、`sync.ts`、`link.ts` |
| **Sync-Server 层** | 第三方 API 调用、凭据管理、Provider 特定逻辑 | `app-gocardless/`、`secrets-service.js` |
| **第三方 API** | 银行数据获取、OAuth 授权 | GoCardless API、SimpleFIN Bridge 等 |

### 4.2 各 Provider 职责边界

#### GoCardless (欧洲)
- **范围**：欧洲银行 PSD2 开放银行
- **凭据**：`gocardless_secretId`、`gocardless_secretKey`
- **银行 ID 策略**：`requisitionId`（授权会话级）
- **同步模式**：单账户同步
- **特点**：支持 90 天历史数据，需要定期重新授权

#### SimpleFIN (北美)
- **范围**：北美银行（通过 SimpleFIN Bridge）
- **凭据**：`simplefin_token`、`simplefin_accessKey`
- **银行 ID 策略**：`orgDomain`（机构级）
- **同步模式**：支持批量同步 (`simplefin-batch-sync`)
- **特点**：一次授权可访问所有账户，批量同步性能更好

#### Pluggy.ai (巴西)
- **范围**：巴西银行
- **凭据**：`pluggyai_clientId`、`pluggyai_clientSecret`、`pluggyai_itemIds`
- **银行 ID 策略**：`orgId`（机构级）
- **同步模式**：单账户同步

#### Enable Banking (欧洲 - 替代方案)
- **范围**：欧洲银行（GoCardless 免费替代）
- **凭据**：`enablebanking_applicationId`、`enablebanking_secretKey`
- **银行 ID 策略**：`account_id`（账户级，每个账户独立会话）
- **同步模式**：单账户同步
- **特点**：功能标志控制 (`enableBanking`)，默认不启用

### 4.3 关键边界

#### 4.3.1 本地预算文件 vs Sync-Server

| 本地预算文件 (SQLite) | Sync-Server (SQLite) |
|----------------------|---------------------|
| accounts 表（账户信息） | secrets 表（API 密钥） |
| banks 表（银行记录） | 临时授权 token |
| transactions 表（交易记录） | Provider 会话状态 |
| 永久存储，用户可导出/备份 | 独立于预算文件，可更换 |
| **不包含**第三方 API 密钥 | **不包含**用户预算数据 |

#### 4.3.2 Provider 之间的隔离

1. **账户级隔离**：每个账户通过 `account_sync_source` 字段明确标记来源
2. **配置独立**：各 Provider 的密钥配置独立，互不影响
3. **同步独立**：每个账户独立同步，一个 Provider 失败不影响其他
4. **银行记录独立**：`banks` 表通过 `bank_id` 区分不同 Provider 的银行

#### 4.3.3 同步服务边界

- **sync-server 是无状态的**：不持久化用户预算数据，仅作为网关
- **多用户支持**：同一 sync-server 可服务多个预算文件
- **凭据共享**：同一 sync-server 上的所有预算文件共享 Provider 配置
- **可插拔**：可以更换 sync-server 而不影响本地预算数据（需要重新配置 Provider）

---

## 五、数据模型

### 5.1 账户表 (accounts)

关键字段：

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | TEXT | 本地账户 ID (UUID) |
| `account_id` | TEXT | 外部银行账户 ID |
| `bank` | TEXT | 关联 banks 表的 ID |
| `account_sync_source` | TEXT | 同步来源: `goCardless`/`simpleFin`/`pluggyai`/`enableBanking`/null |
| `bankName` | TEXT | 银行名称 |
| `bankId` | TEXT | 银行 ID |
| `mask` | TEXT | 账号掩码 |
| `last_sync` | TEXT | 上次同步时间戳 |

### 5.2 银行表 (banks)

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | TEXT | 本地银行 ID (UUID) |
| `bank_id` | TEXT | Provider 级别的银行标识（见 2.2.6） |
| `name` | TEXT | 银行名称 |

### 5.3 凭据表 (secrets, sync-server)

| 字段 | 类型 | 说明 |
|------|------|------|
| `name` | TEXT | 凭据名称 (如: `gocardless_secretId`) |
| `value` | TEXT | 凭据值（加密存储） |

---

## 六、关键技术决策

### 6.1 为什么凭据不存储在本地预算文件？

1. **安全**：预算文件可能被共享或备份，不应包含第三方 API 密钥
2. **多用户**：sync-server 模式下，多个用户共享同一 Provider 配置
3. **隔离**：预算文件是纯财务数据，不应包含基础设施配置
4. **可更换**：可以更换 sync-server 而无需修改预算文件

### 6.2 为什么使用 `account_sync_source` 字段？

1. **Provider 路由**：同步时根据来源选择正确的下载方法
2. **UI 分组**：账户列表按 Provider 分组显示
3. **统计分析**：可统计各 Provider 的使用情况
4. **渐进式迁移**：支持从一个 Provider 迁移到另一个

### 6.3 为什么 SimpleFIN 支持批量同步？

SimpleFIN Bridge 的 API 设计支持一次请求获取多个账户的交易记录，因此可以：
- 减少 HTTP 请求次数
- 提高同步速度
- 降低被限流的风险

其他 Provider（GoCardless、Pluggy.ai、Enable Banking）的 API 是单账户设计，因此只能逐个同步。

---

## 七、错误处理与恢复

### 7.1 常见错误场景

| 错误类型 | 处理方式 |
|----------|----------|
| **Provider 未配置** | 弹出配置模态框，引导用户输入密钥 |
| **授权过期** | 标记账户状态，提示用户重新授权 |
| **速率限制** | 显示友好提示，建议稍后重试 |
| **银行 API 错误** | 保留原始错误信息，便于排查 |
| **网络中断** | 不修改本地数据，下次同步重试 |

### 7.2 账户解绑

```typescript
// app.ts:1466-1543
async function unlinkAccount({ id }) {
  // 1. 清除账户的同步关联字段
  await db.updateAccount({
    id,
    account_id: null,
    bank: null,
    account_sync_source: null,
    // ...
  });
  
  // 2. 如果是 GoCardless，且没有其他账户使用同一银行，
  //    调用 GoCardless API 删除 requisition
  if (isGoCardless && noMoreAccountsLinked) {
    await post(serverConfig.GOCARDLESS_SERVER + '/remove-account', {
      requisitionId: bank.bank_id,
    });
  }
}
```

---

## 八、总结

Actual Budget 的银行连接架构采用了清晰的分层设计：

1. **统一的用户体验**：所有 Provider 共享相同的 UI 模式和交互流程
2. **清晰的职责边界**：UI、本地核心、sync-server、第三方 API 四层分离
3. **安全的凭据管理**：API 密钥与预算数据物理隔离
4. **灵活的扩展能力**：新 Provider 只需实现对应接口即可接入
5. **本地优先的数据模型**：用户始终掌握自己的财务数据

多渠道并存时，通过 `account_sync_source` 字段实现逻辑隔离，通过 sync-server 实现 Provider 特定逻辑的封装，确保架构清晰且易于维护。
