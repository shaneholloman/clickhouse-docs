---
slug: /engines/table-engines/mergetree-family/replication
sidebar_position: 20
sidebar_label: Репликация данных
title: 'Репликация данных'
description: 'Обзор репликации данных в ClickHouse'
---


# Репликация данных

:::note
В ClickHouse Cloud репликация управляется для вас. Пожалуйста, создавайте ваши таблицы без добавления аргументов. Например, в тексте ниже вы замените:

```sql
ENGINE = ReplicatedMergeTree(
    '/clickhouse/tables/{shard}/table_name',
    '{replica}',
    ver
)
```

на:

```sql
ENGINE = ReplicatedMergeTree
```
:::

Репликация поддерживается только для таблиц семейства MergeTree:

- ReplicatedMergeTree
- ReplicatedSummingMergeTree
- ReplicatedReplacingMergeTree
- ReplicatedAggregatingMergeTree
- ReplicatedCollapsingMergeTree
- ReplicatedVersionedCollapsingMergeTree
- ReplicatedGraphiteMergeTree

Репликация работает на уровне отдельной таблицы, а не всего сервера. Сервер может одновременно хранить как реплицированные, так и нереплицированные таблицы.

Репликация не зависит от шардирования. Каждый шард имеет свою независимую репликацию.

Сжатые данные для `INSERT` и `ALTER` запросов реплицируются (для получения дополнительной информации см. документацию по [ALTER](/sql-reference/statements/alter)).

Запросы `CREATE`, `DROP`, `ATTACH`, `DETACH` и `RENAME` выполняются на одном сервере и не реплицируются:

- Запрос `CREATE TABLE` создает новую реплицируемую таблицу на сервере, где выполняется запрос. Если эта таблица уже существует на других серверах, добавляется новая реплика.
- Запрос `DROP TABLE` удаляет реплику, расположенную на сервере, где выполняется запрос.
- Запрос `RENAME` переименовывает таблицу на одной из реплик. Другими словами, реплицированные таблицы могут иметь разные названия на разных репликах.

ClickHouse использует [ClickHouse Keeper](/guides/sre/keeper/index.md) для хранения метаинформации о репликах. Возможно использовать ZooKeeper версии 3.4.5 или новее, но рекомендуется ClickHouse Keeper.

Чтобы использовать репликацию, установите параметры в разделе конфигурации сервера [zookeeper](/operations/server-configuration-parameters/settings#zookeeper).

:::note
Не пренебрегайте настройками безопасности. ClickHouse поддерживает `digest` [схему ACL](https://zookeeper.apache.org/doc/current/zookeeperProgrammers.html#sc_ZooKeeperAccessControl) подсистемы безопасности ZooKeeper.
:::

Пример установки адресов кластера ClickHouse Keeper:

``` xml
<zookeeper>
    <node>
        <host>example1</host>
        <port>2181</port>
    </node>
    <node>
        <host>example2</host>
        <port>2181</port>
    </node>
    <node>
        <host>example3</host>
        <port>2181</port>
    </node>
</zookeeper>
```

ClickHouse также поддерживает хранение метаинформации о репликах в вспомогательном кластере ZooKeeper. Сделайте это, предоставив имя кластера ZooKeeper и путь в качестве аргументов двигателя.
Другими словами, поддерживается хранение метаданных различных таблиц в разных кластерах ZooKeeper.

Пример установки адресов вспомогательного кластера ZooKeeper:

``` xml
<auxiliary_zookeepers>
    <zookeeper2>
        <node>
            <host>example_2_1</host>
            <port>2181</port>
        </node>
        <node>
            <host>example_2_2</host>
            <port>2181</port>
        </node>
        <node>
            <host>example_2_3</host>
            <port>2181</port>
        </node>
    </zookeeper2>
    <zookeeper3>
        <node>
            <host>example_3_1</host>
            <port>2181</port>
        </node>
    </zookeeper3>
</auxiliary_zookeepers>
```

Чтобы хранить метаданные таблицы во вспомогательном кластере ZooKeeper вместо стандартного кластера ZooKeeper, мы можем использовать SQL для создания таблицы с
движком ReplicatedMergeTree следующим образом:

```sql
CREATE TABLE table_name ( ... ) ENGINE = ReplicatedMergeTree('zookeeper_name_configured_in_auxiliary_zookeepers:path', 'replica_name') ...
```
Вы можете указать любой существующий кластер ZooKeeper, и система будет использовать директорию на нем для своих собственных данных (директория указывается при создании реплицируемой таблицы).

Если ZooKeeper не задан в файле конфигурации, вы не сможете создавать реплицированные таблицы, а все существующие реплицированные таблицы будут только для чтения.

ZooKeeper не используется в запросах `SELECT`, потому что репликация не влияет на производительность `SELECT`, и запросы выполняются так же быстро, как для нереплицированных таблиц. При запросах распределенных реплицированных таблиц поведение ClickHouse контролируется настройками [max_replica_delay_for_distributed_queries](/operations/settings/settings.md/#max_replica_delay_for_distributed_queries) и [fallback_to_stale_replicas_for_distributed_queries](/operations/settings/settings.md/#fallback_to_stale_replicas_for_distributed_queries).

Для каждого запроса `INSERT` в ZooKeeper добавляется около десяти записей через несколько транзакций. (Чтобы быть более точным, это для каждого вставленного блока данных; запрос `INSERT` содержит один блок или один блок на `max_insert_block_size = 1048576` строк.) Это приводит к немного более длительным задержкам для `INSERT` по сравнению с нереплицированными таблицами. Но если вы следуете рекомендациям по вставке данных партиями не более одного `INSERT` в секунду, это не создает никаких проблем. Весь кластер ClickHouse, используемый для координации одного кластера ZooKeeper, имеет в общей сложности несколько сотен `INSERTs` в секунду. Пропускная способность вставки данных (количество строк в секунду) такая же высокая, как для нереплицированных данных.

Для очень больших кластеров вы можете использовать разные кластеры ZooKeeper для разных шардов. Однако из нашего опыта это не оказалось необходимым на основе производственных кластеров с примерно 300 серверами.

Репликация является асинхронной и многоуровневой. Запросы `INSERT` (также как и `ALTER`) могут быть отправлены на любой доступный сервер. Данные вставляются на сервер, где выполняется запрос, а затем копируются на другие серверы. Поскольку это асинхронно, недавно вставленные данные появляются на других репликах с некоторой задержкой. Если часть реплик недоступна, данные будут записаны, когда они станут доступными. Если реплика доступна, задержка — это время, необходимое для передачи блока сжатых данных по сети. Количество потоков, выполняющих фоновые задачи для реплицированных таблиц, может быть установлено с помощью настройки [background_schedule_pool_size](/operations/server-configuration-parameters/settings.md/#background_schedule_pool_size).

Движок `ReplicatedMergeTree` использует отдельный пул потоков для реплицированных выборок. Размер пула ограничен настройкой [background_fetches_pool_size](/operations/server-configuration-parameters/settings#background_fetches_pool_size), которую можно настроить только после перезагрузки сервера.

По умолчанию запрос `INSERT` ожидает подтверждения записи данных только от одной реплики. Если данные были успешно записаны только в одну реплику, а сервер с этой репликой перестает существовать, сохраненные данные будут потеряны. Чтобы включить получение подтверждения записи данных от нескольких реплик, используйте параметр `insert_quorum`.

Каждый блок данных записывается атомарно. Запрос `INSERT` делится на блоки по размеру до `max_insert_block_size = 1048576` строк. Другими словами, если запрос `INSERT` содержит менее 1048576 строк, он выполняется атомарно.

Блоки данных дескриптируются. При многократных записях одного и того же блока данных (блоки данных одинакового размера, содержащие одни и те же строки в одном и том же порядке) блок записывается только один раз. Это делается на случай сетевых сбоев, когда клиентское приложение не знает, были ли данные записаны в БД, поэтому запрос `INSERT` может просто повторяться. Не имеет значения, на какие реплики были отправлены `INSERTs` с идентичными данными. `INSERTs` являются идемпотентными. Параметры дедупликации контролируются настройками сервера [merge_tree](/operations/server-configuration-parameters/settings.md/#merge_tree).

Во время репликации только исходные данные для вставки передаются по сети. Дальнейшая трансформация данных (слияние) координируется и выполняется на всех репликах одинаково. Это минимизирует использование сети, что означает, что репликация хорошо работает, когда реплики находятся в разных центрах обработки данных. (Обратите внимание, что дублирование данных в разных центрах обработки данных является основной целью репликации.)

Вы можете иметь любое количество реплик одних и тех же данных. Исходя из нашего опыта, достаточно надежным и удобным решением может быть использование двойной репликации в производственных условиях, при этом каждое сервер использует RAID-5 или RAID-6 (и RAID-10 в некоторых случаях).

Система отслеживает синхронность данных на репликах и может восстанавливать данные после сбоя. Переключение на резервные реплики происходит автоматически (для небольших различий в данных) или полуручно (когда данные отличаются слишком сильно, что может указывать на ошибку конфигурации).

## Создание реплицированных таблиц {#creating-replicated-tables}

:::note
В ClickHouse Cloud репликация управляется для вас. Пожалуйста, создавайте ваши таблицы без добавления аргументов. Например, в тексте ниже вы замените:
```sql
ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/table_name', '{replica}', ver)
```
на:
```sql
ENGINE = ReplicatedMergeTree
```
:::

Префикс `Replicated` добавляется к имени движка таблицы. Например:`ReplicatedMergeTree`.

:::tip
Добавление `Replicated` является необязательным в ClickHouse Cloud, так как все таблицы реплицированы.
:::

### Параметры Replicated*MergeTree {#replicatedmergetree-parameters}

#### zoo_path {#zoo_path}

`zoo_path` — Путь к таблице в ClickHouse Keeper.

#### replica_name {#replica_name}

`replica_name` — Имя реплики в ClickHouse Keeper.

#### other_parameters {#other_parameters}

`other_parameters` — Параметры движка, который используется для создания реплицированной версии, например, версия в `ReplacingMergeTree`.

Пример:

``` sql
CREATE TABLE table_name
(
    EventDate DateTime,
    CounterID UInt32,
    UserID UInt32,
    ver UInt16
ENGINE = ReplicatedReplacingMergeTree('/clickhouse/tables/{layer}-{shard}/table_name', '{replica}', ver)
PARTITION BY toYYYYMM(EventDate)
ORDER BY (CounterID, EventDate, intHash32(UserID))
SAMPLE BY intHash32(UserID);
```

<details markdown="1">

<summary>Пример в устаревшем синтаксисе</summary>

``` sql
CREATE TABLE table_name
(
    EventDate DateTime,
    CounterID UInt32,
    UserID UInt32
) ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/table_name', '{replica}', EventDate, intHash32(UserID), (CounterID, EventDate, intHash32(UserID), EventTime), 8192);
```

</details>

Как показывает пример, эти параметры могут содержать подстановки в фигурных скобках. Подставленные значения берутся из раздела [macros](/operations/server-configuration-parameters/settings.md/#macros) файла конфигурации.

Пример:

``` xml
<macros>
    <shard>02</shard>
    <replica>example05-02-1</replica>
</macros>
```

Путь к таблице в ClickHouse Keeper должен быть уникальным для каждой реплицированной таблицы. Таблицы на разных шарах должны иметь разные пути.
В этом случае путь состоит из следующих частей:

`/clickhouse/tables/` — общий префикс. Мы рекомендуем использовать именно этот.

`{shard}` будет расширен до идентификатора шарда.

`table_name` — имя узла для таблицы в ClickHouse Keeper. Лучше всего, чтобы оно совпадало с именем таблицы. Оно определяется явно, поскольку в отличие от имени таблицы, оно не меняется после запроса `RENAME`.
*ПОДСказка*: вы можете также добавить имя базы данных перед `table_name`. Например, `db_name.table_name`

Две встроенные подстановки `{database}` и `{table}` могут быть использованы, они расширяются в имя таблицы и имя базы данных соответственно (если только эти макросы не определены в разделе `macros`). Таким образом, путь к zoo может быть указан как `'/clickhouse/tables/{shard}/{database}/{table}'`.
Будьте осторожны с переименованиями таблиц при использовании этих встроенных подстановок. Путь в ClickHouse Keeper не может быть изменен, и когда таблица переименовывается, макросы будут расширяться в другой путь, таблица будет ссылаться на путь, который не существует в ClickHouse Keeper, и перейдет в режим только для чтения.

Имя реплики идентифицирует разные реплики одной и той же таблицы. Вы можете использовать имя сервера для этого, как в примере. Имя должно быть уникальным внутри каждого шарда.

Вы можете явно определить параметры вместо того, чтобы использовать подстановки. Это может быть удобно для тестирования и настройки небольших кластеров. Однако в этом случае вы не можете использовать распределенные запросы DDL (`ON CLUSTER`).

При работе с большими кластерами мы рекомендуем использовать подстановки, так как они уменьшают вероятность ошибки.

Вы можете задать значения по умолчанию для движка таблицы `Replicated` в файле конфигурации сервера. Например:

```xml
<default_replica_path>/clickhouse/tables/{shard}/{database}/{table}</default_replica_path>
<default_replica_name>{replica}</default_replica_name>
```

В этом случае вы можете опустить аргументы при создании таблиц:

``` sql
CREATE TABLE table_name (
	x UInt32
) ENGINE = ReplicatedMergeTree
ORDER BY x;
```

Это эквивалентно:

``` sql
CREATE TABLE table_name (
	x UInt32
) ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/{database}/table_name', '{replica}')
ORDER BY x;
```

Запустите запрос `CREATE TABLE` на каждой реплике. Этот запрос создает новую реплицируемую таблицу или добавляет новую реплику к существующей.

Если вы добавите новую реплику после того, как у таблицы уже есть данные на других репликах, данные будут скопированы с других реплик в новую после выполнения запроса. Другими словами, новая реплика синхронизируется с другими.

Чтобы удалить реплику, выполните `DROP TABLE`. Однако удаляется только одна реплика — та, которая находится на сервере, где вы выполняете запрос.

## Восстановление после сбоев {#recovery-after-failures}

Если ClickHouse Keeper недоступен при запуске сервера, реплицированные таблицы переключаются в режим только для чтения. Система периодически пытается подключиться к ClickHouse Keeper.

Если ClickHouse Keeper недоступен во время `INSERT`, или возникает ошибка при взаимодействии с ClickHouse Keeper, выдается исключение.

После подключения к ClickHouse Keeper система проверяет, совпадает ли набор данных в локальной файловой системе с ожидаемым набором данных (ClickHouse Keeper хранит эту информацию). Если есть незначительные несоответствия, система разрешает их, синхронизируя данные с репликами.

Если система обнаруживает поврежденные части данных (с неправильным размером файлов) или нераспознанные части (части, записанные в файловой системе, но не зарегистрированные в ClickHouse Keeper), она перемещает их в подкаталог `detached` (они не удаляются). Все отсутствующие части копируются с реплик.

Обратите внимание, что ClickHouse не выполняет никаких разрушительных действий, таких как автоматическое удаление большого количества данных.

Когда сервер запускается (или устанавливает новую сессию с ClickHouse Keeper), он только проверяет количество и размеры всех файлов. Если размеры файлов совпадают, но байты были изменены где-то посередине, это не обнаруживается сразу, а только при попытке прочитать данные для запроса `SELECT`. Запрос выдает исключение о несоответствии контрольной суммы или размера сжатого блока. В этом случае части данных добавляются в очередь проверки и копируются с реплик, если необходимо.

Если локальный набор данных слишком сильно отличается от ожидаемого, срабатывает защитный механизм. Сервер записывает это в журнал и отказывается от запуска. Причина в том, что этот случай может указывать на ошибку конфигурации, например, если реплика на шарде была случайно настроена как реплика на другом шарде. Однако пороги для этого механизма установлены довольно низко, и эта ситуация может возникнуть во время нормального восстановления после сбоев. В этом случае данные восстанавливаются полуручно - с помощью «нажатия кнопки».

Чтобы начать восстановление, создайте узел `/path_to_table/replica_name/flags/force_restore_data` в ClickHouse Keeper с любым содержанием или выполните команду для восстановления всех реплицированных таблиц:

``` bash
sudo -u clickhouse touch /var/lib/clickhouse/flags/force_restore_data
```

Затем перезапустите сервер. При запуске сервер удаляет эти флаги и начинает восстановление.

## Восстановление после полной потери данных {#recovery-after-complete-data-loss}

Если все данные и метаданные пропали с одного из серверов, выполните следующие шаги для восстановления:

1. Установите ClickHouse на сервере. Правильно определите подстановки в файле конфигурации, который содержит идентификатор шарда и реплики, если вы их используете.
2. Если у вас были нереплицированные таблицы, которые необходимо вручную дублировать на серверах, скопируйте их данные с реплики (в директории `/var/lib/clickhouse/data/db_name/table_name/`).
3. Скопируйте определения таблиц, расположенные в `/var/lib/clickhouse/metadata/` с реплики. Если идентификатор шарда или реплики явно определен в определениях таблиц, исправьте его так, чтобы он соответствовал этой реплике. (В качестве альтернативы, запустите сервер и выполните все запросы `ATTACH TABLE`, которые должны были быть в .sql файлах в `/var/lib/clickhouse/metadata/`).
4. Чтобы начать восстановление, создайте узел ClickHouse Keeper `/path_to_table/replica_name/flags/force_restore_data` с любым содержанием или выполните команду для восстановления всех реплицированных таблиц: `sudo -u clickhouse touch /var/lib/clickhouse/flags/force_restore_data`

Затем запустите сервер (перезапустите, если он уже работает). Данные будут загружены с реплик.

Альтернативный вариант восстановления — удалить информацию о потерянной реплике из ClickHouse Keeper (`/path_to_table/replica_name`), затем создать реплику снова, как описано в "[Создание реплицированных таблиц](#creating-replicated-tables)".

Нет ограничений на сетевую пропускную способность во время восстановления. Имейте это в виду, если вы восстанавливаете много реплик одновременно.

## Конвертация из MergeTree в ReplicatedMergeTree {#converting-from-mergetree-to-replicatedmergetree}

Мы используем термин `MergeTree`, чтобы обозначать все движки таблиц в `MergeTree family`, так же как и для `ReplicatedMergeTree`.

Если у вас есть таблица `MergeTree`, которая была вручную реплицирована, вы можете преобразовать ее в реплицированную таблицу. Вам может понадобиться это сделать, если вы уже собрали большое количество данных в таблице `MergeTree`, и теперь вы хотите включить репликацию.

Заявление [ATTACH TABLE ... AS REPLICATED](/sql-reference/statements/attach.md#attach-mergetree-table-as-replicatedmergetree) позволяет прикрепить отсоединенную таблицу `MergeTree` как `ReplicatedMergeTree`.

Таблица `MergeTree` может быть автоматически конвертирована при перезагрузке сервера, если установлен флаг `convert_to_replicated` в директории данных таблицы (`/store/xxx/xxxyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy/` для базы данных `Atomic`).
Создайте пустой файл `convert_to_replicated`, и таблица будет загружена как реплицированная при следующей перезагрузке сервера.

Этот запрос может быть использован, чтобы получить путь данных таблицы. Если у таблицы есть много путей данных, вам нужно будет использовать первый из них.

```sql
SELECT data_paths FROM system.tables WHERE table = 'table_name' AND database = 'database_name';
```

Обратите внимание, что таблица ReplicatedMergeTree будет создана с значениями настроек `default_replica_path` и `default_replica_name`.
Чтобы создать преобразованную таблицу на других репликах, вам нужно будет явно указать ее путь в первом аргументе движка `ReplicatedMergeTree`. Для этого можно использовать следующий запрос.

```sql
SELECT zookeeper_path FROM system.replicas WHERE table = 'table_name';
```

Существует также ручной способ сделать это.

Если данные различаются на различных репликах, сначала синхронизируйте их или удалите эти данные на всех репликах, кроме одной.

Переименуйте существующую таблицу MergeTree, затем создайте таблицу `ReplicatedMergeTree` с старым именем.
Переместите данные из старой таблицы в подкаталог `detached` внутри директории с новыми данными таблицы (`/var/lib/clickhouse/data/db_name/table_name/`).
Затем выполните `ALTER TABLE ATTACH PARTITION` на одной из реплик, чтобы добавить эти части данных в рабочий набор.

## Конвертация из ReplicatedMergeTree в MergeTree {#converting-from-replicatedmergetree-to-mergetree}

Используйте [ATTACH TABLE ... AS NOT REPLICATED](/sql-reference/statements/attach.md#attach-mergetree-table-as-replicatedmergetree) для прикрепления отсоединенной таблицы `ReplicatedMergeTree` как `MergeTree` на одном сервере.

Другой способ сделать это заключается в перезагрузке сервера. Создайте таблицу MergeTree с другим именем. Переместите все данные из директории с данными таблицы `ReplicatedMergeTree` в директорию новых данных таблицы. Затем удалите таблицу `ReplicatedMergeTree` и перезагрузите сервер.

Если вы хотите избавиться от таблицы `ReplicatedMergeTree` без запуска сервера:

- Удалите соответствующий .sql файл в директории метаданных (`/var/lib/clickhouse/metadata/`).
- Удалите соответствующий путь в ClickHouse Keeper (`/path_to_table/replica_name`).

После этого вы можете запустить сервер, создать таблицу `MergeTree`, переместить данные в ее директорию, а затем перезагрузить сервер.

## Восстановление при потере или повреждении метаданных в кластере ClickHouse Keeper {#recovery-when-metadata-in-the-zookeeper-cluster-is-lost-or-damaged}

Если данные в ClickHouse Keeper были потеряны или повреждены, вы можете сохранить данные, переместив их в нереплицированную таблицу, как описано выше.

**Смотрите также**

- [background_schedule_pool_size](/operations/server-configuration-parameters/settings.md/#background_schedule_pool_size)
- [background_fetches_pool_size](/operations/server-configuration-parameters/settings.md/#background_fetches_pool_size)
- [execute_merges_on_single_replica_time_threshold](/operations/settings/merge-tree-settings#execute_merges_on_single_replica_time_threshold)
- [max_replicated_fetches_network_bandwidth](/operations/settings/merge-tree-settings.md/#max_replicated_fetches_network_bandwidth)
- [max_replicated_sends_network_bandwidth](/operations/settings/merge-tree-settings.md/#max_replicated_sends_network_bandwidth)
