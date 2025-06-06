---
'sidebar_label': 'CSV 和 TSV'
'slug': '/integrations/data-formats/csv-tsv'
'title': '在 ClickHouse 中使用 CSV 和 TSV 数据'
'description': '页面描述如何在 ClickHouse 中使用 CSV 和 TSV 数据'
---


# 在 ClickHouse 中处理 CSV 和 TSV 数据

ClickHouse 支持从 CSV 文件导入和导出数据。由于 CSV 文件可能具有不同的格式特性，包括标题行、自定义分隔符和转义符号，ClickHouse 提供了格式和设置，以便高效处理每种情况。

## 从 CSV 文件导入数据 {#importing-data-from-a-csv-file}

在导入数据之前，让我们创建一个具有相关结构的表：

```sql
CREATE TABLE sometable
(
    `path` String,
    `month` Date,
    `hits` UInt32
)
ENGINE = MergeTree
ORDER BY tuple(month, path)
```

要将数据从 [CSV 文件](assets/data_small.csv) 导入到 `sometable` 表中，我们可以将文件直接通过管道传递给 clickhouse-client：

```bash
clickhouse-client -q "INSERT INTO sometable FORMAT CSV" < data_small.csv
```

请注意，我们使用 [FORMAT CSV](/interfaces/formats.md/#csv) 来让 ClickHouse 知道我们正在摄取 CSV 格式的数据。或者，我们可以使用 [FROM INFILE](/sql-reference/statements/insert-into.md/#inserting-data-from-a-file) 子句从本地文件加载数据：

```sql
INSERT INTO sometable
FROM INFILE 'data_small.csv'
FORMAT CSV
```

在这里，我们使用 `FORMAT CSV` 子句来让 ClickHouse 理解文件格式。我们还可以使用 [url()](/sql-reference/table-functions/url.md) 函数直接从 URL 加载数据或使用 [s3()](/sql-reference/table-functions/s3.md) 函数从 S3 文件加载数据。

:::tip
我们可以跳过对 `file()` 和 `INFILE`/`OUTFILE` 的显式格式设置。在这种情况下，ClickHouse 将根据文件扩展名自动检测格式。
:::

### 带标题的 CSV 文件 {#csv-files-with-headers}

假设我们的 [CSV 文件带有标题](assets/data_small_headers.csv)：

```bash
head data-small-headers.csv
```
```response
"path","month","hits"
"Akiba_Hebrew_Academy","2017-08-01",241
"Aegithina_tiphia","2018-02-01",34
```

要从该文件导入数据，我们可以使用 [CSVWithNames](/interfaces/formats.md/#csvwithnames) 格式：

```bash
clickhouse-client -q "INSERT INTO sometable FORMAT CSVWithNames" < data_small_headers.csv
```

在这种情况下，ClickHouse 在从文件导入数据时会跳过第一行。

:::tip
从 23.1 [版本](https://github.com/ClickHouse/ClickHouse/releases) 开始，当使用 `CSV` 类型时，ClickHouse 将自动检测 CSV 文件中的标题，因此无需使用 `CSVWithNames` 或 `CSVWithNamesAndTypes`。
:::

### 带自定义分隔符的 CSV 文件 {#csv-files-with-custom-delimiters}

如果 CSV 文件使用除了逗号以外的其他分隔符，我们可以使用 [format_csv_delimiter](/operations/settings/settings-formats.md/#format_csv_delimiter) 选项来设置相关符号：

```sql
SET format_csv_delimiter = ';'
```

现在，当我们从 CSV 文件导入时，`;` 符号将作为分隔符使用，而不是逗号。

### 跳过 CSV 文件中的行 {#skipping-lines-in-a-csv-file}

有时，我们可能会在从 CSV 文件导入数据时跳过某些行。这可以使用 [input_format_csv_skip_first_lines](/operations/settings/settings-formats.md/#input_format_csv_skip_first_lines) 选项来完成：

```sql
SET input_format_csv_skip_first_lines = 10
```

在这种情况下，我们将跳过 CSV 文件中的前十行：

```sql
SELECT count(*) FROM file('data-small.csv', CSV)
```
```response
┌─count()─┐
│     990 │
└─────────┘
```

该 [文件](assets/data_small.csv) 有 1000 行，但 ClickHouse 只加载了 990，因为我们要求跳过前 10 行。

:::tip
使用 `file()` 函数时，使用 ClickHouse Cloud，您需要在文件所在的机器上使用 `clickhouse client` 运行命令。另一个选择是使用 [`clickhouse-local`](/operations/utilities/clickhouse-local.md) 在本地探索文件。
:::

### 在 CSV 文件中处理 NULL 值 {#treating-null-values-in-csv-files}

NULL 值的编码方式可能因生成文件的应用程序而异。默认情况下，ClickHouse 在 CSV 中使用 `\N` 表示 Null 值。但我们可以使用 [format_csv_null_representation](/operations/settings/settings-formats.md/#format_tsv_null_representation) 选项来更改此行为。

假设我们有以下 CSV 文件：

```bash
> cat nulls.csv
Donald,90
Joe,Nothing
Nothing,70
```

如果我们从该文件加载数据，ClickHouse 会将 `Nothing` 视为字符串（这是正确的）：

```sql
SELECT * FROM file('nulls.csv')
```
```response
┌─c1──────┬─c2──────┐
│ Donald  │ 90      │
│ Joe     │ Nothing │
│ Nothing │ 70      │
└─────────┴─────────┘
```

如果我们希望 ClickHouse 将 `Nothing` 视为 `NULL`，可以使用以下选项进行定义：

```sql
SET format_csv_null_representation = 'Nothing'
```

现在我们在预期的位置得到了 `NULL`：

```sql
SELECT * FROM file('nulls.csv')
```
```response
┌─c1─────┬─c2───┐
│ Donald │ 90   │
│ Joe    │ ᴺᵁᴸᴸ │
│ ᴺᵁᴸᴸ   │ 70   │
└────────┴──────┘
```

## TSV（制表符分隔）文件 {#tsv-tab-separated-files}

制表符分隔的数据格式广泛用作数据交换格式。要将数据从 [TSV 文件](assets/data_small.tsv) 加载到 ClickHouse，使用 [TabSeparated](/interfaces/formats.md/#tabseparated) 格式：

```bash
clickhouse-client -q "INSERT INTO sometable FORMAT TabSeparated" < data_small.tsv
```

还有一个 [TabSeparatedWithNames](/interfaces/formats.md/#tabseparatedwithnames) 格式，可以处理带有标题的 TSV 文件。同样，我们可以使用 [input_format_tsv_skip_first_lines](/operations/settings/settings-formats.md/#input_format_tsv_skip_first_lines) 选项跳过前 X 行。

### 原始 TSV {#raw-tsv}

有时，TSV 文件保存时没有转义制表符和换行符。我们应该使用 [TabSeparatedRaw](/interfaces/formats.md/#tabseparatedraw) 来处理此类文件。

## 导出到 CSV {#exporting-to-csv}

我们之前示例中的任何格式也可以用于导出数据。要将数据从表（或查询）导出到 CSV 格式，我们使用相同的 `FORMAT` 子句：

```sql
SELECT *
FROM sometable
LIMIT 5
FORMAT CSV
```
```response
"Akiba_Hebrew_Academy","2017-08-01",241
"Aegithina_tiphia","2018-02-01",34
"1971-72_Utah_Stars_season","2016-10-01",1
"2015_UEFA_European_Under-21_Championship_qualification_Group_8","2015-12-01",73
"2016_Greater_Western_Sydney_Giants_season","2017-05-01",86
```

要向 CSV 文件添加标题，我们使用 [CSVWithNames](/interfaces/formats.md/#csvwithnames) 格式：

```sql
SELECT *
FROM sometable
LIMIT 5
FORMAT CSVWithNames
```
```response
"path","month","hits"
"Akiba_Hebrew_Academy","2017-08-01",241
"Aegithina_tiphia","2018-02-01",34
"1971-72_Utah_Stars_season","2016-10-01",1
"2015_UEFA_European_Under-21_Championship_qualification_Group_8","2015-12-01",73
"2016_Greater_Western_Sydney_Giants_season","2017-05-01",86
```

### 将导出数据保存到 CSV 文件 {#saving-exported-data-to-a-csv-file}

要将导出数据保存到文件中，我们可以使用 [INTO...OUTFILE](/sql-reference/statements/select/into-outfile.md) 子句：

```sql
SELECT *
FROM sometable
INTO OUTFILE 'out.csv'
FORMAT CSVWithNames
```
```response
36838935 rows in set. Elapsed: 1.304 sec. Processed 36.84 million rows, 1.42 GB (28.24 million rows/s., 1.09 GB/s.)
```

请注意，ClickHouse 只需 **~1** 秒即可将 3600 万行保存到 CSV 文件中。

### 导出带自定义分隔符的 CSV {#exporting-csv-with-custom-delimiters}

如果我们希望使用除逗号以外的分隔符，我们可以使用 [format_csv_delimiter](/operations/settings/settings-formats.md/#format_csv_delimiter) 设置选项：

```sql
SET format_csv_delimiter = '|'
```

现在 ClickHouse 将使用 `|` 作为 CSV 格式的分隔符：

```sql
SELECT *
FROM sometable
LIMIT 5
FORMAT CSV
```
```response
"Akiba_Hebrew_Academy"|"2017-08-01"|241
"Aegithina_tiphia"|"2018-02-01"|34
"1971-72_Utah_Stars_season"|"2016-10-01"|1
"2015_UEFA_European_Under-21_Championship_qualification_Group_8"|"2015-12-01"|73
"2016_Greater_Western_Sydney_Giants_season"|"2017-05-01"|86
```

### 为 Windows 导出 CSV {#exporting-csv-for-windows}

如果我们希望 CSV 文件在 Windows 环境中正常工作，我们应该考虑启用 [output_format_csv_crlf_end_of_line](/operations/settings/settings-formats.md/#output_format_csv_crlf_end_of_line) 选项。这将使用 `\r\n` 作为换行符，而不是 `\n`：

```sql
SET output_format_csv_crlf_end_of_line = 1;
```

## CSV 文件的模式推断 {#schema-inference-for-csv-files}

在许多情况下，我们可能需要处理未知的 CSV 文件，因此我们必须探索为列使用哪些类型。ClickHouse 默认会根据其对给定 CSV 文件的分析来尝试猜测数据格式。这被称为“模式推断”。检测到的数据类型可以使用 `DESCRIBE` 语句与 [file()](/sql-reference/table-functions/file.md) 函数结合使用进行探索：

```sql
DESCRIBE file('data-small.csv', CSV)
```
```response
┌─name─┬─type─────────────┬─default_type─┬─default_expression─┬─comment─┬─codec_expression─┬─ttl_expression─┐
│ c1   │ Nullable(String) │              │                    │         │                  │                │
│ c2   │ Nullable(Date)   │              │                    │         │                  │                │
│ c3   │ Nullable(Int64)  │              │                    │         │                  │                │
└──────┴──────────────────┴──────────────┴────────────────────┴─────────┴──────────────────┴────────────────┘
```

在这里，ClickHouse 能够有效地猜测我们 CSV 文件的列类型。如果我们不希望 ClickHouse 进行猜测，可以使用以下选项禁用此功能：

```sql
SET input_format_csv_use_best_effort_in_schema_inference = 0
```

在这种情况下，所有列类型将被视为 `String`。

### 导入和导出具有显式列类型的 CSV {#exporting-and-importing-csv-with-explicit-column-types}

ClickHouse 还允许在使用 [CSVWithNamesAndTypes](/interfaces/formats.md/#csvwithnamesandtypes) （以及其他 *WithNames 格式系列）导出数据时显式设置列类型：

```sql
SELECT *
FROM sometable
LIMIT 5
FORMAT CSVWithNamesAndTypes
```
```response
"path","month","hits"
"String","Date","UInt32"
"Akiba_Hebrew_Academy","2017-08-01",241
"Aegithina_tiphia","2018-02-01",34
"1971-72_Utah_Stars_season","2016-10-01",1
"2015_UEFA_European_Under-21_Championship_qualification_Group_8","2015-12-01",73
"2016_Greater_Western_Sydney_Giants_season","2017-05-01",86
```

该格式将包含两行标题 - 一行包含列名称，另一行包含列类型。这将允许 ClickHouse（以及其他应用程序）在从 [这样的文件](assets/data_csv_types.csv) 加载数据时识别列类型：

```sql
DESCRIBE file('data_csv_types.csv', CSVWithNamesAndTypes)
```
```response
┌─name──┬─type───┬─default_type─┬─default_expression─┬─comment─┬─codec_expression─┬─ttl_expression─┐
│ path  │ String │              │                    │         │                  │                │
│ month │ Date   │              │                    │         │                  │                │
│ hits  │ UInt32 │              │                    │         │                  │                │
└───────┴────────┴──────────────┴────────────────────┴─────────┴──────────────────┴────────────────┘
```

现在 ClickHouse 根据第二行标题而不是进行猜测来识别列类型。

## 自定义分隔符、分隔符和转义规则 {#custom-delimiters-separators-and-escaping-rules}

在复杂的情况下，文本数据可以以高度自定义的方式格式化，但仍然具有结构。ClickHouse 有一个特殊的 [CustomSeparated](/interfaces/formats.md/#format-customseparated) 格式，用于此类情况，可以设置自定义转义规则、分隔符、行分隔符和开始/结束符号。

假设我们在文件中有以下数据：

```text
row('Akiba_Hebrew_Academy';'2017-08-01';241),row('Aegithina_tiphia';'2018-02-01';34),...
```

我们可以看到单个行被 `row()` 包裹，行之间用 `,` 分隔，单个值用 `;` 分隔。在这种情况下，我们可以使用以下设置从该文件读取数据：

```sql
SET format_custom_row_before_delimiter = 'row(';
SET format_custom_row_after_delimiter = ')';
SET format_custom_field_delimiter = ';';
SET format_custom_row_between_delimiter = ',';
SET format_custom_escaping_rule = 'Quoted';
```

现在我们可以从自定义格式的 [文件](assets/data_small_custom.txt) 中加载数据：

```sql
SELECT *
FROM file('data_small_custom.txt', CustomSeparated)
LIMIT 3
```
```response
┌─c1────────────────────────┬─────────c2─┬──c3─┐
│ Akiba_Hebrew_Academy      │ 2017-08-01 │ 241 │
│ Aegithina_tiphia          │ 2018-02-01 │  34 │
│ 1971-72_Utah_Stars_season │ 2016-10-01 │   1 │
└───────────────────────────┴────────────┴─────┘
```

我们也可以使用 [CustomSeparatedWithNames](/interfaces/formats.md/#customseparatedwithnames) 以正确导出和导入标题。请探索 [regex 和模板](templates-regex.md) 格式以处理更复杂的情况。

## 处理大 CSV 文件 {#working-with-large-csv-files}

CSV 文件可能很大，ClickHouse 可以高效处理任何大小的文件。大文件通常是压缩的，ClickHouse 在处理之前无需解压缩。我们可以在插入时使用 `COMPRESSION` 子句：

```sql
INSERT INTO sometable
FROM INFILE 'data_csv.csv.gz'
COMPRESSION 'gzip' FORMAT CSV
```

如果省略 `COMPRESSION` 子句，ClickHouse 仍会根据文件扩展名尝试猜测文件压缩。相同的方法也可以用于将文件直接导出为压缩格式：

```sql
SELECT *
FROM for_csv
INTO OUTFILE 'data_csv.csv.gz'
COMPRESSION 'gzip' FORMAT CSV
```

这将创建一个压缩的 `data_csv.csv.gz` 文件。

## 其他格式 {#other-formats}

ClickHouse 引入了对多种格式的支持，包括文本和二进制，以覆盖各种场景和平台。请在以下文章中探索更多格式及其使用方法：

- **CSV 和 TSV 格式**
- [Parquet](parquet.md)
- [JSON 格式](/integrations/data-ingestion/data-formats/json/intro.md)
- [正则表达式和模板](templates-regex.md)
- [原生和二进制格式](binary.md)
- [SQL 格式](sql.md)

同时查看 [clickhouse-local](https://clickhouse.com/blog/extracting-converting-querying-local-files-with-sql-clickhouse-local) - 一款便携式全功能工具，可在无需 Clickhouse 服务器的情况下处理本地/远程文件。
