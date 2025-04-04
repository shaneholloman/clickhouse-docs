---
description: '包含已配置设置配置文件的属性的系统表。'
slug: /operations/system-tables/settings_profiles
title: 'system.settings_profiles'
keywords: ['system table', 'settings_profiles']
---

包含已配置设置配置文件的属性。

列：
- `name` ([String](../../sql-reference/data-types/string.md)) — 设置配置文件名称。

- `id` ([UUID](../../sql-reference/data-types/uuid.md)) — 设置配置文件ID。

- `storage` ([String](../../sql-reference/data-types/string.md)) — 设置配置文件存储的路径。在 `access_control_path` 参数中配置。

- `num_elements` ([UInt64](../../sql-reference/data-types/int-uint.md)) — 此配置文件在 `system.settings_profile_elements` 表中的元素数量。

- `apply_to_all` ([UInt8](/sql-reference/data-types/int-uint#integer-ranges)) — 表示此设置配置文件适用于所有角色和/或用户。

- `apply_to_list` ([Array](../../sql-reference/data-types/array.md)([String](../../sql-reference/data-types/string.md))) — 应用此设置配置文件的角色和/或用户列表。

- `apply_to_except` ([Array](../../sql-reference/data-types/array.md)([String](../../sql-reference/data-types/string.md))) — 此设置配置文件适用于所有角色和/或用户，但不包括列举的角色和/或用户。

## 另见 {#see-also}

- [SHOW PROFILES](/sql-reference/statements/show#show-profiles)
