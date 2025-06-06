---
'description': 'Documentation for cluster discovery in ClickHouse'
'sidebar_label': '集群发现'
'slug': '/operations/cluster-discovery'
'title': '集群发现'
---


# 集群发现

## 概述 {#overview}

ClickHouse 的集群发现功能通过允许节点自动发现并注册自己，简化了集群配置，而无需在配置文件中进行明确的定义。这在手动定义每个节点变得繁琐的情况下尤其有益。

:::note

集群发现是一个实验性功能，未来版本可能会更改或删除。
要启用此功能，请在配置文件中包含 `allow_experimental_cluster_discovery` 设置：

```xml
<clickhouse>
    <!-- ... -->
    <allow_experimental_cluster_discovery>1</allow_experimental_cluster_discovery>
    <!-- ... -->
</clickhouse>
```
:::

## 远程服务器配置 {#remote-servers-configuration}

### 传统手动配置 {#traditional-manual-configuration}

在 ClickHouse 中，传统上需要手动在配置中指定每个分片和副本：

```xml
<remote_servers>
    <cluster_name>
        <shard>
            <replica>
                <host>node1</host>
                <port>9000</port>
            </replica>
            <replica>
                <host>node2</host>
                <port>9000</port>
            </replica>
        </shard>
        <shard>
            <replica>
                <host>node3</host>
                <port>9000</port>
            </replica>
            <replica>
                <host>node4</host>
                <port>9000</port>
            </replica>
        </shard>
    </cluster_name>
</remote_servers>

```

### 使用集群发现 {#using-cluster-discovery}

使用集群发现，而不是明确地定义每个节点，您只需在 ZooKeeper 中指定一个路径。在该路径下注册的所有节点将会被自动发现并添加到集群中。

```xml
<remote_servers>
    <cluster_name>
        <discovery>
            <path>/clickhouse/discovery/cluster_name</path>

            <!-- # Optional configuration parameters: -->

            <!-- ## Authentication credentials to access all other nodes in cluster: -->
            <!-- <user>user1</user> -->
            <!-- <password>pass123</password> -->
            <!-- ### Alternatively to password, interserver secret may be used: -->
            <!-- <secret>secret123</secret> -->

            <!-- ## Shard for current node (see below): -->
            <!-- <shard>1</shard> -->

            <!-- ## Observer mode (see below): -->
            <!-- <observer/> -->
        </discovery>
    </cluster_name>
</remote_servers>
```

如果您想为特定节点指定一个分片编号，可以在 `<discovery>` 部分中包含 `<shard>` 标签：

对于 `node1` 和 `node2`：

```xml
<discovery>
    <path>/clickhouse/discovery/cluster_name</path>
    <shard>1</shard>
</discovery>
```

对于 `node3` 和 `node4`：

```xml
<discovery>
    <path>/clickhouse/discovery/cluster_name</path>
    <shard>2</shard>
</discovery>
```

### 观察者模式 {#observer-mode}

配置为观察者模式的节点不会将自己注册为副本。
它们仅会观察并发现集群中其他活动的副本，而不会主动参与。
要启用观察者模式，请在 `<discovery>` 部分中包含 `<observer/>` 标签：

```xml
<discovery>
    <path>/clickhouse/discovery/cluster_name</path>
    <observer/>
</discovery>
```

### 集群发现 {#discovery-of-clusters}

有时，您可能需要添加和移除的不仅仅是集群中的主机，而是集群本身。您可以使用 `<multicluster_root_path>` 节点来为多个集群指定根路径：

```xml
<remote_servers>
    <some_unused_name>
        <discovery>
            <multicluster_root_path>/clickhouse/discovery</multicluster_root_path>
            <observer/>
        </discovery>
    </some_unused_name>
</remote_servers>
```

在这种情况下，当其他主机使用路径 `/clickhouse/discovery/some_new_cluster` 注册时，将添加一个名为 `some_new_cluster` 的集群。

您可以同时使用这两种功能，主机可以在集群 `my_cluster` 中注册自己，同时发现其他集群：

```xml
<remote_servers>
    <my_cluster>
        <discovery>
            <path>/clickhouse/discovery/my_cluster</path>
        </discovery>
    </my_cluster>
    <some_unused_name>
        <discovery>
            <multicluster_root_path>/clickhouse/discovery</multicluster_root_path>
            <observer/>
        </discovery>
    </some_unused_name>
</remote_servers>
```

限制：
- 您不能在同一个 `remote_servers` 子树中同时使用 `<path>` 和 `<multicluster_root_path>`。
- `<multicluster_root_path>` 只能与 `<observer/>` 一起使用。
- Keeper 中路径的最后部分被用作集群名称，而在注册时名称是从 XML 标签中获取的。

## 用例和限制 {#use-cases-and-limitations}

当节点从指定的 ZooKeeper 路径添加或移除时，它们会自动被发现或从集群中移除，而无需进行配置更改或服务器重启。

然而，变化仅影响集群配置，不影响数据或现有的数据库和表。

考虑以下三个节点的集群示例：

```xml
<remote_servers>
    <default>
        <discovery>
            <path>/clickhouse/discovery/default_cluster</path>
        </discovery>
    </default>
</remote_servers>
```

```sql
SELECT * EXCEPT (default_database, errors_count, slowdowns_count, estimated_recovery_time, database_shard_name, database_replica_name)
FROM system.clusters WHERE cluster = 'default';

┌─cluster─┬─shard_num─┬─shard_weight─┬─replica_num─┬─host_name────┬─host_address─┬─port─┬─is_local─┬─user─┬─is_active─┐
│ default │         1 │            1 │           1 │ 92d3c04025e8 │ 172.26.0.5   │ 9000 │        0 │      │      ᴺᵁᴸᴸ │
│ default │         1 │            1 │           2 │ a6a68731c21b │ 172.26.0.4   │ 9000 │        1 │      │      ᴺᵁᴸᴸ │
│ default │         1 │            1 │           3 │ 8e62b9cb17a1 │ 172.26.0.2   │ 9000 │        0 │      │      ᴺᵁᴸᴸ │
└─────────┴───────────┴──────────────┴─────────────┴──────────────┴──────────────┴──────┴──────────┴──────┴───────────┘
```

```sql
CREATE TABLE event_table ON CLUSTER default (event_time DateTime, value String)
ENGINE = ReplicatedMergeTree('/clickhouse/tables/event_table', '{replica}')
ORDER BY event_time PARTITION BY toYYYYMM(event_time);

INSERT INTO event_table ...
```

然后，我们通过在配置文件的 `remote_servers` 部分中使用相同的条目，向集群添加一个新节点，启动一个新节点：

```response
┌─cluster─┬─shard_num─┬─shard_weight─┬─replica_num─┬─host_name────┬─host_address─┬─port─┬─is_local─┬─user─┬─is_active─┐
│ default │         1 │            1 │           1 │ 92d3c04025e8 │ 172.26.0.5   │ 9000 │        0 │      │      ᴺᵁᴸᴸ │
│ default │         1 │            1 │           2 │ a6a68731c21b │ 172.26.0.4   │ 9000 │        1 │      │      ᴺᵁᴸᴸ │
│ default │         1 │            1 │           3 │ 8e62b9cb17a1 │ 172.26.0.2   │ 9000 │        0 │      │      ᴺᵁᴸᴸ │
│ default │         1 │            1 │           4 │ b0df3669b81f │ 172.26.0.6   │ 9000 │        0 │      │      ᴺᵁᴸᴸ │
└─────────┴───────────┴──────────────┴─────────────┴──────────────┴──────────────┴──────┴──────────┴──────┴───────────┘
```

第四个节点参与集群，但表 `event_table` 仍然只存在于前三个节点上：

```sql
SELECT hostname(), database, table FROM clusterAllReplicas(default, system.tables) WHERE table = 'event_table' FORMAT PrettyCompactMonoBlock

┌─hostname()───┬─database─┬─table───────┐
│ a6a68731c21b │ default  │ event_table │
│ 92d3c04025e8 │ default  │ event_table │
│ 8e62b9cb17a1 │ default  │ event_table │
└──────────────┴──────────┴─────────────┘
```

如果您需要在所有节点上复制表，可以使用 [Replicated](../engines/database-engines/replicated.md) 数据库引擎作为集群发现的替代方案。
