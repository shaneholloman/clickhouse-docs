---
description: 'Вычисляет скользящую сумму входных значений.'
sidebar_position: 144
slug: /sql-reference/aggregate-functions/reference/grouparraymovingsum
title: 'groupArrayMovingSum'
---


# groupArrayMovingSum

Вычисляет скользящую сумму входных значений.

```sql
groupArrayMovingSum(numbers_for_summing)
groupArrayMovingSum(window_size)(numbers_for_summing)
```

Функция может принимать размер окна в качестве параметра. Если не указано, функция берёт размер окна, равный количеству строк в колонке.

**Аргументы**

- `numbers_for_summing` — [Выражение](/sql-reference/syntax#expressions), приводящее к значению числового типа данных.
- `window_size` — Размер окна вычисления.

**Возвращаемые значения**

- Массив того же размера и типа, что и входные данные.

**Пример**

Пример таблицы:

```sql
CREATE TABLE t
(
    `int` UInt8,
    `float` Float32,
    `dec` Decimal32(2)
)
ENGINE = TinyLog
```

```text
┌─int─┬─float─┬──dec─┐
│   1 │   1.1 │ 1.10 │
│   2 │   2.2 │ 2.20 │
│   4 │   4.4 │ 4.40 │
│   7 │  7.77 │ 7.77 │
└─────┴───────┴──────┘
```

Запросы:

```sql
SELECT
    groupArrayMovingSum(int) AS I,
    groupArrayMovingSum(float) AS F,
    groupArrayMovingSum(dec) AS D
FROM t
```

```text
┌─I──────────┬─F───────────────────────────────┬─D──────────────────────┐
│ [1,3,7,14] │ [1.1,3.3000002,7.7000003,15.47] │ [1.10,3.30,7.70,15.47] │
└────────────┴─────────────────────────────────┴────────────────────────┘
```

```sql
SELECT
    groupArrayMovingSum(2)(int) AS I,
    groupArrayMovingSum(2)(float) AS F,
    groupArrayMovingSum(2)(dec) AS D
FROM t
```

```text
┌─I──────────┬─F───────────────────────────────┬─D──────────────────────┐
│ [1,3,6,11] │ [1.1,3.3000002,6.6000004,12.17] │ [1.10,3.30,6.60,12.17] │
└────────────┴─────────────────────────────────┴────────────────────────┘
```
