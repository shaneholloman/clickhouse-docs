---
description: 'Документация для утилиты клиента ClickHouse Keeper'
sidebar_label: 'clickhouse-keeper-client'
slug: /operations/utilities/clickhouse-keeper-client
title: 'Утилита clickhouse-keeper-client'
---

# Утилита clickhouse-keeper-client

Клиентское приложение для взаимодействия с clickhouse-keeper по его собственному протоколу.

## Ключи {#clickhouse-keeper-client}

-   `-q QUERY`, `--query=QUERY` — Запрос для выполнения. Если этот параметр не передан, `clickhouse-keeper-client` начнет в интерактивном режиме.
-   `-h HOST`, `--host=HOST` — Хост сервера. Значение по умолчанию: `localhost`.
-   `-p N`, `--port=N` — Порт сервера. Значение по умолчанию: 9181
-   `-c FILE_PATH`, `--config-file=FILE_PATH` — Установите путь к файлу конфигурации для получения строки подключения. Значение по умолчанию: `config.xml`.
-   `--connection-timeout=TIMEOUT` — Установите таймаут подключения в секундах. Значение по умолчанию: 10s.
-   `--session-timeout=TIMEOUT` — Установите таймаут сессии в секундах. Значение по умолчанию: 10s.
-   `--operation-timeout=TIMEOUT` — Установите таймаут операции в секундах. Значение по умолчанию: 10s.
-   `--history-file=FILE_PATH` — Установите путь к файлу истории. Значение по умолчанию: `~/.keeper-client-history`.
-   `--log-level=LEVEL` — Установите уровень логирования. Значение по умолчанию: `information`.
-   `--no-confirmation` — Если установлено, не требуется подтверждение по нескольким командам. Значение по умолчанию `false` для интерактивного режима и `true` для запроса.
-   `--help` — Показывает справочное сообщение.

## Пример {#clickhouse-keeper-client-example}

```bash
./clickhouse-keeper-client -h localhost -p 9181 --connection-timeout 30 --session-timeout 30 --operation-timeout 30
Connected to ZooKeeper at [::1]:9181 with session_id 137
/ :) ls
keeper foo bar
/ :) cd 'keeper'
/keeper :) ls
api_version
/keeper :) cd 'api_version'
/keeper/api_version :) ls

/keeper/api_version :) cd 'xyz'
Path /keeper/api_version/xyz does not exist
/keeper/api_version :) cd ../../
/ :) ls
keeper foo bar
/ :) get 'keeper/api_version'
2
```

## Команды {#clickhouse-keeper-client-commands}

-   `ls '[path]'` -- Перечисляет узлы для указанного пути (по умолчанию: cwd)
-   `cd '[path]'` -- Изменяет рабочий путь (по умолчанию `.`)
-   `cp '<src>' '<dest>'`  -- Копирует узел 'src' в путь 'dest'
-   `mv '<src>' '<dest>'`  -- Перемещает узел 'src' в путь 'dest'
-   `exists '<path>'` -- Возвращает `1`, если узел существует, `0` в противном случае
-   `set '<path>' <value> [version]` -- Обновляет значение узла. Обновления происходят только если версия совпадает (по умолчанию: -1)
-   `create '<path>' <value> [mode]` -- Создает новый узел с заданным значением
-   `touch '<path>'` -- Создает новый узел с пустой строкой в качестве значения. Не вызывает исключение, если узел уже существует
-   `get '<path>'` -- Возвращает значение узла
-   `rm '<path>' [version]` -- Удаляет узел только если версия совпадает (по умолчанию: -1)
-   `rmr '<path>' [limit]` -- Рекурсивно удаляет путь, если размер поддерева меньше лимита. Требуется подтверждение (лимит по умолчанию = 100)
-   `flwc <command>` -- Выполняет команду из четырех букв
-   `help` -- Печатает это сообщение
-   `get_direct_children_number '[path]'` -- Получает количество прямых дочерних узлов под указанным путем
-   `get_all_children_number '[path]'` -- Получает общее количество дочерних узлов под указанным путем
-   `get_stat '[path]'` -- Возвращает статистику узла (по умолчанию `.`)
-   `find_super_nodes <threshold> '[path]'` -- Находит узлы с количеством дочерних узлов, превышающим некоторый порог для данного пути (по умолчанию `.`)
-   `delete_stale_backups` -- Удаляет узлы ClickHouse, использовавшиеся для резервных копий, которые теперь неактивны
-   `find_big_family [path] [n]` -- Возвращает топ n узлов с самой большой семью в поддереве (путь по умолчанию = `.` и n = 10)
-   `sync '<path>'` -- Синхронизирует узел между процессами и лидером
-   `reconfig <add|remove|set> "<arg>" [version]` -- Перенастраивает кластер Keeper. См. /docs/en/guides/sre/keeper/clickhouse-keeper#reconfiguration
