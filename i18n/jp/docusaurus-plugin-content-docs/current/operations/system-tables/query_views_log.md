---
description: 'System table containing information about the dependent views executed
  when running a query, for example, the view type or the execution time.'
keywords:
- 'system table'
- 'query_views_log'
slug: '/operations/system-tables/query_views_log'
title: 'system.query_views_log'
---

import SystemTableCloud from '@site/i18n/jp/docusaurus-plugin-content-docs/current/_snippets/_system_table_cloud.md';


# system.query_views_log

<SystemTableCloud/>

クエリを実行する際に実行された依存ビューに関する情報を含みます。例えば、ビューのタイプや実行時間などです。

ログの開始方法:

1. [query_views_log](../../operations/server-configuration-parameters/settings.md#query_views_log) セクションでパラメータを設定します。
2. [log_query_views](/operations/settings/settings#log_query_views) を 1 に設定します。

データのフラッシュ期間は、[query_views_log](../../operations/server-configuration-parameters/settings.md#query_views_log) サーバ設定セクションの `flush_interval_milliseconds` パラメータで設定します。フラッシュを強制的に行うには、[SYSTEM FLUSH LOGS](/sql-reference/statements/system#flush-logs) クエリを使用します。

ClickHouseは、テーブルからデータを自動的に削除しません。詳細については、[Introduction](/operations/system-tables/overview#system-tables-introduction) を参照してください。

[log_queries_probability](/operations/settings/settings#log_queries_probability) 設定を使用して、`query_views_log` テーブルに登録されるクエリの数を減少させることができます。

カラム:

- `hostname` ([LowCardinality(String)](../../sql-reference/data-types/string.md)) — クエリを実行しているサーバのホスト名。
- `event_date` ([Date](../../sql-reference/data-types/date.md)) — ビューの最後のイベントが発生した日付。
- `event_time` ([DateTime](../../sql-reference/data-types/datetime.md)) — ビューの実行が終了した日付と時間。
- `event_time_microseconds` ([DateTime](../../sql-reference/data-types/datetime.md)) — マイクロ秒精度でビューの実行が終了した日付と時間。
- `view_duration_ms` ([UInt64](/sql-reference/data-types/int-uint#integer-ranges)) — ビューの実行時間（そのステージの合計）をミリ秒で表したもの。
- `initial_query_id` ([String](../../sql-reference/data-types/string.md)) — 初期クエリのID（分散クエリ実行用）。
- `view_name` ([String](../../sql-reference/data-types/string.md)) — ビューの名前。
- `view_uuid` ([UUID](../../sql-reference/data-types/uuid.md)) — ビューのUUID。
- `view_type` ([Enum8](../../sql-reference/data-types/enum.md)) — ビューのタイプ。値:
    - `'Default' = 1` — [デフォルトビュー](/sql-reference/statements/create/view#normal-view)。このログには表示されるべきではありません。
    - `'Materialized' = 2` — [マテリアライズドビュー](/sql-reference/statements/create/view#materialized-view)。
    - `'Live' = 3` — [ライブビュー](../../sql-reference/statements/create/view.md#live-view)。
- `view_query` ([String](../../sql-reference/data-types/string.md)) — ビューによって実行されたクエリ。
- `view_target` ([String](../../sql-reference/data-types/string.md)) — ビューのターゲットテーブルの名前。
- `read_rows` ([UInt64](/sql-reference/data-types/int-uint#integer-ranges)) — 読み取られた行の数。
- `read_bytes` ([UInt64](/sql-reference/data-types/int-uint#integer-ranges)) — 読み取られたバイトの数。
- `written_rows` ([UInt64](/sql-reference/data-types/int-uint#integer-ranges)) — 書き込まれた行の数。
- `written_bytes` ([UInt64](/sql-reference/data-types/int-uint#integer-ranges)) — 書き込まれたバイトの数。
- `peak_memory_usage` ([Int64](../../sql-reference/data-types/int-uint.md)) — このビューに関する割り当てられたメモリと解放されたメモリの最大差。
- `ProfileEvents` ([Map(String, UInt64)](../../sql-reference/data-types/array.md)) — 異なるメトリクスを測定するProfileEvents。これらの説明はテーブル [system.events](/operations/system-tables/events) で見つけることができます。
- `status` ([Enum8](../../sql-reference/data-types/enum.md)) — ビューの状態。値:
    - `'QueryStart' = 1` — ビューの実行の成功した開始。表示されるべきではありません。
    - `'QueryFinish' = 2` — ビューの実行の成功した終了。
    - `'ExceptionBeforeStart' = 3` — ビューの実行開始前の例外。
    - `'ExceptionWhileProcessing' = 4` — ビューの実行中の例外。
- `exception_code` ([Int32](../../sql-reference/data-types/int-uint.md)) — 例外のコード。
- `exception` ([String](../../sql-reference/data-types/string.md)) — 例外メッセージ。
- `stack_trace` ([String](../../sql-reference/data-types/string.md)) — [スタックトレース](https://en.wikipedia.org/wiki/Stack_trace)。クエリが成功裏に完了した場合は空の文字列。

**例**

クエリ:

```sql
SELECT * FROM system.query_views_log LIMIT 1 \G;
```

結果:

```text
Row 1:
──────
hostname:                clickhouse.eu-central1.internal
event_date:              2021-06-22
event_time:              2021-06-22 13:23:07
event_time_microseconds: 2021-06-22 13:23:07.738221
view_duration_ms:        0
initial_query_id:        c3a1ac02-9cad-479b-af54-9e9c0a7afd70
view_name:               default.matview_inner
view_uuid:               00000000-0000-0000-0000-000000000000
view_type:               Materialized
view_query:              SELECT * FROM default.table_b
view_target:             default.`.inner.matview_inner`
read_rows:               4
read_bytes:              64
written_rows:            2
written_bytes:           32
peak_memory_usage:       4196188
ProfileEvents:           {'FileOpen':2,'WriteBufferFromFileDescriptorWrite':2,'WriteBufferFromFileDescriptorWriteBytes':187,'IOBufferAllocs':3,'IOBufferAllocBytes':3145773,'FunctionExecute':3,'DiskWriteElapsedMicroseconds':13,'InsertedRows':2,'InsertedBytes':16,'SelectedRows':4,'SelectedBytes':48,'ContextLock':16,'RWLockAcquiredReadLocks':1,'RealTimeMicroseconds':698,'SoftPageFaults':4,'OSReadChars':463}
status:                  QueryFinish
exception_code:          0
exception:
stack_trace:
```

**関連情報**

- [system.query_log](/operations/system-tables/query_log) — クエリ実行に関する一般情報を含む `query_log` システムテーブルの説明。
- [system.query_thread_log](/operations/system-tables/query_thread_log) — このテーブルは各クエリ実行スレッドに関する情報を含みます。
