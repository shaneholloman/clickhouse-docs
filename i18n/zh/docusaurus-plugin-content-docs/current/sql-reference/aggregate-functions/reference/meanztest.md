---
slug: /sql-reference/aggregate-functions/reference/meanztest
sidebar_position: 166
sidebar_label: meanZTest
title: 'meanZTest'
description: '对两个总体样本应用均值z检验。'
---


# meanZTest

对两个总体样本应用均值z检验。

**语法**

``` sql
meanZTest(population_variance_x, population_variance_y, confidence_level)(sample_data, sample_index)
```

两个样本的值位于 `sample_data` 列中。如果 `sample_index` 等于 0，则该行的值属于第一个总体的样本。否则，它属于第二个总体的样本。
原假设是两个总体的均值相等。假定服从正态分布。总体可能具有不等的方差且方差是已知的。

**参数**

- `sample_data` — 样本数据。 [整数](../../../sql-reference/data-types/int-uint.md), [浮点数](../../../sql-reference/data-types/float.md) 或 [小数](../../../sql-reference/data-types/decimal.md)。
- `sample_index` — 样本索引。 [整数](../../../sql-reference/data-types/int-uint.md)。

**参数说明**

- `population_variance_x` — 总体x的方差。 [浮点数](../../../sql-reference/data-types/float.md)。
- `population_variance_y` — 总体y的方差。 [浮点数](../../../sql-reference/data-types/float.md)。
- `confidence_level` — 用于计算置信区间的置信水平。 [浮点数](../../../sql-reference/data-types/float.md)。

**返回值**

[元组](../../../sql-reference/data-types/tuple.md)，包含四个元素：

- 计算得到的t统计量。 [Float64](../../../sql-reference/data-types/float.md)。
- 计算得到的p值。 [Float64](../../../sql-reference/data-types/float.md)。
- 计算得到的置信区间低值。 [Float64](../../../sql-reference/data-types/float.md)。
- 计算得到的置信区间高值。 [Float64](../../../sql-reference/data-types/float.md)。

**示例**

输入表：

``` text
┌─sample_data─┬─sample_index─┐
│        20.3 │            0 │
│        21.9 │            0 │
│        22.1 │            0 │
│        18.9 │            1 │
│          19 │            1 │
│        20.3 │            1 │
└─────────────┴──────────────┘
```

查询：

``` sql
SELECT meanZTest(0.7, 0.45, 0.95)(sample_data, sample_index) FROM mean_ztest
```

结果：

``` text
┌─meanZTest(0.7, 0.45, 0.95)(sample_data, sample_index)────────────────────────────┐
│ (3.2841296025548123,0.0010229786769086013,0.8198428246768334,3.2468238419898365) │
└──────────────────────────────────────────────────────────────────────────────────┘
```
