---
'description': 'UNDROP TABLE 的文档'
'sidebar_label': 'UNDROP'
'slug': '/sql-reference/statements/undrop'
'title': 'UNDROP TABLE'
---


# UNDROP TABLE

取消对表的删除。

从 ClickHouse 版本 23.3 开始，可以在发出 DROP TABLE 语句后的 `database_atomic_delay_before_drop_table_sec`（默认情况下为 8 分钟）内对 Atomic 数据库中的表进行 UNDROP。被删除的表会列在名为 `system.dropped_tables` 的系统表中。

如果您有一个没有 `TO` 子句与被删除表关联的物化视图，那么您还需要 UNDROP 那个视图的内部表。

:::tip
另请参见 [DROP TABLE](/sql-reference/statements/drop.md)
:::

语法：

```sql
UNDROP TABLE [db.]name [UUID '<uuid>'] [ON CLUSTER cluster]
```

**示例**

```sql
CREATE TABLE tab
(
    `id` UInt8
)
ENGINE = MergeTree
ORDER BY id;

DROP TABLE tab;

SELECT *
FROM system.dropped_tables
FORMAT Vertical;
```

```response
Row 1:
──────
index:                 0
database:              default
table:                 tab
uuid:                  aa696a1a-1d70-4e60-a841-4c80827706cc
engine:                MergeTree
metadata_dropped_path: /var/lib/clickhouse/metadata_dropped/default.tab.aa696a1a-1d70-4e60-a841-4c80827706cc.sql
table_dropped_time:    2023-04-05 14:12:12

1 row in set. Elapsed: 0.001 sec. 
```

```sql
UNDROP TABLE tab;

SELECT *
FROM system.dropped_tables
FORMAT Vertical;

```response
Ok.

0 rows in set. Elapsed: 0.001 sec. 
```

```sql
DESCRIBE TABLE tab
FORMAT Vertical;
```

```response
Row 1:
──────
name:               id
type:               UInt8
default_type:       
default_expression: 
comment:            
codec_expression:   
ttl_expression:     
```
