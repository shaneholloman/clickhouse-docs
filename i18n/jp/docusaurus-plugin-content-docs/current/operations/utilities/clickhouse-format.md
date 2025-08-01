---
description: 'Guide to using the format utility for working with ClickHouse data
  formats'
slug: '/operations/utilities/clickhouse-format'
title: 'clickhouse-format'
---




# clickhouse-format ユーティリティ

入力クエリをフォーマットすることができます。

キー:

- `--help` または `-h` — ヘルプメッセージを表示します。
- `--query` — 任意の長さおよび複雑さのクエリをフォーマットします。
- `--hilite` — ANSI ターミナルエスケープシーケンスを使用して構文ハイライトを追加します。
- `--oneline` — 単一行でフォーマットします。
- `--max_line_length` — 指定された長さ未満の単一行クエリをフォーマットします。
- `--comments` — 出力にコメントを保持します。
- `--quiet` または `-q` — 構文を確認するだけで、成功時には出力しません。
- `--multiquery` または `-n` — 同じファイル内で複数のクエリを許可します。
- `--obfuscate` — フォーマットの代わりに難読化します。
- `--seed <string>` — 難読化結果を決定する任意の文字列のシード。
- `--backslash` — フォーマットされたクエリの各行の末尾にバックスラッシュを追加します。これは、複数行のクエリをウェブなどからコピーし、コマンドラインで実行したい場合に便利です。

## 例 {#examples}

1. クエリのフォーマット:

```bash
$ clickhouse-format --query "select number from numbers(10) where number%2 order by number desc;"
```

結果:

```bash
SELECT number
FROM numbers(10)
WHERE number % 2
ORDER BY number DESC
```

2. ハイライトおよび単一行:

```bash
$ clickhouse-format --oneline --hilite <<< "SELECT sum(number) FROM numbers(5);"
```

結果:

```sql
SELECT sum(number) FROM numbers(5)
```

3. マルチクエリ:

```bash
$ clickhouse-format -n <<< "SELECT min(number) FROM numbers(5); SELECT max(number) FROM numbers(5);"
```

結果:

```sql
SELECT min(number)
FROM numbers(5)
;

SELECT max(number)
FROM numbers(5)
;

```

4. 難読化:

```bash
$ clickhouse-format --seed Hello --obfuscate <<< "SELECT cost_first_screen BETWEEN a AND b, CASE WHEN x >= 123 THEN y ELSE NULL END;"
```

結果:

```sql
SELECT treasury_mammoth_hazelnut BETWEEN nutmeg AND span, CASE WHEN chive >= 116 THEN switching ELSE ANYTHING END;
```

同じクエリと別のシード文字列:

```bash
$ clickhouse-format --seed World --obfuscate <<< "SELECT cost_first_screen BETWEEN a AND b, CASE WHEN x >= 123 THEN y ELSE NULL END;"
```

結果:

```sql
SELECT horse_tape_summer BETWEEN folklore AND moccasins, CASE WHEN intestine >= 116 THEN nonconformist ELSE FORESTRY END;
```

5. バックスラッシュの追加:

```bash
$ clickhouse-format --backslash <<< "SELECT * FROM (SELECT 1 AS x UNION ALL SELECT 1 UNION DISTINCT SELECT 3);"
```

結果:

```sql
SELECT * \
FROM  \
( \
    SELECT 1 AS x \
    UNION ALL \
    SELECT 1 \
    UNION DISTINCT \
    SELECT 3 \
)
```
