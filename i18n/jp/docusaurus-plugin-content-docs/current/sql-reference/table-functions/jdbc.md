---
description: 'Returns a table that is connected via JDBC driver.'
sidebar_label: 'jdbc'
sidebar_position: 100
slug: '/sql-reference/table-functions/jdbc'
title: 'jdbc'
---




# jdbcテーブル関数

:::note
clickhouse-jdbc-bridgeには実験的なコードが含まれており、もはやサポートされていません。信頼性の問題やセキュリティの脆弱性を含む可能性があります。使用は自己責任で行ってください。  
ClickHouseでは、より良い代替手段を提供するClickHouse内蔵のテーブル関数を使用することを推奨します。これにより、アドホッククエリシナリオ（Postgres、MySQL、MongoDBなど）が実現できます。
:::

`jdbc(datasource, schema, table)` - JDBCドライバーを介して接続されるテーブルを返します。

このテーブル関数は、別の[clickhouse-jdbc-bridge](https://github.com/ClickHouse/clickhouse-jdbc-bridge)プログラムが実行されている必要があります。  
Nullable型をサポートしています（クエリされたリモートテーブルのDDLに基づきます）。

## 例 {#examples}

```sql
SELECT * FROM jdbc('jdbc:mysql://localhost:3306/?user=root&password=root', 'schema', 'table')
```

```sql
SELECT * FROM jdbc('mysql://localhost:3306/?user=root&password=root', 'select * from schema.table')
```

```sql
SELECT * FROM jdbc('mysql-dev?p1=233', 'num Int32', 'select toInt32OrZero(''{{p1}}'') as num')
```

```sql
SELECT *
FROM jdbc('mysql-dev?p1=233', 'num Int32', 'select toInt32OrZero(''{{p1}}'') as num')
```

```sql
SELECT a.datasource AS server1, b.datasource AS server2, b.name AS db
FROM jdbc('mysql-dev?datasource_column', 'show databases') a
INNER JOIN jdbc('self?datasource_column', 'show databases') b ON a.Database = b.name

