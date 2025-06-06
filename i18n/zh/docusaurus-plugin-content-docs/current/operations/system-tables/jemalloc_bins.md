---
'description': '系统表包含通过 jemalloc 分配器在不同大小类（bins）中进行的内存分配的信息，这些信息是从所有竞技场汇总而来的。'
'keywords':
- 'system table'
- 'jemalloc_bins'
'slug': '/operations/system-tables/jemalloc_bins'
'title': 'system.jemalloc_bins'
---

import SystemTableCloud from '@site/i18n/zh/docusaurus-plugin-content-docs/current/_snippets/_system_table_cloud.md';

<SystemTableCloud/>

包含通过 jemalloc 分配器在不同大小类别（箱）中进行的内存分配信息，这些信息是从所有领域中聚合而来的。
由于 jemalloc 中的线程本地缓存，这些统计信息可能并不绝对准确。

列：

- `index` (UInt64) — 按大小排序的箱的索引
- `large` (Bool) — 对于大规模分配为真，对于小规模分配为假
- `size` (UInt64) — 此箱中分配的大小
- `allocations` (UInt64) — 分配的数量
- `deallocations` (UInt64) — 释放的数量

**示例**

查找对当前整体内存使用贡献最大分配的大小。

```sql
SELECT
    *,
    allocations - deallocations AS active_allocations,
    size * active_allocations AS allocated_bytes
FROM system.jemalloc_bins
WHERE allocated_bytes > 0
ORDER BY allocated_bytes DESC
LIMIT 10
```

```text
┌─index─┬─large─┬─────size─┬─allocactions─┬─deallocations─┬─active_allocations─┬─allocated_bytes─┐
│    82 │     1 │ 50331648 │            1 │             0 │                  1 │        50331648 │
│    10 │     0 │      192 │       512336 │        370710 │             141626 │        27192192 │
│    69 │     1 │  5242880 │            6 │             2 │                  4 │        20971520 │
│     3 │     0 │       48 │     16938224 │      16559484 │             378740 │        18179520 │
│    28 │     0 │     4096 │       122924 │        119142 │               3782 │        15491072 │
│    61 │     1 │  1310720 │        44569 │         44558 │                 11 │        14417920 │
│    39 │     1 │    28672 │         1285 │           913 │                372 │        10665984 │
│     4 │     0 │       64 │      2837225 │       2680568 │             156657 │        10026048 │
│     6 │     0 │       96 │      2617803 │       2531435 │              86368 │         8291328 │
│    36 │     1 │    16384 │        22431 │         21970 │                461 │         7553024 │
└───────┴───────┴──────────┴──────────────┴───────────────┴────────────────────┴─────────────────┘
```
