---
slug: /integrations/qstudio
sidebar_label: QStudio
description: 'QStudio - это бесплатный инструмент SQL.'
---
import ConnectionDetails from '@site/i18n/ru/docusaurus-plugin-content-docs/current/_snippets/_gather_your_details_http.mdx';
import qstudio_add_connection from '@site/static/images/integrations/sql-clients/qstudio-add-connection.png';
import qstudio_running_query from '@site/static/images/integrations/sql-clients/qstudio-running-query.png';

QStudio - это бесплатный графический интерфейс SQL, который позволяет запускать SQL-скрипты, легко просматривать таблицы, строить графики и экспортировать результаты. Он работает на любой операционной системе с любой базой данных.


# Подключение QStudio к ClickHouse

QStudio подключается к ClickHouse с использованием JDBC.

## 1. Соберите ваши данные ClickHouse {#1-gather-your-clickhouse-details}

QStudio использует JDBC через HTTP(S) для подключения к ClickHouse; вам нужны:

- конечная точка
- номер порта
- имя пользователя
- пароль

<ConnectionDetails />

## 2. Скачайте QStudio {#2-download-qstudio}

QStudio доступен по адресу https://www.timestored.com/qstudio/download/

## 3. Добавьте базу данных {#3-add-a-database}

- При первом открытии QStudio нажмите на меню **Сервер->Добавить сервер** или на кнопку добавления сервера на панели инструментов.
- Затем задайте детали:

<img src={qstudio_add_connection} alt="Настройка новой базы данных" />

1.   Тип сервера: Clickhouse.com
2.   Обратите внимание, что для хоста нужно УБЕДИТЬСЯ, что вы включили https://
    Хост: https://abc.def.clickhouse.cloud
    Порт: 8443
3.   Имя пользователя: default
    Пароль: `XXXXXXXXXXX`
4. Нажмите Добавить

Если QStudio обнаружит, что у вас не установлен драйвер JDBC для ClickHouse, он предложит скачать его для вас:

## 4. Запрос к ClickHouse {#4-query-clickhouse}

- Откройте редактор запросов и выполните запрос. Вы можете выполнять запросы с помощью 
- Ctrl + e - Выполняет выделенный текст
- Ctrl + Enter - Выполняет текущую строку

- Пример запроса:

<img src={qstudio_running_query} alt="Пример запроса" />

## Следующие шаги {#next-steps}

Посмотрите [QStudio](https://www.timestored.com/qstudio), чтобы узнать о возможностях QStudio, а также [документацию ClickHouse](https://clickhouse.com/docs), чтобы узнать о возможностях ClickHouse.
