---
slug: /sql-reference/aggregate-functions/reference/intervalLengthSum
sidebar_position: 155
sidebar_label: intervalLengthSum
title: intervalLengthSum
description: "すべての範囲（数値軸上のセグメント）の和の合計を計算します。"
---

すべての範囲（数値軸上のセグメント）の和の合計を計算します。

**構文**

``` sql
intervalLengthSum(start, end)
```

**引数**

- `start` — インターバルの開始値。[Int32](/sql-reference/data-types/int-uint#integer-ranges)、[Int64](/sql-reference/data-types/int-uint#integer-ranges)、[UInt32](/sql-reference/data-types/int-uint#integer-ranges)、[UInt64](/sql-reference/data-types/int-uint#integer-ranges)、[Float32](/sql-reference/data-types/float)、[Float64](/sql-reference/data-types/float)、[DateTime](/sql-reference/data-types/datetime) または [Date](/sql-reference/data-types/date)。
- `end` — インターバルの終了値。[Int32](/sql-reference/data-types/int-uint#integer-ranges)、[Int64](/sql-reference/data-types/int-uint#integer-ranges)、[UInt32](/sql-reference/data-types/int-uint#integer-ranges)、[UInt64](/sql-reference/data-types/int-uint#integer-ranges)、[Float32](/sql-reference/data-types/float)、[Float64](/sql-reference/data-types/float)、[DateTime](/sql-reference/data-types/datetime) または [Date](/sql-reference/data-types/date)。

:::note
引数は同じデータ型でなければなりません。そうでない場合は例外がスローされます。
:::

**返される値**

- すべての範囲（数値軸上のセグメント）の和の合計。引数の型に応じて、返される値は [UInt64](/sql-reference/data-types/int-uint#integer-ranges) または [Float64](/sql-reference/data-types/float) 型とすることができます。

**例**

1. 入力テーブル:

``` text
┌─id─┬─start─┬─end─┐
│ a  │   1.1 │ 2.9 │
│ a  │   2.5 │ 3.2 │
│ a  │     4 │   5 │
└────┴───────┴─────┘
```

この例では、Float32 型の引数が使用されています。この関数は Float64 型の値を返します。

結果は、区間 `[1.1, 3.2]` の長さの合計（`[1.1, 2.9]` と `[2.5, 3.2]` の和）と `[4, 5]` です。

クエリ:

``` sql
SELECT id, intervalLengthSum(start, end), toTypeName(intervalLengthSum(start, end)) FROM fl_interval GROUP BY id ORDER BY id;
```

結果:

``` text
┌─id─┬─intervalLengthSum(start, end)─┬─toTypeName(intervalLengthSum(start, end))─┐
│ a  │                           3.1 │ Float64                                   │
└────┴───────────────────────────────┴───────────────────────────────────────────┘
```

2. 入力テーブル:

``` text
┌─id─┬───────────────start─┬─────────────────end─┐
│ a  │ 2020-01-01 01:12:30 │ 2020-01-01 02:10:10 │
│ a  │ 2020-01-01 02:05:30 │ 2020-01-01 02:50:31 │
│ a  │ 2020-01-01 03:11:22 │ 2020-01-01 03:23:31 │
└────┴─────────────────────┴─────────────────────┘
```

この例では、DateTime 型の引数が使用されています。この関数は秒単位の値を返します。

クエリ:

``` sql
SELECT id, intervalLengthSum(start, end), toTypeName(intervalLengthSum(start, end)) FROM dt_interval GROUP BY id ORDER BY id;
```

結果:

``` text
┌─id─┬─intervalLengthSum(start, end)─┬─toTypeName(intervalLengthSum(start, end))─┐
│ a  │                          6610 │ UInt64                                    │
└────┴───────────────────────────────┴───────────────────────────────────────────┘
```

3. 入力テーブル:

``` text
┌─id─┬──────start─┬────────end─┐
│ a  │ 2020-01-01 │ 2020-01-04 │
│ a  │ 2020-01-12 │ 2020-01-18 │
└────┴────────────┴────────────┘
```

この例では、Date 型の引数が使用されています。この関数は日数単位の値を返します。

クエリ:

``` sql
SELECT id, intervalLengthSum(start, end), toTypeName(intervalLengthSum(start, end)) FROM date_interval GROUP BY id ORDER BY id;
```

結果:

``` text
┌─id─┬─intervalLengthSum(start, end)─┬─toTypeName(intervalLengthSum(start, end))─┐
│ a  │                             9 │ UInt64                                    │
└────┴───────────────────────────────┴───────────────────────────────────────────┘
```
