---
description: '同じ名前の下にグループ化された設定のコレクション。'
sidebar_label: '設定プロファイル'
sidebar_position: 61
slug: '/operations/settings/settings-profiles'
title: 'Settings Profiles'
---




# 設定プロファイル

設定プロファイルは、同じ名前の下にグループ化された設定のコレクションです。

:::note
ClickHouse は、設定プロファイルを管理するための [SQL駆動型ワークフロー](/operations/access-rights#access-control-usage) もサポートしています。これを使用することをお勧めします。
:::

プロファイルには任意の名前を付けることができます。異なるユーザーに同じプロファイルを指定することも可能です。設定プロファイルで最も重要なことは、`readonly=1` を記述することです。これにより、読み取り専用アクセスが保証されます。

設定プロファイルは互いに継承することができます。継承を使用するには、プロファイルにリストされている他の設定の前に、一つまたは複数の `profile` 設定を示します。異なるプロファイルで同じ設定が定義されている場合は、最新に定義されたものが使用されます。

プロファイル内のすべての設定を適用するには、`profile` 設定を設定します。

例:

`web` プロファイルをインストールします。

```sql
SET profile = 'web'
```

設定プロファイルはユーザー設定ファイルで宣言されます。これは通常 `users.xml` です。

例:

```xml
<!-- 設定プロファイル -->
<profiles>
    <!-- デフォルト設定 -->
    <default>
        <!-- 単一のクエリを実行する際の最大スレッド数。 -->
        <max_threads>8</max_threads>
    </default>

    <!-- ユーザーインターフェースからのクエリの設定 -->
    <web>
        <max_rows_to_read>1000000000</max_rows_to_read>
        <max_bytes_to_read>100000000000</max_bytes_to_read>

        <max_rows_to_group_by>1000000</max_rows_to_group_by>
        <group_by_overflow_mode>any</group_by_overflow_mode>

        <max_rows_to_sort>1000000</max_rows_to_sort>
        <max_bytes_to_sort>1000000000</max_bytes_to_sort>

        <max_result_rows>100000</max_result_rows>
        <max_result_bytes>100000000</max_result_bytes>
        <result_overflow_mode>break</result_overflow_mode>

        <max_execution_time>600</max_execution_time>
        <min_execution_speed>1000000</min_execution_speed>
        <timeout_before_checking_execution_speed>15</timeout_before_checking_execution_speed>

        <max_columns_to_read>25</max_columns_to_read>
        <max_temporary_columns>100</max_temporary_columns>
        <max_temporary_non_const_columns>50</max_temporary_non_const_columns>

        <max_subquery_depth>2</max_subquery_depth>
        <max_pipeline_depth>25</max_pipeline_depth>
        <max_ast_depth>50</max_ast_depth>
        <max_ast_elements>100</max_ast_elements>

        <max_sessions_for_user>4</max_sessions_for_user>

        <readonly>1</readonly>
    </web>
</profiles>
```

この例では、`default` と `web` の2つのプロファイルが指定されています。

`default` プロファイルは特別な目的を持っています: 常に存在しなければならず、サーバーを起動する際に適用されます。言い換えれば、`default` プロファイルにはデフォルト設定が含まれています。

`web` プロファイルは通常のプロファイルであり、`SET` クエリを使用するか、HTTP クエリの URL パラメータを使用して設定できます。
