---
description: 'The GenerateRandom table engine produces random data for given table
  schema.'
sidebar_label: 'GenerateRandom'
sidebar_position: 140
slug: '/engines/table-engines/special/generate'
title: 'GenerateRandom Table Engine'
---



The GenerateRandom table engine produces random data for given table schema.

Usage examples:

- Use in test to populate reproducible large table.
- Generate random input for fuzzing tests.

## 使用法 in ClickHouse Server {#usage-in-clickhouse-server}

```sql
ENGINE = GenerateRandom([random_seed [,max_string_length [,max_array_length]]])
```

The `max_array_length` and `max_string_length` parameters specify maximum length of all
array or map columns and strings correspondingly in generated data.

Generate table engine supports only `SELECT` queries.

It supports all [DataTypes](../../../sql-reference/data-types/index.md) that can be stored in a table except `AggregateFunction`.

## 例 {#example}

**1.** Set up the `generate_engine_table` table:

```sql
CREATE TABLE generate_engine_table (name String, value UInt32) ENGINE = GenerateRandom(1, 5, 3)
```

**2.** Query the data:

```sql
SELECT * FROM generate_engine_table LIMIT 3
```

```text
┌─name─┬──────value─┐
│ c4xJ │ 1412771199 │
│ r    │ 1791099446 │
│ 7#$  │  124312908 │
└──────┴────────────┘
```

## 実装の詳細 {#details-of-implementation}

- Not supported:
    - `ALTER`
    - `SELECT ... SAMPLE`
    - `INSERT`
    - Indices
    - Replication
