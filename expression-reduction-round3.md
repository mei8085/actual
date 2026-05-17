# 交易筛选器复合表达式归约分析报告（最终勘误版）

## 勘误说明

本报告是对 `expression-reduction-round2.md` 的逐条勘误，重点修正以下三处证据冲突：

1. **hasTags 正则边界示例**：明确逗号、连字符、双井号三类字符串的匹配结论并给出代码依据
2. **isapprox 日期扩展逻辑**：按真实实现重写，不使用与源码不一致的伪代码
3. **余额计算与筛选、排序关系**：确保字段名与判定条件完全对应源码

文末附结论、代码位置、可复核方法对照表。

---

## 1. hasTags 正则边界匹配行为勘误

### 1.1 真实实现代码

```typescript
// transaction-rules.ts:616-641
case 'hasTags': {
  // ========== 阶段一：从用户输入中提取标签 ==========
  const tagValues = [];
  const seenTags = new Set();
  for (const [_, tag] of value.matchAll(/(?<!#)(#[^#\s]+)/g)) {
    if (!seenTags.has(tag)) {
      seenTags.add(tag);
      tagValues.push(tag);
    }
  }

  if (tagValues.length === 0) {
    return { id: null };
  }

  // ========== 阶段二：为每个标签构建匹配正则 ==========
  return {
    $and: tagValues.map(v => {
      const escapedTag = v
        .replace(/[.*+?^${}()|[\]\\]/g, '\\$&')
        .replace(/\\\$/g, '[$]');
      const pattern = `(?<!#)${escapedTag}([\\s#]|$)`;
      return apply(field, '$regexp', pattern);
    }),
  };
}
```

### 1.2 匹配阶段正则精确解析

**正则模板**：`(?<!#)${escapedTag}([\s#]|$)`

| 组成部分 | 精确含义 |
|---------|---------|
| `(?<!#)` | 负向后顾断言，**当前位置前面不能是 `#`** |
| `${escapedTag}` | 转义后的标签字面量（如 `#groceries`） |
| `([\s#]|$)` | **边界只能是以下三者之一**：<br>1. 空白字符 `\s`（空格、制表符、换行等）<br>2. 字符 `#`<br>3. 字符串结尾 `$` |

### 1.3 修正后的匹配结论（重点勘误）

以 `notes hasTags '#groceries'` 为例，实际生成正则：`(?<!#)#groceries([\s#]|$)`

✅ **可匹配的情况**：
| 测试字符串 | 匹配原因 |
|-----------|---------|
| `"#groceries"` | 标签后是字符串结尾 `$` |
| `"test #groceries"` | 标签后是字符串结尾 `$` |
| `"#groceries test"` | 标签后是空格（`\s`） |
| `"#groceries#other"` | 标签后是 `#` |
| `"#groceries\nnext"` | 标签后是换行（`\s`） |

❌ **不可匹配的情况**（**重点修正**）：
| 测试字符串 | 不匹配原因 | 代码依据 |
|-----------|-----------|---------|
| `"##groceries"` | 逐字符分析：<br>位置 0：满足 `(?<!#)`，但后续是 `#groceries`，与标签 `#groceries` 字面量不匹配<br>位置 1：前面是 `#`，不满足 `(?<!#)`，跳过<br>**结论：不匹配** | `(?<!#)` 要求标签前不能是 `#` |
| `"#groceries, #food"` | `#groceries` 后是逗号 `,`，逗号**不属于** `[\s#]`，也不是结尾，不匹配边界 | 边界正则 `([\s#]|$)` 只包含空白、`#`、结尾 |
| `"#groceries-test"` | `#groceries` 后是连字符 `-`，连字符**不属于** `[\s#]`，也不是结尾，不匹配边界 | 边界正则 `([\s#]|$)` 只包含空白、`#`、结尾 |
| `"#groceries2"` | `#groceries` 后是 `2`，数字不属于边界字符 | 边界正则 `([\s#]|$)` |
| `"#Groceries"` | 是否匹配取决于 SQLite REGEXP 实现，通常不区分大小写 | SQLite 编译选项决定 |

### 1.4 关键边界字符的代码证明

边界字符集合 `[\s#]` 的精确定义：
- `\s` 在 JavaScript 正则中匹配：`[ \t\n\r\f\v]`（空格、制表符、换行、回车、换页、垂直制表符）
- `#` 只匹配字面量 `#` 字符
- **不包含**：逗号 `,`、句号 `.`、连字符 `-`、下划线 `_` 等任何其他标点符号

---

## 2. isapprox 日期扩展逻辑勘误

### 2.1 真实实现代码

```typescript
// transaction-rules.ts:503-556
switch (op) {
  case 'isapprox':
  case 'is':
    if (type === 'date') {
      if (value.type === 'recur') {
        // 递归日期（周期性）
        const dates = value.schedule
          .occurrences({ take: recurDateBounds })
          .toArray()
          .map(d => dayFromDate(d.date));

        return {
          $or: dates.map(d => {
            if (op === 'isapprox') {
              return {
                $and: [
                  { date: { $gte: subDays(d, 2) } },
                  { date: { $lte: addDays(d, 2) } },
                ],
              };
            }
            return { date: d };
          }),
        };
      } else {
        if (op === 'isapprox') {
          // ========== 单个日期的 isapprox 处理 ==========
          const fullDate = parseDate(value.date);
          const high = addDays(fullDate, 2);
          const low = subDays(fullDate, 2);

          return {
            $and: [{ date: { $gte: low } }, { date: { $lte: high } }],
          };
        } else {
          // is 操作符的其他情况（精确匹配日期、月份、年份）
          switch (value.type) {
            case 'date':
              return { date: value.date };
            case 'month': {
              const low = value.date + '-00';
              const high = value.date + '-99';
              return {
                $and: [{ date: { $gte: low } }, { date: { $lte: high } }],
              };
            }
            case 'year': {
              const low = value.date + '-00-00';
              const high = value.date + '-99-99';
              return {
                $and: [{ date: { $gte: low } }, { date: { $lte: high } }],
              };
            }
            default:
          }
        }
      }
    } else if (type === 'number') {
      // 数字的 isapprox 处理（7.5% 误差范围）
      const number = value.value;
      if (op === 'isapprox') {
        const threshold = getApproxNumberThreshold(number);
        return {
          $and: [
            apply(field, '$gte', number - threshold),
            apply(field, '$lte', number + threshold),
          ],
        };
      }
      return apply(field, '$eq', number);
    }
    // ...
}
```

### 2.2 依赖函数真实实现

```typescript
// shared/months.ts:13-84
export function _parse(value: DateLike): Date {
  if (typeof value === 'string') {
    const [year, month, day] = value.split('-');
    if (day != null) {
      // 关键：使用中午12点避免时区问题
      return new Date(parseInt(year), parseInt(month) - 1, parseInt(day), 12);
    }
    // ...
  }
  // ...
}
export const parseDate = _parse;

// shared/months.ts:220-226
export function addDays(day: DateLike, n: number): string {
  return d.format(d.addDays(_parse(day), n), 'yyyy-MM-dd');  // 返回字符串
}

export function subDays(day: DateLike, n: number): string {
  return d.format(d.subDays(_parse(day), n), 'yyyy-MM-dd');  // 返回字符串
}

// shared/rules.ts:370-372
export function getApproxNumberThreshold(number) {
  return Math.round(Math.abs(number) * 0.075);  // 7.5% 误差
}
```

### 2.3 修正后的日期扩展逻辑

**输入示例**：`date isapprox '2024-01-15'`

**执行流程**：
1. `parseDate('2024-01-15')` → 返回 `Date` 对象（2024-01-15 中午12点，本地时区）
2. `subDays(fullDate, 2)` → 返回字符串 `'2024-01-13'`
3. `addDays(fullDate, 2)` → 返回字符串 `'2024-01-17'`
4. 生成 AQL 表达式：
   ```json
   {
     "$and": [
       { "date": { "$gte": "2024-01-13" } },
       { "date": { "$lte": "2024-01-17" } }
     ]
   }
   ```

**匹配范围**：包含 2024-01-13、2024-01-14、2024-01-15、2024-01-16、2024-01-17，共 5 天。

### 2.4 数字近似匹配逻辑

**输入示例**：`amount isapprox 100`

**执行流程**：
1. `getApproxNumberThreshold(100)` → `Math.round(100 * 0.075)` = `8`
2. 生成 AQL 表达式：
   ```json
   {
     "$and": [
       { "amount": { "$gte": 92 } },
       { "amount": { "$lte": 108 } }
     ]
   }
   ```

---

## 3. 余额计算与筛选、排序关系勘误

### 3.1 真实实现代码

```typescript
// Account.tsx:648-664
canCalculateBalance = () => {
  const accountId = this.props.accountId;
  const account = this.props.accounts.find(
    account => account.id === accountId,
  );

  if (!account) return false;
  if (this.state.search !== '') return false;           // 搜索框有内容
  if (this.state.filterConditions.length > 0) return false;  // 有筛选条件
  if (this.state.sort === null) {
    return true;
  } else {
    return (
      this.state.sort.field === 'date' && 
      this.state.sort.ascDesc === 'desc'
    );
  }
};
```

### 3.2 修正后的条件判定

| 条件 | 字段名 | 判定逻辑 | 代码位置 |
|------|--------|---------|---------|
| 账户存在 | `this.props.accountId` | `this.props.accounts.find(...)` 必须找到匹配账户 | Account.tsx:649-654 |
| 搜索框为空 | `this.state.search` | **必须** `=== ''` | Account.tsx:655 |
| 无筛选条件 | `this.state.filterConditions` | **必须** `length === 0` | Account.tsx:656 |
| 排序条件 | `this.state.sort` | 要么 `=== null`，<br>要么同时满足：<br>- `sort.field === 'date'`<br>- `sort.ascDesc === 'desc'` | Account.tsx:657-663 |

### 3.3 常见场景判定结果

| 场景 | 搜索 | 筛选条件 | 排序 | 能否计算余额 |
|-----|------|---------|------|-------------|
| 默认状态 | 空 | 空 | null | ✅ 可以 |
| 默认状态 | 空 | 空 | date desc | ✅ 可以 |
| 按金额排序 | 空 | 空 | amount desc | ❌ 不可以 |
| 按日期升序 | 空 | 空 | date asc | ❌ 不可以 |
| 正在搜索 | "groceries" | 空 | date desc | ❌ 不可以 |
| 有筛选条件 | 空 | [category is '食物'] | date desc | ❌ 不可以 |
| 有筛选条件 + 搜索 | "test" | [amount > 100] | date desc | ❌ 不可以 |

### 3.4 字段名勘误对照表

| 错误字段名（前版） | 正确字段名（源码） | 说明 |
|------------------|-------------------|------|
| `searchConditions` | `search` | 搜索框输入内容 |
| `filterConditions` | `filterConditions` | 筛选条件数组（此字段名正确） |
| `sortField` | `sort.field` | 排序字段嵌套在 sort 对象中 |
| 无 | `sort.ascDesc` | 排序方向字段 |

---

## 4. 结论、代码位置、可复核方法对照表

| 结论 | 代码文件 | 行号范围 | 复核方法 |
|-----|---------|---------|---------|
| hasTags 边界只包含 `[\s#]` 和结尾 | `transaction-rules.ts` | 637 | 查看正则模板 `(?<!#)${escapedTag}([\\s#]|$)` |
| `"#groceries, #food"` 不匹配 | `transaction-rules.ts` | 637 | 逗号不在 `[\s#]` 中，用测试用例验证 |
| `"##groceries"` 不匹配 | `transaction-rules.ts` | 637 | 负向后顾 `(?<!#)` 要求标签前不能是 `#` |
| isapprox 日期前后各2天 | `transaction-rules.ts` | 528-534 | 查看 `subDays(fullDate, 2)` 和 `addDays(fullDate, 2)` |
| isapprox 使用中午12点解析 | `shared/months.ts` | 71 | 查看 `new Date(..., 12)` |
| addDays 返回字符串 | `shared/months.ts` | 220-221 | 查看 `d.format(..., 'yyyy-MM-dd')` |
| 数字近似误差 7.5% | `shared/rules.ts` | 370-372 | 查看 `Math.round(Math.abs(number) * 0.075)` |
| 搜索框有内容禁用余额 | `Account.tsx` | 655 | 查看 `this.state.search !== ''` |
| 有筛选条件禁用余额 | `Account.tsx` | 656 | 查看 `this.state.filterConditions.length > 0` |
| 非 date desc 排序禁用余额 | `Account.tsx` | 657-663 | 查看 `sort.field === 'date' && sort.ascDesc === 'desc'` |
| parseDate 从 shared/months 导入 | `transaction-rules.ts` | 33-38 | 查看 import 语句 |
| hasTags 多标签用 $and 连接 | `transaction-rules.ts` | 632-640 | 查看 `$and: tagValues.map(...)` |
| 无标签输入匹配不到任何结果 | `transaction-rules.ts` | 626-630 | 查看 `return { id: null }` |
| 金额 outflow 隐含金额<0 | `transaction-rules.ts` | 472-486 | 查看 `{ amount: { $lt: 0 } }` |
| 空字符串匹配 NULL 和 '' | `transaction-rules.ts` | 571-575 | 查看 `$or: [null, '']` |
| category is null 隐含非转账非父 | `transaction-rules.ts` | 385-415 | 查看 `conditionSpecialCases` |

---

## 5. 关键代码位置索引（最终版）

| 功能模块 | 文件位置 | 关键函数/类 |
|---------|---------|------------|
| 条件到 AQL 转换 | `packages/loot-core/src/server/transactions/transaction-rules.ts` | `conditionsToAQL`, `mapConditionToActualQL` |
| hasTags 实现 | `packages/loot-core/src/server/transactions/transaction-rules.ts` | 616-641 行 |
| isapprox 日期实现 | `packages/loot-core/src/server/transactions/transaction-rules.ts` | 503-556 行 |
| 特殊条件处理 | `packages/loot-core/src/server/transactions/transaction-rules.ts` | `conditionSpecialCases` |
| 日期解析 | `packages/loot-core/src/shared/months.ts` | `parseDate` (`_parse`) |
| 日期运算 | `packages/loot-core/src/shared/months.ts` | `addDays`, `subDays` |
| 数字近似阈值 | `packages/loot-core/src/shared/rules.ts` | `getApproxNumberThreshold` |
| 余额计算条件 | `packages/desktop-client/src/components/accounts/Account.tsx` | `canCalculateBalance` |
| LiveQuery 基类 | `packages/desktop-client/src/queries/liveQuery.ts` | `LiveQuery`, `onUpdate` |
| PagedQuery 分页查询 | `packages/desktop-client/src/queries/pagedQuery.ts` | `PagedQuery` |
| useTransactions Hook | `packages/desktop-client/src/hooks/useTransactions.ts` | `useTransactions` |
| AQL RPC 调用 | `packages/desktop-client/src/queries/aqlQuery.ts` | `aqlQuery` |
