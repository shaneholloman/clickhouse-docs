---
alias: []
description: 'Документация для формата JSONAsObject'
input_format: true
keywords: ['JSONAsObject']
output_format: false
slug: /interfaces/formats/JSONAsObject
title: 'JSONAsObject'
---

## Описание {#description}

В этом формате один объект JSON интерпретируется как одно значение [JSON](/sql-reference/data-types/newjson.md). Если входные данные содержат несколько объектов JSON (разделенных запятыми), они интерпретируются как отдельные строки. Если входные данные заключены в квадратные скобки, это интерпретируется как массив JSON.

Этот формат можно разобрать только для таблицы с одним полем типа [JSON](/sql-reference/data-types/newjson.md). Остальные столбцы должны быть установлены на [`DEFAULT`](/sql-reference/statements/create/table.md/#default) или [`MATERIALIZED`](/sql-reference/statements/create/view#materialized-view).

## Пример использования {#example-usage}

### Базовый пример {#basic-example}

```sql title="Запрос"
SET enable_json_type = 1;
CREATE TABLE json_as_object (json JSON) ENGINE = Memory;
INSERT INTO json_as_object (json) FORMAT JSONAsObject {"foo":{"bar":{"x":"y"},"baz":1}},{},{"any json stucture":1}
SELECT * FROM json_as_object FORMAT JSONEachRow;
```

```response title="Ответ"
{"json":{"foo":{"bar":{"x":"y"},"baz":"1"}}}
{"json":{}}
{"json":{"any json stucture":"1"}}
```

### Массив объектов JSON {#an-array-of-json-objects}

```sql title="Запрос"
SET enable_json_type = 1;
CREATE TABLE json_square_brackets (field JSON) ENGINE = Memory;
INSERT INTO json_square_brackets FORMAT JSONAsObject [{"id": 1, "name": "name1"}, {"id": 2, "name": "name2"}];
SELECT * FROM json_square_brackets FORMAT JSONEachRow;
```

```response title="Ответ"
{"field":{"id":"1","name":"name1"}}
{"field":{"id":"2","name":"name2"}}
```

### Столбцы с значениями по умолчанию {#columns-with-default-values}

```sql title="Запрос"
SET enable_json_type = 1;
CREATE TABLE json_as_object (json JSON, time DateTime MATERIALIZED now()) ENGINE = Memory;
INSERT INTO json_as_object (json) FORMAT JSONAsObject {"foo":{"bar":{"x":"y"},"baz":1}};
INSERT INTO json_as_object (json) FORMAT JSONAsObject {};
INSERT INTO json_as_object (json) FORMAT JSONAsObject {"any json stucture":1}
SELECT time, json FROM json_as_object FORMAT JSONEachRow
```

```response title="Ответ"
{"time":"2024-09-16 12:18:10","json":{}}
{"time":"2024-09-16 12:18:13","json":{"any json stucture":"1"}}
{"time":"2024-09-16 12:18:08","json":{"foo":{"bar":{"x":"y"},"baz":"1"}}}
```

## Настройки формата {#format-settings}
