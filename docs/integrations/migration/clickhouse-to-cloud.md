---
sidebar_position: 10
sidebar_label: 'ClickHouse to ClickHouse Cloud'
slug: /cloud/migration/clickhouse-to-cloud
title: 'Migrating between self-managed ClickHouse and ClickHouse Cloud'
description: 'Page describing how to migrate between self-managed ClickHouse and ClickHouse Cloud'
---

import Image from '@theme/IdealImage';
import AddARemoteSystem from '@site/docs/_snippets/_add_remote_ip_access_list_detail.md';
import self_managed_01 from '@site/static/images/integrations/migration/self-managed-01.png';
import self_managed_02 from '@site/static/images/integrations/migration/self-managed-02.png';
import self_managed_03 from '@site/static/images/integrations/migration/self-managed-03.png';
import self_managed_04 from '@site/static/images/integrations/migration/self-managed-04.png';
import self_managed_05 from '@site/static/images/integrations/migration/self-managed-05.png';
import self_managed_06 from '@site/static/images/integrations/migration/self-managed-06.png';

# Migrating between self-managed ClickHouse and ClickHouse Cloud

<Image img={self_managed_01} size='md' alt='Migrating Self-managed ClickHouse' background='white' />

This guide will show how to migrate from a self-managed ClickHouse server to ClickHouse Cloud, and also how to migrate between ClickHouse Cloud services. The [`remoteSecure`](../../sql-reference/table-functions/remote.md) function is used in `SELECT` and `INSERT` queries to allow access to remote ClickHouse servers, which makes migrating tables as simple as writing an `INSERT INTO` query with an embedded `SELECT`.

## Migrating from Self-managed ClickHouse to ClickHouse Cloud {#migrating-from-self-managed-clickhouse-to-clickhouse-cloud}

<Image img={self_managed_02} size='sm' alt='Migrating Self-managed ClickHouse' background='white' />

:::note
Regardless of if your source table is sharded and/or replicated, on ClickHouse Cloud you just create a destination table (you can leave out the Engine parameter for this table, it will be automatically a ReplicatedMergeTree table),
and ClickHouse Cloud will automatically take care of vertical and horizontal scaling. There is no need from your side to think about how to replicate and shard the table.
:::

In this example the self-managed ClickHouse server is the *source* and the ClickHouse Cloud service is the *destination*.

### Overview {#overview}

The process is:

1. Add a read-only user to the source service
1. Duplicate the source table structure on the destination service
1. Pull the data from source to destination, or push the data from the source, depending on the network availability of the source
1. Remove the source server from the IP Access List on the destination (if applicable)
1. Remove the read-only user from the source service

### Migration of tables from one system to another: {#migration-of-tables-from-one-system-to-another}
This example migrates one table from a self-managed ClickHouse server to ClickHouse Cloud.

### On the source ClickHouse system (the system that currently hosts the data) {#on-the-source-clickhouse-system-the-system-that-currently-hosts-the-data}

- Add a read only user that can read the source table (`db.table` in this example)
```sql
CREATE USER exporter
IDENTIFIED WITH SHA256_PASSWORD BY 'password-here'
SETTINGS readonly = 1;
```

```sql
GRANT SELECT ON db.table TO exporter;
```

- Copy the table definition
```sql
SELECT create_table_query
FROM system.tables
WHERE database = 'db' AND table = 'table'
```

### On the destination ClickHouse Cloud system: {#on-the-destination-clickhouse-cloud-system}

- Create the destination database:
```sql
CREATE DATABASE db
```

- Using the CREATE TABLE statement from the source, create the destination.

:::tip
Change the ENGINE to to ReplicatedMergeTree without any parameters when you run the CREATE statement. ClickHouse Cloud always replicates tables and provides the correct parameters. Keep the `ORDER BY`, `PRIMARY KEY`, `PARTITION BY`, `SAMPLE BY`, `TTL`, and `SETTINGS` clauses though.
:::

```sql
CREATE TABLE db.table ...
```

- Use the `remoteSecure` function to pull the data from the self-managed source

<Image img={self_managed_03} size='sm' alt='Migrating Self-managed ClickHouse' background='white' />

```sql
INSERT INTO db.table SELECT * FROM
remoteSecure('source-hostname', db, table, 'exporter', 'password-here')
```

:::note
If the source system is not available from outside networks then you can push the data rather than pulling it, as the `remoteSecure` function works for both selects and inserts.  See the next option.
:::

- Use the `remoteSecure` function to push the data to the ClickHouse Cloud service

<Image img={self_managed_04} size='sm' alt='Migrating Self-managed ClickHouse' background='white' />

:::tip Add the remote system to your ClickHouse Cloud service IP Access List
In order for the `remoteSecure` function to connect to your ClickHouse Cloud service the IP Address of the remote system will need to be allowed by the IP Access List.  Expand **Manage your IP Access List** below this tip for more information.
:::

  <AddARemoteSystem />

```sql
INSERT INTO FUNCTION
remoteSecure('HOSTNAME.clickhouse.cloud:9440', 'db.table',
'default', 'PASS') SELECT * FROM db.table
```

## Migrating between ClickHouse Cloud services {#migrating-between-clickhouse-cloud-services}

<Image img={self_managed_05} size='lg' alt='Migrating Self-managed ClickHouse' background='white' />

Some example uses for migrating data between ClickHouse Cloud services:
- Migrating data from a restored backup
- Copying data from a development service to a staging service (or staging to production)

In this example there are two ClickHouse Cloud services, and they will be referred to as *source* and *destination*.  The data will be pulled from the source to the destination. Although you could push if you like, pulling is shown as it uses a read-only user.

<Image img={self_managed_06} size='lg' alt='Migrating Self-managed ClickHouse' background='white' />

There are a few steps in the migration:
1. Identify one ClickHouse Cloud service to be the *source*, and the other as the *destination*
1. Add a read-only user to the source service
1. Duplicate the source table structure on the destination service
1. Temporarily allow IP access to the source service
1. Copy the data from source to destination
1. Re-establish the IP Access List on the destination
1. Remove the read-only user from the source service

#### Add a read-only user to the source service {#add-a-read-only-user-to-the-source-service}

- Add a read only user that can read the source table (`db.table` in this example)
  ```sql
  CREATE USER exporter
  IDENTIFIED WITH SHA256_PASSWORD BY 'password-here'
  SETTINGS readonly = 1;
  ```

  ```sql
  GRANT SELECT ON db.table TO exporter;
  ```

- Copy the table definition
  ```sql
  select create_table_query
  from system.tables
  where database = 'db' and table = 'table'
  ```

#### Duplicate the table structure on the destination service {#duplicate-the-table-structure-on-the-destination-service}

On the destination create the database if it is not there already:

- Create the destination database:
  ```sql
  CREATE DATABASE db
  ```

- Using the CREATE TABLE statement from the source, create the destination.

  On the destination create the table using the output of the `select create_table_query...` from the source:

  ```sql
  CREATE TABLE db.table ...
  ```

#### Allow remote access to the source service {#allow-remote-access-to-the-source-service}

In order to pull data from the source to the destination the source service must allow connections. Temporarily disable the "IP Access List" functionality on the source service.

:::tip
If you will continue to use the source ClickHouse Cloud service then export the existing IP Access list to a JSON file before switching to allow access from anywhere; this will allow you to import the access list after the data is migrated.
:::

Modify the allow list and allow access from **Anywhere** temporarily. See the [IP Access List](/cloud/security/setting-ip-filters) docs for details.

#### Copy the data from source to destination {#copy-the-data-from-source-to-destination}

- Use the `remoteSecure` function to pull the data from the source ClickHouse Cloud service
  Connect to the destination.  Run this command on the destination ClickHouse Cloud service:

  ```sql
  INSERT INTO db.table SELECT * FROM
  remoteSecure('source-hostname', db, table, 'exporter', 'password-here')
  ```

- Verify the data in the destination service

#### Re-establish the IP access list on the source {#re-establish-the-ip-access-list-on-the-source}

  If you exported the access list earlier, then you can re-import it using **Share**, otherwise re-add your entries to the access list.

#### Remove the read-only `exporter` user {#remove-the-read-only-exporter-user}

```sql
DROP USER exporter
```

- Switch the service IP Access List to limit access
