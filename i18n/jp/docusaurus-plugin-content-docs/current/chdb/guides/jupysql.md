---
title: JupySQL と chDB
sidebar_label: JupySQL
slug: /chdb/guides/jupysql
description: chDBをBunにインストールする方法
keywords: [chdb, JupySQL]
---

import PlayersPerRank from '@site/static/images/chdb/guides/players_per_rank.png';

[JupySQL](https://jupysql.ploomber.io/en/latest/quick-start.html) は、JupyterノートブックとIPythonシェルでSQLを実行するためのPythonライブラリです。このガイドでは、chDBとJupySQLを使用してデータをクエリする方法を学びます。

<div class='vimeo-container'>
<iframe width="560" height="315" src="https://www.youtube.com/embed/2wjl3OijCto?si=EVf2JhjS5fe4j6Cy" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>
</div>

## セットアップ {#setup}

まず、仮想環境を作成します：

```bash
python -m venv .venv
source .venv/bin/activate
```

次に、JupySQL、IPython、およびJupyter Labをインストールします：

```bash
pip install jupysql ipython jupyterlab
```

JupySQLをIPythonで使用できます。IPythonを起動するには、以下を実行します：

```bash
ipython
```

または、Jupyter Labで実行するには：

```bash
jupyter lab
```

:::note
Jupyter Labを使用している場合は、他のガイドの続きに従う前にノートブックを作成する必要があります。
:::

## データセットのダウンロード {#downloading-a-dataset}

[Jeff Sackmannの tennis_atp](https://github.com/JeffSackmann/tennis_atp) データセットの1つを使用します。このデータセットには、選手のメタデータとそのランキングの履歴が含まれています。まず、ランキングファイルをダウンロードします：

```python
from urllib.request import urlretrieve
```

```python
files = ['00s', '10s', '20s', '70s', '80s', '90s', 'current']
base = "https://raw.githubusercontent.com/JeffSackmann/tennis_atp/master"
for file in files:
  _ = urlretrieve(
    f"{base}/atp_rankings_{file}.csv",
    f"atp_rankings_{file}.csv",
  )
```

## chDB と JupySQL の設定 {#configuring-chdb-and-jupysql}

次に、chDBの `dbapi` モジュールをインポートします：

```python
from chdb import dbapi
```

chDB接続を作成します。永続化したデータは `atp.chdb` ディレクトリに保存されます：

```python
conn = dbapi.connect(path="atp.chdb")
```

次に、`sql` マジックを読み込み、chDBへの接続を作成します：

```python
%load_ext sql
%sql conn --alias chdb
```

次に、クエリの結果が切り捨てられないように表示制限を設定します：

```python
%config SqlMagic.displaylimit = None
```

## CSVファイル内のデータのクエリ {#querying-data-in-csv-files}

`atp_rankings` プレフィックスの付いたファイルをダウンロードしました。スキーマを確認するために `DESCRIBE` クエリを使用しましょう：

```python
%%sql
DESCRIBE file('atp_rankings*.csv')
SETTINGS describe_compact_output=1,
         schema_inference_make_columns_nullable=0
```

```text
+--------------+-------+
|     name     |  type |
+--------------+-------+
| ranking_date | Int64 |
|     rank     | Int64 |
|    player    | Int64 |
|    points    | Int64 |
+--------------+-------+
```

これらのファイルに対して直接 `SELECT` クエリを書いて、データがどうなっているか見てみましょう：

```python
%sql SELECT * FROM file('atp_rankings*.csv') LIMIT 1
```

```text
+--------------+------+--------+--------+
| ranking_date | rank | player | points |
+--------------+------+--------+--------+
|   20000110   |  1   | 101736 |  4135  |
+--------------+------+--------+--------+
```

データの形式は少し奇妙です。日付をきれいにし、`REPLACE` クエリを使用してきれいになった `ranking_date` を返します：

```python
%%sql
SELECT * REPLACE (
  toDate(parseDateTime32BestEffort(toString(ranking_date))) AS ranking_date
)
FROM file('atp_rankings*.csv')
LIMIT 10
SETTINGS schema_inference_make_columns_nullable=0
```

```text
+--------------+------+--------+--------+
| ranking_date | rank | player | points |
+--------------+------+--------+--------+
|  2000-01-10  |  1   | 101736 |  4135  |
|  2000-01-10  |  2   | 102338 |  2915  |
|  2000-01-10  |  3   | 101948 |  2419  |
|  2000-01-10  |  4   | 103017 |  2184  |
|  2000-01-10  |  5   | 102856 |  2169  |
|  2000-01-10  |  6   | 102358 |  2107  |
|  2000-01-10  |  7   | 102839 |  1966  |
|  2000-01-10  |  8   | 101774 |  1929  |
|  2000-01-10  |  9   | 102701 |  1846  |
|  2000-01-10  |  10  | 101990 |  1739  |
+--------------+------+--------+--------+
```

## chDB への CSVファイルのインポート {#importing-csv-files-into-chdb}

これらのCSVファイルからデータをテーブルに格納します。デフォルトのデータベースはディスク上にデータを永続化しないため、最初に別のデータベースを作成する必要があります：

```python
%sql CREATE DATABASE atp
```

次に、CSVファイルのデータ構造に基づいて `rankings` というテーブルを作成します：

```python
%%sql
CREATE TABLE atp.rankings
ENGINE=MergeTree
ORDER BY ranking_date AS
SELECT * REPLACE (
  toDate(parseDateTime32BestEffort(toString(ranking_date))) AS ranking_date
)
FROM file('atp_rankings*.csv')
SETTINGS schema_inference_make_columns_nullable=0
```

テーブル内のデータを簡単に確認しましょう：

```python
%sql SELECT * FROM atp.rankings LIMIT 10
```

```text
+--------------+------+--------+--------+
| ranking_date | rank | player | points |
+--------------+------+--------+--------+
|  2000-01-10  |  1   | 101736 |  4135  |
|  2000-01-10  |  2   | 102338 |  2915  |
|  2000-01-10  |  3   | 101948 |  2419  |
|  2000-01-10  |  4   | 103017 |  2184  |
|  2000-01-10  |  5   | 102856 |  2169  |
|  2000-01-10  |  6   | 102358 |  2107  |
|  2000-01-10  |  7   | 102839 |  1966  |
|  2000-01-10  |  8   | 101774 |  1929  |
|  2000-01-10  |  9   | 102701 |  1846  |
|  2000-01-10  |  10  | 101990 |  1739  |
+--------------+------+--------+--------+
```

良さそうです - 出力は、直接CSVファイルをクエリしたときと期待通り同じです。

選手のメタデータについても同じプロセスを行います。今回はデータが単一のCSVファイルにまとめられているので、そのファイルをダウンロードします：

```python
_ = urlretrieve(
    f"{base}/atp_players.csv",
    "atp_players.csv",
)
```

次に、CSVファイルの内容に基づいて `players` というテーブルを作成します。また、`dob` フィールドを `Date32` 型にします。

> ClickHouseでは、`Date` 型は1970年以降の日付のみをサポートしています。`dob` 列には1970年以前の日付が含まれているため、代わりに `Date32` 型を使用します。

```python
%%sql
CREATE TABLE atp.players
Engine=MergeTree
ORDER BY player_id AS
SELECT * REPLACE (
  makeDate32(
    toInt32OrNull(substring(toString(dob), 1, 4)),
    toInt32OrNull(substring(toString(dob), 5, 2)),
    toInt32OrNull(substring(toString(dob), 7, 2))
  )::Nullable(Date32) AS dob
)
FROM file('atp_players.csv')
SETTINGS schema_inference_make_columns_nullable=0
```

終了したら、取り込んだデータを確認します：

```python
%sql SELECT * FROM atp.players LIMIT 10
```

```text
+-----------+------------+-----------+------+------------+-----+--------+-------------+
| player_id | name_first | name_last | hand |    dob     | ioc | height | wikidata_id |
+-----------+------------+-----------+------+------------+-----+--------+-------------+
|   100001  |  Gardnar   |   Mulloy  |  R   | 1913-11-22 | USA |  185   |    Q54544   |
|   100002  |   Pancho   |   Segura  |  R   | 1921-06-20 | ECU |  168   |    Q54581   |
|   100003  |   Frank    |  Sedgman  |  R   | 1927-10-02 | AUS |  180   |   Q962049   |
|   100004  |  Giuseppe  |   Merlo   |  R   | 1927-10-11 | ITA |   0    |   Q1258752  |
|   100005  |  Richard   |  Gonzalez |  R   | 1928-05-09 | USA |  188   |    Q53554   |
|   100006  |   Grant    |   Golden  |  R   | 1929-08-21 | USA |  175   |   Q3115390  |
|   100007  |    Abe     |   Segal   |  L   | 1930-10-23 | RSA |   0    |   Q1258527  |
|   100008  |    Kurt    |  Nielsen  |  R   | 1930-11-19 | DEN |   0    |   Q552261   |
|   100009  |   Istvan   |   Gulyas  |  R   | 1931-10-14 | HUN |   0    |    Q51066   |
|   100010  |    Luis    |   Ayala   |  R   | 1932-09-18 | CHI |  170   |   Q1275397  |
+-----------+------------+-----------+------+------------+-----+--------+-------------+
```

## chDB に対するクエリ {#querying-chdb}

データの取り込みが完了しました。さて、データをクエリする楽しい部分です！

テニス選手は、参加するトーナメントでのパフォーマンスに基づいてポイントを受け取ります。各選手のポイントは52週間のローリング期間で集計されます。選手ごとに最大ポイントを見つけ、その時のランクを取得するクエリを書きます：

```python
%%sql
SELECT name_first, name_last,
       max(points) as maxPoints,
       argMax(rank, points) as rank,
       argMax(ranking_date, points) as date
FROM atp.players
JOIN atp.rankings ON rankings.player = players.player_id
GROUP BY ALL
ORDER BY maxPoints DESC
LIMIT 10
```

```text
+------------+-----------+-----------+------+------------+
| name_first | name_last | maxPoints | rank |    date    |
+------------+-----------+-----------+------+------------+
|   Novak    |  Djokovic |   16950   |  1   | 2016-06-06 |
|   Rafael   |   Nadal   |   15390   |  1   | 2009-04-20 |
|    Andy    |   Murray  |   12685   |  1   | 2016-11-21 |
|   Roger    |  Federer  |   12315   |  1   | 2012-10-29 |
|   Daniil   |  Medvedev |   10780   |  2   | 2021-09-13 |
|   Carlos   |  Alcaraz  |    9815   |  1   | 2023-08-21 |
|  Dominic   |   Thiem   |    9125   |  3   | 2021-01-18 |
|   Jannik   |   Sinner  |    8860   |  2   | 2024-05-06 |
|  Stefanos  | Tsitsipas |    8350   |  3   | 2021-09-20 |
| Alexander  |   Zverev  |    8240   |  4   | 2021-08-23 |
+------------+-----------+-----------+------+------------+
```

このリストの選手の中には、ポイントが多いにもかかわらず1位でない選手がいることが非常に興味深いです。

## クエリの保存 {#saving-queries}

クエリを保存するには、`%%sql` マジックと同じ行に `--save` パラメータを使用します。`--no-execute` パラメータは、クエリの実行をスキップすることを意味します。

```python
%%sql --save best_points --no-execute
SELECT name_first, name_last,
       max(points) as maxPoints,
       argMax(rank, points) as rank,
       argMax(ranking_date, points) as date
FROM atp.players
JOIN atp.rankings ON rankings.player = players.player_id
GROUP BY ALL
ORDER BY maxPoints DESC
```

保存したクエリを実行すると、実行前に共通テーブル式（CTE）に変換されます。次のクエリでは、選手が1位だったときに達成した最大ポイントを計算します：

```python
%sql select * FROM best_points WHERE rank=1
```

```text
+-------------+-----------+-----------+------+------------+
|  name_first | name_last | maxPoints | rank |    date    |
+-------------+-----------+-----------+------+------------+
|    Novak    |  Djokovic |   16950   |  1   | 2016-06-06 |
|    Rafael   |   Nadal   |   15390   |  1   | 2009-04-20 |
|     Andy    |   Murray  |   12685   |  1   | 2016-11-21 |
|    Roger    |  Federer  |   12315   |  1   | 2012-10-29 |
|    Carlos   |  Alcaraz  |    9815   |  1   | 2023-08-21 |
|     Pete    |  Sampras  |    5792   |  1   | 1997-08-11 |
|    Andre    |   Agassi  |    5652   |  1   | 1995-08-21 |
|   Lleyton   |   Hewitt  |    5205   |  1   | 2002-08-12 |
|   Gustavo   |  Kuerten  |    4750   |  1   | 2001-09-10 |
| Juan Carlos |  Ferrero  |    4570   |  1   | 2003-10-20 |
|    Stefan   |   Edberg  |    3997   |  1   | 1991-02-25 |
|     Jim     |  Courier  |    3973   |  1   | 1993-08-23 |
|     Ivan    |   Lendl   |    3420   |  1   | 1990-02-26 |
|     Ilie    |  Nastase  |     0     |  1   | 1973-08-27 |
+-------------+-----------+-----------+------+------------+
```

## パラメータを使ってクエリする {#querying-with-parameters}

クエリでパラメータを使用することもできます。パラメータは通常の変数です：

```python
rank = 10
```

次に、クエリ内で `{{variable}}` 構文を使用します。次のクエリは、選手がランキングのトップ10に初めて入ってから最後に入ったまでの最小日数を取得します：

```python
%%sql
SELECT name_first, name_last,
       MIN(ranking_date) AS earliest_date,
       MAX(ranking_date) AS most_recent_date,
       most_recent_date - earliest_date AS days,
       1 + (days/7) AS weeks
FROM atp.rankings
JOIN atp.players ON players.player_id = rankings.player
WHERE rank <= {{rank}}
GROUP BY ALL
ORDER BY days
LIMIT 10
```

```text
+------------+-----------+---------------+------------------+------+-------+
| name_first | name_last | earliest_date | most_recent_date | days | weeks |
+------------+-----------+---------------+------------------+------+-------+
|    Alex    | Metreveli |   1974-06-03  |    1974-06-03    |  0   |   1   |
|   Mikael   |  Pernfors |   1986-09-22  |    1986-09-22    |  0   |   1   |
|   Felix    |  Mantilla |   1998-06-08  |    1998-06-08    |  0   |   1   |
|   Wojtek   |   Fibak   |   1977-07-25  |    1977-07-25    |  0   |   1   |
|  Thierry   |  Tulasne  |   1986-08-04  |    1986-08-04    |  0   |   1   |
|   Lucas    |  Pouille  |   2018-03-19  |    2018-03-19    |  0   |   1   |
|    John    | Alexander |   1975-12-15  |    1975-12-15    |  0   |   1   |
|  Nicolas   |   Massu   |   2004-09-13  |    2004-09-20    |  7   |   2   |
|   Arnaud   |  Clement  |   2001-04-02  |    2001-04-09    |  7   |   2   |
|  Ernests   |   Gulbis  |   2014-06-09  |    2014-06-23    |  14  |   3   |
+------------+-----------+---------------+------------------+------+-------+
```

## ヒストグラムのプロット {#plotting-histograms}

JupySQLは、限られたチャート機能も持っています。ボックスプロットやヒストグラムを作成できます。

ヒストグラムを作成しますが、まずは各選手が達成したランキング内のランキングを計算するクエリを書いて（保存して）おきましょう。このクエリを使用して、各ランキングを達成した選手の数をカウントするヒストグラムを作成できます：

```python
%%sql --save players_per_rank --no-execute
select distinct player, rank
FROM atp.rankings
WHERE rank <= 100
```

次に、以下のコードを実行してヒストグラムを作成します：

```python
from sql.ggplot import ggplot, geom_histogram, aes

plot = (
  ggplot(
    table="players_per_rank",
    with_="players_per_rank",
    mapping=aes(x="rank", fill="#69f0ae", color="#fff"),
  ) + geom_histogram(bins=100)
)
```

<img src={PlayersPerRank} alt="ATPデータセットにおけるプレイヤーランキングのヒストグラム" class="image" style={{width: '90%', padding: '30px'}} />

