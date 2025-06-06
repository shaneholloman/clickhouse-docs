---
'title': 'AWS的BYOC (自带云服务)'
'slug': '/cloud/reference/byoc'
'sidebar_label': 'BYOC (自带云服务)'
'keywords':
- 'BYOC'
- 'cloud'
- 'bring your own cloud'
'description': '在您自己的云基础设施上部署 ClickHouse'
---

import Image from '@theme/IdealImage';
import byoc1 from '@site/static/images/cloud/reference/byoc-1.png';
import byoc4 from '@site/static/images/cloud/reference/byoc-4.png';
import byoc3 from '@site/static/images/cloud/reference/byoc-3.png';
import byoc_vpcpeering from '@site/static/images/cloud/reference/byoc-vpcpeering-1.png';
import byoc_vpcpeering2 from '@site/static/images/cloud/reference/byoc-vpcpeering-2.png';
import byoc_vpcpeering3 from '@site/static/images/cloud/reference/byoc-vpcpeering-3.png';
import byoc_vpcpeering4 from '@site/static/images/cloud/reference/byoc-vpcpeering-4.png';
import byoc_plb from '@site/static/images/cloud/reference/byoc-plb.png';
import byoc_security from '@site/static/images/cloud/reference/byoc-securitygroup.png';
import byoc_inbound from '@site/static/images/cloud/reference/byoc-inbound-rule.png';
import byoc_subnet_1 from '@site/static/images/cloud/reference/byoc-subnet-1.png';
import byoc_subnet_2 from '@site/static/images/cloud/reference/byoc-subnet-2.png';

## 概述 {#overview}

BYOC (自带云) 允许您在自己的云基础设施上部署 ClickHouse Cloud。如果您有特定的需求或约束，导致无法使用 ClickHouse Cloud 托管服务，这将非常有用。

**如果您希望获取访问权限，请 [联系我们](https://clickhouse.com/cloud/bring-your-own-cloud)。** 有关更多信息，请参阅我们的 [服务条款](https://clickhouse.com/legal/agreements/terms-of-service)。

目前，BYOC 仅支持 AWS。您可以在这里加入 GCP 和 Azure 的候补名单 [here](https://clickhouse.com/cloud/bring-your-own-cloud)。

:::note 
BYOC 专为大规模部署而设计，需要客户签署承诺合同。
:::

## 术语表 {#glossary}

- **ClickHouse VPC:**  ClickHouse Cloud 所拥有的 VPC。
- **客户 BYOC VPC:** 由客户云账户拥有并由 ClickHouse Cloud 提供和管理的 VPC，专用于 ClickHouse Cloud BYOC 部署。
- **客户 VPC:** 其他由客户云账户拥有的 VPC，用于需要连接到客户 BYOC VPC 的应用程序。

## 架构 {#architecture}

指标和日志存储在客户的 BYOC VPC 中。日志目前在本地存储于 EBS。在未来的更新中，日志将存储在 LogHouse 中，这是客户 BYOC VPC 内的 ClickHouse 服务。指标通过存储在客户 BYOC VPC 中的 Prometheus 和 Thanos 堆栈实现。

<br />

<Image img={byoc1} size="lg" alt="BYOC 架构" background='black'/>

<br />

## 上线流程 {#onboarding-process}

客户可以通过与 [我们](https://clickhouse.com/cloud/bring-your-own-cloud) 联系来启动上线流程。客户需要有一个专用的 AWS 账户，并知道他们将使用的区域。目前，我们仅允许用户在支持 ClickHouse Cloud 的区域内启动 BYOC 服务。

### 准备 AWS 账户 {#prepare-an-aws-account}

建议客户为托管 ClickHouse BYOC 部署准备一个专用的 AWS 账户，以确保更好的隔离。但是，也可以使用共享账户和现有 VPC。请参见下面的 *设置 BYOC 基础设施* 了解详细信息。

使用此账户和初始组织管理员电子邮件，您可以联系 ClickHouse 支持。

### 应用 CloudFormation 模板 {#apply-cloudformation-template}

BYOC 设置通过一个 [CloudFormation 堆栈](https://s3.us-east-2.amazonaws.com/clickhouse-public-resources.clickhouse.cloud/cf-templates/byoc.yaml) 初始化，该堆栈仅创建一个角色，允许 ClickHouse Cloud 的 BYOC 控制器管理基础设施。S3、VPC 和用于运行 ClickHouse 的计算资源不包括在此堆栈中。

<!-- TODO: 添加其余上线的截图，一旦自助上线流程实施。 -->

### 设置 BYOC 基础设施 {#setup-byoc-infrastructure}

创建 CloudFormation 堆栈后，系统会提示您设置基础设施，包括 S3、VPC 和 EKS 集群。某些配置必须在此阶段确定，因为之后无法更改。具体而言：

- **您想使用的区域**，您可以选择我们的 ClickHouse Cloud 可用的任何 [公共区域](/cloud/reference/supported-regions)。
- **BYOC 的 VPC CIDR 范围**：默认情况下，我们为 BYOC VPC CIDR 范围使用 `10.0.0.0/16`。如果您计划与另一个账户进行 VPC 对等连接，请确保 CIDR 范围不重叠。为 BYOC 分配适当的 CIDR 范围，最小大小为 `/22` 以容纳必要的工作负载。
- **BYOC VPC 的可用区**：如果您计划使用 VPC 对等连接，则在源账户和 BYOC 账户之间对齐可用区可以帮助减少跨 AZ 流量费用。在 AWS 中，可用区后缀 (`a, b, c`) 可能表示不同的物理区 ID。有关详细信息，请参见 [AWS 指南](https://docs.aws.amazon.com/prescriptive-guidance/latest/patterns/use-consistent-availability-zones-in-vpcs-across-different-aws-accounts.html)。

#### 客户管理的 VPC {#customer-managed-vpc}
默认情况下，ClickHouse Cloud 会为您的 BYOC 部署配置一个专用 VPC，以确保更好的隔离。但是，您也可以使用您账户中的现有 VPC。这需要特定配置，并且必须通过 ClickHouse 支持进行协调。

**配置您现有的 VPC**
1. 在 3 个不同可用区中分配至少 3 个私有子网以供 ClickHouse Cloud 使用。
2. 确保每个子网的 CIDR 范围至少为 `/23`（例如，10.0.0.0/23），以提供足够的 IP 地址用于 ClickHouse 部署。
3. 为每个子网添加标签 `kubernetes.io/role/internal-elb=1` 以启用适当的负载均衡器配置。

<br />

<Image img={byoc_subnet_1} size="lg" alt="BYOC VPC 子网" background='black'/>

<br />

<br />

<Image img={byoc_subnet_2} size="lg" alt="BYOC VPC 子网标签" background='black'/>

<br />

**联系 ClickHouse 支持**  
创建一个支持工单，其中包含以下信息：

* 您的 AWS 账户 ID
* 您希望部署服务的 AWS 区域
* 您的 VPC ID
* 您为 ClickHouse 分配的私有子网 ID
* 这些子网所在的可用区


### 可选：设置 VPC 对等连接 {#optional-setup-vpc-peering}

要为 ClickHouse BYOC 创建或删除 VPC 对等连接，请按以下步骤操作：

#### 第一步 启用 ClickHouse BYOC 的私有负载均衡器 {#step-1-enable-private-load-balancer-for-clickhouse-byoc}
联系 ClickHouse 支持以启用私有负载均衡器。

#### 第二步 创建对等连接 {#step-2-create-a-peering-connection}
1. 转到 ClickHouse BYOC 账户中的 VPC 仪表板。
2. 选择对等连接。
3. 点击创建对等连接。
4. 将 VPC 请求者设置为 ClickHouse VPC ID。
5. 将 VPC 接受者设置为目标 VPC ID。（如有适用，选择另一个账户）
6. 点击创建对等连接。

<br />

<Image img={byoc_vpcpeering} size="lg" alt="BYOC 创建对等连接" border />

<br />

#### 第三步 接受对等连接请求 {#step-3-accept-the-peering-connection-request}
转到对等账户，在(VPC -> 对等连接 -> 操作 -> 接受请求)页面，客户可以批准此 VPC 对等连接请求。

<br />

<Image img={byoc_vpcpeering2} size="lg" alt="BYOC 接受对等连接" border />

<br />

#### 第四步 将目标添加到 ClickHouse VPC 路由表 {#step-4-add-destination-to-clickhouse-vpc-route-tables}
在 ClickHouse BYOC 账户中，
1. 在 VPC 仪表板中选择路由表。
2. 搜索 ClickHouse VPC ID。编辑附加到私有子网的每个路由表。
3. 点击路由选项卡下的编辑按钮。
4. 点击添加另一条路由。
5. 输入目标 VPC 的 CIDR 范围作为目的地。
6. 选择“对等连接”和对等连接的 ID 作为目标。

<br />

<Image img={byoc_vpcpeering3} size="lg" alt="BYOC 添加路由表" border />

<br />

#### 第五步 将目标添加到目标 VPC 路由表 {#step-5-add-destination-to-the-target-vpc-route-tables}
在对等 AWS 账户中，
1. 在 VPC 仪表板中选择路由表。
2. 搜索目标 VPC ID。
3. 点击路由选项卡下的编辑按钮。
4. 点击添加另一条路由。
5. 输入 ClickHouse VPC 的 CIDR 范围作为目的地。
6. 选择“对等连接”和对等连接的 ID 作为目标。

<br />

<Image img={byoc_vpcpeering4} size="lg" alt="BYOC 添加路由表" border />

<br />

#### 第六步 编辑安全组以允许对等 VPC 访问 {#step-6-edit-security-group-to-allow-peered-vpc-access}
在 ClickHouse BYOC 账户中，您需要更新安全组设置以允许来自对等 VPC 的流量。请联系 ClickHouse 支持以请求添加入站规则，包括您对等 VPC 的 CIDR 范围。

---
现在 ClickHouse 服务应该可以从对等 VPC 访问。

要私密访问 ClickHouse，将为用户的对等 VPC 提供私有负载均衡器和终端。私有终端遵循公共终端格式，带有 `-private` 后缀。例如：
- **公共终端**: `h5ju65kv87.mhp0y4dmph.us-west-2.aws.byoc.clickhouse.cloud`
- **私有终端**: `h5ju65kv87-private.mhp0y4dmph.us-west-2.aws.byoc.clickhouse.cloud`

可选地，在验证对等连接正常工作后，您可以请求移除 ClickHouse BYOC 的公共负载均衡器。

## 升级流程 {#upgrade-process}

我们定期升级软件，包括 ClickHouse 数据库版本升级、ClickHouse Operator、EKS 和其他组件。

虽然我们的目标是实现无缝升级（例如，滚动升级和重启），但某些升级，如 ClickHouse 版本变更和 EKS 节点升级，可能会影响服务。客户可以指定维护窗口（例如，每周二凌晨 1:00 PDT），确保此类升级仅在计划时间内进行。

:::note
维护窗口不适用于安全和漏洞修复。这些处理为非周期性升级，并及时沟通以协调合适的时间，最大限度地减少操作影响。
:::

## CloudFormation IAM 角色 {#cloudformation-iam-roles}

### 引导 IAM 角色 {#bootstrap-iam-role}

引导 IAM 角色拥有以下权限：

- **EC2 和 VPC 操作**：用于设置 VPC 和 EKS 集群。
- **S3 操作（例如，`s3:CreateBucket`）**：需要创建 ClickHouse BYOC 存储桶。
- **`route53:*` 权限**：用于外部 DNS 在 Route 53 中配置记录。
- **IAM 操作（例如，`iam:CreatePolicy`）**：需要控制器创建额外角色（详见下一节）。
- **EKS 操作**：限于名称以 `clickhouse-cloud` 前缀开头的资源。

### 由控制器创建的其他 IAM 角色 {#additional-iam-roles-created-by-the-controller}

除了通过 CloudFormation 创建的 `ClickHouseManagementRole` 之外，控制器还将创建几个其他角色。

这些角色由在客户 EKS 集群内运行的应用程序承担：
- **状态导出器角色**
  - ClickHouse 组件，向 ClickHouse Cloud 报告服务健康信息。
  - 需要写入 ClickHouse Cloud 拥有的 SQS 队列的权限。
- **负载均衡器控制器**
  - 标准 AWS 负载均衡器控制器。
  - EBS CSI 控制器，用于管理 ClickHouse 服务的卷。
- **外部 DNS**
  - 将 DNS 配置传播到 Route 53。
- **证书管理器**
  - 为 BYOC 服务域提供 TLS 证书。
- **集群自动缩放器**
  - 根据需要调整节点组大小。

**K8s-control-plane** 和 **k8s-worker** 角色旨在由 AWS EKS 服务承担。

最后，**`data-plane-mgmt`** 允许 ClickHouse Cloud 控制平面组件调和必要的自定义资源，如 `ClickHouseCluster` 和 Istio 虚拟服务/网关。

## 网络边界 {#network-boundaries}

本节涵盖客户端 BYOC VPC 的不同网络流量：

- **入站**：进入客户 BYOC VPC 的流量。
- **出站**：来自客户 BYOC VPC 并发送到外部目标的流量。
- **公共**：可以从公共互联网访问的网络端点。
- **私有**：只能通过私有连接访问的网络端点，例如 VPC 对等连接、VPC 私有链接或 Tailscale。

**Istio 入口在 AWS NLB 后面部署，以接受 ClickHouse 客户端流量。**

*入站，公有（可以是私有）*

Istio 入口网关终止 TLS。证书由 CertManager 使用 Let's Encrypt 提供，存储在 EKS 集群中的秘密中。由于它们位于同一 VPC 中，Istio 和 ClickHouse 之间的流量经过 [AWS 加密](https://docs.aws.amazon.com/whitepapers/latest/logical-separation/encrypting-data-at-rest-and--in-transit.html#:~:text=All%20network%20traffic%20between%20AWS,supported%20Amazon%20EC2%20instance%20types)。

默认情况下，入口是公开可访问的，并具有 IP 允许名单过滤。客户可以配置 VPC 对等连接以使其变为私有并禁用公共连接。我们强烈建议设置 [IP 过滤器](/cloud/security/setting-ip-filters) 以限制访问。

### 故障排除访问 {#troubleshooting-access}

*入站，公有（可以是私有）*

ClickHouse Cloud 工程师需要通过 Tailscale 进行故障排除访问。它们通过及时的基于证书的身份验证为 BYOC 部署提供服务。

### 计费抓取器 {#billing-scraper}

*出站，私有*

计费抓取器从 ClickHouse 收集计费数据，并将其发送到 ClickHouse Cloud 拥有的 S3 存储桶。

它作为 ClickHouse 服务器容器的附加容器运行，周期性地抓取 CPU 和内存指标。在同一区域内的请求通过 VPC 网关服务端点路由。

### 警报 {#alerts}

*出站，公有*

AlertManager 被配置为在客户的 ClickHouse 集群处于不健康状态时向 ClickHouse Cloud 发送警报。

指标和日志存储在客户的 BYOC VPC 中。日志目前在本地存储于 EBS。在未来的更新中，它们将存储在 LogHouse 中，这是 BYOC VPC 内的 ClickHouse 服务。指标使用存储在 BYOC VPC 中的 Prometheus 和 Thanos 堆栈。

### 服务状态 {#service-state}

*出站*

状态导出器将 ClickHouse 服务状态信息发送到 ClickHouse Cloud 拥有的 SQS。

## 特性 {#features}

### 支持的特性 {#supported-features}

- **SharedMergeTree**: ClickHouse Cloud 和 BYOC 使用相同的二进制和配置。因此，ClickHouse 核心的所有功能在 BYOC 中都得到支持，例如 SharedMergeTree。
- **管理服务状态的控制台访问**：
  - 支持启动、停止和终止等操作。
  - 查看服务和状态。
- **备份和恢复。**
- **手动垂直和水平缩放。**
- **空闲。**
- **仓库**：计算-计算分离
- **通过 Tailscale 的零信任网络。**
- **监控**：
  - Cloud 控制台包含用于监控服务健康的内置健康仪表板。
  - Prometheus 抓取，以便与 Prometheus、Grafana 和 Datadog 进行集中监控。有关设置说明，请参见 [Prometheus 文档](/integrations/prometheus)。
- **VPC 对等连接。**
- **集成**：完整列表请参见 [此页面](/integrations)。
- **安全的 S3。**
- **[AWS PrivateLink](https://aws.amazon.com/privatelink/)。**

### 计划中的特性（目前不支持） {#planned-features-currently-unsupported}

- [AWS KMS](https://aws.amazon.com/kms/) 即 CMEK（客户管理加密密钥）
- ClickPipes 用于摄取
- 自动缩放
- MySQL 接口

## 常见问题解答 {#faq}

### 计算 {#compute}

#### 我可以在这个单一 EKS 集群中创建多个服务吗？ {#can-i-create-multiple-services-in-this-single-eks-cluster}

可以。基础设施只需为每个 AWS 账户和区域组合配置一次。

### 您支持哪些区域的 BYOC？ {#which-regions-do-you-support-for-byoc}

BYOC 支持与 ClickHouse Cloud 相同的一组 [区域](/cloud/reference/supported-regions#aws-regions)。

#### 会有一些资源开销吗？运行 ClickHouse 实例之外的服务需要哪些资源？ {#will-there-be-some-resource-overhead-what-are-the-resources-needed-to-run-services-other-than-clickhouse-instances}

除了 ClickHouse 实例（ClickHouse 服务器和 ClickHouse Keeper）外，我们还运行 `clickhouse-operator`、`aws-cluster-autoscaler`、Istio 等服务，以及我们的监控堆栈。

目前我们在专用节点组中有 3 个 m5.xlarge 节点（每个 AZ 一个）以运行这些工作负载。

### 网络和安全 {#network-and-security}

#### 在安装后可以撤销设置期间设置的权限吗？ {#can-we-revoke-permissions-set-up-during-installation-after-setup-is-complete}

目前无法实现。

#### 您是否考虑过未来的安全控制，以便 ClickHouse 工程师访问客户基础设施进行故障排除？ {#have-you-considered-some-future-security-controls-for-clickhouse-engineers-to-access-customer-infra-for-troubleshooting}

是的。实施客户控制机制，允许客户批准工程师访问集群在我们的路线图上。当时，工程师必须通过我们的内部升级流程，以获取对集群的及时访问。这由我们的安全团队记录和审核。

#### 创建的 VPC IP 范围的大小是多少？ {#what-is-the-size-of-the-vpc-ip-range-created}

默认情况下，我们为 BYOC VPC 使用 `10.0.0.0/16`。我们建议保留至少 /22 供未来潜在扩展，但如果您希望限制大小，如果预计将限制到 30 个服务器 pod，可以使用 /23。

#### 我可以决定维护频率吗？ {#can-i-decide-maintenance-frequency}

联系支持以安排维护窗口。请预计每周至少一次的更新计划。

## 可观测性 {#observability}

### 内置监控工具 {#built-in-monitoring-tools}

#### 可观测性仪表板 {#observability-dashboard}

ClickHouse Cloud 包含一个高级可观测性仪表板，显示内存使用情况、查询率和 I/O 等指标。可以在 ClickHouse Cloud 网络控制台界面的 **监控** 部分访问。

<br />

<Image img={byoc3} size="lg" alt="可观测性仪表板" border />

<br />

#### 高级仪表板 {#advanced-dashboard}

您可以使用 `system.metrics`、`system.events` 和 `system.asynchronous_metrics` 等系统表中的指标自定义仪表板，以详细监控服务器性能和资源利用。

<br />

<Image img={byoc4} size="lg" alt="高级仪表板" border />

<br />

#### Prometheus 集成 {#prometheus-integration}

ClickHouse Cloud 提供了一个 Prometheus 端点，您可以用来抓取监控指标。这允许与 Grafana 和 Datadog 等工具进行可视化集成。

**通过 https 端点 /metrics_all 的示例请求**

```bash
curl --user <username>:<password> https://i6ro4qarho.mhp0y4dmph.us-west-2.aws.byoc.clickhouse.cloud:8443/metrics_all
```

**示例响应**

```bash

# HELP ClickHouse_CustomMetric_StorageSystemTablesS3DiskBytes The amount of bytes stored on disk `s3disk` in system database

# TYPE ClickHouse_CustomMetric_StorageSystemTablesS3DiskBytes gauge
ClickHouse_CustomMetric_StorageSystemTablesS3DiskBytes{hostname="c-jet-ax-16-server-43d5baj-0"} 62660929

# HELP ClickHouse_CustomMetric_NumberOfBrokenDetachedParts The number of broken detached parts

# TYPE ClickHouse_CustomMetric_NumberOfBrokenDetachedParts gauge
ClickHouse_CustomMetric_NumberOfBrokenDetachedParts{hostname="c-jet-ax-16-server-43d5baj-0"} 0

# HELP ClickHouse_CustomMetric_LostPartCount The age of the oldest mutation (in seconds)

# TYPE ClickHouse_CustomMetric_LostPartCount gauge
ClickHouse_CustomMetric_LostPartCount{hostname="c-jet-ax-16-server-43d5baj-0"} 0

# HELP ClickHouse_CustomMetric_NumberOfWarnings The number of warnings issued by the server. It usually indicates about possible misconfiguration

# TYPE ClickHouse_CustomMetric_NumberOfWarnings gauge
ClickHouse_CustomMetric_NumberOfWarnings{hostname="c-jet-ax-16-server-43d5baj-0"} 2

# HELP ClickHouseErrorMetric_FILE_DOESNT_EXIST FILE_DOESNT_EXIST

# TYPE ClickHouseErrorMetric_FILE_DOESNT_EXIST counter
ClickHouseErrorMetric_FILE_DOESNT_EXIST{hostname="c-jet-ax-16-server-43d5baj-0",table="system.errors"} 1

# HELP ClickHouseErrorMetric_UNKNOWN_ACCESS_TYPE UNKNOWN_ACCESS_TYPE

# TYPE ClickHouseErrorMetric_UNKNOWN_ACCESS_TYPE counter
ClickHouseErrorMetric_UNKNOWN_ACCESS_TYPE{hostname="c-jet-ax-16-server-43d5baj-0",table="system.errors"} 8

# HELP ClickHouse_CustomMetric_TotalNumberOfErrors The total number of errors on server since the last restart

# TYPE ClickHouse_CustomMetric_TotalNumberOfErrors gauge
ClickHouse_CustomMetric_TotalNumberOfErrors{hostname="c-jet-ax-16-server-43d5baj-0"} 9
```

**身份验证**

可以使用 ClickHouse 用户名和密码对进行身份验证。我们建议创建一个具有最小权限的专用用户来抓取监控指标。至少，`system.custom_metrics` 表上的 `READ` 权限是必须的，在副本之间。例如：

```sql
GRANT REMOTE ON *.* TO scraping_user
GRANT SELECT ON system.custom_metrics TO scraping_user
```

**配置 Prometheus**

下面显示了一个示例配置。 `targets` 端点与访问 ClickHouse 服务使用的相同。

```bash
global:
 scrape_interval: 15s

scrape_configs:
 - job_name: "prometheus"
   static_configs:
   - targets: ["localhost:9090"]
 - job_name: "clickhouse"
   static_configs:
     - targets: ["<subdomain1>.<subdomain2>.aws.byoc.clickhouse.cloud:8443"]
   scheme: https
   metrics_path: "/metrics_all"
   basic_auth:
     username: <KEY_ID>
     password: <KEY_SECRET>
   honor_labels: true
```

有关更多详细信息，请参见 [这篇博客文章](https://clickhouse.com/blog/clickhouse-cloud-now-supports-prometheus-monitoring) 和 [ClickHouse 的 Prometheus 设置文档](/integrations/prometheus)。
