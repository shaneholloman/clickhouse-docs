---
'description': 'GROUP BY 子句的文档'
'sidebar_label': 'GROUP BY'
'slug': '/sql-reference/statements/select/group-by'
'title': 'GROUP BY 子句'
---


# GROUP BY 子句

`GROUP BY` 子句将 `SELECT` 查询切换到聚合模式，工作原理如下：

- `GROUP BY` 子句包含一个表达式列表（或者一个表达式，视为长度为一的列表）。该列表作为“分组键”，而每个单独的表达式将被称为“键表达式”。
- 所有在 [SELECT](/sql-reference/statements/select/index.md)、[HAVING](/sql-reference/statements/select/having.md) 和 [ORDER BY](/sql-reference/statements/select/order-by.md) 子句中的表达式 **必须** 基于键表达式 **或** 在非键表达式（包括普通列）上的 [聚合函数](../../../sql-reference/aggregate-functions/index.md) 来计算。换句话说，从表中选择的每一列必须要么用于键表达式，要么在聚合函数内使用，但不能同时用于两者。
- 聚合 `SELECT` 查询的结果行数将与源表中“分组键”的唯一值数量相等。通常，这会显著降低行数，往往是几个数量级，但不一定：如果所有“分组键”值都是唯一的，行数保持不变。

当您想根据列号而不是列名对表中的数据进行分组时，可以启用设置 [enable_positional_arguments](/operations/settings/settings#enable_positional_arguments)。

:::note
还有另一种在表上运行聚合的方法。如果查询仅在聚合函数内包含表列，则 `GROUP BY` 子句可以省略，并假定使用空的键集进行聚合。这种查询始终返回正好一行。
:::

## NULL 处理 {#null-processing}

在分组时，ClickHouse 将 [NULL](/sql-reference/syntax#null) 视为一个值，并且 `NULL==NULL`。这与其他上下文中的 `NULL` 处理有所不同。

以下是一个示例，说明这意味着什么。

假设您有以下表：

```text
┌─x─┬────y─┐
│ 1 │    2 │
│ 2 │ ᴺᵁᴸᴸ │
│ 3 │    2 │
│ 3 │    3 │
│ 3 │ ᴺᵁᴸᴸ │
└───┴──────┘
```

查询 `SELECT sum(x), y FROM t_null_big GROUP BY y` 的结果为：

```text
┌─sum(x)─┬────y─┐
│      4 │    2 │
│      3 │    3 │
│      5 │ ᴺᵁᴸᴸ │
└────────┴──────┘
```

您可以看到 `GROUP BY` 对 `y = NULL` 的处理对 `x` 进行了求和，就好像 `NULL` 是这个值一样。

如果您向 `GROUP BY` 传递多个键，结果将给出所选内容的所有组合，就好像 `NULL` 是一个特定的值。

## ROLLUP 修饰符 {#rollup-modifier}

`ROLLUP` 修饰符用于根据 `GROUP BY` 列表中的顺序计算键表达式的子总计。子总计行在结果表之后添加。

子总计是按相反顺序计算的：首先为列表中的最后一个键表达式计算子总计，然后是前一个，以此类推，直到第一个键表达式。

在子总计行中，已“分组”的键表达式的值被设置为 `0` 或空行。

:::note
请注意， [HAVING](/sql-reference/statements/select/having.md) 子句可能会影响子总计结果。
:::

**示例**

考虑表 t：

```text
┌─year─┬─month─┬─day─┐
│ 2019 │     1 │   5 │
│ 2019 │     1 │  15 │
│ 2020 │     1 │   5 │
│ 2020 │     1 │  15 │
│ 2020 │    10 │   5 │
│ 2020 │    10 │  15 │
└──────┴───────┴─────┘
```

查询：

```sql
SELECT year, month, day, count(*) FROM t GROUP BY ROLLUP(year, month, day);
```
由于 `GROUP BY` 部分有三个键表达式，因此结果包含四个带有子总计的表，这些总计是从右到左“滚动”得出的：

- `GROUP BY year, month, day`；
- `GROUP BY year, month`（`day` 列填充为零）；
- `GROUP BY year`（现在 `month, day` 列都填充为零）；
- 和总计（所有三个键表达式列都是零）。

```text
┌─year─┬─month─┬─day─┬─count()─┐
│ 2020 │    10 │  15 │       1 │
│ 2020 │     1 │   5 │       1 │
│ 2019 │     1 │   5 │       1 │
│ 2020 │     1 │  15 │       1 │
│ 2019 │     1 │  15 │       1 │
│ 2020 │    10 │   5 │       1 │
└──────┴───────┴─────┴─────────┘
┌─year─┬─month─┬─day─┬─count()─┐
│ 2019 │     1 │   0 │       2 │
│ 2020 │     1 │   0 │       2 │
│ 2020 │    10 │   0 │       2 │
└──────┴───────┴─────┴─────────┘
┌─year─┬─month─┬─day─┬─count()─┐
│ 2019 │     0 │   0 │       2 │
│ 2020 │     0 │   0 │       4 │
└──────┴───────┴─────┴─────────┘
┌─year─┬─month─┬─day─┬─count()─┐
│    0 │     0 │   0 │       6 │
└──────┴───────┴─────┴─────────┘
```
同样的查询也可以使用 `WITH` 关键字重写。
```sql
SELECT year, month, day, count(*) FROM t GROUP BY year, month, day WITH ROLLUP;
```

**另请参阅**

- [group_by_use_nulls](/operations/settings/settings.md#group_by_use_nulls) 设置以兼容 SQL 标准。

## CUBE 修饰符 {#cube-modifier}

`CUBE` 修饰符用于计算 `GROUP BY` 列表中每个键表达式组合的子总计。子总计行在结果表之后添加。

在子总计行中，所有“分组”键表达式的值被设置为 `0` 或空行。

:::note
请注意， [HAVING](/sql-reference/statements/select/having.md) 子句可能会影响子总计结果。
:::

**示例**

考虑表 t：

```text
┌─year─┬─month─┬─day─┐
│ 2019 │     1 │   5 │
│ 2019 │     1 │  15 │
│ 2020 │     1 │   5 │
│ 2020 │     1 │  15 │
│ 2020 │    10 │   5 │
│ 2020 │    10 │  15 │
└──────┴───────┴─────┘
```

查询：

```sql
SELECT year, month, day, count(*) FROM t GROUP BY CUBE(year, month, day);
```

由于 `GROUP BY` 部分有三个键表达式，因此结果包含八个带有所有键表达式组合的子总计的表：

- `GROUP BY year, month, day`
- `GROUP BY year, month`
- `GROUP BY year, day`
- `GROUP BY year`
- `GROUP BY month, day`
- `GROUP BY month`
- `GROUP BY day`
- 和总计。

未包含在 `GROUP BY` 中的列填充为零。

```text
┌─year─┬─month─┬─day─┬─count()─┐
│ 2020 │    10 │  15 │       1 │
│ 2020 │     1 │   5 │       1 │
│ 2019 │     1 │   5 │       1 │
│ 2020 │     1 │  15 │       1 │
│ 2019 │     1 │  15 │       1 │
│ 2020 │    10 │   5 │       1 │
└──────┴───────┴─────┴─────────┘
┌─year─┬─month─┬─day─┬─count()─┐
│ 2019 │     1 │   0 │       2 │
│ 2020 │     1 │   0 │       2 │
│ 2020 │    10 │   0 │       2 │
└──────┴───────┴─────┴─────────┘
┌─year─┬─month─┬─day─┬─count()─┐
│ 2020 │     0 │   5 │       2 │
│ 2019 │     0 │   5 │       1 │
│ 2020 │     0 │  15 │       2 │
│ 2019 │     0 │  15 │       1 │
└──────┴───────┴─────┴─────────┘
┌─year─┬─month─┬─day─┬─count()─┐
│ 2019 │     0 │   0 │       2 │
│ 2020 │     0 │   0 │       4 │
└──────┴───────┴─────┴─────────┘
┌─year─┬─month─┬─day─┬─count()─┐
│    0 │     1 │   5 │       2 │
│    0 │    10 │  15 │       1 │
│    0 │    10 │   5 │       1 │
│    0 │     1 │  15 │       2 │
└──────┴───────┴─────┴─────────┘
┌─year─┬─month─┬─day─┬─count()─┐
│    0 │     1 │   0 │       4 │
│    0 │    10 │   0 │       2 │
└──────┴───────┴─────┴─────────┘
┌─year─┬─month─┬─day─┬─count()─┐
│    0 │     0 │   5 │       3 │
│    0 │     0 │  15 │       3 │
└──────┴───────┴─────┴─────────┘
┌─year─┬─month─┬─day─┬─count()─┐
│    0 │     0 │   0 │       6 │
└──────┴───────┴─────┴─────────┘
```
同样的查询也可以使用 `WITH` 关键字重写。
```sql
SELECT year, month, day, count(*) FROM t GROUP BY year, month, day WITH CUBE;
```

**另请参阅**

- [group_by_use_nulls](/operations/settings/settings.md#group_by_use_nulls) 设置以兼容 SQL 标准。

## WITH TOTALS 修饰符 {#with-totals-modifier}

如果指定了 `WITH TOTALS` 修饰符，将计算另一行。该行将具有包含默认值（零或空行）的键列，以及在所有行上计算的聚合函数的列（“总计”值）。

此额外行仅在 `JSON*`、`TabSeparated*` 和 `Pretty*` 格式中生成，与其他行分开：

- 在 `XML` 和 `JSON*` 格式中，此行作为单独的 'totals' 字段输出。
- 在 `TabSeparated*`、`CSV*` 和 `Vertical` 格式中，行在主要结果之后，前面有一行空行（在其他数据之后）。
- 在 `Pretty*` 格式中，该行在主要结果后作为单独的表输出。
- 在 `Template` 格式中，行根据指定的模板输出。
- 在其他格式中不可用。

:::note
在 `SELECT` 查询的结果中输出 totals，但在 `INSERT INTO ... SELECT` 中不会输出。
:::

当存在 [HAVING](/sql-reference/statements/select/having.md) 时，`WITH TOTALS` 可以以不同方式运行。该行为取决于 `totals_mode` 设置。

### 配置总计处理 {#configuring-totals-processing}

默认情况下，`totals_mode = 'before_having'`。在这种情况下，'totals' 是基于所有行计算的，包括那些未通过 HAVING 和 `max_rows_to_group_by` 的行。

其他选项包括仅将通过 HAVING 的行包含在 'totals' 中，并在 `max_rows_to_group_by` 和 `group_by_overflow_mode = 'any'` 设置上表现不同。

`after_having_exclusive` – 不包括未通过 `max_rows_to_group_by` 的行。换句话说，'totals' 的行数将少于或等于未通过 `max_rows_to_group_by` 时的行数。

`after_having_inclusive` – 包含所有未通过 'max_rows_to_group_by' 的行。换句话说，'totals' 的行数将多于或等于未通过 `max_rows_to_group_by` 时的行数。

`after_having_auto` – 计算通过 HAVING 的行数。如果超过某个数量（默认值 50%），则将所有未通过 'max_rows_to_group_by' 的行包含在 'totals' 中。否则，不包括它们。

`totals_auto_threshold` – 默认值为 0.5。用于 `after_having_auto` 的系数。

如果未使用 `max_rows_to_group_by` 和 `group_by_overflow_mode = 'any'`，则所有 `after_having` 的变体是相同的，您可以使用它们中的任何一个（例如， `after_having_auto`）。

您可以在子查询中使用 `WITH TOTALS`，包括在 [JOIN](/sql-reference/statements/select/join.md) 子句中的子查询（在这种情况下，相应的总值会组合在一起）。

## GROUP BY ALL {#group-by-all}

`GROUP BY ALL` 等同于列出所有未使用聚合函数的 SELECT 表达式。

例如：

```sql
SELECT
    a * 2,
    b,
    count(c),
FROM t
GROUP BY ALL
```

与

```sql
SELECT
    a * 2,
    b,
    count(c),
FROM t
GROUP BY a * 2, b
```

相同。

对于特殊情况，如果存在一个函数接受聚合函数和其他字段作为其参数，`GROUP BY` 键将包含我们能够提取的最大非聚合字段。

例如：

```sql
SELECT
    substring(a, 4, 2),
    substring(substring(a, 1, 2), 1, count(b))
FROM t
GROUP BY ALL
```

与

```sql
SELECT
    substring(a, 4, 2),
    substring(substring(a, 1, 2), 1, count(b))
FROM t
GROUP BY substring(a, 4, 2), substring(a, 1, 2)
```

相同。

## 示例 {#examples}

示例：

```sql
SELECT
    count(),
    median(FetchTiming > 60 ? 60 : FetchTiming),
    count() - sum(Refresh)
FROM hits
```

与 MySQL 相比（并符合标准 SQL），您不能获取一些不在键或聚合函数中的列的值（常量表达式除外）。为了解决这个问题，您可以使用 'any' 聚合函数（获取首次遇到的值）或 'min/max'。

示例：

```sql
SELECT
    domainWithoutWWW(URL) AS domain,
    count(),
    any(Title) AS title -- getting the first occurred page header for each domain.
FROM hits
GROUP BY domain
```

对于每个遇到的不同键值，`GROUP BY` 计算一组聚合函数值。

## GROUPING SETS 修饰符 {#grouping-sets-modifier}

这是最通用的修饰符。
此修饰符允许手动指定多个聚合键集（分组集）。
聚合分别在每个分组集上执行，之后将所有结果合并。
如果某列未出现在分组集中，则用默认值填充。

换句话说，上述描述的修饰符可以通过 `GROUPING SETS` 表示。
尽管带有 `ROLLUP`、`CUBE` 和 `GROUPING SETS` 修饰符的查询在语法上是相等的，但它们的执行方式可能不同。
当 `GROUPING SETS` 尝试并行执行所有操作时，`ROLLUP` 和 `CUBE` 在单个线程中执行聚合的最终合并。

在源列包含默认值的情况下，可能很难区分一行是否是使用这些列作为键的聚合的一部分。
为了解决此问题，必须使用 `GROUPING` 函数。

**示例**

以下两个查询是等效的。

```sql
-- Query 1
SELECT year, month, day, count(*) FROM t GROUP BY year, month, day WITH ROLLUP;

-- Query 2
SELECT year, month, day, count(*) FROM t GROUP BY
GROUPING SETS
(
    (year, month, day),
    (year, month),
    (year),
    ()
);
```

**另请参阅**

- [group_by_use_nulls](/operations/settings/settings.md#group_by_use_nulls) 设置以兼容 SQL 标准。

## 实现细节 {#implementation-details}

聚合是列式 DBMS 中最重要的功能之一，因此它的实现是 ClickHouse 中优化得最彻底的部分之一。默认情况下，聚合在内存中使用哈希表完成。它具有 40 多种专用实现，可以根据“分组键”数据类型自动选择。

### 根据表排序键优化 GROUP BY {#group-by-optimization-depending-on-table-sorting-key}

如果表按某个键排序，并且 `GROUP BY` 表达式至少包含排序键的前缀或单射函数，则可以更有效地执行聚合。在这种情况下，当从表中读取新的键时，可以最终确定聚合的中间结果并将其发送给客户端。这种行为由 [optimize_aggregation_in_order](../../../operations/settings/settings.md#optimize_aggregation_in_order) 设置开启。此优化在聚合期间减少内存使用，但在某些情况下可能会减慢查询执行速度。

### 在外部内存中执行 GROUP BY {#group-by-in-external-memory}

您可以启用将临时数据转储到磁盘，以限制 `GROUP BY` 期间的内存使用情况。
[ max_bytes_before_external_group_by](/operations/settings/settings#max_bytes_before_external_group_by) 设置确定将 `GROUP BY` 临时数据转储到文件系统的 RAM 使用阈值。如果设置为 0（默认值），则禁用。
另外，您可以设置 [max_bytes_ratio_before_external_group_by](/operations/settings/settings#max_bytes_ratio_before_external_group_by)，这允许在查询达到某个使用内存的阈值时才在外部内存中使用 `GROUP BY`。

在使用 `max_bytes_before_external_group_by` 时，我们建议您将 `max_memory_usage` 设置为大约两倍（或 `max_bytes_ratio_before_external_group_by=0.5`）。这是因为聚合有两个阶段：读取数据并形成中间数据（1）和合并中间数据（2）。数据转储到文件系统只能在阶段 1 中发生。如果临时数据未被转储，则阶段 2 可能需要的内存与阶段 1 相同。

例如，如果 [max_memory_usage](/operations/settings/settings#max_memory_usage) 设置为 10000000000 并且您想使用外部聚合，那么将 `max_bytes_before_external_group_by` 设置为 10000000000，并将 `max_memory_usage` 设置为 20000000000 是合乎逻辑的。当触发外部聚合时（如果至少有一次临时数据转储），最大 RAM 消耗仅略高于 `max_bytes_before_external_group_by`。

在分布式查询处理时，外部聚合在远程服务器上执行。为了使请求服务器仅使用少量 RAM，请将 `distributed_aggregation_memory_efficient` 设置为 1。

当合并已刷新到磁盘的数据时，以及在 `distributed_aggregation_memory_efficient` 设置启用时合并来自远程服务器的结果时，会消耗最多 `1/256 * 线程数` 的总 RAM。

当启用外部聚合时，如果数据少于 `max_bytes_before_external_group_by`（即数据未被刷新），查询的运行速度与没有外部聚合时相同。如果有任何临时数据被刷新，则运行时间将长几倍（大约三倍）。

如果您的 [ORDER BY](/sql-reference/statements/select/order-by.md) 后有一个 [LIMIT](/sql-reference/statements/select/limit.md)，则使用的 RAM 量取决于 `LIMIT` 中的数据量，而不是整个表中的数据量。但如果 `ORDER BY` 没有 `LIMIT`，请记得启用外部排序（`max_bytes_before_external_sort`）。
