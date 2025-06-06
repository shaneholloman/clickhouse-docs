---
'sidebar_label': 'Deepnote'
'sidebar_position': 11
'slug': '/integrations/deepnote'
'keywords':
- 'clickhouse'
- 'Deepnote'
- 'connect'
- 'integrate'
- 'notebook'
'description': '高效查询非常大的数据集，在熟悉的笔记本环境中进行分析和建模。'
'title': '连接 ClickHouse 到 Deepnote'
---

import deepnote_01 from '@site/static/images/integrations/data-visualization/deepnote_01.png';
import deepnote_02 from '@site/static/images/integrations/data-visualization/deepnote_02.png';
import deepnote_03 from '@site/static/images/integrations/data-visualization/deepnote_03.png';
import Image from '@theme/IdealImage';
import CommunityMaintainedBadge from '@theme/badges/CommunityMaintained';
import ConnectionDetails from '@site/i18n/zh/docusaurus-plugin-content-docs/current/_snippets/_gather_your_details_http.mdx';


# 将 ClickHouse 连接到 Deepnote

<CommunityMaintainedBadge/>

<a href="https://www.deepnote.com/" target="_blank">Deepnote</a> 是一个协作数据笔记本，旨在帮助团队发现和分享洞察。除了与 Jupyter 兼容之外，它还在云端工作，为您提供一个集中地点，以高效地协作和处理数据科学项目。

本指南假设您已经有一个 Deepnote 账户，并且您有一个正在运行的 ClickHouse 实例。

## 交互式示例 {#interactive-example}
如果您想探索通过 Deepnote 数据笔记本查询 ClickHouse 的交互式示例，请点击下面的按钮以启动一个连接到 [ClickHouse playground](../../getting-started/playground.md) 的模板项目。

[<Image size="logo" img="https://deepnote.com/buttons/launch-in-deepnote.svg" alt="Launch in Deepnote" />](https://deepnote.com/launch?template=ClickHouse%20and%20Deepnote)

## 连接到 ClickHouse {#connect-to-clickhouse}

1. 在 Deepnote 中，选择“集成”概述并点击 ClickHouse 瓷砖。

<Image size="lg" img={deepnote_01} alt="ClickHouse integration tile" border />

2. 提供您的 ClickHouse 实例的连接详细信息：
<ConnectionDetails />

   <Image size="md" img={deepnote_02} alt="ClickHouse details dialog" border />

   **_注意：_** 如果您的 ClickHouse 连接受 IP 访问列表保护，您可能需要允许 Deepnote 的 IP 地址。请在 [Deepnote 的文档](https://docs.deepnote.com/integrations/authorize-connections-from-deepnote-ip-addresses) 中了解更多信息。

3. 恭喜！您现在已经将 ClickHouse 集成到 Deepnote 中。

## 使用 ClickHouse 集成 {#using-clickhouse-integration}

1. 首先，连接到您笔记本右侧的 ClickHouse 集成。

   <Image size="lg" img={deepnote_03} alt="ClickHouse details dialog" border />

2. 现在创建一个新的 ClickHouse 查询块并查询您的数据库。查询结果将作为 DataFrame 保存，并存储在 SQL 块中指定的变量中。
3. 您还可以将任何现有的 [SQL 块](https://docs.deepnote.com/features/sql-cells) 转换为 ClickHouse 块。
