---
description: 'Возвращает максимум вычисленного экспоненциально сглаженного скользящего среднего
  по индексу `t` во времени по сравнению с `t-1`. '
sidebar_position: 135
slug: /sql-reference/aggregate-functions/reference/exponentialTimeDecayedMax
title: 'exponentialTimeDecayedMax'
---

## exponentialTimeDecayedMax {#exponentialtimedecayedmax}

Возвращает максимум вычисленного экспоненциально сглаженного скользящего среднего по индексу `t` во времени по сравнению с `t-1`. 

**Синтаксис**

```sql
exponentialTimeDecayedMax(x)(value, timeunit)
```

**Аргументы**

- `value` — Значение. [Целое число](../../../sql-reference/data-types/int-uint.md), [Float](../../../sql-reference/data-types/float.md) или [Decimal](../../../sql-reference/data-types/decimal.md).
- `timeunit` — Единица времени. [Целое число](../../../sql-reference/data-types/int-uint.md), [Float](../../../sql-reference/data-types/float.md) или [Decimal](../../../sql-reference/data-types/decimal.md), [DateTime](../../data-types/datetime.md), [DateTime64](../../data-types/datetime64.md).

**Параметры**

- `x` — Период полураспада. [Целое число](../../../sql-reference/data-types/int-uint.md), [Float](../../../sql-reference/data-types/float.md) или [Decimal](../../../sql-reference/data-types/decimal.md).

**Возвращаемые значения**

- Возвращает максимум экспоненциально сглаженного взвешенного скользящего среднего в `t` и `t-1`. [Float64](../../data-types/float.md).

**Пример**

Запрос:

```sql
SELECT
    value,
    time,
    round(exp_smooth, 3),
    bar(exp_smooth, 0, 5, 50) AS bar
FROM
    (
    SELECT
    (number = 0) OR (number >= 25) AS value,
    number AS time,
    exponentialTimeDecayedMax(10)(value, time) OVER (ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS exp_smooth
    FROM numbers(50)
    );
```

Результат:

```response
    ┌─value─┬─time─┬─round(exp_smooth, 3)─┬─bar────────┐
 1. │     1 │    0 │                    1 │ ██████████ │
 2. │     0 │    1 │                0.905 │ █████████  │
 3. │     0 │    2 │                0.819 │ ████████▏  │
 4. │     0 │    3 │                0.741 │ ███████▍   │
 5. │     0 │    4 │                 0.67 │ ██████▋    │
 6. │     0 │    5 │                0.607 │ ██████     │
 7. │     0 │    6 │                0.549 │ █████▍     │
 8. │     0 │    7 │                0.497 │ ████▉      │
 9. │     0 │    8 │                0.449 │ ████▍      │
10. │     0 │    9 │                0.407 │ ████       │
11. │     0 │   10 │                0.368 │ ███▋       │
12. │     0 │   11 │                0.333 │ ███▎       │
13. │     0 │   12 │                0.301 │ ███        │
14. │     0 │   13 │                0.273 │ ██▋        │
15. │     0 │   14 │                0.247 │ ██▍        │
16. │     0 │   15 │                0.223 │ ██▏        │
17. │     0 │   16 │                0.202 │ ██         │
18. │     0 │   17 │                0.183 │ █▊         │
19. │     0 │   18 │                0.165 │ █▋         │
20. │     0 │   19 │                 0.15 │ █▍         │
21. │     0 │   20 │                0.135 │ █▎         │
22. │     0 │   21 │                0.122 │ █▏         │
23. │     0 │   22 │                0.111 │ █          │
24. │     0 │   23 │                  0.1 │ █          │
25. │     0 │   24 │                0.091 │ ▉          │
26. │     1 │   25 │                    1 │ ██████████ │
27. │     1 │   26 │                    1 │ ██████████ │
28. │     1 │   27 │                    1 │ ██████████ │
29. │     1 │   28 │                    1 │ ██████████ │
30. │     1 │   29 │                    1 │ ██████████ │
31. │     1 │   30 │                    1 │ ██████████ │
32. │     1 │   31 │                    1 │ ██████████ │
33. │     1 │   32 │                    1 │ ██████████ │
34. │     1 │   33 │                    1 │ ██████████ │
35. │     1 │   34 │                    1 │ ██████████ │
36. │     1 │   35 │                    1 │ ██████████ │
37. │     1 │   36 │                    1 │ ██████████ │
38. │     1 │   37 │                    1 │ ██████████ │
39. │     1 │   38 │                    1 │ ██████████ │
40. │     1 │   39 │                    1 │ ██████████ │
41. │     1 │   40 │                    1 │ ██████████ │
42. │     1 │   41 │                    1 │ ██████████ │
43. │     1 │   42 │                    1 │ ██████████ │
44. │     1 │   43 │                    1 │ ██████████ │
45. │     1 │   44 │                    1 │ ██████████ │
46. │     1 │   45 │                    1 │ ██████████ │
47. │     1 │   46 │                    1 │ ██████████ │
48. │     1 │   47 │                    1 │ ██████████ │
49. │     1 │   48 │                    1 │ ██████████ │
50. │     1 │   49 │                    1 │ ██████████ │
    └───────┴──────┴──────────────────────┴────────────┘
```
