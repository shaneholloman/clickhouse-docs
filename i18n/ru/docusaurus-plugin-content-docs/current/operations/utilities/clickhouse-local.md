---
slug: /operations/utilities/clickhouse-local
sidebar_position: 60
sidebar_label: clickhouse-local
---


# clickhouse-local

## Связанный контент {#related-content}

- Блог: [Извлечение, преобразование и запрос данных в локальных файлах с помощью clickhouse-local](https://clickhouse.com/blog/extracting-converting-querying-local-files-with-sql-clickhouse-local)

## Когда использовать clickhouse-local против ClickHouse {#when-to-use-clickhouse-local-vs-clickhouse}

`clickhouse-local` — это простая версия ClickHouse, которая идеально подходит для разработчиков, которым необходимо быстро обрабатывать локальные и удаленные файлы с помощью SQL без установки полноценного сервера баз данных. С `clickhouse-local` разработчики могут использовать SQL-команды (используя [диалект SQL ClickHouse](../../sql-reference/index.md)) непосредственно из командной строки, что обеспечивает простой и эффективный способ доступа к функциональности ClickHouse без необходимости полной установки ClickHouse. Одним из главных преимуществ `clickhouse-local` является то, что он уже включен при установке [clickhouse-client](/operations/utilities/clickhouse-local). Это означает, что разработчики могут быстро начать использовать `clickhouse-local`, не проходя через сложный процесс установки.

Хотя `clickhouse-local` является отличным инструментом для разработки и тестирования, а также для обработки файлов, он не подходит для обслуживания конечных пользователей или приложений. В таких сценариях рекомендуется использовать открытую версию [ClickHouse](/install). ClickHouse — это мощная OLAP база данных, предназначенная для обработки крупномасштабных аналитических нагрузок. Она предоставляет быструю и эффективную обработку сложных запросов на больших наборах данных, что делает ее идеальной для использования в производственной среде, где высокопроизводительность критически важна. Кроме того, ClickHouse предлагает широкий спектр функций, таких как репликация, шардирование и высокая доступность, которые необходимы для масштабирования и обработки больших наборов данных, а также для обслуживания приложений. Если вам нужно обрабатывать большие наборы данных или обслуживать конечных пользователей или приложения, мы рекомендуем использовать открытую версию ClickHouse вместо `clickhouse-local`.

Пожалуйста, прочитайте документы ниже, которые показывают примеры использования `clickhouse-local`, такие как [запрос данных из локального файла](#query_data_in_file) или [чтение файла Parquet в S3](#query-data-in-a-parquet-file-in-aws-s3).

## Скачивание clickhouse-local {#download-clickhouse-local}

`clickhouse-local` выполняется с помощью того же бинарного файла `clickhouse`, который запускает сервер ClickHouse и `clickhouse-client`. Самый простой способ скачать последнюю версию — это выполнить следующую команду:

```bash
curl https://clickhouse.com/ | sh
```

:::note
Бинарный файл, который вы только что загрузили, может запускать все виды инструментов и утилит ClickHouse. Если вы хотите запустить ClickHouse как сервер баз данных, посмотрите [Быстрый старт](../../quick-start.mdx).
:::

## Запрос данных из файла с использованием SQL {#query_data_in_file}

Распространенное использование `clickhouse-local` — это выполнение произвольных запросов на файловых данных: когда вам не нужно вставлять данные в таблицу. `clickhouse-local` может загрузить данные из файла во временную таблицу и выполнить ваш SQL-запрос.

Если файл находится на той же машине, что и `clickhouse-local`, вы можете просто указать файл для загрузки. Следующий файл `reviews.tsv` содержит выборку обзоров продуктов Amazon:

```bash
./clickhouse local -q "SELECT * FROM 'reviews.tsv'"
```

Эта команда является сокращенной для:

```bash
./clickhouse local -q "SELECT * FROM file('reviews.tsv')"
```

ClickHouse знает, что файл использует формат с разделителем табуляции по расширению файла. Если вам необходимо явно указать формат, просто добавьте один из [многих форматов ввода ClickHouse](../../interfaces/formats.md):
```bash
./clickhouse local -q "SELECT * FROM file('reviews.tsv', 'TabSeparated')"
```

Функция таблицы `file` создает таблицу, и вы можете использовать `DESCRIBE`, чтобы увидеть выведенную схему:

```bash
./clickhouse local -q "DESCRIBE file('reviews.tsv')"
```

:::tip
Вы можете использовать шаблоны в имени файла (см. [замены шаблонов](/sql-reference/table-functions/file.md/#globs-in-path)).

Примеры:

```bash
./clickhouse local -q "SELECT * FROM 'reviews*.jsonl'"
./clickhouse local -q "SELECT * FROM 'review_?.csv'"
./clickhouse local -q "SELECT * FROM 'review_{1..3}.csv'"
```

:::

```response
marketplace	Nullable(String)
customer_id	Nullable(Int64)
review_id	Nullable(String)
product_id	Nullable(String)
product_parent	Nullable(Int64)
product_title	Nullable(String)
product_category	Nullable(String)
star_rating	Nullable(Int64)
helpful_votes	Nullable(Int64)
total_votes	Nullable(Int64)
vine	Nullable(String)
verified_purchase	Nullable(String)
review_headline	Nullable(String)
review_body	Nullable(String)
review_date	Nullable(Date)
```

Давайте найдем продукт с наивысшим рейтингом:

```bash
./clickhouse local -q "SELECT
    argMax(product_title,star_rating),
    max(star_rating)
FROM file('reviews.tsv')"
```

```response
Monopoly Junior Board Game	5
```

## Запрос данных в файле Parquet в AWS S3 {#query-data-in-a-parquet-file-in-aws-s3}

Если у вас есть файл в S3, используйте `clickhouse-local` и функцию таблицы `s3` для запроса файла на месте (без вставки данных в таблицу ClickHouse). У нас есть файл с именем `house_0.parquet` в публичном бакете, который содержит цены на недвижимость, проданную в Великобритании. Давайте посмотрим, сколько строк в нем:

```bash
./clickhouse local -q "
SELECT count()
FROM s3('https://datasets-documentation.s3.eu-west-3.amazonaws.com/house_parquet/house_0.parquet')"
```

В файле 2.7M строк:

```response
2772030
```

Всегда полезно увидеть, какую схему ClickHouse выводит из файла:

```bash
./clickhouse local -q "DESCRIBE s3('https://datasets-documentation.s3.eu-west-3.amazonaws.com/house_parquet/house_0.parquet')"
```

```response
price	Nullable(Int64)
date	Nullable(UInt16)
postcode1	Nullable(String)
postcode2	Nullable(String)
type	Nullable(String)
is_new	Nullable(UInt8)
duration	Nullable(String)
addr1	Nullable(String)
addr2	Nullable(String)
street	Nullable(String)
locality	Nullable(String)
town	Nullable(String)
district	Nullable(String)
county	Nullable(String)
```

Давайте посмотрим, какие районы самые дорогие:

```bash
./clickhouse local -q "
SELECT
    town,
    district,
    count() AS c,
    round(avg(price)) AS price,
    bar(price, 0, 5000000, 100)
FROM s3('https://datasets-documentation.s3.eu-west-3.amazonaws.com/house_parquet/house_0.parquet')
GROUP BY
    town,
    district
HAVING c >= 100
ORDER BY price DESC
LIMIT 10"
```

```response
LONDON	CITY OF LONDON	886	2271305	█████████████████████████████████████████████▍
LEATHERHEAD	ELMBRIDGE	206	1176680	███████████████████████▌
LONDON	CITY OF WESTMINSTER	12577	1108221	██████████████████████▏
LONDON	KENSINGTON AND CHELSEA	8728	1094496	█████████████████████▉
HYTHE	FOLKESTONE AND HYTHE	130	1023980	████████████████████▍
CHALFONT ST GILES	CHILTERN	113	835754	████████████████▋
AMERSHAM	BUCKINGHAMSHIRE	113	799596	███████████████▉
VIRGINIA WATER	RUNNYMEDE	356	789301	███████████████▊
BARNET	ENFIELD	282	740514	██████████████▊
NORTHWOOD	THREE RIVERS	184	731609	██████████████▋
```

:::tip
Когда вы будете готовы вставить ваши файлы в ClickHouse, запустите сервер ClickHouse и вставьте результаты ваших функций таблиц `file` и `s3` в таблицу `MergeTree`. Ознакомьтесь с [Быстрым стартом](../../quick-start.mdx) для получения дополнительной информации.
:::


## Преобразования форматов {#format-conversions}

Вы можете использовать `clickhouse-local` для преобразования данных между различными форматами. Пример:

``` bash
$ clickhouse-local --input-format JSONLines --output-format CSV --query "SELECT * FROM table" < data.json > data.csv
```

Форматы автоматически определяются по расширениям файлов: 

``` bash
$ clickhouse-local --query "SELECT * FROM table" < data.json > data.csv
```

В качестве ярлыка вы можете написать это с использованием аргумента `--copy`:
``` bash
$ clickhouse-local --copy < data.json > data.csv
```


## Использование {#usage}

По умолчанию `clickhouse-local` имеет доступ к данным сервера ClickHouse на том же хосте, и он не зависит от конфигурации сервера. Он также поддерживает загрузку конфигурации сервера с помощью аргумента `--config-file`. Для временных данных создается уникальный временный каталог данных по умолчанию.

Основное использование (Linux):

``` bash
$ clickhouse-local --structure "table_structure" --input-format "format_of_incoming_data" --query "query"
```

Основное использование (Mac):

``` bash
$ ./clickhouse local --structure "table_structure" --input-format "format_of_incoming_data" --query "query"
```

:::note
`clickhouse-local` также поддерживается на Windows через WSL2.
:::

Аргументы:

- `-S`, `--structure` — структура таблицы для входных данных.
- `--input-format` — входной формат, по умолчанию `TSV`.
- `-F`, `--file` — путь к данным, по умолчанию `stdin`.
- `-q`, `--query` — запросы для выполнения с разделителем `;`. `--query` может быть указан несколько раз, например, `--query "SELECT 1" --query "SELECT 2"`. Нельзя использовать одновременно с `--queries-file`.
- `--queries-file` - путь к файлу с запросами для выполнения. `--queries-file` может быть указан несколько раз, например, `--query queries1.sql --query queries2.sql`. Нельзя использовать одновременно с `--query`.
- `--multiquery, -n` – Если указано, несколько запросов, разделенных точками с запятой, могут быть перечислены после опции `--query`. Для удобства также можно опустить `--query` и передать запросы непосредственно после `--multiquery`.
- `-N`, `--table` — имя таблицы, куда поместить выходные данные, по умолчанию `table`.
- `-f`, `--format`, `--output-format` — выходной формат, по умолчанию `TSV`.
- `-d`, `--database` — база данных по умолчанию, по умолчанию `_local`.
- `--stacktrace` — нужно ли выводить отладочный вывод в случае исключения.
- `--echo` — выводить запрос перед выполнением.
- `--verbose` — больше деталей о выполнении запросов.
- `--logger.console` — Журналировать в консоль.
- `--logger.log` — Имя файла журнала.
- `--logger.level` — Уровень журнала.
- `--ignore-error` — не прекращать обработку, если запрос не удался.
- `-c`, `--config-file` — путь к файлу конфигурации в том же формате, что и для сервера ClickHouse, по умолчанию конфигурация пустая.
- `--no-system-tables` — не подключать системные таблицы.
- `--help` — справка по аргументам для `clickhouse-local`.
- `-V`, `--version` — вывести информацию о версии и выйти.

Также есть аргументы для каждой переменной конфигурации ClickHouse, которые чаще используются вместо `--config-file`.


## Примеры {#examples}

``` bash
$ echo -e "1,2\n3,4" | clickhouse-local --structure "a Int64, b Int64" \
    --input-format "CSV" --query "SELECT * FROM table"
Read 2 rows, 32.00 B in 0.000 sec., 5182 rows/sec., 80.97 KiB/sec.
1   2
3   4
```

Предыдущий пример такой же, как:

``` bash
$ echo -e "1,2\n3,4" | clickhouse-local -n --query "
    CREATE TABLE table (a Int64, b Int64) ENGINE = File(CSV, stdin);
    SELECT a, b FROM table;
    DROP TABLE table;"
Read 2 rows, 32.00 B in 0.000 sec., 4987 rows/sec., 77.93 KiB/sec.
1   2
3   4
```

Вам не обязательно использовать `stdin` или аргумент `--file`, и вы можете открывать любое количество файлов, используя функцию таблицы [`file`](../../sql-reference/table-functions/file.md):

``` bash
$ echo 1 | tee 1.tsv
1

$ echo 2 | tee 2.tsv
2

$ clickhouse-local --query "
    select * from file('1.tsv', TSV, 'a int') t1
    cross join file('2.tsv', TSV, 'b int') t2"
1	2
```

Теперь давайте выведем использование памяти для каждого пользователя Unix:

Запрос:

``` bash
$ ps aux | tail -n +2 | awk '{ printf("%s\t%s\n", $1, $4) }' \
    | clickhouse-local --structure "user String, mem Float64" \
        --query "SELECT user, round(sum(mem), 2) as memTotal
            FROM table GROUP BY user ORDER BY memTotal DESC FORMAT Pretty"
```

Результат:

``` text
Read 186 rows, 4.15 KiB in 0.035 sec., 5302 rows/sec., 118.34 KiB/sec.
┏━━━━━━━━━━┳━━━━━━━━━━┓
┃ user     ┃ memTotal ┃
┡━━━━━━━━━━╇━━━━━━━━━━┩
│ bayonet  │    113.5 │
├──────────┼──────────┤
│ root     │      8.8 │
├──────────┼──────────┤
...
```



## Связанный контент {#related-content-1}

- [Извлечение, преобразование и запрос данных в локальных файлах с помощью clickhouse-local](https://clickhouse.com/blog/extracting-converting-querying-local-files-with-sql-clickhouse-local)
- [Получение данных в ClickHouse - Часть 1](https://clickhouse.com/blog/getting-data-into-clickhouse-part-1)
- [Изучение массивных реальных наборов данных: 100+ лет записей о погоде в ClickHouse](https://clickhouse.com/blog/real-world-data-noaa-climate-data)
