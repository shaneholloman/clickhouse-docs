---
slug: /guides/developer/deduplication
sidebar_label: 'Стратегии дедупликации'
sidebar_position: 3
description: 'Используйте дедупликацию, когда вам нужно выполнять частые обновления, вставки и удаления.'
title: 'Стратегии дедупликации'
---

import deduplication from '@site/static/images/guides/developer/de_duplication.png';
import Image from '@theme/IdealImage';



# Стратегии дедупликации

**Дедупликация** относится к процессу ***удаления дублирующихся строк в наборе данных***. В OLTP базе данных это делается легко, потому что каждая строка имеет уникальный первичный ключ - но за счет более медленных вставок. Каждая вставленная строка сначала должна быть найдена, и, если она найдена, требуется её заменить.

ClickHouse создан для скорости в отношении вставки данных. Файлы хранения являются неизменяемыми, и ClickHouse не проверяет существующий первичный ключ перед вставкой строки, поэтому дедупликация требует немного больше усилий. Это также означает, что дедупликация не является немедленной - она **окончательная**, что имеет несколько побочных эффектов:

- В любой момент времени ваша таблица может всё ещё содержать дубликаты (строки с одинаковым ключом сортировки)
- Фактическое удаление дублирующихся строк происходит во время слияния частей
- Ваши запросы должны учитывать возможность дубликатов

<div class='transparent-table'>

|||
|------|----|
|<Image img={deduplication}  alt="Логотип Дедупликации" size="sm"/>|ClickHouse предоставляет бесплатное обучение по дедупликации и многим другим темам. Модуль обучения [Удаление и обновление данных](https://learn.clickhouse.com/visitor_catalog_class/show/1328954/?utm_source=clickhouse&utm_medium=docs) - хорошее место для начала.|

</div>

## Варианты дедупликации {#options-for-deduplication}

Дедупликация реализована в ClickHouse с использованием следующих движков таблиц:

1. Движок таблицы `ReplacingMergeTree`: с помощью этого движка дублирующиеся строки с одинаковым ключом сортировки удаляются во время слияний. `ReplacingMergeTree` является хорошим вариантом для эмуляции поведения upsert (когда вы хотите, чтобы запросы возвращали последнюю вставленную строку).

2. Слияние строк: движки таблиц `CollapsingMergeTree` и `VersionedCollapsingMergeTree` используют логику, при которой существующая строка "отменяется", и добавляется новая строка. Они более сложные в реализации, чем `ReplacingMergeTree`, но ваши запросы и агрегации могут быть проще для написания без необходимости беспокоиться о том, была ли ещё слияна информация. Эти два движка таблиц полезны, когда вам нужно часто обновлять данные.

Мы рассмотрим обе эти техники ниже. Для получения более подробной информации можете ознакомиться с нашим бесплатным модулем обучения [Удаление и обновление данных](https://learn.clickhouse.com/visitor_catalog_class/show/1328954/?utm_source=clickhouse&utm_medium=docs).

## Использование ReplacingMergeTree для Upserts {#using-replacingmergetree-for-upserts}

Посмотрим на простой пример, где таблица содержит комментарии Hacker News с колонкой views, представляющей количество раз, когда комментарий был просмотрен. Предположим, мы вставляем новую строку, когда статья публикуется, и обновляем строку один раз в день с общим количеством просмотров, если значение увеличивается:

```sql
CREATE TABLE hackernews_rmt (
    id UInt32,
    author String,
    comment String,
    views UInt64
)
ENGINE = ReplacingMergeTree
PRIMARY KEY (author, id)
```

Вставим две строки:

```sql
INSERT INTO hackernews_rmt VALUES
   (1, 'ricardo', 'Это пост #1', 0),
   (2, 'ch_fan', 'Это пост #2', 0)
```

Чтобы обновить колонку `views`, вставим новую строку с тем же первичным ключом (обратите внимание на новые значения колонки `views`):

```sql
INSERT INTO hackernews_rmt VALUES
   (1, 'ricardo', 'Это пост #1', 100),
   (2, 'ch_fan', 'Это пост #2', 200)
```

Теперь в таблице 4 строки:

```sql
SELECT *
FROM hackernews_rmt
```

```response
┌─id─┬─author──┬─comment─────────┬─views─┐
│  2 │ ch_fan  │ Это пост #2     │     0 │
│  1 │ ricardo │ Это пост #1     │     0 │
└────┴─────────┴─────────────────┴───────┘
┌─id─┬─author──┬─comment─────────┬─views─┐
│  2 │ ch_fan  │ Это пост #2     │   200 │
│  1 │ ricardo │ Это пост #1     │   100 │
└────┴─────────┴─────────────────┴───────┘
```

Отдельные блоки выше в выводе демонстрируют две части за кулисами - эти данные ещё не были объединены, поэтому дублирующиеся строки ещё не были удалены. Давайте используем ключевое слово `FINAL` в запросе `SELECT`, что приводит к логическому слиянию результата запроса:

```sql
SELECT *
FROM hackernews_rmt
FINAL
```

```response
┌─id─┬─author──┬─comment─────────┬─views─┐
│  2 │ ch_fan  │ Это пост #2     │   200 │
│  1 │ ricardo │ Это пост #1     │   100 │
└────┴─────────┴─────────────────┴───────┘
```

Результат имеет только 2 строки, и последняя вставленная строка - это та, которая возвращается.

:::note
Использование `FINAL` работает нормально, если у вас небольшое количество данных. Если вы работаете с большим количеством данных, использование `FINAL` вероятно, не лучший вариант. Давайте обсудим более подходящий вариант для нахождения последнего значения колонки...
:::

### Избегание FINAL {#avoiding-final}

Давайте снова обновим колонку `views` для обеих уникальных строк:

```sql
INSERT INTO hackernews_rmt VALUES
   (1, 'ricardo', 'Это пост #1', 150),
   (2, 'ch_fan', 'Это пост #2', 250)
```

Теперь в таблице 6 строк, поскольку фактическое слияние ещё не произошло (только объединение во время запроса, когда мы использовали `FINAL`).

```sql
SELECT *
FROM hackernews_rmt
```

```response
┌─id─┬─author──┬─comment─────────┬─views─┐
│  2 │ ch_fan  │ Это пост #2     │   200 │
│  1 │ ricardo │ Это пост #1     │   100 │
└────┴─────────┴─────────────────┴───────┘
┌─id─┬─author──┬─comment─────────┬─views─┐
│  2 │ ch_fan  │ Это пост #2     │     0 │
│  1 │ ricardo │ Это пост #1     │     0 │
└────┴─────────┴─────────────────┴───────┘
┌─id─┬─author──┬─comment─────────┬─views─┐
│  2 │ ch_fan  │ Это пост #2     │   250 │
│  1 │ ricardo │ Это пост #1     │   150 │
└────┴─────────┴─────────────────┴───────┘
```


Вместо использования `FINAL`, давайте применим некоторую бизнес-логику - мы знаем, что колонка `views` всегда увеличивается, поэтому мы можем выбрать строку с наибольшим значением, используя функцию `max` после группировки по нужным колонкам:

```sql
SELECT
    id,
    author,
    comment,
    max(views)
FROM hackernews_rmt
GROUP BY (id, author, comment)
```

```response
┌─id─┬─author──┬─comment─────────┬─max(views)─┐
│  2 │ ch_fan  │ Это пост #2     │        250 │
│  1 │ ricardo │ Это пост #1     │        150 │
└────┴─────────┴─────────────────┴────────────┘
```

Группировка в приведенном выше запросе на самом деле может быть более эффективной (с точки зрения производительности запроса), чем использование ключевого слова `FINAL`.

Наш [модуль обучения по удалению и обновлению данных](https://learn.clickhouse.com/visitor_catalog_class/show/1328954/?utm_source=clickhouse&utm_medium=docs) расширяет этот пример, включая информацию о том, как использовать колонку `version` с `ReplacingMergeTree`.

## Использование CollapsingMergeTree для частого обновления колонок {#using-collapsingmergetree-for-updating-columns-frequently}

Обновление колонки включает в себя удаление существующей строки и замену её новыми значениями. Как вы уже видели, этот тип мутации в ClickHouse происходит _в конечном итоге_ - во время слияний. Если вам нужно обновить много строк, может оказаться более эффективным избегать команды `ALTER TABLE..UPDATE` и просто вставить новые данные рядом с существующими. Мы можем добавить колонку, которая указывает, являются ли данные устаревшими или новыми... и на самом деле есть движок таблицы, который уже очень хорошо реализует это поведение, особенно учитывая, что он автоматически удаляет устаревшие данные. Давайте посмотрим, как это работает.

Предположим, мы отслеживаем количество просмотров комментариев Hacker News с помощью внешней системы, и каждые несколько часов отправляем данные в ClickHouse. Мы хотим, чтобы старые строки были удалены, а новые строки представляли новое состояние каждого комментария Hacker News. Мы можем использовать `CollapsingMergeTree` для реализации этого поведения.

Давайте определим таблицу для хранения количества просмотров:

```sql
CREATE TABLE hackernews_views (
    id UInt32,
    author String,
    views UInt64,
    sign Int8
)
ENGINE = CollapsingMergeTree(sign)
PRIMARY KEY (id, author)
```

Обратите внимание, что таблица `hackernews_views` имеет колонку `Int8` под названием sign, которая называется **колонкой знака**. Название колонки знака произвольно, но требуется тип данных `Int8`, и обратите внимание, что имя колонки передается в конструктор таблицы `CollapsingMergeTree`.

Что такое колонка знака в таблице `CollapsingMergeTree`? Она представляет _состояние_ строки, и колонка знака может быть только 1 или -1. Вот как это работает:

- Если две строки имеют одинаковый первичный ключ (или порядок сортировки, если он отличается от первичного ключа), но разные значения колонки знака, то последняя вставленная строка с +1 становится строкой состояния, а другие строки отменяются
- Строки, которые отменяют друг друга, удаляются во время слияний
- Строки, у которых нет соответствующей пары, сохраняются

Давайте добавим строку в таблицу `hackernews_views`. Поскольку это единственная строка для этого первичного ключа, мы устанавливаем её состояние на 1:

```sql
INSERT INTO hackernews_views VALUES
   (123, 'ricardo', 0, 1)
```

Теперь предположим, мы хотим изменить колонку views. Вы вставляете две строки: одна, которая отменяет существующую строку, и одна, которая содержит новое состояние строки:

```sql
INSERT INTO hackernews_views VALUES
   (123, 'ricardo', 0, -1),
   (123, 'ricardo', 150, 1)
```

Теперь в таблице 3 строки с первичным ключом `(123, 'ricardo')`:

```sql
SELECT *
FROM hackernews_views
```

```response
┌──id─┬─author──┬─views─┬─sign─┐
│ 123 │ ricardo │     0 │   -1 │
│ 123 │ ricardo │   150 │    1 │
└─────┴─────────┴───────┴──────┘
┌──id─┬─author──┬─views─┬─sign─┐
│ 123 │ ricardo │     0 │    1 │
└─────┴─────────┴───────┴──────┘
```

Обратите внимание, что добавление `FINAL` возвращает текущую строку состояния:

```sql
SELECT *
FROM hackernews_views
FINAL
```

```response
┌──id─┬─author──┬─views─┬─sign─┐
│ 123 │ ricardo │   150 │    1 │
└─────┴─────────┴───────┴──────┘
```

Но, конечно, использовать `FINAL` не рекомендуется для больших таблиц.

:::note
Значение, переданное в колонку `views` в нашем примере, на самом деле не требуется и не обязательно соответствует текущему значению `views` старой строки. Фактически, вы можете отменить строку, указав только первичный ключ и -1:

```sql
INSERT INTO hackernews_views(id, author, sign) VALUES
   (123, 'ricardo', -1)
```
:::

## Обновления в реальном времени из нескольких потоков {#real-time-updates-from-multiple-threads}

С таблицей `CollapsingMergeTree` строки отменяют друг друга, используя колонку знака, и состояние строки определяется последней вставленной строкой. Но это может быть проблематично, если вы вставляете строки из разных потоков, где строки могут быть вставлены не по порядку. Использование "последней" строки не сработает в этой ситуации.

Здесь на помощь приходит `VersionedCollapsingMergeTree` - он объединяет строки так же, как `CollapsingMergeTree`, но вместо того, чтобы сохранять последнюю вставленную строку, он сохраняет строку с наибольшим значением колонки версии, которую вы указываете.

Давайте рассмотрим пример. Предположим, мы хотим отслеживать количество просмотров наших комментариев Hacker News, и данные часто обновляются. Мы хотим, чтобы отчёты использовали последние значения, не заставляя или не ожидая слияний. Мы начинаем с таблицы, аналогичной `CollapsedMergeTree`, за исключением того, что добавляем колонку для хранения версии состояния строки:

```sql
CREATE TABLE hackernews_views_vcmt (
    id UInt32,
    author String,
    views UInt64,
    sign Int8,
    version UInt32
)
ENGINE = VersionedCollapsingMergeTree(sign, version)
PRIMARY KEY (id, author)
```

Обратите внимание, что таблица использует `VersionedCollapsingMergeTree` в качестве движка и передает **колонку знака** и **колонку версии**. Вот как работает таблица:

- Она удаляет каждую пару строк, у которых одинаковый первичный ключ и версия и разные знаки
- Порядок, в котором строки были вставлены, не имеет значения
- Обратите внимание, что если колонка версии не является частью первичного ключа, ClickHouse добавляет её в первичный ключ неявно как последнее поле

Вы используете ту же логику, когда пишете запросы - группируете по первичному ключу и используете умную логику, чтобы избежать строк, которые были отменены, но ещё не удалены. Давайте добавим несколько строк в таблицу `hackernews_views_vcmt`:

```sql
INSERT INTO hackernews_views_vcmt VALUES
   (1, 'ricardo', 0, 1, 1),
   (2, 'ch_fan', 0, 1, 1),
   (3, 'kenny', 0, 1, 1)
```

Теперь мы обновляем две строки и удаляем одну из них. Чтобы отменить строку, убедитесь, что вы включили предыдущий номер версии (так как он является частью первичного ключа):

```sql
INSERT INTO hackernews_views_vcmt VALUES
   (1, 'ricardo', 0, -1, 1),
   (1, 'ricardo', 50, 1, 2),
   (2, 'ch_fan', 0, -1, 1),
   (3, 'kenny', 0, -1, 1),
   (3, 'kenny', 1000, 1, 2)
```

Мы запустим тот же запрос, что и прежде, который умно добавляет и вычитает значения на основе колонки знака:

```sql
SELECT
    id,
    author,
    sum(views * sign)
FROM hackernews_views_vcmt
GROUP BY (id, author)
HAVING sum(sign) > 0
ORDER BY id ASC
```

Результат - две строки:

```response
┌─id─┬─author──┬─sum(multiply(views, sign))─┐
│  1 │ ricardo │                         50 │
│  3 │ kenny   │                       1000 │
└────┴─────────┴────────────────────────────┘
```

Давайте принудительно выполним слияние таблицы:

```sql
OPTIMIZE TABLE hackernews_views_vcmt
```

В результате должно быть только две строки:

```sql
SELECT *
FROM hackernews_views_vcmt
```

```response
┌─id─┬─author──┬─views─┬─sign─┬─version─┐
│  1 │ ricardo │    50 │    1 │       2 │
│  3 │ kenny   │  1000 │    1 │       2 │
└────┴─────────┴───────┴──────┴─────────┘
```

Таблица `VersionedCollapsingMergeTree` весьма полезна, когда вы хотите реализовать дедупликацию при вставке строк от нескольких клиентов и/или потоков.

## Почему мои строки не деседуплицируются? {#why-arent-my-rows-being-deduplicated}

Одна из причин, по которой вставленные строки могут не дедуплицироваться, заключается в том, что вы используете неидемпотентную функцию или выражение в вашем операторе `INSERT`. Например, если вы вставляете строки с колонкой `createdAt DateTime64(3) DEFAULT now()`, ваши строки гарантированно будут уникальными, потому что каждая строка будет иметь уникальное значение по умолчанию для колонки `createdAt`. Движок таблицы MergeTree / ReplicatedMergeTree не будет знать, как дедуплицировать строки, так как каждая вставленная строка будет генерировать уникальную контрольную сумму.

В этом случае вы можете указать свой собственный `insert_deduplication_token` для каждой партии строк, чтобы гарантировать, что несколько вставок одной и той же партии не приведут к повторному введению одних и тех же строк. Пожалуйста, ознакомьтесь с [документацией по `insert_deduplication_token`](/operations/settings/settings#insert_deduplication_token) для получения дополнительной информации о том, как использовать эту настройку.
