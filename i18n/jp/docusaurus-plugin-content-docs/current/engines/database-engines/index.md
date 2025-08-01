---
description: 'Documentation for Database Engines'
slug: '/engines/database-engines/'
toc_folder_title: 'Database Engines'
toc_priority: 27
toc_title: 'Introduction'
title: 'Database Engines'
---




# データベースエンジン

データベースエンジンは、テーブルを操作するためのものです。デフォルトでは、ClickHouseは[Atomic](../../engines/database-engines/atomic.md)データベースエンジンを使用しており、構成可能な[テーブルエンジン](../../engines/table-engines/index.md)と[SQLダイアレクト](../../sql-reference/syntax.md)を提供します。

以下は、利用可能なデータベースエンジンの完全なリストです。詳細についてはリンクを参照してください：

<!-- The table of contents table for this page is automatically generated by 
https://github.com/ClickHouse/clickhouse-docs/blob/main/scripts/autogenerate-table-of-contents.sh
from the YAML front matter fields: slug, description, title.

If you've spotted an error, please edit the YML frontmatter of the pages themselves.
--> | ページ | 説明 |
|-----|-----|
| [Replicated](/engines/database-engines/replicated) | このエンジンはAtomicエンジンに基づいています。DDLログをZooKeeperに書き込み、指定されたデータベースのすべてのレプリカで実行されることによってメタデータのレプリケーションをサポートします。 |
| [MySQL](/engines/database-engines/mysql) | リモートMySQLサーバー上のデータベースに接続し、ClickHouseとMySQL間でデータを交換するために`INSERT`および`SELECT`クエリを実行することを可能にします。 |
| [MaterializedPostgreSQL](/engines/database-engines/materialized-postgresql) | PostgreSQLデータベースからテーブルを持つClickHouseデータベースを作成します。 |
| [Atomic](/engines/database-engines/atomic) | `Atomic`エンジンは、ノンブロッキングの`DROP TABLE`および`RENAME TABLE`クエリ、並びにアトミックな`EXCHANGE TABLES`クエリをサポートします。デフォルトでは`Atomic`データベースエンジンが使用されます。 |
| [Lazy](/engines/database-engines/lazy) | 最後のアクセスから`expiration_time_in_seconds`秒間のみRAMにテーブルを保持します。Logタイプのテーブルでのみ使用可能です。 |
| [PostgreSQL](/engines/database-engines/postgresql) | リモートPostgreSQLサーバー上のデータベースに接続することを可能にします。 |
| [Backup](/engines/database-engines/backup) | 読み取り専用モードでバックアップからテーブル/データベースを即座にアタッチすることを可能にします。 |
| [SQLite](/engines/database-engines/sqlite) | SQLiteデータベースに接続し、ClickHouseとSQLite間でデータを交換するために`INSERT`および`SELECT`クエリを実行することを可能にします。 |
