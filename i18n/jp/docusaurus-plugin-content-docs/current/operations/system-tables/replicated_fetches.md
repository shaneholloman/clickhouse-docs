---
description: 'System table containing information about currently running background
  fetches.'
keywords:
- 'system table'
- 'replicated_fetches'
slug: '/operations/system-tables/replicated_fetches'
title: 'system.replicated_fetches'
---

import SystemTableCloud from '@site/i18n/jp/docusaurus-plugin-content-docs/current/_snippets/_system_table_cloud.md';


# system.replicated_fetches

<SystemTableCloud/>

現在実行中のバックグラウンドフェッチに関する情報が含まれています。

カラム:

- `database` ([String](../../sql-reference/data-types/string.md)) — データベースの名前。

- `table` ([String](../../sql-reference/data-types/string.md)) — テーブルの名前。

- `elapsed` ([Float64](../../sql-reference/data-types/float.md)) — 現在の実行中のバックグラウンドフェッチの表示が開始されてから経過した時間（秒単位）。

- `progress` ([Float64](../../sql-reference/data-types/float.md)) — 完了した作業の進捗割合（0から1まで）。

- `result_part_name` ([String](../../sql-reference/data-types/string.md)) — 現在の実行中のバックグラウンドフェッチの結果として形成されるパーツの名前。

- `result_part_path` ([String](../../sql-reference/data-types/string.md)) — 現在の実行中のバックグラウンドフェッチの結果として形成されるパーツへの絶対パス。

- `partition_id` ([String](../../sql-reference/data-types/string.md)) — パーティションのID。

- `total_size_bytes_compressed` ([UInt64](../../sql-reference/data-types/int-uint.md)) — 結果パーツ内の圧縮データの総サイズ（バイト単位）。

- `bytes_read_compressed` ([UInt64](../../sql-reference/data-types/int-uint.md)) — 結果パーツから読み取った圧縮バイト数。

- `source_replica_path` ([String](../../sql-reference/data-types/string.md)) — ソースレプリカへの絶対パス。

- `source_replica_hostname` ([String](../../sql-reference/data-types/string.md)) — ソースレプリカのホスト名。

- `source_replica_port` ([UInt16](../../sql-reference/data-types/int-uint.md)) — ソースレプリカのポート番号。

- `interserver_scheme` ([String](../../sql-reference/data-types/string.md)) — インターサーバースキームの名前。

- `URI` ([String](../../sql-reference/data-types/string.md)) — 一意のリソース識別子。

- `to_detached` ([UInt8](../../sql-reference/data-types/int-uint.md)) — 現在の実行中のバックグラウンドフェッチが`TO DETACHED`式を使用して行われているかどうかを示すフラグ。

- `thread_id` ([UInt64](../../sql-reference/data-types/int-uint.md)) — スレッド識別子。

**例**

```sql
SELECT * FROM system.replicated_fetches LIMIT 1 FORMAT Vertical;
```

```text
Row 1:
──────
database:                    default
table:                       t
elapsed:                     7.243039876
progress:                    0.41832135995612835
result_part_name:            all_0_0_0
result_part_path:            /var/lib/clickhouse/store/700/70080a04-b2de-4adf-9fa5-9ea210e81766/all_0_0_0/
partition_id:                all
total_size_bytes_compressed: 1052783726
bytes_read_compressed:       440401920
source_replica_path:         /clickhouse/test/t/replicas/1
source_replica_hostname:     node1
source_replica_port:         9009
interserver_scheme:          http
URI:                         http://node1:9009/?endpoint=DataPartsExchange%3A%2Fclickhouse%2Ftest%2Ft%2Freplicas%2F1&part=all_0_0_0&client_protocol_version=4&compress=false
to_detached:                 0
thread_id:                   54
```

**関連情報**

- [ReplicatedMergeTree テーブルの管理](../../sql-reference/statements/system.md/#managing-replicatedmergetree-tables)
