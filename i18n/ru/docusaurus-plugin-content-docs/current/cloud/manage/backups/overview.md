---
sidebar_label: 'Обзор'
sidebar_position: 0
slug: /cloud/manage/backups/overview
title: 'Обзор'
keywords: ['бэкапы', 'облачные бэкапы', 'восстановление']
---

import CloudNotSupportedBadge from '@theme/badges/CloudNotSupportedBadge';
import ScalePlanFeatureBadge from '@theme/badges/ScalePlanFeatureBadge';
import backup_chain from '@site/static/images/cloud/manage/backup-chain.png';
import backup_status_list from '@site/static/images/cloud/manage/backup-status-list.png';
import backup_usage from '@site/static/images/cloud/manage/backup-usage.png';
import backup_restore from '@site/static/images/cloud/manage/backup-restore.png';
import backup_service_provisioning from '@site/static/images/cloud/manage/backup-service-provisioning.png';


# Бэкапы

Бэкапы баз данных обеспечивают защиту, гарантируя, что если данные будут утеряны по какой-либо непредвиденной причине, сервис может быть восстановлен до предыдущего состояния из последнего успешного бэкапа. Это минимизирует время простоя и предотвращает безвозвратную потерю критически важных данных для бизнеса. Этот гид охватывает, как работают бэкапы в ClickHouse Cloud, какие у вас есть параметры для настройки бэкапов для вашего сервиса и как восстановить данные из бэкапа.

## Как работают бэкапы в ClickHouse Cloud {#how-backups-work-in-clickhouse-cloud}

Бэкапы ClickHouse Cloud представляют собой комбинацию «полных» и «инкрементных» бэкапов, которые образуют цепочку бэкапов. Цепочка начинается с полного бэкапа, а затем в течение нескольких запланированных периодов времени выполняются инкрементные бэкапы для создания последовательности бэкапов. Как только цепочка бэкапов достигает определенной длины, начинается новая цепочка. Эта вся цепочка бэкапов затем может быть использована для восстановления данных в новый сервис, если это необходимо. Как только все бэкапы, включенные в конкретную цепочку, превышают установленный для сервиса срок хранения (более подробно о сроке хранения ниже), цепочка утилизируется.

На скриншоте ниже квадратные блоки с непрерывной линией показывают полные бэкапы, а квадратные блоки с пунктирной линией показывают инкрементные бэкапы. Прямоугольник с непрерывной линией вокруг квадратов обозначает срок хранения и бэкапы, которые видны конечному пользователю, которые могут быть использованы для восстановления бэкапа. В приведенном ниже сценарии бэкапы выполняются каждые 24 часа и хранятся в течение 2 дней.

В День 1 выполняется полный бэкап для начала цепочки бэкапов. В День 2 выполняется инкрементный бэкап, и теперь у нас есть полный и инкрементный бэкап, доступные для восстановления. К Дню 7 у нас есть один полный бэкап и шесть инкрементных бэкапов в цепочке, при этом два самых последних инкрементных бэкапа видны пользователю. В День 8 мы делаем новый полный бэкап, а в День 9, когда у нас будет два бэкапа в новой цепочке, предыдущая цепочка утилизируется.

<img src={backup_chain}
    alt="Пример цепочки бэкапов в ClickHouse Cloud"
    class="image"
/>

*Пример сценария бэкапа в ClickHouse Cloud*

## Политика по умолчанию для бэкапов {#default-backup-policy}

В базовом, масштабирующем и корпоративном тарифах бэкапы учитываются и выставляются отдельным счетом от хранения. Все сервисы по умолчанию имеют один бэкап с возможностью настройки дополнительных бэкапов, начиная с масштабируемого тарифа, через вкладку Настройки в облачной консоли.

## Список статусов бэкапов {#backup-status-list}

Ваш сервис будет бэкапиться в соответствии с установленным графиком, будь то стандартный ежедневный график или [пользовательский график](./configurable-backups.md), выбранный вами. Все доступные бэкапы можно просмотреть на вкладке **Бэкапы** сервиса. Отсюда вы можете увидеть статус бэкапа, продолжительность, а также размер бэкапа. Вы также можете восстановить конкретный бэкап, используя колонку **Действия**.

<img src={backup_status_list}
    alt="Список статусов бэкапов в ClickHouse Cloud"
    class="image"
/>

## Понимание стоимости бэкапов {#understanding-backup-cost}

В соответствии с политикой по умолчанию ClickHouse Cloud требует выполнения бэкапа каждый день с 24-часовым сроком хранения. Выбор графика, который требует сохранения большего объема данных или вызывает более частые бэкапы, может привести к дополнительным расходам на хранение бэкапов.

Чтобы понять стоимость бэкапов, вы можете просмотреть стоимость бэкапа для сервиса на экране использования (как показано ниже). После нескольких дней работы бэкапов с пользовательским графиком вы сможете оценить стоимость и экстраполировать, чтобы получить ежемесячную стоимость бэкапов.

<img src={backup_usage}
    alt="График использования бэкапов в ClickHouse Cloud"
    class="image"
/>


Оценка общей стоимости ваших бэкапов требует, чтобы вы установили график. Мы также работаем над обновлением нашего [калькулятора цен](https://clickhouse.com/pricing), чтобы вы могли получить оценку ежемесячной стоимости перед установкой графика. Вам необходимо будет предоставить следующие данные для оценки стоимости:
- Размер полных и инкрементных бэкапов
- Желаемая частота
- Желаемый срок хранения
- Облачный провайдер и регион

:::note
Имейте в виду, что оценочная стоимость бэкапов будет изменяться по мере роста объема данных в сервисе со временем.
:::


## Восстановление бэкапа {#restore-a-backup}

Бэкапы восстанавливаются в новый сервис ClickHouse Cloud, а не в существующий сервис, из которого был сделан бэкап.

Нажав на иконку **Восстановить** бэкап, вы можете указать имя сервиса нового сервиса, который будет создан, а затем восстановить этот бэкап:

<img src={backup_restore}
    alt="Восстановление бэкапа в ClickHouse Cloud"
    class="image"
/>

Новый сервис будет отображаться в списке сервисов как `Provisioning`, пока он не будет готов:

<img src={backup_service_provisioning}
    alt="Процесс подготовки сервиса"
    class="image"
    style={{width: '80%'}}
/>

## Работа с восстановленным сервисом {#working-with-your-restored-service}

После восстановления бэкапа у вас будет два схожих сервиса: **оригинальный сервис**, который необходимо было восстановить, и новый **восстановленный сервис**, который был восстановлен из бэкапа оригинального.

После завершения восстановления бэкапа вам следует сделать одно из следующих действий:
- Использовать новый восстановленный сервис и удалить оригинальный сервис.
- Перенести данные из нового восстановленного сервиса обратно в оригинальный сервис и удалить новый восстановленный сервис.

### Использовать **новый восстановленный сервис** {#use-the-new-restored-service}

Чтобы использовать новый сервис, выполните следующие шаги:

1. Убедитесь, что новый сервис имеет необходимые записи IP Access List для вашего запроса.
1. Убедитесь, что новый сервис содержит необходимые вам данные.
1. Удалите оригинальный сервис.

### Перенести данные из **нового восстановленного сервиса** обратно в **оригинальный сервис** {#migrate-data-from-the-newly-restored-service-back-to-the-original-service}

Предположим, что вы не можете работать с новым восстановленным сервисом по какой-либо причине, например, если у вас все еще есть пользователи или приложения, которые подключаются к существующему сервису. Вы можете решить перенести вновь восстановленные данные в оригинальный сервис. Миграция может быть выполнена следующим образом:

**Разрешите удаленный доступ к новому восстановленному сервису**

Новый сервис должен быть восстановлен из бэкапа с таким же IP Allow List, как и оригинальный сервис. Это необходимо, поскольку подключения к другим сервисам ClickHouse Cloud не будут разрешены, если вы не разрешили доступ с **Anywhere**. Временно измените список разрешений и разрешите доступ с **Anywhere**. Смотрите документацию по [IP Access List](/cloud/security/setting-ip-filters) для деталей.

**На новом восстановленном ClickHouse сервисе (системе, которая хранит восстановленные данные)**

:::note
Вам необходимо сбросить пароль для нового сервиса, чтобы получить к нему доступ. Вы можете сделать это на вкладке **Настройки** в списке сервисов.
:::

Добавьте пользователя только для чтения, который может получать доступ к исходной таблице (`db.table` в этом примере):

  ```sql
  CREATE USER exporter
  IDENTIFIED WITH SHA256_PASSWORD BY 'password-here'
  SETTINGS readonly = 1;
  ```

  ```sql
  GRANT SELECT ON db.table TO exporter;
  ```

Скопируйте определение таблицы:

  ```sql
  SELECT create_table_query
  FROM system.tables
  WHERE database = 'db' AND table = 'table'
  ```

**На системе ClickHouse Cloud назначения (в той, которая имела поврежденную таблицу):**

Создайте базу данных назначения:
  ```sql
  CREATE DATABASE db
  ```

С помощью оператора `CREATE TABLE`, используя источник, создайте базу данных назначения:

:::tip
Измените `ENGINE` на `ReplicatedMergeTree` без каких-либо параметров, когда будете запускать оператор `CREATE`. ClickHouse Cloud всегда реплицирует таблицы и предоставляет правильные параметры.
:::

  ```sql
  CREATE TABLE db.table ...
  ENGINE = ReplicatedMergeTree
  ORDER BY ...
  ```

Используйте функцию `remoteSecure`, чтобы перетянуть данные из вновь восстановленного ClickHouse Cloud сервиса в ваш оригинальный сервис:

  ```sql
  INSERT INTO db.table
  SELECT *
  FROM remoteSecure('source-hostname', db, table, 'exporter', 'password-here')
  ```

После успешной вставки данных в ваш оригинальный сервис убедитесь, что данные корректны. Также убедитесь, что вы удалили новый сервис после проверки данных.

## Восстановление удаленных или сброшенных таблиц {#undeleting-or-undropping-tables}

<CloudNotSupportedBadge/>

Команда `UNDROP` не поддерживается в ClickHouse Cloud. Если вы случайно удалили таблицу с помощью `DROP`, лучше всего восстановить ваш последний бэкап и воссоздать таблицу из бэкапа.

Чтобы предотвратить случайное удаление таблиц пользователями, вы можете использовать [`GRANT` statements](/sql-reference/statements/grant), чтобы отозвать права на команду [`DROP TABLE` command](/sql-reference/statements/drop#drop-table) для конкретного пользователя или роли.

:::note
Чтобы предотвратить случайное удаление данных, обратите внимание, что по умолчанию в ClickHouse Cloud невозможно удалять таблицы размером более `1TB`. Если вы хотите удалить таблицы крупнее этого порога, вы можете использовать настройку `max_table_size_to_drop` для этого:

```sql
DROP TABLE IF EXISTS table_to_drop
SYNC SETTINGS max_table_size_to_drop=2097152 -- увеличивает лимит до 2TB
```
:::

## Настраиваемые бэкапы {#configurable-backups}

Если вы хотите установить график бэкапов, отличный от стандартного графика бэкапов, загляните в [Настраиваемые бэкапы](./configurable-backups.md).

## Экспорт бэкапов в свою учетную запись облака {#export-backups-to-your-own-cloud-account}

Для пользователей, желающих экспортировать бэкапы в свою учетную запись облака, смотрите [здесь](./export-backups-to-own-cloud-account.md).
