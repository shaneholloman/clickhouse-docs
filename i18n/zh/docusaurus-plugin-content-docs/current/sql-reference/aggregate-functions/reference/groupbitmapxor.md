---
slug: /sql-reference/aggregate-functions/reference/groupbitmapxor
sidebar_position: 151
title: groupBitmapXor
description: "计算位图列的异或，并返回 UInt64 类型的基数，如果与后缀 -State 一起使用，则返回一个 [位图对象](../../../sql-reference/functions/bitmap-functions.md)"
---


# groupBitmapXor

`groupBitmapXor` 计算位图列的异或，并返回 UInt64 类型的基数，如果与后缀 -State 一起使用，则返回一个 [位图对象](../../../sql-reference/functions/bitmap-functions.md)。

``` sql
groupBitmapOr(expr)
```

**参数**

`expr` – 结果为 `AggregateFunction(groupBitmap, UInt*)` 类型的表达式。

**返回值**

`UInt64` 类型的值。

**示例**

``` sql
DROP TABLE IF EXISTS bitmap_column_expr_test2;
CREATE TABLE bitmap_column_expr_test2
(
    tag_id String,
    z AggregateFunction(groupBitmap, UInt32)
)
ENGINE = MergeTree
ORDER BY tag_id;

INSERT INTO bitmap_column_expr_test2 VALUES ('tag1', bitmapBuild(cast([1,2,3,4,5,6,7,8,9,10] as Array(UInt32))));
INSERT INTO bitmap_column_expr_test2 VALUES ('tag2', bitmapBuild(cast([6,7,8,9,10,11,12,13,14,15] as Array(UInt32))));
INSERT INTO bitmap_column_expr_test2 VALUES ('tag3', bitmapBuild(cast([2,4,6,8,10,12] as Array(UInt32))));

SELECT groupBitmapXor(z) FROM bitmap_column_expr_test2 WHERE like(tag_id, 'tag%');
┌─groupBitmapXor(z)─┐
│              10   │
└───────────────────┘

SELECT arraySort(bitmapToArray(groupBitmapXorState(z))) FROM bitmap_column_expr_test2 WHERE like(tag_id, 'tag%');
┌─arraySort(bitmapToArray(groupBitmapXorState(z)))─┐
│ [1,3,5,6,8,10,11,13,14,15]                       │
└──────────────────────────────────────────────────┘
```
