---
description: 'Documentation for the `EXISTS` operator'
slug: '/sql-reference/operators/exists'
title: 'EXISTS'
---




# EXISTS

`EXISTS` 演算子は、サブクエリの結果にレコードがいくつあるかを確認します。もし結果が空であれば、演算子は `0` を返します。そうでない場合は `1` を返します。

`EXISTS` は [WHERE](../../sql-reference/statements/select/where.md) 句でも使用できます。

:::tip    
サブクエリ内でのメインクエリのテーブルやカラムへの参照はサポートされていません。
:::

**構文**

```sql
EXISTS(subquery)
```

**例**

サブクエリ内の値の存在を確認するクエリ:

```sql
SELECT EXISTS(SELECT * FROM numbers(10) WHERE number > 8), EXISTS(SELECT * FROM numbers(10) WHERE number > 11)
```

結果:

```text
┌─in(1, _subquery1)─┬─in(1, _subquery2)─┐
│                 1 │                 0 │
└───────────────────┴───────────────────┘
```

複数行を返すサブクエリを含むクエリ:

```sql
SELECT count() FROM numbers(10) WHERE EXISTS(SELECT number FROM numbers(10) WHERE number > 8);
```

結果:

```text
┌─count()─┐
│      10 │
└─────────┘
```

空の結果を返すサブクエリを含むクエリ:

```sql
SELECT count() FROM numbers(10) WHERE EXISTS(SELECT number FROM numbers(10) WHERE number > 11);
```

結果:

```text
┌─count()─┐
│       0 │
└─────────┘
```
