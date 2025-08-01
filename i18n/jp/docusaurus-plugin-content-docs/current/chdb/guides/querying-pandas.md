---
title: 'How to query Pandas DataFrames with chDB'
sidebar_label: 'Querying Pandas'
slug: '/chdb/guides/pandas'
description: 'Learn how to query Pandas DataFrames with chDB'
keywords:
- 'chdb'
- 'pandas'
---



[Pandas](https://pandas.pydata.org/) は、Python におけるデータ操作と分析のための人気のあるライブラリです。  
chDB のバージョン 2 では、Pandas DataFrame のクエリ性能を向上させ、`Python` テーブル関数を導入しました。  
このガイドでは、`Python` テーブル関数を使用して Pandas にクエリを実行する方法を学びます。

## セットアップ {#setup}

まず、仮想環境を作成しましょう：

```bash
python -m venv .venv
source .venv/bin/activate
```

次に、chDB をインストールします。  
バージョン 2.0.2 以上を持っていることを確認してください：

```bash
pip install "chdb>=2.0.2"
```

次に、Pandas といくつかの他のライブラリをインストールします：

```bash
pip install pandas requests ipython
```

これからのガイドのコマンドを実行するために `ipython` を使用します。以下のコマンドで起動できます：

```bash
ipython
```

Python スクリプトやあなたのお気に入りのノートブックでもコードを使用できます。

## URL から Pandas DataFrame を作成する {#creating-a-pandas-dataframe-from-a-url}

[StatsBomb GitHub リポジトリ](https://github.com/statsbomb/open-data/tree/master?tab=readme-ov-file) からデータをクエリします。  
まず、requests と pandas をインポートします：

```python
import requests
import pandas as pd
```

次に、1 つの試合の JSON ファイルを DataFrame に読み込みます：

```python
response = requests.get(
  "https://raw.githubusercontent.com/statsbomb/open-data/master/data/matches/223/282.json"
)
matches_df = pd.json_normalize(response.json(), sep='_')
```

どのデータを扱うのか見てみましょう：

```python
matches_df.iloc[0]
```

```text
match_id                                                                  3943077
match_date                                                             2024-07-15
kick_off                                                             04:15:00.000
home_score                                                                      1
away_score                                                                      0
match_status                                                            available
match_status_360                                                      unscheduled
last_updated                                           2024-07-15T15:50:08.671355
last_updated_360                                                             None
match_week                                                                      6
competition_competition_id                                                    223
competition_country_name                                            South America
competition_competition_name                                         Copa America
season_season_id                                                              282
season_season_name                                                           2024
home_team_home_team_id                                                        779
home_team_home_team_name                                                Argentina
home_team_home_team_gender                                                   male
home_team_home_team_group                                                    None
home_team_country_id                                                           11
home_team_country_name                                                  Argentina
home_team_managers              [{'id': 5677, 'name': 'Lionel Sebastián Scalon...
away_team_away_team_id                                                        769
away_team_away_team_name                                                 Colombia
away_team_away_team_gender                                                   male
away_team_away_team_group                                                    None
away_team_country_id                                                           49
away_team_country_name                                                   Colombia
away_team_managers              [{'id': 5905, 'name': 'Néstor Gabriel Lorenzo'...
metadata_data_version                                                       1.1.0
metadata_shot_fidelity_version                                                  2
metadata_xy_fidelity_version                                                    2
competition_stage_id                                                           26
competition_stage_name                                                      Final
stadium_id                                                                   5337
stadium_name                                                    Hard Rock Stadium
stadium_country_id                                                            241
stadium_country_name                                     United States of America
referee_id                                                                   2638
referee_name                                                        Raphael Claus
referee_country_id                                                             31
referee_country_name                                                       Brazil
Name: 0, dtype: object
```

次に、1 つのイベントの JSON ファイルを読み込み、その DataFrame に `match_id` という列を追加します：

```python
response = requests.get(
  "https://raw.githubusercontent.com/statsbomb/open-data/master/data/events/3943077.json"
)
events_df = pd.json_normalize(response.json(), sep='_')
events_df["match_id"] = 3943077
```

再度、最初の行を見てみましょう：

```python
with pd.option_context("display.max_rows", None):
    first_row = events_df.iloc[0]
    non_nan_columns = first_row[first_row.notna()].T
    display(non_nan_columns)
```

```text
id                                   279b7d66-92b5-4daa-8ff6-cba8fce271d9
index                                                                   1
period                                                                  1
timestamp                                                    00:00:00.000
minute                                                                  0
second                                                                  0
possession                                                              1
duration                                                              0.0
type_id                                                                35
type_name                                                     Starting XI
possession_team_id                                                    779
possession_team_name                                            Argentina
play_pattern_id                                                         1
play_pattern_name                                            Regular Play
team_id                                                               779
team_name                                                       Argentina
tactics_formation                                                   442.0
tactics_lineup          [{'player': {'id': 6909, 'name': 'Damián Emili...
match_id                                                          3943077
Name: 0, dtype: object
```

## Pandas DataFrame をクエリする {#querying-pandas-dataframes}

次に、chDB を使ってこれらの DataFrame にクエリを実行する方法を見てみましょう。  
ライブラリをインポートします：

```python
import chdb
```

Pandas DataFrame を `Python` テーブル関数を使用してクエリすることができます：

```sql
SELECT *
FROM Python(<name-of-variable>)
```

したがって、`matches_df` のカラムをリストアップしたい場合、次のように書くことができます：

```python
chdb.query("""
DESCRIBE Python(matches_df)
SETTINGS describe_compact_output=1
""", "DataFrame")
```

```text
                              name    type
0                         match_id   Int64
1                       match_date  String
2                         kick_off  String
3                       home_score   Int64
4                       away_score   Int64
5                     match_status  String
6                 match_status_360  String
7                     last_updated  String
8                 last_updated_360  String
9                       match_week   Int64
10      competition_competition_id   Int64
11        competition_country_name  String
12    competition_competition_name  String
13                season_season_id   Int64
14              season_season_name  String
15          home_team_home_team_id   Int64
16        home_team_home_team_name  String
17      home_team_home_team_gender  String
18       home_team_home_team_group  String
19            home_team_country_id   Int64
20          home_team_country_name  String
21              home_team_managers  String
22          away_team_away_team_id   Int64
23        away_team_away_team_name  String
24      away_team_away_team_gender  String
25       away_team_away_team_group  String
26            away_team_country_id   Int64
27          away_team_country_name  String
28              away_team_managers  String
29           metadata_data_version  String
30  metadata_shot_fidelity_version  String
31    metadata_xy_fidelity_version  String
32            competition_stage_id   Int64
33          competition_stage_name  String
34                      stadium_id   Int64
35                    stadium_name  String
36              stadium_country_id   Int64
37            stadium_country_name  String
38                      referee_id   Int64
39                    referee_name  String
40              referee_country_id   Int64
41            referee_country_name  String
```

次に、過去に 1 回以上の試合を裁いた審判を見つけるために、以下のクエリを書くことができます：

```python
chdb.query("""
SELECT referee_name, count() AS count
FROM Python(matches_df)
GROUP BY ALL
HAVING count > 1
ORDER BY count DESC
""", "DataFrame")
```

```text
                    referee_name  count
0  César Arturo Ramos Palazuelos      3
1               Maurizio Mariani      3
2               Piero Maza Gomez      3
3     Mario Alberto Escobar Toca      2
4  Wilmar Alexander Roldán Pérez      2
5          Jesús Valenzuela Sáez      2
6         Wilton Pereira Sampaio      2
7                  Darío Herrera      2
8                 Andrés Matonte      2
9                  Raphael Claus      2
```

次に、`events_df` を見てみましょう。

```python
chdb.query("""
SELECT pass_recipient_name, count()
FROM Python(events_df)
WHERE type_name = 'Pass' AND pass_recipient_name <> ''
GROUP BY ALL
ORDER BY count() DESC
LIMIT 10
""", "DataFrame")
```

```text
               pass_recipient_name  count()
0            Davinson Sánchez Mina       76
1  Ángel Fabián Di María Hernández       64
2              Alexis Mac Allister       62
3                   Enzo Fernandez       57
4      James David Rodríguez Rubio       56
5      Johan Andrés Mojica Palacio       55
6           Rodrigo Javier De Paul       54
7     Jefferson Andrés Lerma Solís       53
8        Jhon Adolfo Arias Andrade       52
9  Carlos Eccehomo Cuesta Figueroa       50
```

## Pandas DataFrame を結合する {#joining-pandas-dataframes}

クエリ内で DataFrame を結合することもできます。  
たとえば、試合の概要を得るために、以下のクエリを書くことができます：

```python
chdb.query("""
SELECT home_team_home_team_name, away_team_away_team_name, home_score, away_score,
       countIf(type_name = 'Pass' AND possession_team_id=home_team_home_team_id) AS home_passes,
       countIf(type_name = 'Pass' AND possession_team_id=away_team_away_team_id) AS away_passes,
       countIf(type_name = 'Shot' AND possession_team_id=home_team_home_team_id) AS home_shots,
       countIf(type_name = 'Shot' AND possession_team_id=away_team_away_team_id) AS away_shots
FROM Python(matches_df) AS matches
JOIN Python(events_df) AS events ON events.match_id = matches.match_id
GROUP BY ALL
LIMIT 5
""", "DataFrame").iloc[0]
```

```text
home_team_home_team_name    Argentina
away_team_away_team_name     Colombia
home_score                          1
away_score                          0
home_passes                       527
away_passes                       669
home_shots                         11
away_shots                         19
Name: 0, dtype: object
```

## DataFrame からテーブルを作成する {#populating-a-table-from-a-dataframe}

DataFrame から ClickHouse テーブルを作成して populate することも可能です。  
chDB にテーブルを作成するには Stateful Session API を使用する必要があります。

セッションモジュールをインポートしましょう：

```python
from chdb import session as chs
```

セッションを初期化します：

```python
sess = chs.Session()
```

次に、データベースを作成します：

```python
sess.query("CREATE DATABASE statsbomb")
```

次に、`events_df` に基づいて `events` テーブルを作成します：

```python
sess.query("""
CREATE TABLE statsbomb.events ORDER BY id AS
SELECT * 
FROM Python(events_df)
""")
```

その後、最も多くのパスを受け取った選手を返すクエリを実行します：

```python
sess.query("""
SELECT pass_recipient_name, count()
FROM statsbomb.events
WHERE type_name = 'Pass' AND pass_recipient_name <> ''
GROUP BY ALL
ORDER BY count() DESC
LIMIT 10
""", "DataFrame")
```

```text
               pass_recipient_name  count()
0            Davinson Sánchez Mina       76
1  Ángel Fabián Di María Hernández       64
2              Alexis Mac Allister       62
3                   Enzo Fernandez       57
4      James David Rodríguez Rubio       56
5      Johan Andrés Mojica Palacio       55
6           Rodrigo Javier De Paul       54
7     Jefferson Andrés Lerma Solís       53
8        Jhon Adolfo Arias Andrade       52
9  Carlos Eccehomo Cuesta Figueroa       50
```

## Pandas DataFrame とテーブルを結合する {#joining-a-pandas-dataframe-and-table}

最後に、結合クエリを更新して `matches_df` DataFrame を `statsbomb.events` テーブルと結合することもできます：

```python
sess.query("""
SELECT home_team_home_team_name, away_team_away_team_name, home_score, away_score,
       countIf(type_name = 'Pass' AND possession_team_id=home_team_home_team_id) AS home_passes,
       countIf(type_name = 'Pass' AND possession_team_id=away_team_away_team_id) AS away_passes,
       countIf(type_name = 'Shot' AND possession_team_id=home_team_home_team_id) AS home_shots,
       countIf(type_name = 'Shot' AND possession_team_id=away_team_away_team_id) AS away_shots
FROM Python(matches_df) AS matches
JOIN statsbomb.events AS events ON events.match_id = matches.match_id
GROUP BY ALL
LIMIT 5
""", "DataFrame").iloc[0]
```

```text
home_team_home_team_name    Argentina
away_team_away_team_name     Colombia
home_score                          1
away_score                          0
home_passes                       527
away_passes                       669
home_shots                         11
away_shots                         19
Name: 0, dtype: object
```
