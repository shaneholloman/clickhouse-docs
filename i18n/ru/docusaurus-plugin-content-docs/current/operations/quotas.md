---
description: 'Руководство по настройке и управлению пользовательскими квотами в ClickHouse'
sidebar_label: 'Квоты'
sidebar_position: 51
slug: /operations/quotas
title: 'Квоты'
---

:::note Квоты в ClickHouse Cloud
Квоты поддерживаются в ClickHouse Cloud, но должны создаваться с использованием [DDL синтаксиса](/sql-reference/statements/create/quota). Подход с конфигурацией XML, описанный ниже, **не поддерживается**.
:::

Квоты позволяют ограничивать использование ресурсов в течение определенного периода времени или отслеживать использование ресурсов.  
Квоты настраиваются в конфигурации пользователя, которая, как правило, является 'users.xml'.

Система также имеет возможность ограничивать сложность одного запроса. См. раздел [Ограничения на сложность запросов](../operations/settings/query-complexity.md).

В отличие от ограничений на сложность запросов, квоты:

- Накладывают ограничения на набор запросов, которые могут быть выполнены в течение определенного времени, вместо того чтобы ограничивать один запрос.
- Учитывают ресурсы, потраченные на всех удаленных серверах при распределенной обработке запросов.

Рассмотрим раздел файла 'users.xml', который определяет квоты.

```xml
<!-- Квоты -->
<quotas>
    <!-- Имя квоты. -->
    <default>
        <!-- Ограничения для временного периода. Вы можете установить много интервалов с разными ограничениями. -->
        <interval>
            <!-- Длина интервала. -->
            <duration>3600</duration>

            <!-- Неограниченно. Просто собирайте данные за указанный временной интервал. -->
            <queries>0</queries>
            <query_selects>0</query_selects>
            <query_inserts>0</query_inserts>
            <errors>0</errors>
            <result_rows>0</result_rows>
            <read_rows>0</read_rows>
            <execution_time>0</execution_time>
        </interval>
    </default>
```

По умолчанию квота отслеживает потребление ресурсов каждый час, не ограничивая использование.  
Потребление ресурсов, рассчитанное для каждого интервала, выводится в журнал сервера после каждого запроса.

```xml
<statbox>
    <!-- Ограничения для временного периода. Вы можете установить много интервалов с разными ограничениями. -->
    <interval>
        <!-- Длина интервала. -->
        <duration>3600</duration>

        <queries>1000</queries>
        <query_selects>100</query_selects>
        <query_inserts>100</query_inserts>
        <errors>100</errors>
        <result_rows>1000000000</result_rows>
        <read_rows>100000000000</read_rows>
        <execution_time>900</execution_time>
    </interval>

    <interval>
        <duration>86400</duration>

        <queries>10000</queries>
        <query_selects>10000</query_selects>
        <query_inserts>10000</query_inserts>
        <errors>1000</errors>
        <result_rows>5000000000</result_rows>
        <read_rows>500000000000</read_rows>
        <execution_time>7200</execution_time>
    </interval>
</statbox>
```

Для квоты 'statbox' ограничения установлены на каждый час и каждые 24 часа (86,400 секунд). Временной интервал считается, начиная с фиксированного момента времени, определенного реализацией. Другими словами, 24-часовой интервал не обязательно начинается с полуночи.

Когда интервал заканчивается, все собранные значения очищаются. Для следующего часа расчет квоты начинается заново.

Вот что можно ограничить:

`queries` – Общее количество запросов.

`query_selects` – Общее количество запросов select.

`query_inserts` – Общее количество запросов insert.

`errors` – Количество запросов, которые вызвали исключение.

`result_rows` – Общее количество строк, возвращенных в результате.

`read_rows` – Общее количество исходных строк, прочитанных из таблиц для выполнения запроса на всех удаленных серверах.

`execution_time` – Общее время выполнения запроса в секундах (время на стенде).

Если предел превышен хотя бы для одного временного интервала, выбрасывается исключение с сообщением о том, какое ограничение было превышено, за какой интервал и когда начинается новый интервал (когда запросы можно будет отправлять снова).

Квоты могут использовать функцию "quota key", чтобы независимо отслеживать ресурсы для нескольких ключей. Вот пример этого:

```xml
<!-- Для глобального конструктора отчетов. -->
<web_global>
    <!-- keyed – Ключ квоты "key" передается в параметре запроса,
            и квота отслеживается отдельно для каждого значения ключа.
        Например, вы можете передать имя пользователя в качестве ключа,
            чтобы квота считалась отдельно для каждого имени пользователя.
        Использование ключей имеет смысл только в том случае, если quota_key передается программой, а не пользователем.

        Вы также можете написать <keyed_by_ip />, так что IP-адрес будет использоваться в качестве ключа квоты.
        (Но учитывайте, что пользователи могут довольно легко изменить адрес IPv6.)
    -->
    <keyed />
```

Квота назначается пользователям в разделе 'users' конфигурации. См. раздел "Права доступа".

Для распределенной обработки запросов накопленные суммы хранятся на сервере запросов. Так что если пользователь переходит на другой сервер, квота там "начнется заново".

Когда сервер перезапускается, квоты сбрасываются.

## Связанный контент {#related-content}

- Блог: [Создание одностраничных приложений с ClickHouse](https://clickhouse.com/blog/building-single-page-applications-with-clickhouse-and-http)
