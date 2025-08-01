---
description: 'Documentation for Check Table'
sidebar_label: 'CHECK TABLE'
sidebar_position: 41
slug: '/sql-reference/statements/check-table'
title: 'CHECK TABLE Statement'
---



The `CHECK TABLE` クエリは ClickHouse で特定のテーブルまたはそのパーティションのバリデーションチェックを実行するために使用されます。これは、チェックサムとその他の内部データ構造を確認することにより、データの整合性を確保します。

特に、実際のファイルサイズとサーバーに保存されている期待値とを比較します。ファイルサイズが保存された値と一致しない場合、データが破損していることを意味します。これは、例えば、クエリの実行中にシステムがクラッシュした場合に起こり得ます。

:::warning
`CHECK TABLE` クエリはテーブル内のすべてのデータを読み取る可能性があり、いくつかのリソースを消費します。そのため、リソースに負荷がかかります。
このクエリを実行する前に、パフォーマンスやリソース利用への影響を考慮してください。
このクエリはシステムのパフォーマンスを向上させるものではなく、実行する際に何をしているのか不明な場合は実行しないでください。
:::

## 構文 {#syntax}

クエリの基本構文は次のとおりです。

```sql
CHECK TABLE table_name [PARTITION partition_expression | PART part_name] [FORMAT format] [SETTINGS check_query_single_value_result = (0|1) [, other_settings]]
```

- `table_name`: チェックしたいテーブルの名前を指定します。
- `partition_expression`: （オプション）テーブルの特定のパーティションをチェックしたい場合、この式を使用してパーティションを指定できます。
- `part_name`: （オプション）テーブル内の特定のパートをチェックしたい場合、文字列リテラルを追加してパート名を指定できます。
- `FORMAT format`: （オプション）結果の出力形式を指定します。
- `SETTINGS`: （オプション）追加の設定を可能にします。
    - **`check_query_single_value_result`**: （オプション）この設定を使用すると、詳細な結果（`0`）または要約された結果（`1`）を切り替えできます。
    - その他の設定も適用できます。結果の決定的な順序が必要ない場合、`max_threads` を 1 より大きな値に設定してクエリを高速化できます。

クエリの応答は、`check_query_single_value_result` 設定の値に依存します。
`check_query_single_value_result = 1` の場合、単一行の `result` カラムのみが返されます。この行内の値は、整合性チェックが通れば `1`、データが破損していれば `0` です。

`check_query_single_value_result = 0` の場合、クエリは次のカラムを返します：
    - `part_path`: データパートまたはファイル名へのパスを示します。
    - `is_passed`: このパートのチェックが成功した場合は `1`、それ以外の場合は `0` を返します。
    - `message`: チェックに関連する追加メッセージ、例えばエラーや成功メッセージなど。

`CHECK TABLE` クエリは次のテーブルエンジンをサポートしています：

- [Log](../../engines/table-engines/log-family/log.md)
- [TinyLog](../../engines/table-engines/log-family/tinylog.md)
- [StripeLog](../../engines/table-engines/log-family/stripelog.md)
- [MergeTree family](../../engines/table-engines/mergetree-family/mergetree.md)

他のテーブルエンジン上で実行すると `NOT_IMPLEMENTED` 例外が発生します。

`*Log` ファミリーのエンジンは、障害発生時に自動データ回復を提供しません。`CHECK TABLE` クエリを使用して、タイムリーにデータ損失を追跡できます。

## 例 {#examples}

デフォルトの `CHECK TABLE` クエリは、一般的なテーブルチェックステータスを表示します：

```sql
CHECK TABLE test_table;
```

```text
┌─result─┐
│      1 │
└────────┘
```

個々のデータパートごとのチェックステータスを確認したい場合は、`check_query_single_value_result` 設定を使用できます。

また、テーブルの特定のパーティションをチェックするために、`PARTITION` キーワードを使用できます。

```sql
CHECK TABLE t0 PARTITION ID '201003'
FORMAT PrettyCompactMonoBlock
SETTINGS check_query_single_value_result = 0
```

出力：

```text
┌─part_path────┬─is_passed─┬─message─┐
│ 201003_7_7_0 │         1 │         │
│ 201003_3_3_0 │         1 │         │
└──────────────┴───────────┴─────────┘
```

同様に、`PART` キーワードを使用してテーブルの特定のパートをチェックできます。

```sql
CHECK TABLE t0 PART '201003_7_7_0'
FORMAT PrettyCompactMonoBlock
SETTINGS check_query_single_value_result = 0
```

出力：

```text
┌─part_path────┬─is_passed─┬─message─┐
│ 201003_7_7_0 │         1 │         │
└──────────────┴───────────┴─────────┘
```

存在しないパートがある場合、クエリはエラーを返します：

```sql
CHECK TABLE t0 PART '201003_111_222_0'
```

```text
DB::Exception: No such data part '201003_111_222_0' to check in table 'default.t0'. (NO_SUCH_DATA_PART)
```

### '破損'の結果を受け取る {#receiving-a-corrupted-result}

:::warning
免責事項：ここに記載されている手続き、データディレクトリからのファイルの手動操作や削除を含むものは、実験的または開発環境専用です。プロダクションサーバーでこれを試みないでください。データ損失やその他の意図しない結果を引き起こす可能性があります。
:::

既存のチェックサムファイルを削除します：

```bash
rm /var/lib/clickhouse-server/data/default/t0/201003_3_3_0/checksums.txt
```

```sql
CHECK TABLE t0 PARTITION ID '201003'
FORMAT PrettyCompactMonoBlock
SETTINGS check_query_single_value_result = 0
```

出力：

```text
┌─part_path────┬─is_passed─┬─message──────────────────────────────────┐
│ 201003_7_7_0 │         1 │                                          │
│ 201003_3_3_0 │         1 │ Checksums recounted and written to disk. │
└──────────────┴───────────┴──────────────────────────────────────────┘
```

checksums.txt ファイルが失われている場合、それは復元できます。特定のパーティションに対して CHECK TABLE コマンドを実行する際に再計算され、書き込まれ、ステータスは依然として 'is_passed = 1' と報告されます。

`(Replicated)MergeTree` テーブルを一度にすべてチェックするために、`CHECK ALL TABLES` クエリを使用できます。

```sql
CHECK ALL TABLES
FORMAT PrettyCompactMonoBlock
SETTINGS check_query_single_value_result = 0
```

```text
┌─database─┬─table────┬─part_path───┬─is_passed─┬─message─┐
│ default  │ t2       │ all_1_95_3  │         1 │         │
│ db1      │ table_01 │ all_39_39_0 │         1 │         │
│ default  │ t1       │ all_39_39_0 │         1 │         │
│ db1      │ t1       │ all_39_39_0 │         1 │         │
│ db1      │ table_01 │ all_1_6_1   │         1 │         │
│ default  │ t1       │ all_1_6_1   │         1 │         │
│ db1      │ t1       │ all_1_6_1   │         1 │         │
│ db1      │ table_01 │ all_7_38_2  │         1 │         │
│ db1      │ t1       │ all_7_38_2  │         1 │         │
│ default  │ t1       │ all_7_38_2  │         1 │         │
└──────────┴──────────┴─────────────┴───────────┴─────────┘
```

## データが破損した場合 {#if-the-data-is-corrupted}

テーブルが破損している場合、非破損のデータを別のテーブルにコピーできます。これを行うには：

1.  破損したテーブルと同じ構造の新しいテーブルを作成します。これを実行するには、`CREATE TABLE <new_table_name> AS <damaged_table_name>` クエリを実行します。
2.  次のクエリを単一スレッドで処理するために `max_threads` の値を 1 に設定します。これを行うには、`SET max_threads = 1` クエリを実行します。
3.  `INSERT INTO <new_table_name> SELECT * FROM <damaged_table_name>` クエリを実行します。このリクエストは、破損したテーブルから非破損のデータを別のテーブルにコピーします。破損したパート以前のデータのみがコピーされます。
4.  `clickhouse-client` を再起動して `max_threads` の値をリセットします。
