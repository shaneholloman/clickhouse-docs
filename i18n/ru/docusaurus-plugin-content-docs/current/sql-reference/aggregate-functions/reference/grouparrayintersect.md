---
description: 'Возвращает пересечение заданных массивов (Возвращает все элементы массивов, 
  которые присутствуют во всех данных массивах).'
sidebar_position: 141
slug: /sql-reference/aggregate-functions/reference/grouparrayintersect
title: 'groupArrayIntersect'
---


# groupArrayIntersect

Возвращает пересечение заданных массивов (Возвращает все элементы массивов, которые присутствуют во всех данных массивах).

**Синтаксис**

```sql
groupArrayIntersect(x)
```

**Аргументы**

- `x` — Аргумент (имя колонки или выражение).

**Возвращаемые значения**

- Массив, содержащий элементы, которые присутствуют во всех массивах.

Тип: [Array](../../data-types/array.md).

**Примеры**

Рассмотрим таблицу `numbers`:

```text
┌─a──────────────┐
│ [1,2,4]        │
│ [1,5,2,8,-1,0] │
│ [1,5,7,5,8,2]  │
└────────────────┘
```

Запрос с именем колонки в качестве аргумента:

```sql
SELECT groupArrayIntersect(a) as intersection FROM numbers;
```

Результат:

```text
┌─intersection──────┐
│ [1, 2]            │
└───────────────────┘
```
