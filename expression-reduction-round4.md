# 交易筛选器复合表达式归约分析报告（最终定版）

## 修订说明

本报告是对 `expression-reduction-round3.md` 的定点核查，重点解决以下问题：

1. **明确标签匹配大小写行为**：基于真实 REGEXP 实现确认大小写敏感性
2. **统一边界样例**：删除所有"通常"、"取决于实现"等不确定说法
3. **添加自检结论**：逐条确认所有示例与代码一致

---

## 1. hasTags 正则边界匹配行为（最终定版）

### 1.1 真实实现代码

```typescript
// transaction-rules.ts:616-641
case 'hasTags': {
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

### 1.2 REGEXP 执行机制确认

**SQLite REGEXP 函数实现**：

```typescript
// platform/server/sqlite/index.ts:186-188
function regexp(regex: string, text: string) {
  return new RegExp(regex).test(text || '') ? 1 : 0;
}

// platform/server/sqlite/index.electron.ts:99-101
function regexp(regex: string, text: string | null) {
  return new RegExp(regex).test(text || '') ? 1 : 0;
}
```

**关键结论**：
- ✅ 使用 JavaScript 原生 `new RegExp(regex)`，**无 `i` 标志**
- ✅ **严格区分大小写**
- ✅ 行为在所有平台（浏览器 sql.js、Electron better-sqlite3）完全一致
- ✅ 没有"取决于实现"的不确定性

### 1.3 匹配阶段正则精确解析

**正则模板**：`(?<!#)${escapedTag}([\s#]|$)`

| 组成部分 | 精确含义 |
|---------|---------|
| `(?<!#)` | 负向后顾断言，**当前位置前面不能是 `#`** |
| `${escapedTag}` | 转义后的标签字面量（如 `#groceries`） |
| `([\s#]|$)` | **边界只能是以下三者之一**：<br>1. 空白字符 `\s`（空格、制表符、换行、回车、换页、垂直制表符）<br>2. 字符 `#`<br>3. 字符串结尾 `$` |

### 1.4 匹配结论表（零歧义）

以 `notes hasTags '#groceries'` 为例，实际生成正则：`(?<!#)#groceries([\s#]|$)`

✅ **可匹配的情况**：
| 测试字符串 | 匹配原因 |
|-----------|---------|
| `"#groceries"` | 标签后是字符串结尾 `$` |
| `"test #groceries"` | 标签后是字符串结尾 `$` |
| `"#groceries test"` | 标签后是空格（`\s`） |
| `"#groceries#other"` | 标签后是 `#` |
| `"#groceries\nnext"` | 标签后是换行（`\s`） |
| `"#groceries\tmore"` | 标签后是制表符（`\s`） |

❌ **不可匹配的情况**：
| 测试字符串 | 不匹配原因 | 代码依据 |
|-----------|-----------|---------|
| `"##groceries"` | 逐字符分析：<br>位置 0：满足 `(?<!#)`，但后续是 `#groceries`，与标签 `#groceries` 字面量不匹配<br>位置 1：前面是 `#`，不满足 `(?<!#)`，跳过<br>**结论：不匹配** | `(?<!#)` 要求标签前不能是 `#` |
| `"#groceries, #food"` | `#groceries` 后是逗号 `,`，逗号**不属于** `[\s#]`，也不是结尾，不匹配边界 | 边界正则 `([\s#]|$)` 只包含空白、`#`、结尾 |
| `"#groceries-test"` | `#groceries` 后是连字符 `-`，连字符**不属于** `[\s#]`，也不是结尾，不匹配边界 | 边界正则 `([\s#]|$)` 只包含空白、`#`、结尾 |
| `"#groceries2"` | `#groceries` 后是 `2`，数字不属于边界字符 | 边界正则 `([\s#]|$)` |
| `"#Groceries"` | 大写 `G` 与小写 `g` 不匹配，**严格区分大小写** | `new RegExp(regex)` 无 `i` 标志 |
| `"#GROCERIES"` | 全大写，不匹配 | 区分大小写 |

---

## 2. isapprox 日期扩展逻辑（确认无变化）

### 2.1 真实实现代码

```typescript
// transaction-rules.ts:503-556
switch (op) {
  case 'isapprox':
  case 'is':
    if (type === 'date') {
      if (value.type === 'recur') {
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
          const fullDate = parseDate(value.date);
          const high = addDays(fullDate, 2);
          const low = subDays(fullDate, 2);

          return {
            $and: [{ date: { $gte: low } }, { date: { $lte: high } }],
          };
        }
        // ... is 操作符的其他情况
      }
    } else if (type === 'number') {
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
}
```

### 2.2 依赖函数

```typescript
// shared/months.ts:13-84
export function _parse(value: DateLike): Date {
  if (typeof value === 'string') {
    const [year, month, day] = value.split('-');
    if (day != null) {
      return new Date(parseInt(year), parseInt(month) - 1, parseInt(day), 12);
    }
  }
  // ...
}
export const parseDate = _parse;

// shared/months.ts:220-226
export function addDays(day: DateLike, n: number): string {
  return d.format(d.addDays(_parse(day), n), 'yyyy-MM-dd');
}

export function subDays(day: DateLike, n: number): string {
  return d.format(d.subDays(_parse(day), n), 'yyyy-MM-dd');
}

// shared/rules.ts:370-372
export function getApproxNumberThreshold(number) {
  return Math.round(Math.abs(number) * 0.075);
}
```

### 2.3 日期扩展流程

**输入**：`date isapprox '2024-01-15'`

**执行**：
1. `parseDate('2024-01-15')` → `Date` 对象（2024-01-15 中午12点，本地时区）
2. `subDays(fullDate, 2)` → 字符串 `'2024-01-13'`
3. `addDays(fullDate, 2)` → 字符串 `'2024-01-17'`
4. 生成 AQL：
   ```json
   {
     "$and": [
       { "date": { "$gte": "2024-01-13" } },
       { "date": { "$lte": "2024-01-17" } }
     ]
   }
   ```

**匹配范围**：2024-01-13 至 2024-01-17，共 5 天。

---

## 3. 余额计算与筛选、排序关系（确认无变化）

### 3.1 真实实现代码

```typescript
// Account.tsx:648-664
canCalculateBalance = () => {
  const accountId = this.props.accountId;
  const account = this.props.accounts.find(
    account => account.id === accountId,
  );

  if (!account) return false;
  if (this.state.search !== '') return false;
  if (this.state.filterConditions.length > 0) return false;
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

### 3.2 条件判定表

| 条件 | 字段名 | 判定逻辑 |
|------|--------|---------|
| 账户存在 | `this.props.accountId` | `this.props.accounts.find(...)` 必须找到 |
| 搜索框为空 | `this.state.search` | 必须 `=== ''` |
| 无筛选条件 | `this.state.filterConditions` | 必须 `length === 0` |
| 排序条件 | `this.state.sort` | 要么 `=== null`，要么同时满足：<br>- `sort.field === 'date'`<br>- `sort.ascDesc === 'desc'` |

---

## 4. 结论、代码位置、可复核方法对照表（最终版）

| 结论 | 代码文件 | 行号 | 复核方法 |
|-----|---------|------|---------|
| hasTags 边界只包含 `[\s#]` 和结尾 | `transaction-rules.ts` | 637 | 查看正则模板 `(?<!#)${escapedTag}([\\s#]|$)` |
| `"#groceries, #food"` 不匹配 | `transaction-rules.ts` | 637 | 逗号不在 `[\s#]` 中 |
| `"#groceries-test"` 不匹配 | `transaction-rules.ts` | 637 | 连字符不在 `[\s#]` 中 |
| `"##groceries"` 不匹配 | `transaction-rules.ts` | 637 | `(?<!#)` 要求标签前不能是 `#` |
| hasTags 区分大小写 | `platform/server/sqlite/index.ts` | 187 | `new RegExp(regex)` 无 `i` 标志 |
| `"#Groceries"` 不匹配 `"#groceries"` | `platform/server/sqlite/index.ts` | 187 | 区分大小写 |
| REGEXP 行为各平台一致 | `index.ts` + `index.electron.ts` | 186-188, 99-101 | 两处实现完全相同 |
| isapprox 日期前后各2天 | `transaction-rules.ts` | 528-534 | `subDays(fullDate, 2)` + `addDays(fullDate, 2)` |
| isapprox 使用中午12点解析 | `shared/months.ts` | 71 | `new Date(..., 12)` |
| addDays 返回字符串 | `shared/months.ts` | 220-221 | `d.format(..., 'yyyy-MM-dd')` |
| 数字近似误差 7.5% | `shared/rules.ts` | 370-372 | `Math.round(Math.abs(number) * 0.075)` |
| 搜索框有内容禁用余额 | `Account.tsx` | 655 | `this.state.search !== ''` |
| 有筛选条件禁用余额 | `Account.tsx` | 656 | `this.state.filterConditions.length > 0` |
| 非 date desc 排序禁用余额 | `Account.tsx` | 657-663 | `sort.field === 'date' && sort.ascDesc === 'desc'` |
| hasTags 多标签用 $and 连接 | `transaction-rules.ts` | 632-640 | `$and: tagValues.map(...)` |
| 无标签输入匹配不到任何结果 | `transaction-rules.ts` | 626-630 | `return { id: null }` |
| 金额 outflow 隐含金额<0 | `transaction-rules.ts` | 472-486 | `{ amount: { $lt: 0 } }` |
| 空字符串匹配 NULL 和 '' | `transaction-rules.ts` | 571-575 | `$or: [null, '']` |
| category is null 隐含非转账非父 | `transaction-rules.ts` | 385-415 | `conditionSpecialCases` |

---

## 5. 自检结论

### 5.1 边界样例自检

| 样例类型 | 测试字符串 | 结论 | 代码依据 | 一致性确认 |
|---------|-----------|------|---------|-----------|
| 逗号 | `"#groceries, #food"` | ❌ 不匹配 | 边界正则 `([\s#]|$)` 不包含逗号 `,` | ✅ 一致 |
| 连字符 | `"#groceries-test"` | ❌ 不匹配 | 边界正则 `([\s#]|$)` 不包含连字符 `-` | ✅ 一致 |
| 双井号 | `"##groceries"` | ❌ 不匹配 | `(?<!#)` 要求标签前不能是 `#` | ✅ 一致 |

### 5.2 大小写自检

| 测试字符串 | 预期匹配 | 实际匹配 | 原因 | 一致性确认 |
|-----------|---------|---------|------|-----------|
| `"#groceries"` 匹配 `"#groceries"` | ✅ 是 | ✅ 是 | 完全相同 | ✅ 一致 |
| `"#Groceries"` 匹配 `"#groceries"` | ❌ 否 | ❌ 否 | `RegExp` 无 `i` 标志，区分大小写 | ✅ 一致 |
| `"#GROCERIES"` 匹配 `"#groceries"` | ❌ 否 | ❌ 否 | 区分大小写 | ✅ 一致 |

### 5.3 最终确认

所有示例均已与源代码逐条核对：
- ✅ 逗号、连字符、双井号三类边界样例结论与代码一致
- ✅ 大小写匹配结论与 `new RegExp(regex)` 行为一致
- ✅ 所有不确定表述（"通常"、"取决于实现"）已全部删除
- ✅ REGEXP 函数实现已在浏览器和 Electron 两个平台确认，行为完全一致

---

## 6. 关键代码位置索引

| 功能模块 | 文件位置 | 关键函数/类 |
|---------|---------|------------|
| 条件到 AQL 转换 | `packages/loot-core/src/server/transactions/transaction-rules.ts` | `conditionsToAQL`, `mapConditionToActualQL` |
| hasTags 实现 | `packages/loot-core/src/server/transactions/transaction-rules.ts` | 616-641 行 |
| isapprox 日期实现 | `packages/loot-core/src/server/transactions/transaction-rules.ts` | 503-556 行 |
| 特殊条件处理 | `packages/loot-core/src/server/transactions/transaction-rules.ts` | `conditionSpecialCases` |
| SQLite REGEXP 实现（浏览器） | `packages/loot-core/src/platform/server/sqlite/index.ts` | `regexp` 函数 |
| SQLite REGEXP 实现（Electron） | `packages/loot-core/src/platform/server/sqlite/index.electron.ts` | `regexp` 函数 |
| 日期解析 | `packages/loot-core/src/shared/months.ts` | `parseDate` (`_parse`) |
| 日期运算 | `packages/loot-core/src/shared/months.ts` | `addDays`, `subDays` |
| 数字近似阈值 | `packages/loot-core/src/shared/rules.ts` | `getApproxNumberThreshold` |
| 余额计算条件 | `packages/desktop-client/src/components/accounts/Account.tsx` | `canCalculateBalance` |
| LiveQuery 基类 | `packages/desktop-client/src/queries/liveQuery.ts` | `LiveQuery`, `onUpdate` |
| PagedQuery 分页查询 | `packages/desktop-client/src/queries/pagedQuery.ts` | `PagedQuery` |
| useTransactions Hook | `packages/desktop-client/src/hooks/useTransactions.ts` | `useTransactions` |
