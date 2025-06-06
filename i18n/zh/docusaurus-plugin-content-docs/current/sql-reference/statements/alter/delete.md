---
'description': 'ALTER TABLE ... DELETE 语句的文档'
'sidebar_label': 'DELETE'
'sidebar_position': 39
'slug': '/sql-reference/statements/alter/delete'
'title': 'ALTER TABLE ... DELETE 语句'
---


# ALTER TABLE ... DELETE 语句

```sql
ALTER TABLE [db.]table [ON CLUSTER cluster] DELETE WHERE filter_expr
```

删除与指定过滤表达式匹配的数据。实现为 [mutation](/sql-reference/statements/alter/index.md#mutations)。

:::note
`ALTER TABLE` 前缀使得此语法与大多数其他支持 SQL 的系统不同。它旨在表明，与 OLTP 数据库中的类似查询不同，这是一项不设计为频繁使用的重操作。`ALTER TABLE` 被认为是一个重量级操作，需要在删除之前合并底层数据。对于 MergeTree 表，建议使用 [`DELETE FROM` 查询](/sql-reference/statements/delete.md)，它执行轻量级删除，并且可以快得多。
:::

`filter_expr` 必须为 `UInt8` 类型。该查询删除表中该表达式取非零值的行。

一个查询可以包含多个用逗号分隔的命令。

查询处理的同步性由 [mutations_sync](/operations/settings/settings.md/#mutations_sync) 设置定义。默认情况下，它是异步的。

**另见**

- [Mutations](/sql-reference/statements/alter/index.md#mutations)
- [ALTER 查询的同步性](/sql-reference/statements/alter/index.md#synchronicity-of-alter-queries)
- [mutations_sync](/operations/settings/settings.md/#mutations_sync) 设置

## 相关内容 {#related-content}

- 博客: [在 ClickHouse 中处理更新和删除](https://clickhouse.com/blog/handling-updates-and-deletes-in-clickhouse)
