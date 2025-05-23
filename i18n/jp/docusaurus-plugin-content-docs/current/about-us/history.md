---
slug: /about-us/history
sidebar_label: ClickHouseの歴史
sidebar_position: 40
description: ClickHouse開発の歴史
tags: ['歴史', '開発', 'Metrica']
---


# ClickHouseの歴史 {#clickhouse-history}

ClickHouseは初めて開発されたのは、[Yandex.Metrica](https://metrica.yandex.com/)を支えるためであり、[世界で2番目に大きなウェブ解析プラットフォーム](http://w3techs.com/technologies/overview/traffic_analysis/all)です。ClickHouseは現在でもそのコアコンポーネントとして機能しています。データベースには13兆件以上のレコードがあり、毎日200億件以上のイベントを処理するClickHouseは、非集約データから即座にカスタムレポートを生成することを可能にします。本記事は、ClickHouseの初期開発段階における目標を簡単にカバーします。

Yandex.Metricaは、ヒット数やセッションに基づいて、ユーザーが定義した任意のセグメントに基づいてカスタマイズされたレポートをリアルタイムで作成します。これには、ユニークユーザーの数などの複雑な集計を構築することがよく必要で、新しいデータはレポート作成のためにリアルタイムで到着します。

2014年4月の時点で、Yandex.Metricaは毎日約120億件のイベント（ページビューやクリック）を追跡していました。これらすべてのイベントはカスタムレポートを作成するために保存する必要がありました。単一のクエリは、数百ミリ秒で数百万の行をスキャンすることや、数秒で数億の行をスキャンすることを要求する場合がありました。

## Yandex.Metricaおよび他のYandexサービスでの使用 {#usage-in-yandex-metrica-and-other-yandex-services}

ClickHouseはYandex.Metricaにおいて複数の目的で使用されています。
その主なタスクは、非集約データを使用してオンラインモードでレポートを構築することです。374台のサーバークラスターを使用しており、データベースには20.3兆行を超えるデータを保存しています。圧縮データのボリュームは約2PBで、重複やレプリカは考慮していません。非圧縮データのボリューム（TSV形式）は約17PBです。

ClickHouseは以下のプロセスでも重要な役割を果たしています：

- Yandex.Metricaからのセッションリプレイのためのデータ保存。
- 中間データの処理。
- 分析によるグローバルレポートの作成。
- Yandex.Metricaエンジンのデバッグ用クエリの実行。
- APIやユーザーインターフェースからのログ分析。

現在、Yandexの他のサービスや部門には数十のClickHouseインストールがあります：検索縦列、Eコマース、広告、ビジネス分析、モバイル開発、パーソナルサービスなどがあります。

## 集約データと非集約データ {#aggregated-and-non-aggregated-data}

統計を効果的に計算するためにはデータを集約しなければならないという広く受け入れられている意見がありますが、これはデータのボリュームを減らすからです。

しかし、データ集約には多くの制限があります：

- 必要なレポートの事前定義リストが必要です。
- ユーザーはカスタムレポートを作成できません。
- 大量の異なるキーで集約すると、データのボリュームはほとんど減少せず、集約が無駄になります。
- 多数のレポートがある場合、集約のバリエーションが多すぎます（組み合わせ爆発）。
- 高いカーディナリティを持つキー（たとえばURL）を集約する際、データのボリュームはあまり減少しません（2倍未満）。
- そのため、集約を行うとデータのボリュームが増加することがあります。
- ユーザーは、私たちが生成するすべてのレポートを見ているわけではありません。多くの計算は無駄です。
- さまざまな集約によってデータの論理的整合性が侵害されることがあります。

何も集約せず、非集約データで作業すると、計算のボリュームが減少する可能性があります。

しかし、集約が行われると、作業の大部分がオフラインで行われ、比較的落ち着いて完了します。それに対して、オンライン計算は、ユーザーが結果を待っているため、可能な限り早く計算する必要があります。

Yandex.Metricaには、Metrageと呼ばれるデータを集約するための専門的なシステムがあり、これは大部分のレポートで使用されていました。
2009年以降、Yandex.Metricaは、非集約データ専用のOLAPデータベースであるOLAPServerも使用しました。これは、レポートビルダーで以前に使用されていました。
OLAPServerは非集約データに対してうまく機能しましたが、すべてのレポートに必要な制限が多くありました。これにはデータ型のサポートの欠如（数値のみ）、リアルタイムでのデータの増分更新ができないこと（データを日々書き換えることしかできませんでした）が含まれます。OLAPServerはDBMSではなく、特殊なDBです。

ClickHouseの初期目標は、OLAPServerの制限を取り除き、すべてのレポートに対して非集約データで作業する課題を解決することでしたが、年月が経つにつれて、さまざまな分析タスクに適した汎用データベース管理システムに成長しました。
