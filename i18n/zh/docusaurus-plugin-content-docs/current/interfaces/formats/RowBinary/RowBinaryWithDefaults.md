---
'alias': []
'description': 'RowBinaryWithDefaults 格式的 Documentation'
'input_format': true
'keywords':
- 'RowBinaryWithDefaults'
'output_format': false
'slug': '/interfaces/formats/RowBinaryWithDefaults'
'title': 'RowBinaryWithDefaults'
---

import RowBinaryFormatSettings from './_snippets/common-row-binary-format-settings.md'

| 输入 | 输出 | 别名 |
|------|------|------|
| ✔    | ✗    |      |

## 描述 {#description}

类似于 [`RowBinary`](./RowBinary.md) 格式，但在每个列之前增加了一个额外的字节，用于指示是否应使用默认值。

## 示例用法 {#example-usage}

示例：

```sql title="Query"
SELECT * FROM FORMAT('RowBinaryWithDefaults', 'x UInt32 default 42, y UInt32', x'010001000000')
```
```response title="Response"
┌──x─┬─y─┐
│ 42 │ 1 │
└────┴───┘
```

- 对于列 `x`，只有一个字节 `01` 表示应使用默认值，并且该字节之后没有提供其他数据。
- 对于列 `y`，数据以字节 `00` 开头，表示该列有实际值，应从后续数据 `01000000` 中读取。

## 格式设置 {#format-settings}

<RowBinaryFormatSettings/>
