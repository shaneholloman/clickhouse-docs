---
'sidebar_label': 'R2DBC Driver'
'sidebar_position': 5
'keywords':
- 'clickhouse'
- 'java'
- 'driver'
- 'integrate'
- 'r2dbc'
'description': 'ClickHouse R2DBC 驱动程序'
'slug': '/integrations/java/r2dbc'
'title': 'R2DBC 驱动程序'
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';
import CodeBlock from '@theme/CodeBlock';


# R2DBC 驱动程序

## R2DBC 驱动程序 {#r2dbc-driver}

[R2DBC](https://r2dbc.io/) 的 ClickHouse 异步 Java 客户端包装器。

### 环境要求 {#environment-requirements}

- [OpenJDK](https://openjdk.java.net) 版本 >= 8

### 安装 {#setup}

```xml
<dependency>
    <groupId>com.clickhouse</groupId>
    <!-- change to clickhouse-r2dbc_0.9.1 for SPI 0.9.1.RELEASE -->
    <artifactId>clickhouse-r2dbc</artifactId>
    <version>0.7.1</version>
    <!-- use uber jar with all dependencies included, change classifier to http or grpc for smaller jar -->
    <classifier>all</classifier>
    <exclusions>
        <exclusion>
            <groupId>*</groupId>
            <artifactId>*</artifactId>
        </exclusion>
    </exclusions>
</dependency>
```

### 连接到 ClickHouse {#connect-to-clickhouse}

```java showLineNumbers
ConnectionFactory connectionFactory = ConnectionFactories
    .get("r2dbc:clickhouse:http://{username}:{password}@{host}:{port}/{database}");

    Mono.from(connectionFactory.create())
        .flatMapMany(connection -> connection
```

### 查询 {#query}

```java showLineNumbers
connection
    .createStatement("select domain, path,  toDate(cdate) as d, count(1) as count from clickdb.clicks where domain = :domain group by domain, path, d")
    .bind("domain", domain)
    .execute()
    .flatMap(result -> result
    .map((row, rowMetadata) -> String.format("%s%s[%s]:%d", row.get("domain", String.class),
        row.get("path", String.class),
        row.get("d", LocalDate.class),
        row.get("count", Long.class)))
    )
    .doOnNext(System.out::println)
    .subscribe();
```

### 插入 {#insert}

```java showLineNumbers
connection
    .createStatement("insert into clickdb.clicks values (:domain, :path, :cdate, :count)")
    .bind("domain", click.getDomain())
    .bind("path", click.getPath())
    .bind("cdate", LocalDateTime.now())
    .bind("count", 1)
    .execute();
```
