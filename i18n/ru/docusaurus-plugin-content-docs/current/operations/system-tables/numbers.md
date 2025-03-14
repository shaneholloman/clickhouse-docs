---
description: 'Системная таблица, содержащая одну колонку UInt64 с именем `number`, в которой находятся почти все натуральные числа, начиная с нуля.'
slug: /operations/system-tables/numbers
title: 'system.numbers'
keywords: ['системная таблица', 'числа']
---

Эта таблица содержит одну колонку UInt64 с именем `number`, в которой находятся почти все натуральные числа, начиная с нуля.

Вы можете использовать эту таблицу для тестов или если вам нужно выполнить перебор.

Чтения из этой таблицы не параллелизуются.

**Пример**

```sql
SELECT * FROM system.numbers LIMIT 10;
```

```response
┌─number─┐
│      0 │
│      1 │
│      2 │
│      3 │
│      4 │
│      5 │
│      6 │
│      7 │
│      8 │
│      9 │
└────────┘

10 строк в наборе. Время выполнения: 0.001 сек.
```

Вы также можете ограничить вывод с помощью предикатов.

```sql
SELECT * FROM system.numbers < 10;
```

```response
┌─number─┐
│      0 │
│      1 │
│      2 │
│      3 │
│      4 │
│      5 │
│      6 │
│      7 │
│      8 │
│      9 │
└────────┘

10 строк в наборе. Время выполнения: 0.001 сек.
```
