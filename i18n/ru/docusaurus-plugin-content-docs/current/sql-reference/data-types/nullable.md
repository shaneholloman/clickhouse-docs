---
slug: /sql-reference/data-types/nullable
sidebar_position: 44
sidebar_label: Nullable(T)
---


# Nullable(T)

Позволяет хранить специальный маркер ([NULL](../../sql-reference/syntax.md)), который обозначает "отсутствующее значение", наряду с нормальными значениями, разрешенными типом `T`. Например, колонка типа `Nullable(Int8)` может хранить значения типа `Int8`, а строки, не имеющие значения, будут хранить `NULL`.

Тип `T` не может быть одним из составных типов данных [Array](../../sql-reference/data-types/array.md), [Map](../../sql-reference/data-types/map.md) и [Tuple](../../sql-reference/data-types/tuple.md), но составные типы данных могут содержать значения типа `Nullable`, например, `Array(Nullable(Int8))`.

Поле типа `Nullable` не может быть включено в индексы таблицы.

`NULL` является значением по умолчанию для любого типа `Nullable`, если не указано иное в конфигурации сервера ClickHouse.

## Storage Features {#storage-features}

Для хранения значений типа `Nullable` в колонке таблицы ClickHouse использует отдельный файл с масками `NULL` в дополнение к обычному файлу со значениями. Записи в файле масок позволяют ClickHouse отличать `NULL` от значения по умолчанию соответствующего типа данных для каждой строки таблицы. Из-за дополнительного файла колонка `Nullable` потребляет дополнительное пространство для хранения по сравнению с аналогичной обычной колонкой.

:::note    
Использование `Nullable` почти всегда негативно сказывается на производительности, имейте это в виду при проектировании ваших баз данных.
:::

## Finding NULL {#finding-null}

Можно найти значения `NULL` в колонке, используя подколонку `null` без чтения всей колонки. Она возвращает `1`, если соответствующее значение равно `NULL`, и `0` в противном случае.

**Пример**

Запрос:

``` sql
CREATE TABLE nullable (`n` Nullable(UInt32)) ENGINE = MergeTree ORDER BY tuple();

INSERT INTO nullable VALUES (1), (NULL), (2), (NULL);

SELECT n.null FROM nullable;
```

Результат:

``` text
┌─n.null─┐
│      0 │
│      1 │
│      0 │
│      1 │
└────────┘
```

## Usage Example {#usage-example}

``` sql
CREATE TABLE t_null(x Int8, y Nullable(Int8)) ENGINE TinyLog
```

``` sql
INSERT INTO t_null VALUES (1, NULL), (2, 3)
```

``` sql
SELECT x + y FROM t_null
```

``` text
┌─plus(x, y)─┐
│       ᴺᵁᴸᴸ │
│          5 │
└────────────┘
```
