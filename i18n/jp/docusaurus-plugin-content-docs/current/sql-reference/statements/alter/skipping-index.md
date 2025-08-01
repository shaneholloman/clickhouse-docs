---
description: 'Documentation for Manipulating Data Skipping Indices'
sidebar_label: 'インデックス'
sidebar_position: 42
slug: '/sql-reference/statements/alter/skipping-index'
title: 'Manipulating Data Skipping Indices'
toc_hidden_folder: true
---




# データスキッピングインデックスの操作

以下の操作が利用可能です。

## ADD INDEX {#add-index}

`ALTER TABLE [db.]table_name [ON CLUSTER cluster] ADD INDEX [IF NOT EXISTS] name expression TYPE type [GRANULARITY value] [FIRST|AFTER name]` - インデックスの説明をテーブルのメタデータに追加します。

## DROP INDEX {#drop-index}

`ALTER TABLE [db.]table_name [ON CLUSTER cluster] DROP INDEX [IF EXISTS] name` - テーブルのメタデータからインデックスの説明を削除し、ディスクからインデックスファイルを削除します。[mutation](/sql-reference/statements/alter/index.md#mutations)として実装されています。

## MATERIALIZE INDEX {#materialize-index}

`ALTER TABLE [db.]table_name [ON CLUSTER cluster] MATERIALIZE INDEX [IF EXISTS] name [IN PARTITION partition_name]` - 指定された `partition_name` に対して二次インデックス `name` を再構築します。[mutation](/sql-reference/statements/alter/index.md#mutations)として実装されています。`IN PARTITION` 部分が省略された場合、テーブル全体のデータに対してインデックスを再構築します。

## CLEAR INDEX {#clear-index}

`ALTER TABLE [db.]table_name [ON CLUSTER cluster] CLEAR INDEX [IF EXISTS] name [IN PARTITION partition_name]` - 説明を削除せずにディスクから二次インデックスファイルを削除します。[mutation](/sql-reference/statements/alter/index.md#mutations)として実装されています。

コマンド `ADD`、`DROP`、および `CLEAR` は、メタデータを変更するかファイルを削除するだけであり、軽量です。また、ClickHouse KeeperまたはZooKeeperを介してインデックスのメタデータを同期するため、レプリケートされます。

:::note    
インデックス操作は、[`*MergeTree`](/engines/table-engines/mergetree-family/mergetree.md) エンジン（[replicated](/engines/table-engines/mergetree-family/replication.md) バリエーションを含む）を持つテーブルに対してのみサポートされています。
:::
