---
slug: /sql-reference/aggregate-functions/reference/avg
sidebar_position: 112
title: 'avg'
description: '计算算术平均值。'
---


# avg

计算算术平均值。

**语法**

``` sql
avg(x)
```

**参数**

- `x` — 输入值，必须是 [Integer](../../../sql-reference/data-types/int-uint.md)、[Float](../../../sql-reference/data-types/float.md) 或 [Decimal](../../../sql-reference/data-types/decimal.md)。

**返回值**

- 算术平均值，始终为 [Float64](../../../sql-reference/data-types/float.md) 类型。
- 如果输入参数 `x` 为空，则返回 `NaN`。

**示例**

查询：

``` sql
SELECT avg(x) FROM values('x Int8', 0, 1, 2, 3, 4, 5);
```

结果：

``` text
┌─avg(x)─┐
│    2.5 │
└────────┘
```

**示例**

创建一个临时表：

查询：

``` sql
CREATE table test (t UInt8) ENGINE = Memory;
```

获取算术平均值：

查询：

```sql
SELECT avg(t) FROM test;
```

结果：

``` text
┌─avg(x)─┐
│    nan │
└────────┘
```
