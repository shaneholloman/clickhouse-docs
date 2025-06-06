---
'slug': '/use-cases/observability/clickstack/alerts'
'title': '使用 ClickStack 搜索'
'sidebar_label': '警报'
'pagination_prev': null
'pagination_next': null
'description': '使用 ClickStack 的警报'
---

import Image from '@theme/IdealImage';
import search_alert from '@site/static/images/use-cases/observability/search_alert.png';

## 搜索警报 {#search-alerts}

在输入一个 [搜索](/use-cases/observability/clickstack/search) 后，您可以创建一个警报，当匹配搜索的事件（日志或跨度）数量超过或低于某个阈值时通知您。

### 创建警报 {#creating-an-alert}

您可以通过点击 `Search` 页面右上角的 `Alerts` 按钮来创建警报。

在此，您可以为警报命名，并设置阈值、持续时间和通知方法（Slack、电子邮件、PagerDuty 或 Slack webhook）。

`grouped by` 值允许搜索处于聚合状态，例如 `ServiceName`，从而允许同一搜索触发多个潜在警报。

<Image img={search_alert} alt="搜索警报" size="lg"/>

### 常见警报场景 {#common-alert-scenarios}

以下是一些您可以使用 HyperDX 的常见警报场景：

**错误:** 我们首先建议为默认的 `All Error Events` 和 `HTTP Status >= 400` 保存的搜索设置警报，以便在发生过多错误时通知您。

**慢操作:** 您可以设置一个搜索以查找慢操作（例如 `duration:>5000`），并在发生过多慢操作时发出警报。

**用户事件:** 您还可以设置警报，以便在新用户注册或执行关键用户操作时通知面向客户的团队。
