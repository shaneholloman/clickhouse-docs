---
slug: /managing-data/update_mutations
sidebar_label: 'Update mutations'
title: 'Update mutations'
hide_title: false
description: 'Page describing update mutations - ALTER queries that manipulate table data through updates'
---

Update mutations refers to `ALTER` queries that manipulate table data through updates. Most notably they are queries like `ALTER TABLE UPDATE`, etc. Performing such queries will produce new mutated versions of the data parts. This means that such statements would trigger a rewrite of whole data parts for all data that was inserted before the mutation, translating to a large amount of write requests.

:::info
For updates, you can avoid these large amounts of write requests by using specialised table engines like [ReplacingMergeTree](/guides/replacing-merge-tree) or [CollapsingMergeTree](/engines/table-engines/mergetree-family/collapsingmergetree) instead of the default MergeTree table engine.
:::

import UpdateMutations from '@site/docs/sql-reference/statements/alter/update.md';

<UpdateMutations/>
