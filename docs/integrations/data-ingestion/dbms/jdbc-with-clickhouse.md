---
sidebar_label: 'JDBC'
sidebar_position: 2
keywords: ['clickhouse', 'jdbc', 'connect', 'integrate']
slug: /integrations/jdbc/jdbc-with-clickhouse
description: 'The ClickHouse JDBC Bridge allows ClickHouse to access data from any external data source for which a JDBC driver is available'
title: 'Connecting ClickHouse to external data sources with JDBC'
---

import Image from '@theme/IdealImage';
import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';
import Jdbc01 from '@site/static/images/integrations/data-ingestion/dbms/jdbc-01.png';
import Jdbc02 from '@site/static/images/integrations/data-ingestion/dbms/jdbc-02.png';
import Jdbc03 from '@site/static/images/integrations/data-ingestion/dbms/jdbc-03.png';

# Connecting ClickHouse to external data sources with JDBC

:::note
Using JDBC requires the ClickHouse JDBC bridge, so you will need to use `clickhouse-local` on a local machine to stream the data from your database to ClickHouse Cloud. Visit the [**Using clickhouse-local**](/integrations/migration/clickhouse-local-etl.md#example-2-migrating-from-mysql-to-clickhouse-cloud-with-the-jdbc-bridge) page in the **Migrate** section of the docs for details.
:::

**Overview:** The <a href="https://github.com/ClickHouse/clickhouse-jdbc-bridge" target="_blank">ClickHouse JDBC Bridge</a> in combination with the [jdbc table function](/sql-reference/table-functions/jdbc.md) or the [JDBC table engine](/engines/table-engines/integrations/jdbc.md) allows ClickHouse to access data from any external data source for which a <a href="https://en.wikipedia.org/wiki/JDBC_driver" target="_blank">JDBC driver</a> is available:

<Image img={Jdbc01} size="lg" alt="ClickHouse JDBC Bridge architecture diagram" background='white'/>
This is handy when there is no native built-in [integration engine](/engines/table-engines/integrations), table function, or external dictionary for the external data source available, but a JDBC driver for the data source exists.

You can use the ClickHouse JDBC Bridge for both reads and writes. And in parallel for multiple external data sources, e.g. you can run distributed queries on ClickHouse across multiple external and internal data sources in real time.

In this lesson we will show you how easy it is to install, configure, and run the ClickHouse JDBC Bridge in order to connect ClickHouse with an external data source. We will use MySQL as the external data source for this lesson.

Let's get started!

:::note Prerequisites
You have access to a machine that has:
1. a Unix shell and internet access
2. <a href="https://www.gnu.org/software/wget/" target="_blank">wget</a> installed
3. a current version of **Java** (e.g. <a href="https://openjdk.java.net" target="_blank">OpenJDK</a> Version >= 17) installed
4. a current version of **MySQL** (e.g. <a href="https://www.mysql.com" target="_blank">MySQL</a> Version >=8) installed and running
5. a current version of **ClickHouse** [installed](/getting-started/install/install.mdx) and running
:::

## Install the ClickHouse JDBC Bridge locally {#install-the-clickhouse-jdbc-bridge-locally}

The easiest way to use the ClickHouse JDBC Bridge is to install and run it on the same host where also ClickHouse is running:<Image img={Jdbc02} size="lg" alt="ClickHouse JDBC Bridge locally deployment diagram" background='white'/>

Let's start by connecting to the Unix shell on the machine where ClickHouse is running and create a local folder where we will later install the ClickHouse JDBC Bridge into (feel free to name the folder anything you like and put it anywhere you like):
```bash
mkdir ~/clickhouse-jdbc-bridge
```

Now we download the <a href="https://github.com/ClickHouse/clickhouse-jdbc-bridge/releases/" target="_blank">current version</a> of the ClickHouse JDBC Bridge into that folder:

```bash
cd ~/clickhouse-jdbc-bridge
wget https://github.com/ClickHouse/clickhouse-jdbc-bridge/releases/download/v2.0.7/clickhouse-jdbc-bridge-2.0.7-shaded.jar
```

In order to be able to connect to MySQL we are creating a named data source:

 ```bash
 cd ~/clickhouse-jdbc-bridge
 mkdir -p config/datasources
 touch config/datasources/mysql8.json
 ```

 You can now copy and paste the following configuration into the file `~/clickhouse-jdbc-bridge/config/datasources/mysql8.json`:

 ```json
 {
   "mysql8": {
   "driverUrls": [
     "https://repo1.maven.org/maven2/mysql/mysql-connector-java/8.0.28/mysql-connector-java-8.0.28.jar"
   ],
   "jdbcUrl": "jdbc:mysql://<host>:<port>",
   "username": "<username>",
   "password": "<password>"
   }
 }
 ```

:::note
in the config file above
- you are free to use any name you like for the datasource, we used `mysql8`
- in the value for the `jdbcUrl` you need to replace `<host>`, and `<port>` with appropriate values according to your running MySQL instance, e.g. `"jdbc:mysql://localhost:3306"`
- you need to replace `<username>` and `<password>` with your MySQL credentials, if you don't use a password, you can delete the `"password": "<password>"` line in the config file above
- in the value for `driverUrls` we just specified a URL from which the <a href="https://repo1.maven.org/maven2/mysql/mysql-connector-java/" target="_blank">current version</a> of the MySQL JDBC driver can be downloaded. That's all we have to do, and the ClickHouse JDBC Bridge will automatically download that JDBC driver (into a OS specific directory).
:::

<br/>

Now we are ready to start the ClickHouse JDBC Bridge:
 ```bash
 cd ~/clickhouse-jdbc-bridge
 java -jar clickhouse-jdbc-bridge-2.0.7-shaded.jar
 ```
:::note
We started the ClickHouse JDBC Bridge in foreground mode. In order to stop the Bridge you can bring the Unix shell window from above in foreground and press `CTRL+C`.
:::

## Use the JDBC connection from within ClickHouse {#use-the-jdbc-connection-from-within-clickhouse}

ClickHouse can now access MySQL data by either using the [jdbc table function](/sql-reference/table-functions/jdbc.md) or the [JDBC table engine](/engines/table-engines/integrations/jdbc.md).

The easiest way to execute the following examples is to copy and paste them into the [`clickhouse-client`](/interfaces/cli.md) or into the [Play UI](/interfaces/http.md).

- jdbc Table Function:

 ```sql
 SELECT * FROM jdbc('mysql8', 'mydatabase', 'mytable');
 ```
:::note
As the first parameter for the jdbc table function we are using the name of the named data source that we configured above.
:::

- JDBC Table Engine:
 ```sql
 CREATE TABLE mytable (
      <column> <column_type>,
      ...
 )
 ENGINE = JDBC('mysql8', 'mydatabase', 'mytable');

 SELECT * FROM mytable;
 ```
:::note
 As the first parameter for the jdbc engine clause we are using the name of the named data source that we configured above

 The schema of the ClickHouse JDBC engine table and schema of the connected MySQL table must be aligned, e.g. the column names and order must be the same, and the column data types must be compatible
:::

## Install the ClickHouse JDBC Bridge externally {#install-the-clickhouse-jdbc-bridge-externally}

For a distributed ClickHouse cluster (a cluster with more than one ClickHouse host) it makes sense to install and run the ClickHouse JDBC Bridge externally on its own host:
<Image img={Jdbc03} size="lg" alt="ClickHouse JDBC Bridge external deployment diagram" background='white'/>
This has the advantage that each ClickHouse host can access the JDBC Bridge. Otherwise the JDBC Bridge would need to be installed locally for each ClickHouse instance that is supposed to access external data sources via the Bridge.

In order to install the ClickHouse JDBC Bridge externally, we do the following steps:

1. We install, configure and run the ClickHouse JDBC Bridge on a dedicated host by following the steps described in section 1 of this guide.

2. On each ClickHouse Host we add the following configuration block to the <a href="https://clickhouse.com/docs/operations/configuration-files/#configuration_files" target="_blank">ClickHouse server configuration</a> (depending on your chosen configuration format, use either the XML or YAML version):

<Tabs>
<TabItem value="xml" label="XML">

```xml
<jdbc_bridge>
   <host>JDBC-Bridge-Host</host>
   <port>9019</port>
</jdbc_bridge>
```

</TabItem>
<TabItem value="yaml" label="YAML">

```yaml
jdbc_bridge:
    host: JDBC-Bridge-Host
    port: 9019
```

</TabItem>
</Tabs>

:::note
- you need to replace `JDBC-Bridge-Host` with the hostname or ip address of the dedicated ClickHouse JDBC Bridge host
- we specified the default ClickHouse JDBC Bridge port `9019`, if you are using a different port for the JDBC Bridge then you must adapt the configuration above accordingly
:::

[//]: # (## 4. Additional Info)

[//]: # ()
[//]: # (TODO: )

[//]: # (- mention that for jdbc table function it is more performant &#40;not two queries each time&#41; to also specify the schema as a parameter)

[//]: # ()
[//]: # (- mention ad hoc query vs table query, saved query, named query)

[//]: # ()
[//]: # (- mention insert into )
