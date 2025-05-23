---
slug: /guides/replacing-merge-tree
title: 'ReplacingMergeTree'
description: 'Использование движка ReplacingMergeTree в ClickHouse'
keywords: ['replacingmergetree', 'вставки', 'дедупликация']
---

import postgres_replacingmergetree from '@site/static/images/migrations/postgres-replacingmergetree.png';
import Image from '@theme/IdealImage';

В то время как транзакционные базы данных оптимизированы для обработки транзакционных операций обновления и удаления, OLAP базы данных предлагают сниженные гарантии для таких операций. Вместо этого они оптимизируют работу с неизменяемыми данными, вставляемыми пакетами, что значительно ускоряет аналитические запросы. Хотя ClickHouse и предоставляет операции обновления через мутации, а также легковесный способ удаления строк, его ориентированная на колонки структура означает, что эти операции следует планировать с осторожностью, как описано выше. Эти операции обрабатываются асинхронно, выполняются одним потоком и требуют (в случае обновлений) перезаписи данных на диске. Поэтому их не следует использовать для большого числа мелких изменений. 
Чтобы обрабатывать поток строк обновления и удаления, избегая вышеописанных паттернов использования, мы можем использовать движок таблиц ClickHouse ReplacingMergeTree.

## Автоматические обновления вставленных строк {#automatic-upserts-of-inserted-rows}

[Движок таблиц ReplacingMergeTree](/engines/table-engines/mergetree-family/replacingmergetree) позволяет выполнять операции обновления, не прибегая к неэффективным операторам `ALTER` или `DELETE`, предлагая пользователям возможность вставлять несколько копий одной и той же строки и помечать одну из них как последнюю версию. В фоновом режиме асинхронно удаляются старые версии одной и той же строки, эффективно имитируя операцию обновления за счет использования неизменяемых вставок.
Это зависит от способности движка таблицы идентифицировать дублирующиеся строки. Это достигается с помощью оператора `ORDER BY`, который определяет уникальность, то есть, если две строки имеют одинаковые значения для колонок, указанных в `ORDER BY`, они считаются дубликатами. Колонка `version`, указанная при определении таблицы, позволяет сохранить последнюю версию строки, когда две строки идентифицируются как дубликаты, т.е. строка с наибольшим значением версии сохраняется.
Мы иллюстрируем этот процесс в примере ниже. Здесь строки уникально идентифицируются колонкой A (это `ORDER BY` для таблицы). Мы предполагаем, что эти строки были вставлены в два пакета, что привело к образованию двух частей данных на диске. Позже, в ходе асинхронного фонового процесса, эти части сливаются вместе.

Кроме того, ReplacingMergeTree позволяет указать колонку удаления. Эта колонка может содержать значение 0 или 1, где значение 1 указывает на то, что строка (и ее дубликаты) были удалены, а ноль используется в противном случае. **Примечание: удаленные строки не будут удалены во время слияния.**

Во время этого процесса происходит следующее во время слияния частей:

- Строка, идентифицированная значением 1 для колонки A, имеет как обновленную строку с версией 2, так и удаленную строку с версией 3 (и значением колонки удаления 1). Таким образом, последняя строка, помеченная как удаленная, сохраняется.
- Строка, идентифицированная значением 2 для колонки A, имеет две обновленные строки. Последняя строка сохраняется со значением 6 для колонки цены.
- Строка, идентифицированная значением 3 для колонки A, имеет строку с версией 1 и удаленную строку с версией 2. Эта удаленная строка сохраняется.

В результате этого процесса слияния у нас есть четыре строки, представляющие окончательное состояние:

<br />

<Image img={postgres_replacingmergetree} size="md" alt="Процесс ReplacingMergeTree"/>

<br />

Обратите внимание, что удаленные строки никогда не удаляются. Их можно принудительно удалить с помощью `OPTIMIZE table FINAL CLEANUP`. Это требует установки экспериментальной настройки `allow_experimental_replacing_merge_with_cleanup=1`. Это следует делать только при соблюдении следующих условий:

1. Вы можете быть уверены, что старые версии строк (для тех, которые будут удалены с очисткой) не будут вставлены после выполнения операции. Если они будут вставлены, они будут неверно сохранены, так как удаленные строки больше не будут присутствовать.
2. Убедитесь, что все реплики синхронизированы перед выполнением очистки. Это можно сделать с помощью команды:

<br />

```sql
SYSTEM SYNC REPLICA table
```

Мы рекомендуем приостановить вставки после гарантии выполнения (1) и до завершения этой команды и последующей очистки.

> Обработка удалений с использованием ReplacingMergeTree рекомендуется только для таблиц с низким или умеренным количеством удалений (менее 10%), если не удается запланировать периоды для очистки с вышеописанными условиями.

> Совет: пользователи также могут использовать `OPTIMIZE FINAL CLEANUP` для определенных партиций, которые больше не подлежат изменениям.

## Выбор первичного/ключа дедупликации {#choosing-a-primarydeduplication-key}

Выше мы выделили важное дополнительное ограничение, которое также должно быть выполнено в случае ReplacingMergeTree: значения колонок `ORDER BY` должны уникально идентифицировать строку при изменениях. Если вы переходите с транзакционной базы данных, такой как Postgres, то оригинальный первичный ключ Postgres должен быть включен в оператор `ORDER BY` ClickHouse.

Пользователи ClickHouse будут знакомы с выбором колонок в своем операторе `ORDER BY`, чтобы [оптимизировать производительность запросов](/data-modeling/schema-design#choosing-an-ordering-key). Обычно эти колонки должны выбираться на основе ваших [часто используемых запросов и перечислены в порядке возрастания кардинальности](/guides/best-practices/sparse-primary-indexes#an-index-design-for-massive-data-scales). Важно, что ReplacingMergeTree накладывает дополнительное ограничение - эти колонки должны быть неизменяемыми, т.е. при репликации из Postgres добавляйте колонки в этот оператор только в том случае, если они не меняются в основной базе данных Postgres. Несмотря на то, что другие колонки могут изменяться, эти должны оставаться последовательными для уникальной идентификации строк.
Для аналитических рабочих нагрузок первичный ключ Postgres, как правило, имеет небольшое значение, поскольку пользователи редко выполняют точечные запросы по строкам. Учитывая, что мы рекомендуем упорядочивать колонки по возрастанию кардинальности, а также тот факт, что совпадения по [колонкам, перечисленным ранее в операторе ORDER BY, обычно будут быстрее](/guides/best-practices/sparse-primary-indexes#ordering-key-columns-efficiently), первичный ключ Postgres должен быть добавлен в конец `ORDER BY` (если у него нет аналитической ценности). В случае, если несколько колонок формируют первичный ключ в Postgres, они должны быть добавлены в `ORDER BY`, учитывая кардинальность и вероятность значения запроса. Пользователи могут также захотеть сгенерировать уникальный первичный ключ, используя конкатенацию значений через `MATERIALIZED` колонку.

Рассмотрим таблицу постов из набора данных Stack Overflow.

```sql
CREATE TABLE stackoverflow.posts_updateable
(
       `Version` UInt32,
       `Deleted` UInt8,
        `Id` Int32 CODEC(Delta(4), ZSTD(1)),
        `PostTypeId` Enum8('Question' = 1, 'Answer' = 2, 'Wiki' = 3, 'TagWikiExcerpt' = 4, 'TagWiki' = 5, 'ModeratorNomination' = 6, 'WikiPlaceholder' = 7, 'PrivilegeWiki' = 8),
        `AcceptedAnswerId` UInt32,
        `CreationDate` DateTime64(3, 'UTC'),
        `Score` Int32,
        `ViewCount` UInt32 CODEC(Delta(4), ZSTD(1)),
        `Body` String,
        `OwnerUserId` Int32,
        `OwnerDisplayName` String,
        `LastEditorUserId` Int32,
        `LastEditorDisplayName` String,
        `LastEditDate` DateTime64(3, 'UTC') CODEC(Delta(8), ZSTD(1)),
        `LastActivityDate` DateTime64(3, 'UTC'),
        `Title` String,
        `Tags` String,
        `AnswerCount` UInt16 CODEC(Delta(2), ZSTD(1)),
        `CommentCount` UInt8,
        `FavoriteCount` UInt8,
        `ContentLicense` LowCardinality(String),
        `ParentId` String,
        `CommunityOwnedDate` DateTime64(3, 'UTC'),
        `ClosedDate` DateTime64(3, 'UTC')
)
ENGINE = ReplacingMergeTree(Version, Deleted)
PARTITION BY toYear(CreationDate)
ORDER BY (PostTypeId, toDate(CreationDate), CreationDate, Id)
```

Мы используем ключ `ORDER BY` `(PostTypeId, toDate(CreationDate), CreationDate, Id)`. Колонка `Id`, уникальная для каждого поста, гарантирует, что строки могут быть дедуплицированы. Колонки `Version` и `Deleted` добавляются в схему по мере необходимости.

## Запрос к ReplacingMergeTree {#querying-replacingmergetree}

Во время слияния ReplacingMergeTree идентифицирует дублирующиеся строки, используя значения колонок `ORDER BY` в качестве уникального идентификатора, и либо сохраняет только наивысшую версию, либо удаляет все дубликаты, если последняя версия указывает на удаление. Однако это обеспечивает лишь конечную корректность - это не гарантирует, что строки будут дедуплицированы, и вы не должны полагаться на это. Запросы могут, следовательно, давать неправильные ответы из-за учета строк обновления и удаления в запросах.

Чтобы получить правильные ответы, пользователи должны дополнить фоновые слияния дедупликацией и удалением во время выполнения запроса. Это можно достичь с помощью оператора `FINAL`.

Рассмотрим таблицу постов выше. Мы можем использовать обычный метод загрузки этого набора данных, но указываем колонку удаления и колонку версии, дополнительно задавая значения 0. Для примера мы загружаем только 10000 строк.

```sql
INSERT INTO stackoverflow.posts_updateable SELECT 0 AS Version, 0 AS Deleted, *
FROM s3('https://datasets-documentation.s3.eu-west-3.amazonaws.com/stackoverflow/parquet/posts/*.parquet') WHERE AnswerCount > 0 LIMIT 10000

0 rows in set. Elapsed: 1.980 sec. Processed 8.19 thousand rows, 3.52 MB (4.14 thousand rows/s., 1.78 MB/s.)
```

Давайте подтвердим количество строк:

```sql
SELECT count() FROM stackoverflow.posts_updateable

┌─count()─┐
│   10000 │
└─────────┘

1 row in set. Elapsed: 0.002 sec.
```

Теперь мы обновим нашу статистику по ответам на посты. Вместо обновления этих значений мы вставим новые копии 5000 строк и добавим единицу к их номеру версии (это означает, что в таблице будет 150 строк). Мы можем смоделировать это с помощью простого `INSERT INTO SELECT`:

```sql
INSERT INTO posts_updateable SELECT
        Version + 1 AS Version,
        Deleted,
        Id,
        PostTypeId,
        AcceptedAnswerId,
        CreationDate,
        Score,
        ViewCount,
        Body,
        OwnerUserId,
        OwnerDisplayName,
        LastEditorUserId,
        LastEditorDisplayName,
        LastEditDate,
        LastActivityDate,
        Title,
        Tags,
        AnswerCount,
        CommentCount,
        FavoriteCount,
        ContentLicense,
        ParentId,
        CommunityOwnedDate,
        ClosedDate
FROM posts_updateable --выбрать 100 случайных строк
WHERE (Id % toInt32(floor(randUniform(1, 11)))) = 0
LIMIT 5000

0 rows in set. Elapsed: 4.056 sec. Processed 1.42 million rows, 2.20 GB (349.63 thousand rows/s., 543.39 MB/s.)
```

Кроме того, мы удаляем 1000 случайных постов, повторно вставляя строки, но с значением колонки удаления 1. Снова, моделирование этого можно осуществить с помощью простого `INSERT INTO SELECT`.

```sql
INSERT INTO posts_updateable SELECT
        Version + 1 AS Version,
        1 AS Deleted,
        Id,
        PostTypeId,
        AcceptedAnswerId,
        CreationDate,
        Score,
        ViewCount,
        Body,
        OwnerUserId,
        OwnerDisplayName,
        LastEditorUserId,
        LastEditorDisplayName,
        LastEditDate,
        LastActivityDate,
        Title,
        Tags,
        AnswerCount + 1 AS AnswerCount,
        CommentCount,
        FavoriteCount,
        ContentLicense,
        ParentId,
        CommunityOwnedDate,
        ClosedDate
FROM posts_updateable --выбрать 100 случайных строк
WHERE (Id % toInt32(floor(randUniform(1, 11)))) = 0 AND AnswerCount > 0
LIMIT 1000

0 rows in set. Elapsed: 0.166 sec. Processed 135.53 thousand rows, 212.65 MB (816.30 thousand rows/s., 1.28 GB/s.)
```

Результатом вышеописанных операций будет 16000 строк, т.е. 10000 + 5000 + 1000. Правильная общая сумма здесь - на самом деле, мы должны были иметь только 1000 строк меньше, чем наша оригинальная сумма, т.е. 10000 - 1000 = 9000.

```sql
SELECT count()
FROM posts_updateable

┌─count()─┐
│   10000 │
└─────────┘

1 row in set. Elapsed: 0.002 sec.
```

Ваши результаты могут различаться в зависимости от выполненных слияний. Мы видим, что итоговая сумма здесь отличается, поскольку у нас есть дублированные строки. Применение `FINAL` к таблице дает правильный результат.

```sql
SELECT count()
FROM posts_updateable
FINAL

┌─count()─┐
│    9000 │
└─────────┘

1 row in set. Elapsed: 0.006 sec. Processed 11.81 thousand rows, 212.54 KB (2.14 million rows/s., 38.61 MB/s.)
Пиковое использование памяти: 8.14 MiB.
```

## Производительность FINAL {#final-performance}

Оператор `FINAL` будет иметь накладные расходы на производительность запросов, несмотря на постоянные улучшения. Это будет особенно заметно, когда запросы не фильтруют по колонкам первичного ключа, что приводит к считыванию большего объема данных и увеличивает накладные расходы на дедупликацию. Если пользователи фильтруют по ключевым колонкам с использованием условия `WHERE`, объем загружаемых данных и передаваемых для дедупликации будет уменьшен.

Если условие `WHERE` не использует ключевую колонку, ClickHouse в данный момент не использует оптимизацию `PREWHERE` при использовании `FINAL`. Эта оптимизация направлена на сокращение количества строк, считываемых для неотфильтрованных колонок. Примеры эмитации этой `PREWHERE` и, следовательно, потенциального улучшения производительности можно найти [здесь](https://clickhouse.com/blog/clickhouse-postgresql-change-data-capture-cdc-part-1#final-performance).

## Использование партиций с ReplacingMergeTree {#exploiting-partitions-with-replacingmergetree}

Слияние данных в ClickHouse происходит на уровне партиций. При использовании ReplacingMergeTree мы рекомендуем пользователям разбивать свою таблицу на партиции в соответствии с наилучшими практиками, при условии, что пользователи могут гарантировать, что **ключ партиционирования не изменяется для строки**. Это обеспечит, что обновления, относящиеся к одной и той же строке, будут отправлены в одну и ту же партицию ClickHouse. Вы можете повторно использовать тот же ключ партиционирования, что и в Postgres, при условии, что вы придерживаетесь наилучших практик, изложенных здесь.

При предположении, что это так, пользователи могут использовать настройку `do_not_merge_across_partitions_select_final=1`, чтобы улучшить производительность запросов с `FINAL`. Эта настройка заставляет партиции сливать и обрабатывать независимо при использовании FINAL.

Рассмотрим следующую таблицу постов, где мы не используем партиционирование:

```sql
CREATE TABLE stackoverflow.posts_no_part
(
        `Version` UInt32,
        `Deleted` UInt8,
        `Id` Int32 CODEC(Delta(4), ZSTD(1)),
        ...
)
ENGINE = ReplacingMergeTree
ORDER BY (PostTypeId, toDate(CreationDate), CreationDate, Id)

INSERT INTO stackoverflow.posts_no_part SELECT 0 AS Version, 0 AS Deleted, *
FROM s3('https://datasets-documentation.s3.eu-west-3.amazonaws.com/stackoverflow/parquet/posts/*.parquet')

0 rows in set. Elapsed: 182.895 sec. Processed 59.82 million rows, 38.07 GB (327.07 thousand rows/s., 208.17 MB/s.)
```

Чтобы гарантировать, что `FINAL` требуется выполнить некоторую работу, мы обновляем 1 миллион строк, увеличивая их `AnswerCount`, вставляя дублирующие строки.

```sql
INSERT INTO posts_no_part SELECT Version + 1 AS Version, Deleted, Id, PostTypeId, AcceptedAnswerId, CreationDate, Score, ViewCount, Body, OwnerUserId, OwnerDisplayName, LastEditorUserId, LastEditorDisplayName, LastEditDate, LastActivityDate, Title, Tags, AnswerCount + 1 AS AnswerCount, CommentCount, FavoriteCount, ContentLicense, ParentId, CommunityOwnedDate, ClosedDate
FROM posts_no_part
LIMIT 1000000
```

Подсчитываем сумму ответов по годам с `FINAL`:

```sql
SELECT toYear(CreationDate) AS year, sum(AnswerCount) AS total_answers
FROM posts_no_part
FINAL
GROUP BY year
ORDER BY year ASC

┌─year─┬─total_answers─┐
│ 2008 │        371480 │
...
│ 2024 │        127765 │
└──────┴───────────────┘

17 rows in set. Elapsed: 2.338 sec. Processed 122.94 million rows, 1.84 GB (52.57 million rows/s., 788.58 MB/s.)
Пиковое использование памяти: 2.09 GiB.
```

Повторяя эти же шаги для таблицы, разделенной по годам, и повторяя вышеописанный запрос с `do_not_merge_across_partitions_select_final=1`.

```sql
CREATE TABLE stackoverflow.posts_with_part
(
        `Version` UInt32,
        `Deleted` UInt8,
        `Id` Int32 CODEC(Delta(4), ZSTD(1)),
        ...
)
ENGINE = ReplacingMergeTree
PARTITION BY toYear(CreationDate)
ORDER BY (PostTypeId, toDate(CreationDate), CreationDate, Id)

// заполнение и обновление опущено

SELECT toYear(CreationDate) AS year, sum(AnswerCount) AS total_answers
FROM posts_with_part
FINAL
GROUP BY year
ORDER BY year ASC

┌─year─┬─total_answers─┐
│ 2008 │       387832  │
│ 2009 │       1165506 │
│ 2010 │       1755437 │
...
│ 2023 │       787032  │
│ 2024 │       127765  │
└──────┴───────────────┘

17 rows in set. Elapsed: 0.994 sec. Processed 64.65 million rows, 983.64 MB (65.02 million rows/s., 989.23 MB/s.)
```

Как показано, партиционирование значительно улучшило производительность запроса в этом случае, позволяя процессу дедупликации проходить на уровне раздела параллельно.

## Соображения по поведению слияния {#merge-behavior-considerations}

Механизм выбора слияния ClickHouse превосходит простое объединение частей. Ниже мы рассматриваем это поведение в контексте ReplacingMergeTree, включая параметры конфигурации для включения более агрессивного слияния старых данных и соображения для больших частей.

### Логика выбора слияния {#merge-selection-logic}

Хотя слияние направлено на минимизацию числа частей, оно также балансирует эту цель с учетом затрат на увеличение записи. Следовательно, некоторые диапазоны частей исключаются из слияния, если они приведут к чрезмерному увеличению записи, основываясь на внутренних расчетах. Это поведение помогает предотвратить ненужное использование ресурсов и продлевает срок службы компонентов хранения.

### Поведение слияния больших частей {#merging-behavior-on-large-parts}

Движок ReplacingMergeTree в ClickHouse оптимизирован для управления дублированными строками путем слияния частей данных, сохраняя только последнюю версию каждой строки на основе указанного уникального ключа. Однако, когда объединенная часть достигает предела `max_bytes_to_merge_at_max_space_in_pool`, она больше не будет выбрана для дальнейшего слияния, даже если `min_age_to_force_merge_seconds` установлен. В результате нельзя полагаться на автоматические слияния для удаления дубликатов, которые могут накапливаться при непрерывной вставке данных.

Чтобы решить эту проблему, пользователи могут вызвать `OPTIMIZE FINAL`, чтобы вручную объединить части и удалить дубликаты. В отличие от автоматических слияний, `OPTIMIZE FINAL` не учитывает предел `max_bytes_to_merge_at_max_space_in_pool`, объединяя части на основе доступных ресурсов, особенно дискового пространства, пока в каждой партиции не останется одна часть. Однако этот подход может потребовать много памяти для больших таблиц и может потребовать повторного выполнения по мере добавления новых данных.

Для более устойчивого решения, которое поддерживает производительность, рекомендуется партиционирование таблицы. Это может помочь предотвратить достижение частями максимального размера слияния и уменьшить необходимость в постоянных ручных оптимизациях.

### Партиционирование и слияние по партициям {#partitioning-and-merging-across-partitions}

Как обсуждалось в разделе Использование партиций с ReplacingMergeTree, мы рекомендуем партиционирование таблиц как наилучшую практику. Партиционирование изолирует данные для более эффективных слияний и предотвращает слияние по партициям, особенно во время выполнения запроса. Это поведение улучшено в версиях с 23.12 и далее: если ключ партиционирования является префиксом ключа сортировки, слияние по партициям не выполняется во время запроса, что приводит к более высокой производительности запросов.

### Настройка слияния для улучшения производительности запросов {#tuning-merges-for-better-query-performance}

По умолчанию `min_age_to_force_merge_seconds` и `min_age_to_force_merge_on_partition_only` установлены в 0 и false соответственно, что отключает эти функции. В этой конфигурации ClickHouse применяет стандартное поведение слияния без принуждения слияния на основе возраста партиции.

Если задано значение для `min_age_to_force_merge_seconds`, ClickHouse будет игнорировать нормальные эвристики слияния для частей старше указанного периода. Хотя это обычно эффективно только в том случае, если цель - минимизировать общее количество частей, это может улучшить производительность запросов в ReplacingMergeTree, уменьшая количество частей, подлежащих слиянию во время выполнения запроса.

Это поведение можно дополнительно настроить, установив `min_age_to_force_merge_on_partition_only=true`, требуя, чтобы все части в партиции были старше `min_age_to_force_merge_seconds` для агрессивного слияния. Эта конфигурация позволяет старым партициям сливаться до одной части со временем, что консолидирует данные и поддерживает производительность запросов.

### Рекомендуемые настройки {#recommended-settings}

:::warning
Настройка поведения слияния - это сложная операция. Мы рекомендуем проконсультироваться со службой поддержки ClickHouse перед тем, как включить эти настройки в производственных нагрузках.
:::

В большинстве случаев рекомендуется установить `min_age_to_force_merge_seconds` на низкое значение — значительно меньше периода партиционирования. Это минимизирует количество частей и предотвращает ненужное слияние во время выполнения запроса с оператором `FINAL`.

Например, рассмотрим ежемесячную партицию, которая уже была объединена в одну часть. Если небольшая, случайная вставка создает новую часть в этой партиции, производительность запроса может пострадать, поскольку ClickHouse должен считывать несколько частей, пока слияние завершится. Установка `min_age_to_force_merge_seconds` может гарантировать, что эти части объединяются агрессивно, предотвращая ухудшение производительности запросов.
