---
'description': 'OPTIMIZE 的文档'
'sidebar_label': 'OPTIMIZE'
'sidebar_position': 47
'slug': '/sql-reference/statements/optimize'
'title': 'OPTIMIZE 语句'
---

这个查询尝试初始化表的数据部分的无计划合并。请注意，我们通常不推荐使用 `OPTIMIZE TABLE ... FINAL`（请参见这些 [文档](/optimize/avoidoptimizefinal)），因为它的用例主要用于管理，而非日常操作。

:::note
`OPTIMIZE` 不能修复 `太多部分` 错误。
:::

**语法**

```sql
OPTIMIZE TABLE [db.]name [ON CLUSTER cluster] [PARTITION partition | PARTITION ID 'partition_id'] [FINAL | FORCE] [DEDUPLICATE [BY expression]]
```

`OPTIMIZE` 查询支持 [MergeTree](../../engines/table-engines/mergetree-family/mergetree.md) 家族（包括 [物化视图](/sql-reference/statements/create/view#materialized-view)）和 [Buffer](../../engines/table-engines/special/buffer.md) 引擎。其他表引擎不被支持。

当 `OPTIMIZE` 与 [ReplicatedMergeTree](../../engines/table-engines/mergetree-family/replication.md) 家族的表引擎一起使用时，ClickHouse 会创建一个合并任务并等待在所有副本上执行（如果 [alter_sync](/operations/settings/settings#alter_sync) 设置为 `2`）或在当前副本上执行（如果 [alter_sync](/operations/settings/settings#alter_sync) 设置为 `1`）。

- 如果 `OPTIMIZE` 因任何原因未执行合并，它不会通知客户端。要启用通知，请使用 [optimize_throw_if_noop](/operations/settings/settings#optimize_throw_if_noop) 设置。
- 如果您指定了 `PARTITION`，则仅优化指定的分区。[如何设置分区表达式](alter/partition.md#how-to-set-partition-expression)。
- 如果您指定了 `FINAL` 或 `FORCE`，即使所有数据已经合并到一个部分，仍然会执行优化。您可以使用 [optimize_skip_merged_partitions](/operations/settings/settings#optimize_skip_merged_partitions) 设置来控制此行为。此外，即使正在执行并发合并，也会强制执行合并。
- 如果您指定了 `DEDUPLICATE`，则完全相同的行（除非指定了 by 子句）将被去重（比较所有列），这仅对 MergeTree 引擎有意义。

您可以通过 [replication_wait_for_inactive_replica_timeout](/operations/settings/settings#replication_wait_for_inactive_replica_timeout) 设置指定等待非活动副本执行 `OPTIMIZE` 查询的时间（以秒为单位）。

:::note    
如果 `alter_sync` 设置为 `2`，并且某些副本在指定的 `replication_wait_for_inactive_replica_timeout` 设置时间内没有变为活动状态，则会抛出异常 `未完成`。
:::

## BY 表达式 {#by-expression}

如果您想根据自定义列集而非所有列进行去重，可以显式指定列的列表，或使用 [`*`](../../sql-reference/statements/select/index.md#asterisk)、[`COLUMNS`](/sql-reference/statements/select#select-clause) 或 [`EXCEPT`](/sql-reference/statements/select#except) 表达式的任意组合。显式编写或隐式扩展的列列表必须包括行排序表达式（主键和排序键）和分区表达式（分区键）中指定的所有列。

:::note    
注意，`*` 的行为与 `SELECT` 中相同：[MATERIALIZED](/sql-reference/statements/create/view#materialized-view) 和 [ALIAS](../../sql-reference/statements/create/table.md#alias) 列不用于扩展。

此外，指定空列列表或写出导致列列表为空的表达式，或通过 `ALIAS` 列进行去重都是错误的。
:::

**语法**

```sql
OPTIMIZE TABLE table DEDUPLICATE; -- all columns
OPTIMIZE TABLE table DEDUPLICATE BY *; -- excludes MATERIALIZED and ALIAS columns
OPTIMIZE TABLE table DEDUPLICATE BY colX,colY,colZ;
OPTIMIZE TABLE table DEDUPLICATE BY * EXCEPT colX;
OPTIMIZE TABLE table DEDUPLICATE BY * EXCEPT (colX, colY);
OPTIMIZE TABLE table DEDUPLICATE BY COLUMNS('column-matched-by-regex');
OPTIMIZE TABLE table DEDUPLICATE BY COLUMNS('column-matched-by-regex') EXCEPT colX;
OPTIMIZE TABLE table DEDUPLICATE BY COLUMNS('column-matched-by-regex') EXCEPT (colX, colY);
```

**示例**

考虑以下表：

```sql
CREATE TABLE example (
    primary_key Int32,
    secondary_key Int32,
    value UInt32,
    partition_key UInt32,
    materialized_value UInt32 MATERIALIZED 12345,
    aliased_value UInt32 ALIAS 2,
    PRIMARY KEY primary_key
) ENGINE=MergeTree
PARTITION BY partition_key
ORDER BY (primary_key, secondary_key);
```

```sql
INSERT INTO example (primary_key, secondary_key, value, partition_key)
VALUES (0, 0, 0, 0), (0, 0, 0, 0), (1, 1, 2, 2), (1, 1, 2, 3), (1, 1, 3, 3);
```

```sql
SELECT * FROM example;
```
结果：

```sql

┌─primary_key─┬─secondary_key─┬─value─┬─partition_key─┐
│           0 │             0 │     0 │             0 │
│           0 │             0 │     0 │             0 │
└─────────────┴───────────────┴───────┴───────────────┘
┌─primary_key─┬─secondary_key─┬─value─┬─partition_key─┐
│           1 │             1 │     2 │             2 │
└─────────────┴───────────────┴───────┴───────────────┘
┌─primary_key─┬─secondary_key─┬─value─┬─partition_key─┐
│           1 │             1 │     2 │             3 │
│           1 │             1 │     3 │             3 │
└─────────────┴───────────────┴───────┴───────────────┘
```

以下所有示例均在包含 5 行的状态下执行。

#### `DEDUPLICATE` {#deduplicate}
当未指定去重列时，所有列都会被考虑。只有当所有列中的值都等于前一行的对应值时，行才会被移除：

```sql
OPTIMIZE TABLE example FINAL DEDUPLICATE;
```

```sql
SELECT * FROM example;
```

结果：

```response
┌─primary_key─┬─secondary_key─┬─value─┬─partition_key─┐
│           1 │             1 │     2 │             2 │
└─────────────┴───────────────┴───────┴───────────────┘
┌─primary_key─┬─secondary_key─┬─value─┬─partition_key─┐
│           0 │             0 │     0 │             0 │
└─────────────┴───────────────┴───────┴───────────────┘
┌─primary_key─┬─secondary_key─┬─value─┬─partition_key─┐
│           1 │             1 │     2 │             3 │
│           1 │             1 │     3 │             3 │
└─────────────┴───────────────┴───────┴───────────────┘
```

#### `DEDUPLICATE BY *` {#deduplicate-by-}

当隐式指定列时，表将根据所有非 `ALIAS` 或 `MATERIALIZED` 的列进行去重。考虑上面的表，这些列是 `primary_key`、`secondary_key`、`value` 和 `partition_key`：

```sql
OPTIMIZE TABLE example FINAL DEDUPLICATE BY *;
```

```sql
SELECT * FROM example;
```

结果：

```response
┌─primary_key─┬─secondary_key─┬─value─┬─partition_key─┐
│           1 │             1 │     2 │             2 │
└─────────────┴───────────────┴───────┴───────────────┘
┌─primary_key─┬─secondary_key─┬─value─┬─partition_key─┐
│           0 │             0 │     0 │             0 │
└─────────────┴───────────────┴───────┴───────────────┘
┌─primary_key─┬─secondary_key─┬─value─┬─partition_key─┐
│           1 │             1 │     2 │             3 │
│           1 │             1 │     3 │             3 │
└─────────────┴───────────────┴───────┴───────────────┘
```

#### `DEDUPLICATE BY * EXCEPT` {#deduplicate-by--except}
根据所有非 `ALIAS` 或 `MATERIALIZED` 的列去重，并显式不包括 `value`：`primary_key`、`secondary_key` 和 `partition_key` 列。

```sql
OPTIMIZE TABLE example FINAL DEDUPLICATE BY * EXCEPT value;
```

```sql
SELECT * FROM example;
```

结果：

```response
┌─primary_key─┬─secondary_key─┬─value─┬─partition_key─┐
│           1 │             1 │     2 │             2 │
└─────────────┴───────────────┴───────┴───────────────┘
┌─primary_key─┬─secondary_key─┬─value─┬─partition_key─┐
│           0 │             0 │     0 │             0 │
└─────────────┴───────────────┴───────┴───────────────┘
┌─primary_key─┬─secondary_key─┬─value─┬─partition_key─┐
│           1 │             1 │     2 │             3 │
└─────────────┴───────────────┴───────┴───────────────┘
```

#### `DEDUPLICATE BY <列列表>` {#deduplicate-by-list-of-columns}

显式根据 `primary_key`、`secondary_key` 和 `partition_key` 列去重：

```sql
OPTIMIZE TABLE example FINAL DEDUPLICATE BY primary_key, secondary_key, partition_key;
```

```sql
SELECT * FROM example;
```
结果：

```response
┌─primary_key─┬─secondary_key─┬─value─┬─partition_key─┐
│           1 │             1 │     2 │             2 │
└─────────────┴───────────────┴───────┴───────────────┘
┌─primary_key─┬─secondary_key─┬─value─┬─partition_key─┐
│           0 │             0 │     0 │             0 │
└─────────────┴───────────────┴───────┴───────────────┘
┌─primary_key─┬─secondary_key─┬─value─┬─partition_key─┐
│           1 │             1 │     2 │             3 │
└─────────────┴───────────────┴───────┴───────────────┘
```

#### `DEDUPLICATE BY COLUMNS(<正则表达式>)` {#deduplicate-by-columnsregex}

根据匹配正则表达式的所有列进行去重：`primary_key`、`secondary_key` 和 `partition_key` 列：

```sql
OPTIMIZE TABLE example FINAL DEDUPLICATE BY COLUMNS('.*_key');
```

```sql
SELECT * FROM example;
```

结果：

```response
┌─primary_key─┬─secondary_key─┬─value─┬─partition_key─┐
│           0 │             0 │     0 │             0 │
└─────────────┴───────────────┴───────┴───────────────┘
┌─primary_key─┬─secondary_key─┬─value─┬─partition_key─┐
│           1 │             1 │     2 │             2 │
└─────────────┴───────────────┴───────┴───────────────┘
┌─primary_key─┬─secondary_key─┬─value─┬─partition_key─┐
│           1 │             1 │     2 │             3 │
└─────────────┴───────────────┴───────┴───────────────┘
```
