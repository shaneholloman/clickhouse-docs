---
description: 'Документация для оператора SAMPLE'
sidebar_label: 'SAMPLE'
slug: /sql-reference/statements/select/sample
title: 'Оператор SAMPLE'
---


# Оператор SAMPLE

Оператор `SAMPLE` позволяет выполнять обработку запросов `SELECT` с приближёнными данными.

Когда включено выборочное извлечение данных, запрос выполняется не на всех данных, а только на определённой их части (выборке). Например, если вам нужно рассчитать статистику по всем визитам, достаточно выполнить запрос на 1/10 всех визитов и затем умножить результат на 10.

Приближённая обработка запросов может быть полезной в следующих случаях:

- Когда у вас строгие требования по задержке (например, менее 100 мс), но вы не можете обосновать затраты на дополнительные аппаратные ресурсы для их соблюдения.
- Когда ваши исходные данные неточные, поэтому аппроксимация не ухудшает качество на заметное значение.
- Бизнес-требования требуют приблизительных результатов (для экономии затрат или для предложения точных результатов пользователям премиум-класса).

:::note    
Вы можете использовать выборку только с таблицами из семейства [MergeTree](../../../engines/table-engines/mergetree-family/mergetree.md), и только если выражение выборки было указано во время создания таблицы (см. [движок MergeTree](../../../engines/table-engines/mergetree-family/mergetree.md#table_engine-mergetree-creating-a-table)).
:::

Особенности выборочного извлечения данных перечислены ниже:

- Выборка данных является детерминированным механизмом. Результат одного и того же запроса `SELECT .. SAMPLE` всегда одинаков.
- Выборка работает последовательно для разных таблиц. Для таблиц с единственным ключом выборки выборка с одинаковым коэффициентом всегда выбирает один и тот же подмассив возможных данных. Например, выборка идентификаторов пользователей берёт строки с одним и тем же подмножеством всех возможных идентификаторов пользователей из разных таблиц. Это означает, что вы можете использовать выборку в подзапросах в операторе [IN](../../../sql-reference/operators/in.md). Также вы можете объединять выборки с помощью оператора [JOIN](../../../sql-reference/statements/select/join.md).
- Выборка позволяет читать меньше данных с диска. Учтите, что вы должны правильно указать ключ выборки. Для получения дополнительной информации см. [Создание таблицы MergeTree](../../../engines/table-engines/mergetree-family/mergetree.md#table_engine-mergetree-creating-a-table).

Для оператора `SAMPLE` поддерживается следующий синтаксис:

| Синтаксис оператора SAMPLE | Описание                                                                                                                                                                                                                                    |
|-----------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `SAMPLE k`                  | Здесь `k` — число от 0 до 1. Запрос выполняется на `k` доле данных. Например, `SAMPLE 0.1` выполняет запрос на 10% данных. [Читать далее](#sample-k)                                                                                     |
| `SAMPLE n`                  | Здесь `n` — достаточно большое целое число. Запрос выполняется на выборке не менее `n` строк (но не значительно больше этого). Например, `SAMPLE 10000000` выполняет запрос на минимум 10,000,000 строк. [Читать далее](#sample-n) |
| `SAMPLE k OFFSET m`        | Здесь `k` и `m` — числа от 0 до 1. Запрос выполняется на выборке `k` доли данных. Данные, используемые для выборки, смещены на `m` долю. [Читать далее](#sample-k-offset-m)                                                                   |


## SAMPLE K {#sample-k}

Здесь `k` — число от 0 до 1 (поддерживаются как дробные, так и десятичные записи). Например, `SAMPLE 1/2` или `SAMPLE 0.5`.

В операторе `SAMPLE k` выборка берётся из `k` доли данных. Пример показан ниже:

```sql
SELECT
    Title,
    count() * 10 AS PageViews
FROM hits_distributed
SAMPLE 0.1
WHERE
    CounterID = 34
GROUP BY Title
ORDER BY PageViews DESC LIMIT 1000
```

В этом примере запрос выполняется на выборке из 0.1 (10%) данных. Значения агрегатных функций не исправляются автоматически, поэтому для получения приблизительного результата значение `count()` умножается вручную на 10.

## SAMPLE N {#sample-n}

Здесь `n` — достаточно большое целое число. Например, `SAMPLE 10000000`.

В этом случае запрос выполняется на выборке не менее `n` строк (но не значительно больше этого). Например, `SAMPLE 10000000` выполняет запрос на минимум 10,000,000 строк.

Поскольку минимальной единицей для чтения данных является одна гранула (её размер задаётся настройкой `index_granularity`), имеет смысл устанавливать выборку, которая значительно больше размера гранулы.

При использовании оператора `SAMPLE n` вы не знаете, какой относительный процент данных был обработан. Поэтому вы не знаете, на какой коэффициент агрегатные функции должны быть умножены. Используйте виртуальную колонку `_sample_factor`, чтобы получить приблизительный результат.

Колонка `_sample_factor` содержит относительные коэффициенты, которые вычисляются динамически. Эта колонка создаётся автоматически, когда вы [создаёте](../../../engines/table-engines/mergetree-family/mergetree.md#table_engine-mergetree-creating-a-table) таблицу с указанным ключом выборки. Примеры использования колонки `_sample_factor` показаны ниже.

Рассмотрим таблицу `visits`, которая содержит статистику по визитам на сайт. Первый пример показывает, как рассчитать количество просмотров страниц:

```sql
SELECT sum(PageViews * _sample_factor)
FROM visits
SAMPLE 10000000
```

Следующий пример показывает, как рассчитать общее количество визитов:

```sql
SELECT sum(_sample_factor)
FROM visits
SAMPLE 10000000
```

Следующий пример показывает, как рассчитать среднюю продолжительность сессии. Обратите внимание, что вам не нужно использовать относительный коэффициент для вычисления средних значений.

```sql
SELECT avg(Duration)
FROM visits
SAMPLE 10000000
```

## SAMPLE K OFFSET M {#sample-k-offset-m}

Здесь `k` и `m` — числа от 0 до 1. Примеры показаны ниже.

**Пример 1**

```sql
SAMPLE 1/10
```

В этом примере выборка составляет 1/10 всех данных:

`[++------------]`

**Пример 2**

```sql
SAMPLE 1/10 OFFSET 1/2
```

Здесь выборка в 10% берётся из второй половины данных.

`[------++------]`
