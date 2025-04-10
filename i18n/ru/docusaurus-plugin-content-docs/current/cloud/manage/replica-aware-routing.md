---
title: 'Маршрутизация с учетом реплик'
slug: /manage/replica-aware-routing
description: 'Как использовать маршрутизацию с учетом реплик для увеличения повторного использования кэша'
keywords: ['cloud', 'липкие конечные точки', 'липкие', 'конечные точки', 'липкая маршрутизация', 'маршрутизация', 'маршрутизация с учетом реплик']
---


# Маршрутизация с учетом реплик (Приватный предварительный просмотр)

Маршрутизация с учетом реплик (также известная как липкие сессии, липкая маршрутизация или привязанность сессии) использует [балансировку нагрузки с кольцевым хешированием прокси Envoy](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/load_balancing/load_balancers#ring-hash). Основная цель маршрутизации с учетом реплик — увеличить вероятность повторного использования кэша. Это не гарантирует изоляцию.

При включении маршрутизации с учетом реплик для сервиса мы允许 поддомен с подстановочным знаком над именем хоста сервиса. Для сервиса с именем хоста `abcxyz123.us-west-2.aws.clickhouse.cloud`, вы можете использовать любое имя хоста, совпадающее с `*.sticky.abcxyz123.us-west-2.aws.clickhouse.cloud`, чтобы зайти на сервис:

|Пример имен хостов|
|---|
|`aaa.sticky.abcxyz123.us-west-2.aws.clickhouse.cloud`|
|`000.sticky.abcxyz123.us-west-2.aws.clickhouse.cloud`|
|`clickhouse-is-the-best.sticky.abcxyz123.us-west-2.aws.clickhouse.cloud`|

Когда Envoy получает имя хоста, совпадающее с таким шаблоном, он вычисляет хеш маршрутизации на основе имени хоста и находит соответствующий сервер ClickHouse на кольцевом хеше, основываясь на вычисленном хеше. При условии, что нет текущих изменений в сервисе (например, перезапуск серверов, масштабирование вверх/вниз), Envoy всегда будет выбирать один и тот же сервер ClickHouse для подключения.

Обратите внимание, что оригинальное имя хоста все равно будет использовать балансировку нагрузки `LEAST_CONNECTION`, что является алгоритмом маршрутизации по умолчанию.

## Ограничения маршрутизации с учетом реплик {#limitations-of-replica-aware-routing}

### Маршрутизация с учетом реплик не гарантирует изоляцию {#replica-aware-routing-does-not-guarantee-isolation}

Любое нарушение работы сервиса, например, перезапуск пода сервера (по любой причине, такой как обновление версии, сбой, вертикальное масштабирование и т.д.), масштабирование сервера вверх / вниз, вызовет нарушение кольца хеширования маршрутизации. Это приведет к тому, что соединения с одним и тем же именем хоста будут направлены на другой под сервера.

### Маршрутизация с учетом реплик не работает из коробки с частной ссылкой {#replica-aware-routing-does-not-work-out-of-the-box-with-private-link}

Клиентам необходимо вручную добавить запись DNS, чтобы разрешение имен работало для нового шаблона имени хоста. Возможно, это может вызвать дисбаланс в загрузке серверов, если клиенты используют его неправильно.

## Настройка маршрутизации с учетом реплик {#configuring-replica-aware-routing}

Чтобы включить маршрутизацию с учетом реплик, пожалуйста, свяжитесь с [нашей службой поддержки](https://clickhouse.com/support).
