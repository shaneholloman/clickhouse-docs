---
description: 'Kafkaコンシューマに関する情報を含むシステムテーブル。'
keywords:
- 'system table'
- 'kafka_consumers'
slug: '/operations/system-tables/kafka_consumers'
title: 'system.kafka_consumers'
---

import SystemTableCloud from '@site/i18n/jp/docusaurus-plugin-content-docs/current/_snippets/_system_table_cloud.md';

<SystemTableCloud/>

Kafkaコンシューマに関する情報を含みます。
[Kafkaテーブルエンジン](../../engines/table-engines/integrations/kafka)（ネイティブClickHouse統合）に適用されます。

カラム：

- `database` (String) - Kafkaエンジンを持つテーブルのデータベース。
- `table` (String) - Kafkaエンジンを持つテーブルの名前。
- `consumer_id` (String) - Kafkaコンシューマ識別子。テーブルは多くのコンシューマを持つことができる点に注意してください。これは`kafka_num_consumers`パラメータで指定されます。
- `assignments.topic` (Array(String)) - Kafkaトピック。
- `assignments.partition_id` (Array(Int32)) - KafkaパーティションID。1つのパーティションには1つのコンシューマのみが割り当てられる点に注意してください。
- `assignments.current_offset` (Array(Int64)) - 現在のオフセット。
- `exceptions.time` (Array(DateTime)) - 最も最近生成された10件の例外のタイムスタンプ。
- `exceptions.text` (Array(String)) - 最も最近の10件の例外のテキスト。
- `last_poll_time` (DateTime) - 最も最近のポーリングのタイムスタンプ。
- `num_messages_read` (UInt64) - コンシューマが読み取ったメッセージの数。
- `last_commit_time` (DateTime) - 最も最近のコミットのタイムスタンプ。
- `num_commits` (UInt64) - コンシューマのコミット総数。
- `last_rebalance_time` (DateTime) - 最も最近のKafkaのリバランスのタイムスタンプ。
- `num_rebalance_revocations` (UInt64) - コンシューマからパーティションの権限が剥奪された回数。
- `num_rebalance_assignments` (UInt64) - コンシューマがKafkaクラスターに割り当てられた回数。
- `is_currently_used` (UInt8) - コンシューマが使用中
- `last_used` (UInt64) - このコンシューマが最後に使用された時間、マイクロ秒のUnix時間
- `rdkafka_stat` (String) - ライブラリ内部の統計。詳細はhttps://github.com/ClickHouse/librdkafka/blob/master/STATISTICS.mdを参照してください。`statistics_interval_ms`を0に設定すると無効化され、デフォルトは3000（3秒ごと）です。

例：

```sql
SELECT *
FROM system.kafka_consumers
FORMAT Vertical
```

```text
Row 1:
──────
database:                   test
table:                      kafka
consumer_id:                ClickHouse-instance-test-kafka-1caddc7f-f917-4bb1-ac55-e28bd103a4a0
assignments.topic:          ['system_kafka_cons']
assignments.partition_id:   [0]
assignments.current_offset: [18446744073709550615]
exceptions.time:            []
exceptions.text:            []
last_poll_time:             2006-11-09 18:47:47
num_messages_read:          4
last_commit_time:           2006-11-10 04:39:40
num_commits:                1
last_rebalance_time:        1970-01-01 00:00:00
num_rebalance_revocations:  0
num_rebalance_assignments:  1
is_currently_used:          1
rdkafka_stat:               {...}

```
