---
slug: /tutorial
sidebar_label: 'Расширенный учебник'
sidebar_position: 0.5
keywords: ['clickhouse', 'установка', 'учебник', 'словарь', 'словари']
title: 'Расширенный учебник'
description: 'В этом учебнике вы создадите таблицу и вставите большой набор данных. Затем вы выполните запросы к набору данных, включая пример создания словаря и использования его для выполнения JOIN.'
---

import SQLConsoleDetail from '@site/docs/_snippets/_launch_sql_console.md';


# Расширенный учебник

## Чего ожидать от этого учебника? {#what-to-expect-from-this-tutorial}

В этом учебнике вы создадите таблицу и вставите большой набор данных (два миллиона строк данных о такси Нью-Йорка [New York taxi data](/getting-started/example-datasets/nyc-taxi.md)). Затем вы выполните запросы к набору данных, включая пример создания словаря и использования его для выполнения JOIN.

:::note
Этот учебник предполагает, что у вас есть доступ к работающей службе ClickHouse. Если нет, ознакомьтесь с [Быстрым стартом](./quick-start.mdx).
:::

## 1. Создание новой таблицы {#1-create-a-new-table}

Данные о такси Нью-Йорка содержат сведения о миллионах поездок на такси, с колонками, такими как время и место посадки и высадки, стоимость, сумма чаевых, сборы и тип оплаты и так далее. Давайте создадим таблицу для хранения этих данных...

1. Подключитесь к SQL-консоле

  <SQLConsoleDetail />

  Если вы используете самоуправляемый ClickHouse, вы можете подключиться к SQL-консоли по адресу https://_hostname_:8443/play (узнайте у вашего администратора ClickHouse детали подключения).

2. Создайте следующую таблицу `trips` в базе данных `default`:
    ```sql
    CREATE TABLE trips
    (
        `trip_id` UInt32,
        `vendor_id` Enum8('1' = 1, '2' = 2, '3' = 3, '4' = 4, 'CMT' = 5, 'VTS' = 6, 'DDS' = 7, 'B02512' = 10, 'B02598' = 11, 'B02617' = 12, 'B02682' = 13, 'B02764' = 14, '' = 15),
        `pickup_date` Date,
        `pickup_datetime` DateTime,
        `dropoff_date` Date,
        `dropoff_datetime` DateTime,
        `store_and_fwd_flag` UInt8,
        `rate_code_id` UInt8,
        `pickup_longitude` Float64,
        `pickup_latitude` Float64,
        `dropoff_longitude` Float64,
        `dropoff_latitude` Float64,
        `passenger_count` UInt8,
        `trip_distance` Float64,
        `fare_amount` Float32,
        `extra` Float32,
        `mta_tax` Float32,
        `tip_amount` Float32,
        `tolls_amount` Float32,
        `ehail_fee` Float32,
        `improvement_surcharge` Float32,
        `total_amount` Float32,
        `payment_type` Enum8('UNK' = 0, 'CSH' = 1, 'CRE' = 2, 'NOC' = 3, 'DIS' = 4),
        `trip_type` UInt8,
        `pickup` FixedString(25),
        `dropoff` FixedString(25),
        `cab_type` Enum8('yellow' = 1, 'green' = 2, 'uber' = 3),
        `pickup_nyct2010_gid` Int8,
        `pickup_ctlabel` Float32,
        `pickup_borocode` Int8,
        `pickup_ct2010` String,
        `pickup_boroct2010` String,
        `pickup_cdeligibil` String,
        `pickup_ntacode` FixedString(4),
        `pickup_ntaname` String,
        `pickup_puma` UInt16,
        `dropoff_nyct2010_gid` UInt8,
        `dropoff_ctlabel` Float32,
        `dropoff_borocode` UInt8,
        `dropoff_ct2010` String,
        `dropoff_boroct2010` String,
        `dropoff_cdeligibil` String,
        `dropoff_ntacode` FixedString(4),
        `dropoff_ntaname` String,
        `dropoff_puma` UInt16
    )
    ENGINE = MergeTree
    PARTITION BY toYYYYMM(pickup_date)
    ORDER BY pickup_datetime;
    ```

## 2. Вставка набора данных {#2-insert-the-dataset}

Теперь, когда у вас создана таблица, давайте добавим данные о такси Нью-Йорка. Они находятся в CSV-файлах в S3, и вы можете загрузить данные оттуда.

1. Следующая команда вставляет около 2,000,000 строк в вашу таблицу `trips` из двух разных файлов в S3: `trips_1.tsv.gz` и `trips_2.tsv.gz`:

    ```sql
    INSERT INTO trips
    SELECT * FROM s3(
        'https://datasets-documentation.s3.eu-west-3.amazonaws.com/nyc-taxi/trips_{1..2}.gz',
        'TabSeparatedWithNames', "
        `trip_id` UInt32,
        `vendor_id` Enum8('1' = 1, '2' = 2, '3' = 3, '4' = 4, 'CMT' = 5, 'VTS' = 6, 'DDS' = 7, 'B02512' = 10, 'B02598' = 11, 'B02617' = 12, 'B02682' = 13, 'B02764' = 14, '' = 15),
        `pickup_date` Date,
        `pickup_datetime` DateTime,
        `dropoff_date` Date,
        `dropoff_datetime` DateTime,
        `store_and_fwd_flag` UInt8,
        `rate_code_id` UInt8,
        `pickup_longitude` Float64,
        `pickup_latitude` Float64,
        `dropoff_longitude` Float64,
        `dropoff_latitude` Float64,
        `passenger_count` UInt8,
        `trip_distance` Float64,
        `fare_amount` Float32,
        `extra` Float32,
        `mta_tax` Float32,
        `tip_amount` Float32,
        `tolls_amount` Float32,
        `ehail_fee` Float32,
        `improvement_surcharge` Float32,
        `total_amount` Float32,
        `payment_type` Enum8('UNK' = 0, 'CSH' = 1, 'CRE' = 2, 'NOC' = 3, 'DIS' = 4),
        `trip_type` UInt8,
        `pickup` FixedString(25),
        `dropoff` FixedString(25),
        `cab_type` Enum8('yellow' = 1, 'green' = 2, 'uber' = 3),
        `pickup_nyct2010_gid` Int8,
        `pickup_ctlabel` Float32,
        `pickup_borocode` Int8,
        `pickup_ct2010` String,
        `pickup_boroct2010` String,
        `pickup_cdeligibil` String,
        `pickup_ntacode` FixedString(4),
        `pickup_ntaname` String,
        `pickup_puma` UInt16,
        `dropoff_nyct2010_gid` UInt8,
        `dropoff_ctlabel` Float32,
        `dropoff_borocode` UInt8,
        `dropoff_ct2010` String,
        `dropoff_boroct2010` String,
        `dropoff_cdeligibil` String,
        `dropoff_ntacode` FixedString(4),
        `dropoff_ntaname` String,
        `dropoff_puma` UInt16
    ") SETTINGS input_format_try_infer_datetimes = 0
    ```

2. Подождите, пока `INSERT` завершится - это может занять некоторое время, чтобы загрузить 150 MB данных.

    :::note
    Функция `s3` умело знает, как распаковать данные, а формат `TabSeparatedWithNames` сообщает ClickHouse, что данные разделены табуляцией и также пропускает заголовок каждой строки файла.
    :::

3. После завершения вставки проверьте, что все прошло успешно:
    ```sql
    SELECT count() FROM trips
    ```

    Вы должны увидеть около 2M строк (точно 1,999,657 строк).

    :::note
    Обратите внимание, как быстро и как несколько строк ClickHouse пришлось обработать, чтобы определить количество? Вы можете получить это количество за 0.001 секунды с обработкой всего лишь 6 строк.
    :::

4. Если вы выполните запрос, который нужно обратиться ко всем строкам, вы заметите, что обработать нужно будет значительно больше строк, но время выполнения по-прежнему будет стремительно быстрым:
    ```sql
    SELECT DISTINCT(pickup_ntaname) FROM trips
    ```

    Этот запрос должен обработать 2M строк и вернуть 190 значений, но обратите внимание, что это происходит за примерно 1 секунду. Колонка `pickup_ntaname` представляет собой название района в Нью-Йорке, откуда началась поездка на такси.

## 3. Анализ данных {#3-analyze-the-data}

Давайте выполните несколько запросов для анализа 2M строк данных...

1. Начнем с простых вычислений, например, вычислим среднюю сумму чаевых:
    ```sql
    SELECT round(avg(tip_amount), 2) FROM trips
    ```

    Ответ:
    ```response
    ┌─round(avg(tip_amount), 2)─┐
    │                      1.68 │
    └───────────────────────────┘
    ```

2. Этот запрос вычисляет среднюю стоимость в зависимости от количества пассажиров:
    ```sql
    SELECT
        passenger_count,
        ceil(avg(total_amount),2) AS average_total_amount
    FROM trips
    GROUP BY passenger_count
    ```

    `passenger_count` варьируется от 0 до 9:
    ```response
    ┌─passenger_count─┬─average_total_amount─┐
    │               0 │                22.69 │
    │               1 │                15.97 │
    │               2 │                17.15 │
    │               3 │                16.76 │
    │               4 │                17.33 │
    │               5 │                16.35 │
    │               6 │                16.04 │
    │               7 │                 59.8 │
    │               8 │                36.41 │
    │               9 │                 9.81 │
    └─────────────────┴──────────────────────┘
    ```

3. Вот запрос, который вычисляет количество посадок по дням в каждом районе:
    ```sql
    SELECT
        pickup_date,
        pickup_ntaname,
        SUM(1) AS number_of_trips
    FROM trips
    GROUP BY pickup_date, pickup_ntaname
    ORDER BY pickup_date ASC
    ```

    Результат выглядит так:
    ```response
    ┌─pickup_date─┬─pickup_ntaname───────────────────────────────────────────┬─number_of_trips─┐
    │  2015-07-01 │ Brooklyn Heights-Cobble Hill                             │              13 │
    │  2015-07-01 │ Old Astoria                                              │               5 │
    │  2015-07-01 │ Flushing                                                 │               1 │
    │  2015-07-01 │ Yorkville                                                │             378 │
    │  2015-07-01 │ Gramercy                                                 │             344 │
    │  2015-07-01 │ Fordham South                                            │               2 │
    │  2015-07-01 │ SoHo-TriBeCa-Civic Center-Little Italy                   │             621 │
    │  2015-07-01 │ Park Slope-Gowanus                                       │              29 │
    │  2015-07-01 │ Bushwick South                                           │               5 │
    ```

4. Этот запрос вычисляет продолжительность поездки и группирует результаты по этому значению:
    ```sql
    SELECT
        avg(tip_amount) AS avg_tip,
        avg(fare_amount) AS avg_fare,
        avg(passenger_count) AS avg_passenger,
        count() AS count,
        truncate(date_diff('second', pickup_datetime, dropoff_datetime)/60) as trip_minutes
    FROM trips
    WHERE trip_minutes > 0
    GROUP BY trip_minutes
    ORDER BY trip_minutes DESC
    ```

    Результат выглядит так:
    ```response
    ┌──────────────avg_tip─┬───────────avg_fare─┬──────avg_passenger─┬──count─┬─trip_minutes─┐
    │   1.9600000381469727 │                  8 │                  1 │      1 │        27511 │
    │                    0 │                 12 │                  2 │      1 │        27500 │
    │    0.542166673981895 │ 19.716666666666665 │ 1.9166666666666667 │     60 │         1439 │
    │    0.902499997522682 │ 11.270625001192093 │            1.95625 │    160 │         1438 │
    │   0.9715789457909146 │ 13.646616541353383 │ 2.0526315789473686 │    133 │         1437 │
    │   0.9682692398245518 │ 14.134615384615385 │  2.076923076923077 │    104 │         1436 │
    │   1.1022105210705808 │ 13.778947368421052 │  2.042105263157895 │     95 │         1435 │
    ```

5. Этот запрос показывает количество посадок в каждом районе, разбитое по часам дня:
    ```sql
    SELECT
        pickup_ntaname,
        toHour(pickup_datetime) as pickup_hour,
        SUM(1) AS pickups
    FROM trips
    WHERE pickup_ntaname != ''
    GROUP BY pickup_ntaname, pickup_hour
    ORDER BY pickup_ntaname, pickup_hour
    ```

    Результат выглядит так:
    ```response
    ┌─pickup_ntaname───────────────────────────────────────────┬─pickup_hour─┬─pickups─┐
    │ Airport                                                  │           0 │    3509 │
    │ Airport                                                  │           1 │    1184 │
    │ Airport                                                  │           2 │     401 │
    │ Airport                                                  │           3 │     152 │
    │ Airport                                                  │           4 │     213 │
    │ Airport                                                  │           5 │     955 │
    │ Airport                                                  │           6 │    2161 │
    │ Airport                                                  │           7 │    3013 │
    │ Airport                                                  │           8 │    3601 │
    │ Airport                                                  │           9 │    3792 │
    │ Airport                                                  │          10 │    4546 │
    │ Airport                                                  │          11 │    4659 │
    │ Airport                                                  │          12 │    4621 │
    │ Airport                                                  │          13 │    5348 │
    │ Airport                                                  │          14 │    5889 │
    │ Airport                                                  │          15 │    6505 │
    │ Airport                                                  │          16 │    6119 │
    │ Airport                                                  │          17 │    6341 │
    │ Airport                                                  │          18 │    6173 │
    │ Airport                                                  │          19 │    6329 │
    │ Airport                                                  │          20 │    6271 │
    │ Airport                                                  │          21 │    6649 │
    │ Airport                                                  │          22 │    6356 │
    │ Airport                                                  │          23 │    6016 │
    │ Allerton-Pelham Gardens                                  │           4 │       1 │
    │ Allerton-Pelham Gardens                                  │           6 │       1 │
    │ Allerton-Pelham Gardens                                  │           7 │       1 │
    │ Allerton-Pelham Gardens                                  │           9 │       5 │
    │ Allerton-Pelham Gardens                                  │          10 │       3 │
    │ Allerton-Pelham Gardens                                  │          15 │       1 │
    │ Allerton-Pelham Gardens                                  │          20 │       2 │
    │ Allerton-Pelham Gardens                                  │          23 │       1 │
    │ Annadale-Huguenot-Prince's Bay-Eltingville               │          23 │       1 │
    │ Arden Heights                                            │          11 │       1 │
    ```

7. Давайте посмотрим на поездки в аэропорты ЛаГуардиа и JFK:
    ```sql
    SELECT
        pickup_datetime,
        dropoff_datetime,
        total_amount,
        pickup_nyct2010_gid,
        dropoff_nyct2010_gid,
        CASE
            WHEN dropoff_nyct2010_gid = 138 THEN 'LGA'
            WHEN dropoff_nyct2010_gid = 132 THEN 'JFK'
        END AS airport_code,
        EXTRACT(YEAR FROM pickup_datetime) AS year,
        EXTRACT(DAY FROM pickup_datetime) AS day,
        EXTRACT(HOUR FROM pickup_datetime) AS hour
    FROM trips
    WHERE dropoff_nyct2010_gid IN (132, 138)
    ORDER BY pickup_datetime
    ```

    Ответ:
    ```response
    ┌─────pickup_datetime─┬────dropoff_datetime─┬─total_amount─┬─pickup_nyct2010_gid─┬─dropoff_nyct2010_gid─┬─airport_code─┬─year─┬─day─┬─hour─┐
    │ 2015-07-01 00:04:14 │ 2015-07-01 00:15:29 │         13.3 │                 -34 │                  132 │ JFK          │ 2015 │   1 │    0 │
    │ 2015-07-01 00:09:42 │ 2015-07-01 00:12:55 │          6.8 │                  50 │                  138 │ LGA          │ 2015 │   1 │    0 │
    │ 2015-07-01 00:23:04 │ 2015-07-01 00:24:39 │          4.8 │                -125 │                  132 │ JFK          │ 2015 │   1 │    0 │
    │ 2015-07-01 00:27:51 │ 2015-07-01 00:39:02 │        14.72 │                -101 │                  138 │ LGA          │ 2015 │   1 │    0 │
    │ 2015-07-01 00:32:03 │ 2015-07-01 00:55:39 │        39.34 │                  48 │                  138 │ LGA          │ 2015 │   1 │    0 │
    │ 2015-07-01 00:34:12 │ 2015-07-01 00:40:48 │         9.95 │                 -93 │                  132 │ JFK          │ 2015 │   1 │    0 │
    │ 2015-07-01 00:38:26 │ 2015-07-01 00:49:00 │         13.3 │                 -11 │                  138 │ LGA          │ 2015 │   1 │    0 │
    │ 2015-07-01 00:41:48 │ 2015-07-01 00:44:45 │          6.3 │                 -94 │                  132 │ JFK          │ 2015 │   1 │    0 │
    │ 2015-07-01 01:06:18 │ 2015-07-01 01:14:43 │        11.76 │                  37 │                  132 │ JFK          │ 2015 │   1 │    1 │
    ```

## 4. Создание словаря {#4-create-a-dictionary}

Если вы новички в ClickHouse, важно понять, как работают ***словаря***. Простой способ думать о словаре - это отображение пар ключ->значение, которое хранится в памяти. Подробности и все параметры словарей приведены в конце учебника.

1. Давайте посмотрим, как создать словарь, связанный с таблицей в вашей службе ClickHouse. Таблица и соответственно словарь будут основаны на CSV-файле, который содержит 265 строк, по одной строке для каждого района в Нью-Йорке. Районы сопоставлены с названиями боро Нью-Йорка (в Нью-Йорке 5 боро: Бронкс, Бруклин, Манхэттен, Квинс и Статен-Айленд), и этот файл также включает аэропорт Ньюарка (EWR) в качестве боро.

  Вот часть CSV-файла (показана в виде таблицы для ясности). Колонка `LocationID` в файле соответствует колонкам `pickup_nyct2010_gid` и `dropoff_nyct2010_gid` в вашей таблице `trips`:

    | LocationID      | Borough |  Zone      | service_zone |
    | ----------- | ----------- |   ----------- | ----------- |
    | 1      | EWR       |  Newark Airport   | EWR        |
    | 2    |   Queens     |   Jamaica Bay   |      Boro Zone   |
    | 3   |   Bronx     |  Allerton/Pelham Gardens    |    Boro Zone     |
    | 4     |    Manhattan    |    Alphabet City  |     Yellow Zone    |
    | 5     |  Staten Island      |   Arden Heights   |    Boro Zone     |


2. URL для файла: `https://datasets-documentation.s3.eu-west-3.amazonaws.com/nyc-taxi/taxi_zone_lookup.csv`. Выполните следующий SQL-запрос, который создает словарь с именем `taxi_zone_dictionary` и заполняет его из CSV-файла в S3:
  ```sql
  CREATE DICTIONARY taxi_zone_dictionary
  (
    `LocationID` UInt16 DEFAULT 0,
    `Borough` String,
    `Zone` String,
    `service_zone` String
  )
  PRIMARY KEY LocationID
  SOURCE(HTTP(URL 'https://datasets-documentation.s3.eu-west-3.amazonaws.com/nyc-taxi/taxi_zone_lookup.csv' FORMAT 'CSVWithNames'))
  LIFETIME(MIN 0 MAX 0)
  LAYOUT(HASHED_ARRAY())
  ```

  :::note
  Установка `LIFETIME` на 0 означает, что этот словарь никогда не будет обновляться из источника. Это сделано здесь, чтобы не отправлять ненужный трафик к нашему S3-бака, но в целом вы можете указать любые значения времени жизни, которые вам подходят.

    Например:

    ```sql
    LIFETIME(MIN 1 MAX 10)
    ```
    указывает, что словарь будет обновляться через некоторое случайное время между 1 и 10 секундами. (Случайное время необходимо для распределения нагрузки на источник словаря при обновлении на большом количестве серверов.)
  :::

3. Проверьте, что все прошло успешно - вы должны получить 265 строк (по одной строке для каждого района):
    ```sql
    SELECT * FROM taxi_zone_dictionary
    ```

4. Используйте функцию `dictGet` ([или ее вариации](./sql-reference/functions/ext-dict-functions.md)), чтобы получить значение из словаря. Вы передаете имя словаря, значение, которое хотите, и ключ (в нашем примере это колонка `LocationID` таблицы `taxi_zone_dictionary`).

    Например, следующий запрос возвращает `Borough`, чей `LocationID` равен 132 (как мы увидели выше, это аэропорт JFK):
    ```sql
    SELECT dictGet('taxi_zone_dictionary', 'Borough', 132)
    ```

    JFK находится в Квинсе, и обратите внимание, что время на получение значения практически равно 0:
    ```response
    ┌─dictGet('taxi_zone_dictionary', 'Borough', 132)─┐
    │ Queens                                          │
    └─────────────────────────────────────────────────┘

    1 rows in set. Elapsed: 0.004 sec.
    ```

5. Используйте функцию `dictHas`, чтобы проверить, присутствует ли ключ в словаре. Например, следующий запрос возвращает 1 (что означает "истина" в ClickHouse):
    ```sql
    SELECT dictHas('taxi_zone_dictionary', 132)
    ```

6. Следующий запрос возвращает 0, поскольку 4567 не является значением `LocationID` в словаре:
    ```sql
    SELECT dictHas('taxi_zone_dictionary', 4567)
    ```

7. Используйте функцию `dictGet`, чтобы получить название боро в запросе. Например:
    ```sql
    SELECT
        count(1) AS total,
        dictGetOrDefault('taxi_zone_dictionary','Borough', toUInt64(pickup_nyct2010_gid), 'Unknown') AS borough_name
    FROM trips
    WHERE dropoff_nyct2010_gid = 132 OR dropoff_nyct2010_gid = 138
    GROUP BY borough_name
    ORDER BY total DESC
    ```

    Этот запрос суммирует количество поездок на такси по боро, которые заканчиваются либо в аэропорту ЛаГуардиа, либо в JFK. Результат выглядит следующим образом, и обратите внимание, что есть довольно много поездок, где район садов неизвестен:
    ```response
    ┌─total─┬─borough_name──┐
    │ 23683 │ Unknown       │
    │  7053 │ Manhattan     │
    │  6828 │ Brooklyn      │
    │  4458 │ Queens        │
    │  2670 │ Bronx         │
    │   554 │ Staten Island │
    │    53 │ EWR           │
    └───────┴───────────────┘

    7 rows in set. Elapsed: 0.019 sec. Processed 2.00 million rows, 4.00 MB (105.70 million rows/s., 211.40 MB/s.)
    ```

## 5. Выполнение соединения {#5-perform-a-join}

Давайте напишем несколько запросов, которые объединяют `taxi_zone_dictionary` с вашей таблицей `trips`.

1. Мы можем начать с простого JOIN, который действует аналогично предыдущему запросу по аэропорту:
    ```sql
    SELECT
        count(1) AS total,
        Borough
    FROM trips
    JOIN taxi_zone_dictionary ON toUInt64(trips.pickup_nyct2010_gid) = taxi_zone_dictionary.LocationID
    WHERE dropoff_nyct2010_gid = 132 OR dropoff_nyct2010_gid = 138
    GROUP BY Borough
    ORDER BY total DESC
    ```

    Ответ выглядит знакомо:
    ```response
    ┌─total─┬─Borough───────┐
    │  7053 │ Manhattan     │
    │  6828 │ Brooklyn      │
    │  4458 │ Queens        │
    │  2670 │ Bronx         │
    │   554 │ Staten Island │
    │    53 │ EWR           │
    └───────┴───────────────┘

    6 rows in set. Elapsed: 0.034 sec. Processed 2.00 million rows, 4.00 MB (59.14 million rows/s., 118.29 MB/s.)
    ```

    :::note
    Обратите внимание, что вывод вышеуказанного запроса `JOIN` совпадает с выводом запроса, который использовал `dictGetOrDefault` (за исключением того, что значения `Unknown` не включены). За кулисами ClickHouse фактически вызывает функцию `dictGet` для словаря `taxi_zone_dictionary`, но синтаксис `JOIN` более привычен для разработчиков SQL.
    :::

2. Мы не часто используем `SELECT *` в ClickHouse - вы должны извлекать только те колонки, которые вам действительно нужны! Но сложно найти запрос, который требует много времени, поэтому этот запрос целенаправленно выбирает каждую колонку и возвращает каждую строку (за исключением того, что по умолчанию есть встроенный максимум в 10,000 строк в ответе), а также выполняет правое соединение каждой строки со словарем:
    ```sql
    SELECT *
    FROM trips
    JOIN taxi_zone_dictionary
        ON trips.dropoff_nyct2010_gid = taxi_zone_dictionary.LocationID
    WHERE tip_amount > 0
    ORDER BY tip_amount DESC
    LIMIT 1000
    ```

#### Поздравляем! {#congrats}

Отлично - вы прошли через учебник, и, надеюсь, у вас появилось лучшее понимание того, как использовать ClickHouse. Вот несколько вариантов того, что делать дальше:

- Прочитайте [как работают первичные ключи в ClickHouse](./guides/best-practices/sparse-primary-indexes.md) - эти знания помогут вам продвинуться вперед на пути к становлению экспертом по ClickHouse.
- [Интегрируйте внешний источник данных](/integrations/index.mdx), такой как файлы, Kafka, PostgreSQL, конвейеры данных или множество других источников данных.
- [Подключите свой любимый инструмент UI/BI](./integrations/data-visualization/index.md) к ClickHouse.
- Ознакомьтесь с [SQL справочником](./sql-reference/index.md) и просмотрите различные функции. У ClickHouse потрясающая коллекция функций для трансформации, обработки и анализа данных.
- Узнайте больше о [Словарях](/sql-reference/dictionaries/index.md).
