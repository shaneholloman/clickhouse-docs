---
slug: /guides/sizing-and-hardware-recommendations
sidebar_label: '硬件规模与推荐'
sidebar_position: 4
---

# 硬件规模与推荐

本指南讨论我们对开源用户的硬件、计算、内存和磁盘配置的一般建议。如果您想简化设置，我们建议使用 [ClickHouse Cloud](https://clickhouse.com/cloud)，因为它能够自动扩展并适应您的工作负载，同时最小化与基础设施管理相关的成本。

您的 ClickHouse 集群的配置高度依赖于您应用程序的用例和工作负载模式。在规划架构时，您必须考虑以下因素：

- 并发性（每秒请求数）
- 吞吐量（每秒处理的行数）
- 数据量
- 数据保留策略
- 硬件成本
- 维护成本

## 磁盘 {#disk}

您应该使用的磁盘类型取决于数据量、延迟或吞吐量的要求。

### 针对性能的优化 {#optimizing-for-performance}

为了最大化性能，我们建议直接连接 [AWS 的预置 IOPS SSD 卷](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/provisioned-iops.html) 或您云提供商的等效产品，这些产品优化了 IO。

### 针对存储成本的优化 {#optimizing-for-storage-costs}

为了降低成本，您可以使用 [通用 SSD EBS 卷](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/general-purpose.html)。

您还可以在 [热/温/冷架构](/guides/developer/ttl#implementing-a-hotwarmcold-architecture) 中实现使用 SSD 和 HDD 的分层存储。或者，您还可以使用 [AWS S3](https://aws.amazon.com/s3/) 进行存储，以实现计算和存储的分离。有关如何使用开源 ClickHouse 进行计算和存储分离的指南，请参见 [这里](/guides/separation-storage-compute)。 ClickHouse Cloud 中默认支持计算和存储分离。

## CPU {#cpu}

### 我应该使用哪种 CPU？ {#which-cpu-should-i-use}

您应该使用的 CPU 类型取决于您的使用模式。一般来说，然而，频繁的并发查询、处理更多数据或使用计算密集型 UDF 的应用程序将需要更多的 CPU 核心。

**低延迟或面向客户的应用**

对于延迟要求为 10 毫秒的场景，例如面向客户的工作负载，我们建议使用 AWS 的 EC2 [i3 系列](https://aws.amazon.com/ec2/instance-types/i3/) 或 [i4i 系列](https://aws.amazon.com/ec2/instance-types/i4i/) ，或云提供商的等效产品，它们经过 IO 优化。

**高并发应用**

对于需要优化并发（每秒 100 个以上查询）的工作负载，我们建议使用 AWS 的 [计算优化 C 系列](https://aws.amazon.com/ec2/instance-types/#Compute_Optimized) 或云提供商的等效产品。

**数据仓库用例**

对于数据仓库工作负载和临时分析查询，我们建议使用 AWS 的 [R 型系列](https://aws.amazon.com/ec2/instance-types/#Memory_Optimized) 或云提供商的等效产品，因为它们经过内存优化。

### CPU 利用率应是多少？ {#what-should-cpu-utilization-be}

ClickHouse 没有标准的 CPU 利用率目标。使用工具如 [iostat](https://linux.die.net/man/1/iostat) 来测量平均 CPU 使用率，并相应地调整服务器的规模以管理意外的流量峰值。然而，对于临时查询的分析或数据仓库用例，您应该目标为 10-20% 的 CPU 利用率。

### 我应该使用多少个 CPU 核心？ {#how-many-cpu-cores-should-i-use}

您应该使用的 CPU 数量取决于您的工作负载。然而，我们一般建议根据 CPU 类型使用以下内存与 CPU 核心的比例：

- **[M 型](https://aws.amazon.com/ec2/instance-types/)（通用使用案例）：** 4:1 的内存与 CPU 核心比例
- **[R 型](https://aws.amazon.com/ec2/instance-types/#Memory_Optimized)（数据仓库使用案例）：** 8:1 的内存与 CPU 核心比例
- **[C 型](https://aws.amazon.com/ec2/instance-types/#Compute_Optimized)（计算优化使用案例）：** 2:1 的内存与 CPU 核心比例

例如，当使用 M 型 CPU 时，我们建议每 25 个 CPU 核心配置 100GB 的内存。要确定适合您应用程序的内存量，需要对内存使用情况进行分析。您可以阅读 [关于调试内存问题的指南](/guides/developer/debugging-memory-issues) 或使用 [内置可观察性仪表板](/operations/monitoring) 来监控 ClickHouse。

## 内存 {#memory}

与选择 CPU 一样，您选择的内存与存储比以及内存与 CPU 比例也依赖于您的案例。然而，一般来说，内存越多，查询运行得越快。如果您的用例对价格敏感，使用较少的内存也可以，因为可以启用设置（[max_bytes_before_external_group_by](/operations/settings/query-complexity#settings-max_bytes_before_external_group_by) 和 [max_bytes_before_external_sort](/operations/settings/query-complexity#settings-max_bytes_before_external_sort)）来允许将数据溢出到磁盘，但请注意，这可能会显著影响查询性能。

### 内存与存储的比例应是多少？ {#what-should-the-memory-to-storage-ratio-be}

对于较低数据量，1:1 的内存与存储比例是可以接受的，但总内存不应低于 8GB。

对于数据保留期较长或数据量较大的用例，我们建议内存与存储的比例为 1:100 至 1:130。例如，如果您存储 10TB 数据，则每个副本需要 100GB 的 RAM。

对于频繁访问的用例，例如面向客户的工作负载，我们建议使用更高的内存，比例为 1:30 至 1:50 的内存与存储比例。

## 副本 {#replicas}

我们建议每个分片至少有三个副本（或使用 [Amazon EBS](https://aws.amazon.com/ebs/) 的两个副本）。此外，我们建议在添加额外副本（水平扩展）之前，先对所有副本进行纵向扩展。

ClickHouse 并不会自动分片，重新分片您的数据集将需要大量计算资源。因此，我们通常建议使用可用的最大服务器，以防将来需要重新分片您的数据。

考虑使用 [ClickHouse Cloud](https://clickhouse.com/cloud)，它能够自动扩展，并允许您轻松控制适合您用例的副本数量。

## 大工作负载的示例配置 {#example-configurations-for-large-workloads}

ClickHouse 的配置高度依赖于您特定应用的需求。如果您希望我们帮助优化您的架构以降低成本和提高性能，请 [联系我们的销售团队](https://clickhouse.com/company/contact?loc=docs-sizing-and-hardware-recommendations)。

为了提供指导（不是推荐），以下是生产中使用 ClickHouse 的一些用户的示例配置：

### 财富 500 强 B2B SaaS {#fortune-500-b2b-saas}

<table>
    <tr>
        <td col="2"><strong><em>存储</em></strong></td>
    </tr>
    <tr>
        <td><strong>每月新增数据量</strong></td>
        <td>30TB</td>
    </tr>
    <tr>
        <td><strong>总存储（压缩后）</strong></td>
        <td>540TB</td>
    </tr>
    <tr>
        <td><strong>数据保留期</strong></td>
        <td>18 个月</td>
    </tr>
    <tr>
        <td><strong>每个节点的磁盘</strong></td>
        <td>25TB</td>
    </tr>
    <tr>
        <td col="2"><strong><em>CPU</em></strong></td>
    </tr>
    <tr>
        <td><strong>并发性</strong></td>
        <td>200+ 并发查询</td>
    </tr>
    <tr>
        <td><strong>副本数量（包括高可用性对）</strong></td>
        <td>44</td>
    </tr>
    <tr>
        <td><strong>每个节点的 vCPU</strong></td>
        <td>62</td>
    </tr>
    <tr>
        <td><strong>总 vCPU</strong></td>
        <td>2700</td>
    </tr>
    <tr>
        <td col="2"><strong><em>内存</em></strong></td>
    </tr>
    <tr>
        <td><strong>总 RAM</strong></td>
        <td>11TB</td>
    </tr>
    <tr>
        <td><strong>每个副本的 RAM</strong></td>
        <td>256GB</td>
    </tr>
    <tr>
        <td><strong>RAM 与 vCPU 比例</strong></td>
        <td>4:1</td>
    </tr>
    <tr>
        <td><strong>RAM 与磁盘比例</strong></td>
        <td>1:50</td>
    </tr>
</table>

### 财富 500 强电信运营商的日志使用案例 {#fortune-500-telecom-operator-for-a-logging-use-case}

<table>
    <tr>
        <td col="2"><strong><em>存储</em></strong></td>
    </tr>
    <tr>
        <td><strong>每月日志数据量</strong></td>
        <td>4860TB</td>
    </tr>
    <tr>
        <td><strong>总存储（压缩后）</strong></td>
        <td>608TB</td>
    </tr>
    <tr>
        <td><strong>数据保留期</strong></td>
        <td>30 天</td>
    </tr>
    <tr>
        <td><strong>每个节点的磁盘</strong></td>
        <td>13TB</td>
    </tr>
    <tr>
        <td col="2"><strong><em>CPU</em></strong></td>
    </tr>
    <tr>
        <td><strong>副本数量（包括高可用性对）</strong></td>
        <td>38</td>
    </tr>
    <tr>
        <td><strong>每个节点的 vCPU</strong></td>
        <td>42</td>
    </tr>
    <tr>
        <td><strong>总 vCPU</strong></td>
        <td>1600</td>
    </tr>
    <tr>
        <td col="2"><strong><em>内存</em></strong></td>
    </tr>
    <tr>
        <td><strong>总 RAM</strong></td>
        <td>10TB</td>
    </tr>
    <tr>
        <td><strong>每个副本的 RAM</strong></td>
        <td>256GB</td>
    </tr>
    <tr>
        <td><strong>RAM 与 vCPU 比例</strong></td>
        <td>6:1</td>
    </tr>
    <tr>
        <td><strong>RAM 与磁盘比例</strong></td>
        <td>1:60</td>
    </tr>
</table>

## 深入阅读 {#further-reading}

以下是一些使用开源 ClickHouse 的公司的架构相关博客文章：

- [Cloudflare](https://blog.cloudflare.com/http-analytics-for-6m-requests-per-second-using-clickhouse/?utm_source=linkedin&utm_medium=social&utm_campaign=blog)
- [eBay](https://innovation.ebayinc.com/tech/engineering/ou-online-analytical-processing/)
- [GitLab](https://handbook.gitlab.com/handbook/engineering/development/ops/monitor/observability/#clickhouse-datastore)
- [Lyft](https://eng.lyft.com/druid-deprecation-and-clickhouse-adoption-at-lyft-120af37651fd)
- [MessageBird](https://clickhouse.com/blog/how-messagebird-uses-clickhouse-to-monitor-the-delivery-of-billions-of-messages)
- [Microsoft](https://clickhouse.com/blog/self-service-data-analytics-for-microsofts-biggest-web-properties)
- [Uber](https://www.uber.com/en-ES/blog/logging/)
- [Zomato](https://blog.zomato.com/building-a-cost-effective-logging-platform-using-clickhouse-for-petabyte-scale)
