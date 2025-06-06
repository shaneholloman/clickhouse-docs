---
'title': '存储效率 - 时间序列'
'sidebar_label': '存储效率'
'description': '提高时间序列存储效率'
'slug': '/use-cases/time-series/storage-efficiency'
'keywords':
- 'time-series'
'show_related_blogs': true
---


# 时间序列存储效率

在探索如何查询我们的维基百科统计数据集后，接下来让我们关注在 ClickHouse 中优化其存储效率。 
本节演示了在保持查询性能的同时减少存储需求的实际技术。

## 类型优化 {#time-series-type-optimization}

优化存储效率的一般方法是使用最优的数据类型。 
让我们来看一下 `project` 和 `subproject` 列。这些列的数据类型为 String，但具有相对较少的唯一值：

```sql
SELECT
    uniq(project),
    uniq(subproject)
FROM wikistat;
```

```text
┌─uniq(project)─┬─uniq(subproject)─┐
│          1332 │              130 │
└───────────────┴──────────────────┘
```

这意味着我们可以使用 LowCardinality() 数据类型，它使用基于字典的编码。这使得 ClickHouse 存储内部值 ID 而不是原始字符串值，从而节省了大量空间：

```sql
ALTER TABLE wikistat
MODIFY COLUMN `project` LowCardinality(String),
MODIFY COLUMN `subproject` LowCardinality(String)
```

我们还对 hits 列使用了 UInt64 类型，该类型占用 8 字节，但其最大值相对较小：

```sql
SELECT max(hits)
FROM wikistat;
```

```text
┌─max(hits)─┐
│    449017 │
└───────────┘
```

考虑到这个值，我们可以使用 UInt32 替代，它只占用 4 字节，并允许我们存储最大值约为 ~4b：

```sql
ALTER TABLE wikistat
MODIFY COLUMN `hits` UInt32;
```

这将使该列在内存中的大小至少减少 2 倍。 请注意，由于压缩，磁盘上的大小将保持不变。 但是要小心，选择不太小的数据类型！

## 专用编解码器 {#time-series-specialized-codecs}

当我们处理序列数据时，例如时间序列，我们可以通过使用特殊的编解码器进一步提高存储效率。 
其一般思想是存储值之间的变化，而不是绝对值本身，这在处理缓慢变化的数据时所需空间大大减少：

```sql
ALTER TABLE wikistat
MODIFY COLUMN `time` CODEC(Delta, ZSTD);
```

我们对时间列使用了 Delta 编解码器，这是适合时间序列数据的良好选择。 

正确的排序键也可以节省磁盘空间。 
由于我们通常希望按照路径进行过滤，因此我们将 `path` 添加到排序键中。 
这需要重新创建表。

下面我们可以看到初始表和优化表的 `CREATE` 命令：

```sql
CREATE TABLE wikistat
(
    `time` DateTime,
    `project` String,
    `subproject` String,
    `path` String,
    `hits` UInt64
)
ENGINE = MergeTree
ORDER BY (time);
```

```sql
CREATE TABLE optimized_wikistat
(
    `time` DateTime CODEC(Delta(4), ZSTD(1)),
    `project` LowCardinality(String),
    `subproject` LowCardinality(String),
    `path` String,
    `hits` UInt32
)
ENGINE = MergeTree
ORDER BY (path, time);
```

让我们看一下每个表中的数据占用的空间量：

```sql
SELECT
    table,
    formatReadableSize(sum(data_uncompressed_bytes)) AS uncompressed,
    formatReadableSize(sum(data_compressed_bytes)) AS compressed,
    count() AS parts
FROM system.parts
WHERE table LIKE '%wikistat%'
GROUP BY ALL;
```

```text
┌─table──────────────┬─uncompressed─┬─compressed─┬─parts─┐
│ wikistat           │ 35.28 GiB    │ 12.03 GiB  │     1 │
│ optimized_wikistat │ 30.31 GiB    │ 2.84 GiB   │     1 │
└────────────────────┴──────────────┴────────────┴───────┘
```

优化后的表在压缩形式下占用的空间仅略超过原来的 4 倍。
