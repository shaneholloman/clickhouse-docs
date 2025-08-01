---
description: 'nth_value ウィンドウ関数の文書化'
sidebar_label: 'nth_value'
sidebar_position: 5
slug: '/sql-reference/window-functions/nth_value'
title: 'nth_value'
---




# nth_value

nth_value は、順序付けられたフレーム内の nth 行（オフセット）に対して評価された最初の非 NULL 値を返します。

**構文**

```sql
nth_value (x, offset)
  OVER ([[PARTITION BY grouping_column] [ORDER BY sorting_column] 
        [ROWS または RANGE expression_to_bound_rows_withing_the_group]] | [window_name])
FROM table_name
WINDOW window_name as ([[PARTITION BY grouping_column] [ORDER BY sorting_column])
```

ウィンドウ関数の構文の詳細については、[ウィンドウ関数 - 構文](./index.md/#syntax)をご覧ください。

**パラメーター**

- `x` — カラム名。
- `offset` — 現在の行に対して評価する nth 行。

**戻り値**

- 順序付けられたフレーム内の nth 行（オフセット）に対して評価された最初の非 NULL 値。

**例**

この例では、`nth-value` 関数を使用して、プレミアリーグのサッカー選手の架空の給与データセットから三番目に高い給与を見つけます。

クエリ:

```sql
DROP TABLE IF EXISTS salaries;
CREATE TABLE salaries
(
    `team` String,
    `player` String,
    `salary` UInt32,
    `position` String
)
Engine = Memory;

INSERT INTO salaries FORMAT Values
    ('Port Elizabeth Barbarians', 'Gary Chen', 195000, 'F'),
    ('New Coreystad Archdukes', 'Charles Juarez', 190000, 'F'),
    ('Port Elizabeth Barbarians', 'Michael Stanley', 100000, 'D'),
    ('New Coreystad Archdukes', 'Scott Harrison', 180000, 'D'),
    ('Port Elizabeth Barbarians', 'Robert George', 195000, 'M'),
    ('South Hampton Seagulls', 'Douglas Benson', 150000, 'M'),
    ('South Hampton Seagulls', 'James Henderson', 140000, 'M');
```

```sql
SELECT player, salary, nth_value(player,3) OVER(ORDER BY salary DESC) AS third_highest_salary FROM salaries;
```

結果:

```response
   ┌─player──────────┬─salary─┬─third_highest_salary─┐
1. │ Gary Chen       │ 195000 │                      │
2. │ Robert George   │ 195000 │                      │
3. │ Charles Juarez  │ 190000 │ Charles Juarez       │
4. │ Scott Harrison  │ 180000 │ Charles Juarez       │
5. │ Douglas Benson  │ 150000 │ Charles Juarez       │
6. │ James Henderson │ 140000 │ Charles Juarez       │
7. │ Michael Stanley │ 100000 │ Charles Juarez       │
   └─────────────────┴────────┴──────────────────────┘
```
