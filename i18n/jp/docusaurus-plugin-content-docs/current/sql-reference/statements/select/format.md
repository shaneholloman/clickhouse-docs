---
description: 'FORMAT 句のドキュメント'
sidebar_label: 'FORMAT'
slug: '/sql-reference/statements/select/format'
title: 'FORMAT Clause'
---




# FORMAT 句

ClickHouse は、クエリ結果を含むさまざまな [シリアル化フォーマット](../../../interfaces/formats.md) をサポートしています。`SELECT` の出力フォーマットを選択する方法はいくつかあり、その一つはクエリの最後に `FORMAT format` を指定して特定のフォーマットで結果データを取得することです。

特定のフォーマットは、便利さ、他のシステムとの統合、またはパフォーマンス向上のために使用されることがあります。

## デフォルトフォーマット {#default-format}

`FORMAT` 句が省略された場合、デフォルトフォーマットが使用されます。このフォーマットは、ClickHouse サーバーへのアクセスに使用される設定とインターフェースの両方に依存します。[HTTP インターフェース](../../../interfaces/http.md) およびバッチモードの [コマンドラインクライアント](../../../interfaces/cli.md) では、デフォルトフォーマットは `TabSeparated` です。インタラクティブモードのコマンドラインクライアントでは、デフォルトフォーマットは `PrettyCompact` です（これは、コンパクトで人間が読みやすいテーブルを生成します）。

## 実装の詳細 {#implementation-details}

コマンドラインクライアントを使用する際、データは常に内部効率的なフォーマット（`Native`）でネットワークを介して送信されます。クライアントはクエリの `FORMAT` 句を独自に解釈し、データを自らフォーマットします（したがって、ネットワークとサーバーの追加負荷を軽減します）。
