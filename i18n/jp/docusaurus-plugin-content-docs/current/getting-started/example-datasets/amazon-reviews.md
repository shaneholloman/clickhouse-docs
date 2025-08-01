---
description: 'Over 150M customer reviews of Amazon products'
sidebar_label: 'Amazon customer reviews'
slug: '/getting-started/example-datasets/amazon-reviews'
title: 'Amazon Customer Review'
---



This dataset contains over 150M customer reviews of Amazon products. The data is in snappy-compressed Parquet files in AWS S3 that total 49GB in size (compressed). Let's walk through the steps to insert it into ClickHouse.

:::note
The queries below were executed on a **Production** instance of ClickHouse Cloud. For more information see
["Playground specifications"](/getting-started/playground#specifications).
:::

## データセットの読み込み {#loading-the-dataset}

1. ClickHouseにデータを挿入せずに、その場でクエリを実行できます。いくつかの行を取得して、どのようなものか見てみましょう：

```sql
SELECT *
FROM s3('https://datasets-documentation.s3.eu-west-3.amazonaws.com/amazon_reviews/amazon_reviews_2015.snappy.parquet')
LIMIT 3
```

行は次のようになります：

```response
Row 1:
──────
review_date:       16462
marketplace:       US
customer_id:       25444946 -- 25.44百万
review_id:         R146L9MMZYG0WA
product_id:        B00NV85102
product_parent:    908181913 -- 908.18百万
product_title:     XIKEZAN iPhone 6 Plus 5.5インチ防水ケース、衝撃防止、防塵、防雪フルボディスキンケース、ハンドストラップ＆ヘッドフォンアダプタ＆キックスタンド付き
product_category:  Wireless
star_rating:       4
helpful_votes:     0
total_votes:       0
vine:              false
verified_purchase: true
review_headline:   ケースは頑丈で、私が望む通りに保護します
review_body:       防水部分は過信しません（下のゴムシールは私の神経を使ったので外しました）。でも、このケースは頑丈で、私が望む通りに保護します。

Row 2:
──────
review_date:       16462
marketplace:       US
customer_id:       1974568 -- 1.97百万
review_id:         R2LXDXT293LG1T
product_id:        B00OTFZ23M
product_parent:    951208259 -- 951.21百万
product_title:     Season.C シカゴ・ブルズ マリリン・モンロー No.1 ハードバックケースカバー サムスンギャラクシーS5 i9600用
product_category:  Wireless
star_rating:       1
helpful_votes:     0
total_votes:       0
vine:              false
verified_purchase: true
review_headline:   一つ星
review_body:       ケースが電話に合わないので使えません。お金の無駄です！

Row 3:
──────
review_date:       16462
marketplace:       US
customer_id:       24803564 -- 24.80百万
review_id:         R7K9U5OEIRJWR
product_id:        B00LB8C4U4
product_parent:    524588109 -- 524.59百万
product_title:     iPhone 5s ケース、BUDDIBOX [Shield] 薄型デュアルレイヤー保護ケース キックスタンド付き Apple iPhone 5および5s用
product_category:  Wireless
star_rating:       4
helpful_votes:     0
total_votes:       0
vine:              false
verified_purchase: true
review_headline:   しかし全体的にこのケースはかなり頑丈で、電話を良く保護します
review_body:       最初は前面の部分を電話に固定するのが少し難しかったですが、全体的にこのケースはかなり頑丈で、電話を良く保護します。これは私が必要なことです。このケースを再度購入するつもりです。
```

2. データをClickHouseに保存するために、新しい `MergeTree` テーブル `amazon_reviews` を定義しましょう：

```sql
CREATE DATABASE amazon

CREATE TABLE amazon.amazon_reviews
(
    `review_date` Date,
    `marketplace` LowCardinality(String),
    `customer_id` UInt64,
    `review_id` String,
    `product_id` String,
    `product_parent` UInt64,
    `product_title` String,
    `product_category` LowCardinality(String),
    `star_rating` UInt8,
    `helpful_votes` UInt32,
    `total_votes` UInt32,
    `vine` Bool,
    `verified_purchase` Bool,
    `review_headline` String,
    `review_body` String,
    PROJECTION helpful_votes
    (
        SELECT *
        ORDER BY helpful_votes
    )
)
ENGINE = MergeTree
ORDER BY (review_date, product_category)
```

3. 次の `INSERT` コマンドは、`s3Cluster` テーブル関数を使用しており、これによりクラスタのすべてのノードを使用して複数のS3ファイルを同時に処理できます。また、`https://datasets-documentation.s3.eu-west-3.amazonaws.com/amazon_reviews/amazon_reviews_*.snappy.parquet` という名前で始まるファイルを挿入するためにワイルドカードも使用しています：

```sql
INSERT INTO amazon.amazon_reviews SELECT *
FROM s3Cluster('default', 
'https://datasets-documentation.s3.eu-west-3.amazonaws.com/amazon_reviews/amazon_reviews_*.snappy.parquet')
```

:::tip
ClickHouse Cloudでは、クラスタの名前は `default` です。 `default` をあなたのクラスタ名に変更するか、クラスタがない場合は `s3Cluster` の代わりに `s3` テーブル関数を使用してください。
:::

5. このクエリは時間がかからず、平均して毎秒約300,000行の速度で処理されます。5分ほどの間にすべての行が挿入されるはずです：

```sql runnable
SELECT formatReadableQuantity(count())
FROM amazon.amazon_reviews
```

6. データがどれだけのスペースを使用しているか見てみましょう：

```sql runnable
SELECT
    disk_name,
    formatReadableSize(sum(data_compressed_bytes) AS size) AS compressed,
    formatReadableSize(sum(data_uncompressed_bytes) AS usize) AS uncompressed,
    round(usize / size, 2) AS compr_rate,
    sum(rows) AS rows,
    count() AS part_count
FROM system.parts
WHERE (active = 1) AND (table = 'amazon_reviews')
GROUP BY disk_name
ORDER BY size DESC
```

元のデータは約70Gでしたが、ClickHouseでは約30Gのサイズを占めました。

## 例のクエリ {#example-queries}

7. いくつかのクエリを実行してみましょう。データセット内で最も役立つレビューのトップ10はこちらです：

```sql runnable
SELECT
    product_title,
    review_headline
FROM amazon.amazon_reviews
ORDER BY helpful_votes DESC
LIMIT 10
```

:::note
このクエリは、パフォーマンスを向上させるために [プロジェクション](/data-modeling/projections) を使用しています。
:::

8. Amazonでレビューが最も多いトップ10製品はこちらです：

```sql runnable
SELECT
    any(product_title),
    count()
FROM amazon.amazon_reviews
GROUP BY product_id
ORDER BY 2 DESC
LIMIT 10;
```

9. 各製品の月ごとの平均レビュー評価を示します（実際の [Amazonの就職面接質問](https://datalemur.com/questions/sql-avg-review-ratings)！）：

```sql runnable
SELECT
    toStartOfMonth(review_date) AS month,
    any(product_title),
    avg(star_rating) AS avg_stars
FROM amazon.amazon_reviews
GROUP BY
    month,
    product_id
ORDER BY
    month DESC,
    product_id ASC
LIMIT 20;
```

10. 各製品カテゴリごとの投票総数を示します。このクエリは、`product_category` が主キーに含まれているため高速です：

```sql runnable
SELECT
    sum(total_votes),
    product_category
FROM amazon.amazon_reviews
GROUP BY product_category
ORDER BY 1 DESC
```

11. レビュー内で最も頻繁に**"awful"**という単語が出現する製品を探します。これは大きな作業です - 1.51億以上の文字列を解析して単語を探す必要があります：

```sql runnable settings={'enable_parallel_replicas':1}
SELECT
    product_id,
    any(product_title),
    avg(star_rating),
    count() AS count
FROM amazon.amazon_reviews
WHERE position(review_body, 'awful') > 0
GROUP BY product_id
ORDER BY count DESC
LIMIT 50;
```

このような大量のデータに対するクエリ時間に注目してください。結果も読むのが楽しいです！

12. 同じクエリを再度実行できますが、今回はレビュー内で**awesome**を検索します：

```sql runnable settings={'enable_parallel_replicas':1}
SELECT 
    product_id,
    any(product_title),
    avg(star_rating),
    count() AS count
FROM amazon.amazon_reviews
WHERE position(review_body, 'awesome') > 0
GROUP BY product_id
ORDER BY count DESC
LIMIT 50;
```
