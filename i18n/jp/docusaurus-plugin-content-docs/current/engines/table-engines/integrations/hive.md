---
description: 'The Hive engine allows you to perform `SELECT` queries on HDFS Hive
  table.'
sidebar_label: 'Hive'
sidebar_position: 84
slug: '/engines/table-engines/integrations/hive'
title: 'Hive'
---

import CloudNotSupportedBadge from '@theme/badges/CloudNotSupportedBadge';



# Hive

<CloudNotSupportedBadge/>

Hiveエンジンを利用することで、HDFS Hiveテーブルに対して`SELECT`クエリを実行することができます。現在、以下の入力フォーマットがサポートされています。

- テキスト: `binary`を除くシンプルなスカラー型のみをサポート

- ORC: `char`を除くシンプルなスカラー型をサポート; `array`のような複雑な型のみをサポート

- Parquet: すべてのシンプルなスカラー型をサポート; `array`のような複雑な型のみをサポート

## テーブルの作成 {#creating-a-table}

```sql
CREATE TABLE [IF NOT EXISTS] [db.]table_name [ON CLUSTER cluster]
(
    name1 [type1] [ALIAS expr1],
    name2 [type2] [ALIAS expr2],
    ...
) ENGINE = Hive('thrift://host:port', 'database', 'table');
PARTITION BY expr
```
[CREATE TABLE](/sql-reference/statements/create/table) クエリの詳細な説明を参照してください。

テーブルの構造は元のHiveテーブルの構造と異なることがあります:
- カラム名は元のHiveテーブル内のものと同じである必要がありますが、これらのカラムの一部のみを使用することができ、順序も任意で、他のカラムから計算されたエイリアスカラムを使用することもできます。
- カラムタイプは元のHiveテーブルのものと同じである必要があります。
- パーティションによる式は元のHiveテーブルと一貫性を保ち、パーティションによる式のカラムはテーブル構造内に含まれている必要があります。

**エンジンパラメータ**

- `thrift://host:port` — Hiveメタストアのアドレス

- `database` — リモートデータベース名。

- `table` — リモートテーブル名。

## 使用例 {#usage-example}

### HDFSファイルシステムのローカルキャッシュの使用方法 {#how-to-use-local-cache-for-hdfs-filesystem}

リモートファイルシステムのローカルキャッシュを有効にすることを強くお勧めします。ベンチマークによると、キャッシュを使用した場合、ほぼ2倍の速度向上が見られます。

キャッシュを使用する前に、`config.xml`に追加してください。
```xml
<local_cache_for_remote_fs>
    <enable>true</enable>
    <root_dir>local_cache</root_dir>
    <limit_size>559096952</limit_size>
    <bytes_read_before_flush>1048576</bytes_read_before_flush>
</local_cache_for_remote_fs>
```

- enable: trueの場合、ClickHouseは起動後にリモートファイルシステム（HDFS）のローカルキャッシュを維持します。
- root_dir: 必須。リモートファイルシステム用のローカルキャッシュファイルを保存するルートディレクトリ。
- limit_size: 必須。ローカルキャッシュファイルの最大サイズ（バイト単位）。
- bytes_read_before_flush: リモートファイルシステムからファイルをダウンロードする際にローカルファイルシステムにフラッシュする前のバイト数を制御します。デフォルト値は1MBです。

### ORC入力フォーマットでHiveテーブルにクエリを実行する {#query-hive-table-with-orc-input-format}

#### Hiveでのテーブル作成 {#create-table-in-hive}

```text
hive > CREATE TABLE `test`.`test_orc`(
  `f_tinyint` tinyint,
  `f_smallint` smallint,
  `f_int` int,
  `f_integer` int,
  `f_bigint` bigint,
  `f_float` float,
  `f_double` double,
  `f_decimal` decimal(10,0),
  `f_timestamp` timestamp,
  `f_date` date,
  `f_string` string,
  `f_varchar` varchar(100),
  `f_bool` boolean,
  `f_binary` binary,
  `f_array_int` array<int>,
  `f_array_string` array<string>,
  `f_array_float` array<float>,
  `f_array_array_int` array<array<int>>,
  `f_array_array_string` array<array<string>>,
  `f_array_array_float` array<array<float>>)
PARTITIONED BY (
  `day` string)
ROW FORMAT SERDE
  'org.apache.hadoop.hive.ql.io.orc.OrcSerde'
STORED AS INPUTFORMAT
  'org.apache.hadoop.hive.ql.io.orc.OrcInputFormat'
OUTPUTFORMAT
  'org.apache.hadoop.hive.ql.io.orc.OrcOutputFormat'
LOCATION
  'hdfs://testcluster/data/hive/test.db/test_orc'

OK
Time taken: 0.51 seconds

hive > insert into test.test_orc partition(day='2021-09-18') select 1, 2, 3, 4, 5, 6.11, 7.22, 8.333, current_timestamp(), current_date(), 'hello world', 'hello world', 'hello world', true, 'hello world', array(1, 2, 3), array('hello world', 'hello world'), array(float(1.1), float(1.2)), array(array(1, 2), array(3, 4)), array(array('a', 'b'), array('c', 'd')), array(array(float(1.11), float(2.22)), array(float(3.33), float(4.44)));
OK
Time taken: 36.025 seconds

hive > select * from test.test_orc;
OK
1    2    3    4    5    6.11    7.22    8    2021-11-05 12:38:16.314    2021-11-05    hello world    hello world    hello world                                                                                             true    hello world    [1,2,3]    ["hello world","hello world"]    [1.1,1.2]    [[1,2],[3,4]]    [["a","b"],["c","d"]]    [[1.11,2.22],[3.33,4.44]]    2021-09-18
Time taken: 0.295 seconds, Fetched: 1 row(s)
```

#### ClickHouseでのテーブル作成 {#create-table-in-clickhouse}

ClickHouseでのテーブル、上記で作成したHiveテーブルからデータを取得:
```sql
CREATE TABLE test.test_orc
(
    `f_tinyint` Int8,
    `f_smallint` Int16,
    `f_int` Int32,
    `f_integer` Int32,
    `f_bigint` Int64,
    `f_float` Float32,
    `f_double` Float64,
    `f_decimal` Float64,
    `f_timestamp` DateTime,
    `f_date` Date,
    `f_string` String,
    `f_varchar` String,
    `f_bool` Bool,
    `f_binary` String,
    `f_array_int` Array(Int32),
    `f_array_string` Array(String),
    `f_array_float` Array(Float32),
    `f_array_array_int` Array(Array(Int32)),
    `f_array_array_string` Array(Array(String)),
    `f_array_array_float` Array(Array(Float32)),
    `day` String
)
ENGINE = Hive('thrift://202.168.117.26:9083', 'test', 'test_orc')
PARTITION BY day

```

```sql
SELECT * FROM test.test_orc settings input_format_orc_allow_missing_columns = 1\G
```

```text
SELECT *
FROM test.test_orc
SETTINGS input_format_orc_allow_missing_columns = 1

Query id: c3eaffdc-78ab-43cd-96a4-4acc5b480658

Row 1:
──────
f_tinyint:            1
f_smallint:           2
f_int:                3
f_integer:            4
f_bigint:             5
f_float:              6.11
f_double:             7.22
f_decimal:            8
f_timestamp:          2021-12-04 04:00:44
f_date:               2021-12-03
f_string:             hello world
f_varchar:            hello world
f_bool:               true
f_binary:             hello world
f_array_int:          [1,2,3]
f_array_string:       ['hello world','hello world']
f_array_float:        [1.1,1.2]
f_array_array_int:    [[1,2],[3,4]]
f_array_array_string: [['a','b'],['c','d']]
f_array_array_float:  [[1.11,2.22],[3.33,4.44]]
day:                  2021-09-18


1 rows in set. Elapsed: 0.078 sec.
```

### Parquet入力フォーマットでHiveテーブルにクエリを実行する {#query-hive-table-with-parquet-input-format}

#### Hiveでのテーブル作成 {#create-table-in-hive-1}

```text
hive >
CREATE TABLE `test`.`test_parquet`(
  `f_tinyint` tinyint,
  `f_smallint` smallint,
  `f_int` int,
  `f_integer` int,
  `f_bigint` bigint,
  `f_float` float,
  `f_double` double,
  `f_decimal` decimal(10,0),
  `f_timestamp` timestamp,
  `f_date` date,
  `f_string` string,
  `f_varchar` varchar(100),
  `f_char` char(100),
  `f_bool` boolean,
  `f_binary` binary,
  `f_array_int` array<int>,
  `f_array_string` array<string>,
  `f_array_float` array<float>,
  `f_array_array_int` array<array<int>>,
  `f_array_array_string` array<array<string>>,
  `f_array_array_float` array<array<float>>)
PARTITIONED BY (
  `day` string)
ROW FORMAT SERDE
  'org.apache.hadoop.hive.ql.io.parquet.serde.ParquetHiveSerDe'
STORED AS INPUTFORMAT
  'org.apache.hadoop.hive.ql.io.parquet.MapredParquetInputFormat'
OUTPUTFORMAT
  'org.apache.hadoop.hive.ql.io.parquet.MapredParquetOutputFormat'
LOCATION
  'hdfs://testcluster/data/hive/test.db/test_parquet'
OK
Time taken: 0.51 seconds

hive >  insert into test.test_parquet partition(day='2021-09-18') select 1, 2, 3, 4, 5, 6.11, 7.22, 8.333, current_timestamp(), current_date(), 'hello world', 'hello world', 'hello world', true, 'hello world', array(1, 2, 3), array('hello world', 'hello world'), array(float(1.1), float(1.2)), array(array(1, 2), array(3, 4)), array(array('a', 'b'), array('c', 'd')), array(array(float(1.11), float(2.22)), array(float(3.33), float(4.44)));
OK
Time taken: 36.025 seconds

hive > select * from test.test_parquet;
OK
1    2    3    4    5    6.11    7.22    8    2021-12-14 17:54:56.743    2021-12-14    hello world    hello world    hello world                                                                                             true    hello world    [1,2,3]    ["hello world","hello world"]    [1.1,1.2]    [[1,2],[3,4]]    [["a","b"],["c","d"]]    [[1.11,2.22],[3.33,4.44]]    2021-09-18
Time taken: 0.766 seconds, Fetched: 1 row(s)
```

#### ClickHouseでのテーブル作成 {#create-table-in-clickhouse-1}

ClickHouseでのテーブル、上記で作成したHiveテーブルからデータを取得:
```sql
CREATE TABLE test.test_parquet
(
    `f_tinyint` Int8,
    `f_smallint` Int16,
    `f_int` Int32,
    `f_integer` Int32,
    `f_bigint` Int64,
    `f_float` Float32,
    `f_double` Float64,
    `f_decimal` Float64,
    `f_timestamp` DateTime,
    `f_date` Date,
    `f_string` String,
    `f_varchar` String,
    `f_char` String,
    `f_bool` Bool,
    `f_binary` String,
    `f_array_int` Array(Int32),
    `f_array_string` Array(String),
    `f_array_float` Array(Float32),
    `f_array_array_int` Array(Array(Int32)),
    `f_array_array_string` Array(Array(String)),
    `f_array_array_float` Array(Array(Float32)),
    `day` String
)
ENGINE = Hive('thrift://localhost:9083', 'test', 'test_parquet')
PARTITION BY day
```

```sql
SELECT * FROM test.test_parquet settings input_format_parquet_allow_missing_columns = 1\G
```

```text
SELECT *
FROM test_parquet
SETTINGS input_format_parquet_allow_missing_columns = 1

Query id: 4e35cf02-c7b2-430d-9b81-16f438e5fca9

Row 1:
──────
f_tinyint:            1
f_smallint:           2
f_int:                3
f_integer:            4
f_bigint:             5
f_float:              6.11
f_double:             7.22
f_decimal:            8
f_timestamp:          2021-12-14 17:54:56
f_date:               2021-12-14
f_string:             hello world
f_varchar:            hello world
f_char:               hello world
f_bool:               true
f_binary:             hello world
f_array_int:          [1,2,3]
f_array_string:       ['hello world','hello world']
f_array_float:        [1.1,1.2]
f_array_array_int:    [[1,2],[3,4]]
f_array_array_string: [['a','b'],['c','d']]
f_array_array_float:  [[1.11,2.22],[3.33,4.44]]
day:                  2021-09-18

1 rows in set. Elapsed: 0.357 sec.
```

### テキスト入力フォーマットでHiveテーブルにクエリを実行する {#query-hive-table-with-text-input-format}

#### Hiveでのテーブル作成 {#create-table-in-hive-2}

```text
hive >
CREATE TABLE `test`.`test_text`(
  `f_tinyint` tinyint,
  `f_smallint` smallint,
  `f_int` int,
  `f_integer` int,
  `f_bigint` bigint,
  `f_float` float,
  `f_double` double,
  `f_decimal` decimal(10,0),
  `f_timestamp` timestamp,
  `f_date` date,
  `f_string` string,
  `f_varchar` varchar(100),
  `f_char` char(100),
  `f_bool` boolean,
  `f_binary` binary,
  `f_array_int` array<int>,
  `f_array_string` array<string>,
  `f_array_float` array<float>,
  `f_array_array_int` array<array<int>>,
  `f_array_array_string` array<array<string>>,
  `f_array_array_float` array<array<float>>)
PARTITIONED BY (
  `day` string)
ROW FORMAT SERDE
  'org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe'
STORED AS INPUTFORMAT
  'org.apache.hadoop.mapred.TextInputFormat'
OUTPUTFORMAT
  'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
LOCATION
  'hdfs://testcluster/data/hive/test.db/test_text'
Time taken: 0.1 seconds, Fetched: 34 row(s)


hive >  insert into test.test_text partition(day='2021-09-18') select 1, 2, 3, 4, 5, 6.11, 7.22, 8.333, current_timestamp(), current_date(), 'hello world', 'hello world', 'hello world', true, 'hello world', array(1, 2, 3), array('hello world', 'hello world'), array(float(1.1), float(1.2)), array(array(1, 2), array(3, 4)), array(array('a', 'b'), array('c', 'd')), array(array(float(1.11), float(2.22)), array(float(3.33), float(4.44)));
OK
Time taken: 36.025 seconds

hive > select * from test.test_text;
OK
1    2    3    4    5    6.11    7.22    8    2021-12-14 18:11:17.239    2021-12-14    hello world    hello world    hello world                                                                                             true    hello world    [1,2,3]    ["hello world","hello world"]    [1.1,1.2]    [[1,2],[3,4]]    [["a","b"],["c","d"]]    [[1.11,2.22],[3.33,4.44]]    2021-09-18
Time taken: 0.624 seconds, Fetched: 1 row(s)
```

#### ClickHouseでのテーブル作成 {#create-table-in-clickhouse-2}

ClickHouseでのテーブル、上記で作成したHiveテーブルからデータを取得:
```sql
CREATE TABLE test.test_text
(
    `f_tinyint` Int8,
    `f_smallint` Int16,
    `f_int` Int32,
    `f_integer` Int32,
    `f_bigint` Int64,
    `f_float` Float32,
    `f_double` Float64,
    `f_decimal` Float64,
    `f_timestamp` DateTime,
    `f_date` Date,
    `f_string` String,
    `f_varchar` String,
    `f_char` String,
    `f_bool` Bool,
    `day` String
)
ENGINE = Hive('thrift://localhost:9083', 'test', 'test_text')
PARTITION BY day
```

```sql
SELECT * FROM test.test_text settings input_format_skip_unknown_fields = 1, input_format_with_names_use_header = 1, date_time_input_format = 'best_effort'\G
```

```text
SELECT *
FROM test.test_text
SETTINGS input_format_skip_unknown_fields = 1, input_format_with_names_use_header = 1, date_time_input_format = 'best_effort'

Query id: 55b79d35-56de-45b9-8be6-57282fbf1f44

Row 1:
──────
f_tinyint:   1
f_smallint:  2
f_int:       3
f_integer:   4
f_bigint:    5
f_float:     6.11
f_double:    7.22
f_decimal:   8
f_timestamp: 2021-12-14 18:11:17
f_date:      2021-12-14
f_string:    hello world
f_varchar:   hello world
f_char:      hello world
f_bool:      true
day:         2021-09-18
```
