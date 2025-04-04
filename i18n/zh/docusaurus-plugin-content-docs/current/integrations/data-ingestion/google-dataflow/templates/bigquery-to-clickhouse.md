---
sidebar_label: 'BigQuery 到 ClickHouse'
sidebar_position: 1
slug: /integrations/google-dataflow/templates/bigquery-to-clickhouse
description: '用户可以使用 Google Dataflow 模板将数据从 BigQuery 导入 ClickHouse'
---

import TOCInline from '@theme/TOCInline';
import dataflow_inqueue_job from '@site/static/images/integrations/data-ingestion/google-dataflow/dataflow-inqueue-job.png'


# Dataflow BigQuery 到 ClickHouse 模板

BigQuery 到 ClickHouse 模板是一个批处理管道，将数据从 BigQuery 表导入 ClickHouse 表。
该模板可以读取整个表或使用提供的查询读取特定记录。

<TOCInline toc={toc}></TOCInline>

## 管道要求 {#pipeline-requirements}

* 源 BigQuery 表必须存在。
* 目标 ClickHouse 表必须存在。
* ClickHouse 主机必须可以从 Dataflow 工作节点访问。

## 模板参数 {#template-parameters}

<br/>
<br/>

| 参数名称                  | 参数描述                                                                                                                                                                                                                                                                                                                              | 必需   | 备注                                                                                                                                                                                                                                                            |
|---------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `jdbcUrl`                 | ClickHouse 的 JDBC URL，格式为 `jdbc:clickhouse://<host>:<port>/<schema>`。                                                                                                                                                                                                                                                                  | ✅      | 不要在 JDBC 选项中添加用户名和密码。可以将其他 JDBC 选项添加到 JDBC URL 的末尾。对于 ClickHouse Cloud 用户，在 `jdbcUrl` 中添加 `ssl=true&sslmode=NONE`。                                                                  |
| `clickHouseUsername`      | 用于身份验证的 ClickHouse 用户名。                                                                                                                                                                                                                                                                                                      | ✅      |                                                                                                                                                                                                                                                                  |
| `clickHousePassword`      | 用于身份验证的 ClickHouse 密码。                                                                                                                                                                                                                                                                                                      | ✅      |                                                                                                                                                                                                                                                                  |
| `clickHouseTable`         | 要插入数据的目标 ClickHouse 表名。                                                                                                                                                                                                                                                                                            | ✅      |                                                                                                                                                                                                                                                                  |
| `maxInsertBlockSize`      | 插入的最大块大小，如果我们控制插入的块创建（ClickHouseIO 选项）。                                                                                                                                                                                                                                    |        | 一个 `ClickHouseIO` 选项。                                                                                                                                                                                                                                         |
| `insertDistributedSync`   | 如果启用设置，插入查询到分布式等待，直到数据发送到集群中的所有节点。（ClickHouseIO 选项）。                                                                                                                                                                                                                 |        | 一个 `ClickHouseIO` 选项。                                                                                                                                                                                                                                         |
| `insertQuorum`            | 对于复制表中的 INSERT 查询，等待为指定数量的副本写入并线性化数据的添加。0 - 禁用。                                                                                                                                                                                                |        | 一个 `ClickHouseIO` 选项。该设置在默认服务器设置中禁用。                                                                                                                                                                                    |
| `insertDeduplicate`       | 对于复制表中的 INSERT 查询，指定应执行插入块的去重。                                                                                                                                                                                                                                  |        | 一个 `ClickHouseIO` 选项。                                                                                                                                                                                                                                         |
| `maxRetries`              | 每次插入的最大重试次数。                                                                                                                                                                                                                                                                                                              |        | 一个 `ClickHouseIO` 选项。                                                                                                                                                                                                                                         |
| `InputTableSpec`          | 要读取的 BigQuery 表。指定 `inputTableSpec` 或 `query` 之一。当两者都设置时，`query` 参数优先。示例：`<BIGQUERY_PROJECT>:<DATASET_NAME>.<INPUT_TABLE>`。                                                                                                                                                |        | 直接使用 [BigQuery Storage Read API](https://cloud.google.com/bigquery/docs/reference/storage) 从 BigQuery 存储读取数据。要注意 [Storage Read API 的限制](https://cloud.google.com/bigquery/docs/reference/storage#limitations)。 |
| `outputDeadletterTable`   | 出现故障的消息的 BigQuery 表。如果表不存在，它会在管道执行期间创建。如果未指定，将使用 `<outputTableSpec>_error_records`。例如，`<PROJECT_ID>:<DATASET_NAME>.<DEADLETTER_TABLE>`。                                                                              |        |                                                                                                                                                                                                                                                                  |
| `query`                   | 用于从 BigQuery 读取数据的 SQL 查询。如果 BigQuery 数据集位于与 Dataflow 作业不同的项目中，请在 SQL 查询中指定完整的数据集名称，例如：`<PROJECT_ID>.<DATASET_NAME>.<TABLE_NAME>`。默认为 [GoogleSQL](https://cloud.google.com/bigquery/docs/introduction-sql)，除非 `useLegacySql` 为 true。 |        | 必须指定 `inputTableSpec` 或 `query` 之一。如果设置了两个参数，则模板使用 `query` 参数。示例：`SELECT * FROM sampledb.sample_table`。                                                                                        |
| `useLegacySql`            | 设置为 `true` 以使用传统 SQL。此参数仅在使用 `query` 参数时适用。默认为 `false`。                                                                                                                                                                                                                                |        |                                                                                                                                                                                                                                                                  |
| `queryLocation`           | 在没有底层表的权限的情况下读取授权视图时需要。例如，`US`。                                                                                                                                                                                                                                          |        |                                                                                                                                                                                                                                                                  |
| `queryTempDataset`        | 设置现有数据集以创建临时表以存储查询结果。例如，`temp_dataset`。                                                                                                                                                                                                                              |        |                                                                                                                                                                                                                                                                  |
| `KMSEncryptionKey`        | 如果通过查询源从 BigQuery 读取，请使用此 Cloud KMS 密钥加密创建的任何临时表。例如，`projects/your-project/locations/global/keyRings/your-keyring/cryptoKeys/your-key`。                                                                                                                                  |        |                                                                                                                                                                                                                                                                  |


:::note
所有 `ClickHouseIO` 参数的默认值可以在 [`ClickHouseIO` Apache Beam Connector](/integrations/apache-beam#clickhouseiowrite-parameters) 中找到
:::

## 源和目标表模式 {#source-and-target-tables-schema}

为了有效地将 BigQuery 数据集加载到 ClickHouse，并进行列感染过程，进行了以下阶段：

1. 模板基于目标 ClickHouse 表构建模式对象。
2. 模板迭代 BigQuery 数据集，并尝试根据它们的名称匹配列。

<br/>

:::important
需要说明的是，您的 BigQuery 数据集（表或查询）必须与 ClickHouse 目标表具有完全相同的列名。
:::

## 数据类型映射 {#data-types-mapping}

BigQuery 类型根据您的 ClickHouse 表定义进行转换。因此，上表列出了您应该在目标 ClickHouse 表中具有的推荐映射（针对给定的 BigQuery 表/查询）：

| BigQuery 类型                                                                                                         | ClickHouse 类型                                                 | 备注                                                                                                                                                                                                                                                                                                                                                                                                                  |
|-----------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| [**数组类型**](https://cloud.google.com/bigquery/docs/reference/standard-sql/data-types#array_type)                 | [**数组类型**](../../../sql-reference/data-types/array)       | 内部类型必须是此表中列出的支持的基本数据类型之一。                                                                                                                                                                                                                                                                                                                                 |
| [**布尔类型**](https://cloud.google.com/bigquery/docs/reference/standard-sql/data-types#boolean_type)             | [**布尔类型**](../../../sql-reference/data-types/boolean)      |                                                                                                                                                                                                                                                                                                                                                                                                                        |
| [**日期类型**](https://cloud.google.com/bigquery/docs/reference/standard-sql/data-types#date_type)                   | [**日期类型**](../../../sql-reference/data-types/date)         |                                                                                                                                                                                                                                                                                                                                                                                                                        |
| [**日期时间类型**](https://cloud.google.com/bigquery/docs/reference/standard-sql/data-types#datetime_type)           | [**日期时间类型**](../../../sql-reference/data-types/datetime) | 也适用于 `Enum8`，`Enum16` 和 `FixedString`。                                                                                                                                                                                                                                                                                                                                                                |
| [**字符串类型**](https://cloud.google.com/bigquery/docs/reference/standard-sql/data-types#string_type)               | [**字符串类型**](../../../sql-reference/data-types/string)     | 在 BigQuery 中，所有整数类型（`INT`，`SMALLINT`，`INTEGER`，`BIGINT`，`TINYINT`，`BYTEINT`）都是 `INT64` 的别名。我们建议您在 ClickHouse 中设置正确的整数大小，因为模板会根据定义的列类型（`Int8`，`Int16`，`Int32`，`Int64`）转换列。                                                                                                                          |
| [**数字 - 整数类型**](https://cloud.google.com/bigquery/docs/reference/standard-sql/data-types#numeric_types) | [**整数类型**](../../../sql-reference/data-types/int-uint) | 在 BigQuery 中，所有整数类型（`INT`，`SMALLINT`，`INTEGER`，`BIGINT`，`TINYINT`，`BYTEINT`）都是 `INT64` 的别名。我们建议您在 ClickHouse 中设置正确的整数大小，因为模板会根据定义的列类型（`Int8`，`Int16`，`Int32`，`Int64`）转换列。模板还将转换在 ClickHouse 表中使用的未分配整数类型（`UInt8`，`UInt16`，`UInt32`，`UInt64`）。 |
| [**数字 - 浮点类型**](https://cloud.google.com/bigquery/docs/reference/standard-sql/data-types#numeric_types)   | [**浮点类型**](../../../sql-reference/data-types/float)      | 支持的 ClickHouse 类型：`Float32` 和 `Float64`                                                                                                                                                                                                                                                                                                                                                                    |

## 运行模板 {#running-the-template}

BigQuery 到 ClickHouse 模板可以通过 Google Cloud CLI 执行。

:::note
请务必查看本文档，特别是上述部分，以充分了解模板的配置要求和前提条件。

:::

### 安装和配置 `gcloud` CLI {#install--configure-gcloud-cli}

- 如果尚未安装，请安装 [`gcloud` CLI](https://cloud.google.com/sdk/docs/install)。
- 请遵循 [本指南](https://cloud.google.com/dataflow/docs/guides/templates/using-flex-templates#before-you-begin) 中的 `开始之前` 部分，以设置运行 DataFlow 模板所需的配置、设置和权限。

### 运行命令 {#run-command}

使用 [`gcloud dataflow flex-template run`](https://cloud.google.com/sdk/gcloud/reference/dataflow/flex-template/run) 命令运行使用 Flex 模板的 Dataflow 作业。

以下是命令的示例：

```bash
gcloud dataflow flex-template run "bigquery-clickhouse-dataflow-$(date +%Y%m%d-%H%M%S)" \
 --template-file-gcs-location "gs://clickhouse-dataflow-templates/bigquery-clickhouse-metadata.json" \
 --parameters inputTableSpec="<bigquery table id>",jdbcUrl="jdbc:clickhouse://<clickhouse host>:<clickhouse port>/<schema>?ssl=true&sslmode=NONE",clickHouseUsername="<username>",clickHousePassword="<password>",clickHouseTable="<clickhouse target table>"
```

### 命令分解 {#command-breakdown}

- **作业名称：** `run` 关键字后面的文本是唯一的作业名称。
- **模板文件：** 由 `--template-file-gcs-location` 指定的 JSON 文件定义模板结构以及接受的参数的详细信息。提及的文件路径是公开的，可以使用。
- **参数：** 参数用逗号分隔。对于基于字符串的参数，使用双引号将值括起来。

### 预期响应 {#expected-response}

运行命令后，您应该会看到类似于以下的响应：

```bash
job:
  createTime: '2025-01-26T14:34:04.608442Z'
  currentStateTime: '1970-01-01T00:00:00Z'
  id: 2025-01-26_06_34_03-13881126003586053150
  location: us-central1
  name: bigquery-clickhouse-dataflow-20250126-153400
  projectId: ch-integrations
  startTime: '2025-01-26T14:34:04.608442Z'
```

### 监控作业 {#monitor-the-job}

在 Google Cloud 控制台中导航到 [Dataflow 作业选项卡](https://console.cloud.google.com/dataflow/jobs) 以监控作业的状态。您将找到作业的详细信息，包括进度和错误：

<img src={dataflow_inqueue_job} class="image" alt="DataFlow 正在运行的作业" style={{width: '100%', 'background-color': 'transparent'}}/>

## 故障排除 {#troubleshooting}

### 代码：241. DB::Exception: 内存限制（总计）超出 {#code-241-dbexception-memory-limit-total-exceeded}

当 ClickHouse 在处理大量数据批时耗尽内存时，会出现此错误。要解决此问题：

* 增加实例资源：将您的 ClickHouse 服务器升级到具有更多内存的大型实例，以处理数据处理负载。
* 减少批量大小：在您的 Dataflow 作业配置中调整批量大小，将较小的数据块发送到 ClickHouse，从而减少每个批次的内存消耗。
这些更改可能有助于在数据摄取期间平衡资源使用。

## 模板源代码 {#template-source-code}

模板的源代码可以在 ClickHouse 的 [DataflowTemplates](https://github.com/ClickHouse/DataflowTemplates) 分支中找到。
