---
slug: /best-practices/selecting-an-insert-strategy
sidebar_position: 10
sidebar_label: 'Выбор стратегии вставки'
title: 'Выбор стратегии вставки'
description: 'Страница, описывающая, как выбрать стратегию вставки в ClickHouse'
---

import Image from '@theme/IdealImage';
import insert_process from '@site/static/images/bestpractices/insert_process.png';
import async_inserts from '@site/static/images/bestpractices/async_inserts.png';
import AsyncInserts from '@site/docs/best-practices/_snippets/_async_inserts.md';
import BulkInserts from '@site/docs/best-practices/_snippets/_bulk_inserts.md';

Эффективный прием данных является основой высокопроизводительных развертываний ClickHouse. Выбор правильной стратегии вставки может значительно повлиять на пропускную способность, стоимость и надежность. Этот раздел описывает лучшие практики, компромиссы и варианты конфигурации, которые помогут вам принять правильное решение для вашей рабочей нагрузки.

:::note
Следующее предполагает, что вы передаете данные в ClickHouse через клиент. Если вы загружаете данные в ClickHouse, например, с помощью встроенных табличных функций, таких как [s3](/sql-reference/table-functions/s3) и [gcs](/sql-reference/table-functions/gcs), мы рекомендуем ознакомиться с нашим руководством ["Оптимизация производительности вставок и чтений для S3"](/integrations/s3/performance).
:::

## Синхронные вставки по умолчанию {#synchronous-inserts-by-default}

По умолчанию вставки в ClickHouse являются синхронными. Каждый запрос вставки немедленно создает часть хранения на диске, включая метаданные и индексы.

:::note Используйте синхронные вставки, если вы можете пакетировать данные на стороне клиента
Если нет, смотрите [Асинхронные вставки](#asynchronous-inserts) ниже.
:::

 Мы кратко рассмотрим механику вставок в MergeTree ClickHouse ниже:

<Image img={insert_process} size="lg" alt="Процессы вставки" background="black"/>

#### Шаги на стороне клиента {#client-side-steps}

Для достижения оптимальной производительности данные должны быть ①[пакетированы](https://clickhouse.com/blog/asynchronous-data-inserts-in-clickhouse#data-needs-to-be-batched-for-optimal-performance), что делает размер пакета **первым решением**.

ClickHouse хранит вставленные данные на диске,[упорядоченные](/guides/best-practices/sparse-primary-indexes#data-is-stored-on-disk-ordered-by-primary-key-columns) по столбцу(ам) первичного ключа таблицы. **Второе решение** заключается в том, чтобы ② предварительно отсортировать данные перед передачей серверу. Если пакет приходит предварительно отсортированным по столбцу(ам) первичного ключа, ClickHouse может [пропустить](https://github.com/ClickHouse/ClickHouse/blob/94ce8e95404e991521a5608cd9d636ff7269743d/src/Storages/MergeTree/MergeTreeDataWriter.cpp#L595) ⑨ шаг сортировки, ускоряя прием.

Если данные для вставки не имеют предопределенного формата, **ключевое решение** заключается в выборе формата. ClickHouse поддерживает вставку данных в [более чем 70 форматах](/interfaces/formats). Однако при использовании клиентской командной строки ClickHouse или клиентов на языках программирования этот выбор часто обрабатывается автоматически. При необходимости этот автоматический выбор также можно явно переопределить.

Следующее **значительное решение** — это ④ следует ли сжимать данные до передачи серверу ClickHouse. Сжатие уменьшает размер передачи и улучшает сетевую эффективность, что приводит к более быстрому трансферу данных и меньшему использованию пропускной способности, особенно для больших наборов данных.

Данные ⑤ передаются на сетевой интерфейс ClickHouse - либо на [native](/interfaces/tcp), либо на [HTTP](/interfaces/http) интерфейс (который мы [сравниваем](https://clickhouse.com/blog/clickhouse-input-format-matchup-which-is-fastest-most-efficient#clickhouse-client-defaults) позже в этом посте).

#### Шаги на стороне сервера {#server-side-steps}

После ⑥ получения данных ClickHouse ⑦ распаковывает их, если сжатие использовалось, затем ⑧ разбирает их из исходного формата.

Используя значения из этих отформатированных данных и оператор [DDL](/sql-reference/statements/create/table) целевой таблицы, ClickHouse ⑨ строит в памяти [блок](/development/architecture#block) в формате MergeTree, ⑩ [сортирует](/parts#what-are-table-parts-in-clickhouse) строки по столбцам первичного ключа, если они еще не отсортированы, ⑪ создает [разреженный первичный индекс](/guides/best-practices/sparse-primary-indexes), ⑫ применяет [сжатие по столбцам](/parts#what-are-table-parts-in-clickhouse) и ⑬ записывает данные как новую ⑭ [часть данных](/parts) на диск.


### Пакетные вставки, если синхронные {#batch-inserts-if-synchronous}

<BulkInserts/>

### Обеспечьте идемпотентные повторные попытки {#ensure-idempotent-retries}

Синхронные вставки также являются **идемпотентными**. При использовании движков MergeTree ClickHouse будет дедуплицировать вставки по умолчанию. Это защищает от неоднозначных случаев сбоев, таких как:

* Вставка прошла успешно, но клиент никогда не получил подтверждение из-за сетевого прерывания.
* Вставка завершилась неудачей на стороне сервера и превысила время ожидания.

В обоих случаях безопасно **повторить вставку** - при условии, что содержимое и порядок пакета остаются идентичными. По этой причине критически важно, чтобы клиенты повторяли попытки последовательно, не изменяя и не перекрывая данные.

### Выберите правильное место вставки {#choose-the-right-insert-target}

Для шардированных кластеров у вас есть два варианта:

* Вставьте напрямую в таблицу **MergeTree** или **ReplicatedMergeTree**. Это наиболее эффективный вариант, когда клиент может выполнять балансировку нагрузки между шардми. При `internal_replication = true` ClickHouse обрабатывает репликацию прозрачно.
* Вставка в [Распределенную таблицу](/engines/table-engines/special/distributed). Это позволяет клиентам отправлять данные на любой узел и позволить ClickHouse перенаправить их на правильный шард. Это проще, но немного менее эффективно из-за дополнительного шага перенаправления. Рекомендуется также `internal_replication = true`.

**В ClickHouse Cloud все узлы читают и записывают в один и тот же шард. Вставки автоматически сбалансированы между узлами. Пользователи могут просто отправлять вставки на открытый конечный пункт.**

### Выберите правильный формат {#choose-the-right-format}

Выбор правильного формата ввода имеет решающее значение для эффективного приема данных в ClickHouse. С более чем 70 поддерживаемыми форматами выбор наиболее производительного варианта может значительно повлиять на скорость вставки, использование CPU и памяти, а также на общую эффективность системы. 

Хотя гибкость полезна для обработки данных и импорта на основе файлов, **приложения должны приоритизировать форматы, ориентированные на производительность**:

* **Нативный формат** (рекомендуется): Наиболее эффективный. Столбцовый, требует минимального разбора на стороне сервера. Используется по умолчанию в клиентах Go и Python.
* **RowBinary**: Эффективный строковый формат, идеален, если столбцовая трансформация сложна на стороне клиента. Используется клиентом Java.
* **JSONEachRow**: Легок в использовании, но дорогостоящ в разборе. Подходит для случаев с низким объемом данных или быстрых интеграций.

### Используйте сжатие {#use-compression}

Сжатие играет ключевую роль в снижении сетевой нагрузки, ускорении вставок и снижении затрат на хранение в ClickHouse. При эффективном использовании оно повышает производительность приема данных без необходимости в изменениях формата или схемы данных.

Сжатие данных вставки уменьшает размер полезной нагрузки, отправляемой по сети, минимизируя использование пропускной способности и ускоряя передачу.

Для вставок сжатие особенно эффективно при использовании Нативного формата, который уже соответствует внутренней столбцовой модели хранения ClickHouse. В этом настройке сервер может эффективно распаковывать и напрямую сохранять данные с минимальным преобразованием.

#### Используйте LZ4 для скорости, ZSTD для соотношения сжатия {#use-lz4-for-speed-zstd-for-compression-ratio}

ClickHouse поддерживает несколько кодеков сжатия во время передачи данных. Две распространенные опции:

* **LZ4**: Быстрый и легковесный. Существенно уменьшает размер данных с минимальными накладными расходами на CPU, что делает его идеальным для вставок с высокой пропускной способностью и по умолчанию в большинстве клиентов ClickHouse.
* **ZSTD**: Более высокое соотношение сжатия, но более ресурсоемкий для CPU. Полезно, когда затраты на передачу по сети высоки — такие как в сценариях с межрегиональной передачей или при использовании облачных провайдеров, хотя это немного увеличивает время вычислений на стороне клиента и время распаковки на стороне сервера.

Рекомендация по лучшим практикам: Используйте LZ4, если только у вас нет ограниченной пропускной способности или если вы не несете затраты на исходящие данные — тогда рассмотрите возможность использования ZSTD.

:::note
В тестах из [бенчмарка FastFormats](https://clickhouse.com/blog/clickhouse-input-format-matchup-which-is-fastest-most-efficient) вставки, сжатые с помощью LZ4, уменьшили размер данных более чем на 50%, сократив время приема с 150s до 131s для набора данных размером 5.6 GiB. Переход на ZSTD сжал тот же набор данных до 1.69 GiB, но немного увеличил время обработки на стороне сервера.
:::

#### Сжатие снижает использование ресурсов {#compression-reduces-resource-usage}

Сжатие не только снижает сетевой трафик — оно также улучшает эффективность CPU и памяти на сервере. Сжатые данные позволяют ClickHouse получать меньше байтов и тратить меньше времени на разбор больших входных данных. Это преимущество особенно важно при приеме данных от нескольких одновременно работающих клиентов, таких как в сценариях мониторинга.

Влияние сжатия на CPU и память невелико для LZ4 и умеренное для ZSTD. Даже под нагрузкой эффективность на стороне сервера улучшается за счет уменьшенного объема данных.

**Комбинирование сжатия с пакетированием и эффективным форматом ввода (таким как Нативный) приводит к лучшей производительности приема.**

При использовании нативного интерфейса (например, [clickhouse-client](/interfaces/cli)) сжатие LZ4 включает по умолчанию. Вы также можете при необходимости переключиться на ZSTD через настройки.

С [HTTP интерфейсом](/interfaces/http) используйте заголовок Content-Encoding, чтобы применить сжатие (например, Content-Encoding: lz4). Вся полезная нагрузка должна быть сжата перед отправкой.

### Предварительная сортировка при низкой стоимости {#pre-sort-if-low-cost}

Предварительная сортировка данных по первичному ключу перед вставкой может повысить эффективность приема в ClickHouse, особенно для больших пакетов. 

Когда данные приходят предварительно отсортированными, ClickHouse может пропустить или упростить внутренний шаг сортировки при создании части, снижая использование CPU и ускоряя процесс вставки. Предварительная сортировка также улучшает эффективность сжатия, поскольку похожие значения группируются вместе - позволяя кодекам, таким как LZ4 или ZSTD, достигать лучшего соотношения сжатия. Это особенно полезно в сочетании с крупными пакетными вставками и сжатием, поскольку снижает как затраты на обработку, так и объем передаваемых данных.

**Тем не менее, предварительная сортировка является необязательной оптимизацией — не требованием.** ClickHouse сортирует данные очень эффективно, используя параллельную обработку, и во многих случаях сортировка на стороне сервера быстрее или удобнее, чем предварительная сортировка на стороне клиента. 

**Мы рекомендуем предварительную сортировку только в том случае, если данные уже почти отсортированы или если ресурсы на стороне клиента (CPU, память) достаточны и недостаточно используются.** В ситуациях, чувствительных к задержкам или с высокой пропускной способностью, таких как мониторинг, когда данные приходят не в порядке или от множества агентов, часто лучше пропустить предварительную сортировку и полагаться на встроенную производительность ClickHouse.

## Асинхронные вставки {#asynchronous-inserts}

<AsyncInserts />

## Выберите интерфейс - HTTP или Native {#choose-an-interface}

### Native {#choose-an-interface-native}

ClickHouse предлагает два основных интерфейса для приема данных: **нативный интерфейс** и **HTTP интерфейс** - каждый из которых имеет свои компромиссы между производительностью и гибкостью. Нативный интерфейс, используемый [clickhouse-client](/interfaces/cli) и некоторыми языковыми клиентами, такими как Go и C++, создан специально для производительности. Он всегда передает данные в высокоэффективном Нативном формате ClickHouse, поддерживает пакетное сжатие с помощью LZ4 или ZSTD и минимизирует обработку на стороне сервера, перегружая такие задачи, как разбор и преобразование формата, на клиента. 

Он даже позволяет выполнять вычисления на стороне клиента для значений MATERIALIZED и DEFAULT столбцов, позволяя серверу полностью пропускать эти шаги. Это делает нативный интерфейс идеальным для сценариев с высокой пропускной способностью, где эффективность критически важна.

### HTTP {#choose-an-interface-http}

В отличие от многих традиционных баз данных, ClickHouse также поддерживает HTTP интерфейс. **Это, наоборот, приоритизирует совместимость и гибкость.** Он позволяет отправлять данные в [любом поддерживаемом формате](/integrations/data-formats) - включая JSON, CSV, Parquet и другие - и широко поддерживается большинством клиентов ClickHouse, включая Python, Java, JavaScript и Rust. 

Это часто предпочтительнее нативного протокола ClickHouse, так как позволяет легко переключать трафик с помощью балансировщиков нагрузки. Мы ожидаем небольшие различия в производительности вставок с нативным протоколом, который имеет немного меньшие накладные расходы.

Тем не менее, он лишен более глубокой интеграции нативного протокола и не может выполнять оптимизации на стороне клиента, такие как вычисление значений материализованных или автоматическое преобразование в Нативный формат. Хотя вставки через HTTP можно сжимать с использованием стандартных HTTP заголовков (например, `Content-Encoding: lz4`), сжатие применяется ко всей полезной нагрузке, а не к отдельным блокам данных. Этот интерфейс часто предпочитается в средах, где простота протокола, балансировка нагрузки или широкая совместимость форматов важнее, чем сырая производительность.

Для более подробного описания этих интерфейсов смотрите [здесь](/interfaces/overview).
