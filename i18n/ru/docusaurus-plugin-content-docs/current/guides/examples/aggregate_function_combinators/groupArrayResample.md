---
slug: '/examples/aggregate-function-combinators/groupArrayResample'
title: 'groupArrayResample'
description: 'Пример использования комбинатора Resample с groupArray'
keywords: ['groupArray', 'Resample', 'combinator', 'examples', 'groupArrayResample']
sidebar_label: 'groupArrayResample'
---


# groupArrayResample {#grouparrayresample}

## Description {#description}

Комбинатор [`Resample`](/sql-reference/aggregate-functions/combinators#-resample) 
может быть применен к агрегатной функции [`groupArray`](/sql-reference/aggregate-functions/reference/sum) для
разделения диапазона указанной ключевой колонки на фиксированное количество интервалов (`N`) 
и построения результирующего массива, выбирая одно представительское значение 
(соответствующее минимальному ключу) из точек данных, попадающих в каждый интервал.
Он создает уменьшенную выборку данных вместо того, чтобы собирать все значения.

## Example Usage {#example-usage}

Рассмотрим пример. Мы создадим таблицу, которая содержит `name`, `age` и
`wage` сотрудников и вставим в нее данные:

```sql
CREATE TABLE employee_data 
(
    name String,
    age UInt8,
    wage Float32
) ENGINE = MergeTree()
ORDER BY tuple()

INSERT INTO employee_data (name, age, wage) VALUES
    ('John', 16, 10.0),
    ('Alice', 30, 15.0),
    ('Mary', 35, 8.0),
    ('Evelyn', 48, 11.5),
    ('David', 62, 9.9),
    ('Brian', 60, 16.0);
```

Теперь получим имена людей, чей возраст находится в интервалах `[30,60)` 
и `[60,75)`. Поскольку мы используем целочисленное представление для возраста, мы получаем возраста в интервалах
`[30, 59]` и `[60,74]`.

Чтобы агрегировать имена в массив, мы используем агрегатную функцию `groupArray`. 
Она принимает один аргумент. В нашем случае это колонка имени. Функция `groupArrayResample`
должна использовать колонку возраста для агрегирования имен по возрасту. Чтобы определить
необходимые интервалы, мы передаем `30`, `75`, `30` в качестве аргументов в функцию `groupArrayResample`:

```sql
SELECT groupArrayResample(30, 75, 30)(name, age) FROM employee_data
```

```response
┌─groupArrayResample(30, 75, 30)(name, age)─────┐
│ [['Alice','Mary','Evelyn'],['David','Brian']] │
└───────────────────────────────────────────────┘
```

## See also {#see-also}
- [`groupArray`](/sql-reference/aggregate-functions/reference/grouparray)
- [`Resample combinator`](/sql-reference/aggregate-functions/combinators#-resample)
