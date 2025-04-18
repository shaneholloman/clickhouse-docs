---
description: 'Вычисляет энтропию Шеннона для колонки значений.'
sidebar_position: 131
slug: /sql-reference/aggregate-functions/reference/entropy
title: 'энтропия'
---


# энтропия

Вычисляет [энтропию Шеннона](https://en.wikipedia.org/wiki/Entropy_(information_theory)) для колонки значений.

**Синтаксис**

```sql
entropy(val)
```

**Аргументы**

- `val` — Колонка значений любого типа.

**Возвращаемое значение**

- Этропия Шеннона.

Тип: [Float64](../../../sql-reference/data-types/float.md).

**Пример**

Запрос:

```sql
CREATE TABLE entropy (`vals` UInt32,`strings` String) ENGINE = Memory;

INSERT INTO entropy VALUES (1, 'A'), (1, 'A'), (1,'A'), (1,'A'), (2,'B'), (2,'B'), (2,'C'), (2,'D');

SELECT entropy(vals), entropy(strings) FROM entropy;
```

Результат:

```text
┌─entropy(vals)─┬─entropy(strings)─┐
│             1 │             1.75 │
└───────────────┴──────────────────┘
```
