---
slug: '/examples/aggregate-function-combinators/avgMergeState'
title: 'avgMergeState'
description: 'avgMergeState combinator の使用例'
keywords:
- 'avg'
- 'MergeState'
- 'combinator'
- 'examples'
- 'avgMergeState'
sidebar_label: 'avgMergeState'
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';



# avgMergeState {#avgMergeState}

## 説明 {#description}

[`MergeState`](/sql-reference/aggregate-functions/combinators#-state) コンビネータは
[`avg`](/sql-reference/aggregate-functions/reference/avg) 関数に適用され、`AverageFunction(avg, T)` 型の部分集約状態を結合し、新しい中間集約状態を返します。

## 使用例 {#example-usage}

`MergeState` コンビネータは、事前に集約された状態を結合し、それをさらなる処理のために状態として保持したいマルチレベルの集約シナリオに特に役立ちます。ここでは、個々のサーバーのパフォーマンス指標を複数のレベルにわたる階層的集約に変換する例を見てみましょう：サーバーレベル → リージョンレベル → データセンターレベル。

まず、原データを格納するテーブルを作成します。

```sql
CREATE TABLE raw_server_metrics
(
    timestamp DateTime DEFAULT now(),
    server_id UInt32,
    region String,
    datacenter String,
    response_time_ms UInt32
)
ENGINE = MergeTree()
ORDER BY (region, server_id, timestamp);
```

サーバーレベルの集約ターゲットテーブルを作成し、そこに挿入トリガーとして機能するインクリメンタル Materialized View を定義します。

```sql
CREATE TABLE server_performance
(
    server_id UInt32,
    region String,
    datacenter String,
    avg_response_time AggregateFunction(avg, UInt32)
)
ENGINE = AggregatingMergeTree()
ORDER BY (region, server_id);

CREATE MATERIALIZED VIEW server_performance_mv
TO server_performance
AS SELECT
    server_id,
    region,
    datacenter,
    avgState(response_time_ms) AS avg_response_time
FROM raw_server_metrics
GROUP BY server_id, region, datacenter;
```

地域およびデータセンターレベルについても同様に行います。

```sql
CREATE TABLE region_performance
(
    region String,
    datacenter String,
    avg_response_time AggregateFunction(avg, UInt32)
)
ENGINE = AggregatingMergeTree()
ORDER BY (datacenter, region);

CREATE MATERIALIZED VIEW region_performance_mv
TO region_performance
AS SELECT
    region,
    datacenter,
    avgMergeState(avg_response_time) AS avg_response_time
FROM server_performance
GROUP BY region, datacenter;

-- データセンターレベルのテーブルと Materialized View

CREATE TABLE datacenter_performance
(
    datacenter String,
    avg_response_time AggregateFunction(avg, UInt32)
)
ENGINE = AggregatingMergeTree()
ORDER BY datacenter;

CREATE MATERIALIZED VIEW datacenter_performance_mv
TO datacenter_performance
AS SELECT
      datacenter,
      avgMergeState(avg_response_time) AS avg_response_time
FROM region_performance
GROUP BY datacenter;
```

次に、ソーステーブルにサンプルの生データを挿入します。

```sql
INSERT INTO raw_server_metrics (timestamp, server_id, region, datacenter, response_time_ms) VALUES
    (now(), 101, 'us-east', 'dc1', 120),
    (now(), 101, 'us-east', 'dc1', 130),
    (now(), 102, 'us-east', 'dc1', 115),
    (now(), 201, 'us-west', 'dc1', 95),
    (now(), 202, 'us-west', 'dc1', 105),
    (now(), 301, 'eu-central', 'dc2', 145),
    (now(), 302, 'eu-central', 'dc2', 155);
```

各レベルに対して3つのクエリを書きます。

<Tabs>
  <TabItem value="Service level" label="Service level" default>
```sql
SELECT
    server_id,
    region,
    avgMerge(avg_response_time) AS avg_response_ms
FROM server_performance
GROUP BY server_id, region
ORDER BY region, server_id;
```
```response
┌─server_id─┬─region─────┬─avg_response_ms─┐
│       301 │ eu-central │             145 │
│       302 │ eu-central │             155 │
│       101 │ us-east    │             125 │
│       102 │ us-east    │             115 │
│       201 │ us-west    │              95 │
│       202 │ us-west    │             105 │
└───────────┴────────────┴─────────────────┘
```
  </TabItem>
  <TabItem value="Regional level" label="Regional level">
```sql
SELECT
    region,
    datacenter,
    avgMerge(avg_response_time) AS avg_response_ms
FROM region_performance
GROUP BY region, datacenter
ORDER BY datacenter, region;
```
```response
┌─region─────┬─datacenter─┬────avg_response_ms─┐
│ us-east    │ dc1        │ 121.66666666666667 │
│ us-west    │ dc1        │                100 │
│ eu-central │ dc2        │                150 │
└────────────┴────────────┴────────────────────┘
```
  </TabItem>
  <TabItem value="Datacenter level" label="Datacenter level">
```sql
SELECT
    datacenter,
    avgMerge(avg_response_time) AS avg_response_ms
FROM datacenter_performance
GROUP BY datacenter
ORDER BY datacenter;
```
```response
┌─datacenter─┬─avg_response_ms─┐
│ dc1        │             113 │
│ dc2        │             150 │
└────────────┴─────────────────┘
```
  </TabItem>
</Tabs>

さらにデータを挿入します。

```sql
INSERT INTO raw_server_metrics (timestamp, server_id, region, datacenter, response_time_ms) VALUES
    (now(), 101, 'us-east', 'dc1', 140),
    (now(), 201, 'us-west', 'dc1', 85),
    (now(), 301, 'eu-central', 'dc2', 135);
```

データセンターレベルのパフォーマンスを再度確認します。集約チェーン全体が自動的に更新される様子に注意してください。

```sql
SELECT
    datacenter,
    avgMerge(avg_response_time) AS avg_response_ms
FROM datacenter_performance
GROUP BY datacenter
ORDER BY datacenter;
```

```response
┌─datacenter─┬────avg_response_ms─┐
│ dc1        │ 112.85714285714286 │
│ dc2        │                145 │
└────────────┴────────────────────┘
```

## 参考 {#see-also}
- [`avg`](/sql-reference/aggregate-functions/reference/avg)
- [`AggregateFunction`](/sql-reference/data-types/aggregatefunction)
- [`Merge`](/sql-reference/aggregate-functions/combinators#-merge)
- [`MergeState`](/sql-reference/aggregate-functions/combinators#-mergestate)
