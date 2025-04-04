---
slug: /sql-reference/aggregate-functions/reference/groupbitor
sidebar_position: 152
title: 'groupBitOr'
description: 'Применяет побитовую операцию `OR` к серии чисел.'
---


# groupBitOr

Применяет побитовую операцию `OR` к серии чисел.

``` sql
groupBitOr(expr)
```

**Аргументы**

`expr` – Выражение, которое дает результат типа `UInt*` или `Int*`.

**Возвращаемое значение**

Значение типа `UInt*` или `Int*`.

**Пример**

Тестовые данные:

``` text
binary     decimal
00101100 = 44
00011100 = 28
00001101 = 13
01010101 = 85
```

Запрос:

``` sql
SELECT groupBitOr(num) FROM t
```

Где `num` – это колонка с тестовыми данными.

Результат:

``` text
binary     decimal
01111101 = 125
```
