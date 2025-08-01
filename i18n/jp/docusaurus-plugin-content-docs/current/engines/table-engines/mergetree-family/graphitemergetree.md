---
description: 'Designed for thinning and aggregating/averaging (rollup) Graphite
  data.'
sidebar_label: 'GraphiteMergeTree'
sidebar_position: 90
slug: '/engines/table-engines/mergetree-family/graphitemergetree'
title: 'GraphiteMergeTree'
---




# GraphiteMergeTree

このエンジンは、[Graphite](http://graphite.readthedocs.io/en/latest/index.html)データのスリムと集約/平均（ロールアップ）のために設計されています。ClickHouseをGraphiteのデータストアとして使用したい開発者にとって便利です。

ロールアップが不要な場合は、任意のClickHouseテーブルエンジンを使用してGraphiteデータを保存できますが、ロールアップが必要な場合は`GraphiteMergeTree`を使用してください。このエンジンは、ストレージのボリュームを削減し、Graphiteからのクエリの効率を向上させます。

このエンジンは、[MergeTree](../../../engines/table-engines/mergetree-family/mergetree.md)からプロパティを継承します。

## テーブルの作成 {#creating-table}

```sql
CREATE TABLE [IF NOT EXISTS] [db.]table_name [ON CLUSTER cluster]
(
    Path String,
    Time DateTime,
    Value Float64,
    Version <Numeric_type>
    ...
) ENGINE = GraphiteMergeTree(config_section)
[PARTITION BY expr]
[ORDER BY expr]
[SAMPLE BY expr]
[SETTINGS name=value, ...]
```

[CREATE TABLE](/sql-reference/statements/create/table) クエリの詳細な説明を参照してください。

Graphiteデータ用のテーブルは、以下のデータのために次のカラムを持つ必要があります：

- メトリック名（Graphiteセンサー）。データタイプ：`String`。

- メトリックを測定した時間。データタイプ：`DateTime`。

- メトリックの値。データタイプ：`Float64`。

- メトリックのバージョン。データタイプ：任意の数値（ClickHouseは、最高バージョンの行またはバージョンが同じ場合は最後に書き込まれた行を保存します。他の行はデータ部分のマージ中に削除されます）。

これらのカラムの名前は、ロールアップ設定で設定する必要があります。

**GraphiteMergeTreeのパラメータ**

- `config_section` — ロールアップのルールが設定されている設定ファイルのセクション名。

**クエリの句**

`GraphiteMergeTree`テーブルを作成する際には、`MergeTree`テーブルを作成する際と同じ[句](../../../engines/table-engines/mergetree-family/mergetree.md#table_engine-mergetree-creating-a-table)が必要です。

<details markdown="1">

<summary>テーブル作成のための非推奨メソッド</summary>

:::note
新しいプロジェクトではこのメソッドを使用せず、可能であれば古いプロジェクトを上記のメソッドに切り替えてください。
:::

```sql
CREATE TABLE [IF NOT EXISTS] [db.]table_name [ON CLUSTER cluster]
(
    EventDate Date,
    Path String,
    Time DateTime,
    Value Float64,
    Version <Numeric_type>
    ...
) ENGINE [=] GraphiteMergeTree(date-column [, sampling_expression], (primary, key), index_granularity, config_section)
```

`config_section`を除くすべてのパラメータは、`MergeTree`と同じ意味を持ちます。

- `config_section` — ロールアップのルールが設定されている設定ファイルのセクション名。

</details>

## ロールアップ設定 {#rollup-configuration}

ロールアップの設定は、サーバー設定の[graphite_rollup](../../../operations/server-configuration-parameters/settings.md#graphite)パラメータによって定義されます。このパラメータの名前は何でも構いません。複数の設定を作成し、異なるテーブルで使用することができます。

ロールアップ設定の構造：

      required-columns
      patterns

### 必須カラム {#required-columns}

#### path_column_name {#path_column_name}

`path_column_name` — メトリック名（Graphiteセンサー）を保存するカラムの名前。デフォルト値：`Path`。

#### time_column_name {#time_column_name}

`time_column_name` — メトリックの測定時刻を保存するカラムの名前。デフォルト値：`Time`。

#### value_column_name {#value_column_name}

`value_column_name` — `time_column_name`で設定した時刻におけるメトリックの値を保存するカラムの名前。デフォルト値：`Value`。

#### version_column_name {#version_column_name}

`version_column_name` — メトリックのバージョンを保存するカラムの名前。デフォルト値：`Timestamp`。

### パターン {#patterns}

`patterns`セクションの構造：

```text
pattern
    rule_type
    regexp
    function
pattern
    rule_type
    regexp
    age + precision
    ...
pattern
    rule_type
    regexp
    function
    age + precision
    ...
pattern
    ...
default
    function
    age + precision
    ...
```

:::important
パターンは厳密に順序付けられている必要があります：

1. `function`または`retention`のないパターン。
1. 両方の`function`と`retention`を持つパターン。
1. `default`パターン。
:::

行を処理する際、ClickHouseは`pattern`セクション内のルールをチェックします。それぞれの`pattern`（`default`を含む）セクションには、集約用の`function`パラメータ、`retention`パラメータのいずれかまたは両方が含まれている可能性があります。メトリック名が`regexp`に一致する場合、`pattern`セクション（またはセクション）のルールが適用されます。そうでない場合は、`default`セクションのルールが使用されます。

`pattern`および`default`セクションのフィールド：

- `rule_type` - ルールのタイプ。特定のメトリックにのみ適用されます。エンジンはこれを使用して、単純なメトリックとタグ付けされたメトリックを区別します。オプションのパラメータ。デフォルト値：`all`。
パフォーマンスが重要でない場合、または単一のメトリックタイプが使用されている場合（例えば、単純なメトリック）、これは必要ありません。デフォルトでは、ルールセットは1つだけが作成されます。別の特殊なタイプが定義されている場合は、異なる2つのセットが作成されます。1つは単純なメトリック（root.branch.leaf）用、もう1つはタグ付けされたメトリック（root.branch.leaf;tag1=value1）用です。
デフォルトルールは両方のセットに終了します。
有効な値：
    - `all`（デフォルト） - ルールのタイプが省略されたときに使用される普遍的なルール。
    - `plain` - 単純メトリック用のルール。フィールド`regexp`は正規表現として処理されます。
    - `tagged` - タグ付けされたメトリック用のルール（メトリックは`someName?tag1=value1&tag2=value2&tag3=value3`形式でDBに保存されます）。正規表現はタグ名でソートされる必要があり、最初のタグは`__name__`である必要があります（存在する場合）。フィールド`regexp`は正規表現として処理されます。
    - `tag_list` - タグ付けされたメトリック用のルール、Graphiteフォーマットでメトリック記述を簡素化するためのシンプルなDSL `someName;tag1=value1;tag2=value2`、`someName`、または`tag1=value1;tag2=value2`。フィールド`regexp`は`tagged`ルールに変換されます。タグ名によるソートは不要で、自動的に行われます。タグの値（名前ではなく）は正規表現として設定できます。例：`env=(dev|staging)`。
- `regexp` – メトリック名のパターン（通常のまたはDSL）。
- `age` – データの最小年齢（秒単位）。
- `precision`– データの年齢を定義する精度（秒単位）。86400（1日の秒数）の約数である必要があります。
- `function` – `[age, age + precision]`範囲内にあるデータに適用する集約関数の名前。受け入れられる関数：min / max / any / avg。平均は不正確に計算され、平均の平均値として算出されます。

### ルールタイプなしの設定例 {#configuration-example}

```xml
<graphite_rollup>
    <version_column_name>Version</version_column_name>
    <pattern>
        <regexp>click_cost</regexp>
        <function>any</function>
        <retention>
            <age>0</age>
            <precision>5</precision>
        </retention>
        <retention>
            <age>86400</age>
            <precision>60</precision>
        </retention>
    </pattern>
    <default>
        <function>max</function>
        <retention>
            <age>0</age>
            <precision>60</precision>
        </retention>
        <retention>
            <age>3600</age>
            <precision>300</precision>
        </retention>
        <retention>
            <age>86400</age>
            <precision>3600</precision>
        </retention>
    </default>
</graphite_rollup>
```

### ルールタイプありの設定例 {#configuration-typed-example}

```xml
<graphite_rollup>
    <version_column_name>Version</version_column_name>
    <pattern>
        <rule_type>plain</rule_type>
        <regexp>click_cost</regexp>
        <function>any</function>
        <retention>
            <age>0</age>
            <precision>5</precision>
        </retention>
        <retention>
            <age>86400</age>
            <precision>60</precision>
        </retention>
    </pattern>
    <pattern>
        <rule_type>tagged</rule_type>
        <regexp>^((.*)|.)min\?</regexp>
        <function>min</function>
        <retention>
            <age>0</age>
            <precision>5</precision>
        </retention>
        <retention>
            <age>86400</age>
            <precision>60</precision>
        </retention>
    </pattern>
    <pattern>
        <rule_type>tagged</rule_type>
        <regexp><![CDATA[^someName\?(.*&)*tag1=value1(&|$)]]></regexp>
        <function>min</function>
        <retention>
            <age>0</age>
            <precision>5</precision>
        </retention>
        <retention>
            <age>86400</age>
            <precision>60</precision>
        </retention>
    </pattern>
    <pattern>
        <rule_type>tag_list</rule_type>
        <regexp>someName;tag2=value2</regexp>
        <retention>
            <age>0</age>
            <precision>5</precision>
        </retention>
        <retention>
            <age>86400</age>
            <precision>60</precision>
        </retention>
    </pattern>
    <default>
        <function>max</function>
        <retention>
            <age>0</age>
            <precision>60</precision>
        </retention>
        <retention>
            <age>3600</age>
            <precision>300</precision>
        </retention>
        <retention>
            <age>86400</age>
            <precision>3600</precision>
        </retention>
    </default>
</graphite_rollup>
```

:::note
データのロールアップはマージ中に実行されます。通常、古いパーティションについてはマージは開始されないため、ロールアップには[optimize](../../../sql-reference/statements/optimize.md)を使用して予定外のマージをトリガーする必要があります。または、[graphite-ch-optimizer](https://github.com/innogames/graphite-ch-optimizer)などの追加ツールを使用します。
:::
