---
title: RowBinary
slug: /interfaces/formats/RowBinary
keywords: [RowBinary]
input_format: true
output_format: true
alias: []
---

import RowBinaryFormatSettings from './_snippets/common-row-binary-format-settings.md'

| 入力 | 出力 | エイリアス |
|-------|--------|-------|
| ✔     | ✔      |       |

## 説明 {#description}

`RowBinary`フォーマットは、データをバイナリフォーマットで行単位で解析します。  
行と値は連続してリストされ、区切り文字はありません。  
データがバイナリフォーマットであるため、`FORMAT RowBinary`の後の区切り文字は次のように厳密に指定されています：

- 任意の数の空白：
  - `' '`（スペース - コード `0x20`）
  - `'\t'`（タブ - コード `0x09`）
  - `'\f'`（フォームフィード - コード `0x0C`） 
- 正確に1つの改行シーケンスが続く：
  - Windowsスタイル `"\r\n"` 
  - またはUnixスタイル `'\n'`
- すぐにバイナリデータが続きます。

:::note
このフォーマットは行ベースであるため、[Native](../Native.md)フォーマットよりも効率が低くなります。
:::

次のデータ型においては、次の点に注意することが重要です：

- [整数](../../../sql-reference/data-types/int-uint.md)は固定長のリトルエンディアン表現を使用します。例えば、`UInt64`は8バイトを使用します。
- [DateTime](../../../sql-reference/data-types/datetime.md)は、Unixタイムスタンプを値として含む`UInt32`として表現されます。
- [Date](../../../sql-reference/data-types/date.md)は、`1970-01-01`以降の日数を値として含むUInt16オブジェクトとして表現されます。
- [String](../../../sql-reference/data-types/string.md)は、可変幅整数（varint）（符号なしの[`LEB128`](https://en.wikipedia.org/wiki/LEB128)）として表現され、その後に文字列のバイトが続きます。
- [FixedString](../../../sql-reference/data-types/fixedstring.md)は、単にバイトのシーケンスとして表現されます。
- [配列](../../../sql-reference/data-types/array.md)は、可変幅整数（varint）（符号なしの[LEB128](https://en.wikipedia.org/wiki/LEB128)）として表現され、その後に配列の要素が続きます。

[NULL](/sql-reference/syntax#null)サポートのために、各[Nullable](/sql-reference/data-types/nullable.md)値の前に`1`または`0`を含む追加のバイトが追加されます。  
- `1`の場合、その値は`NULL`であり、このバイトは別の値として解釈されます。  
- `0`の場合、バイトの後の値は`NULL`ではありません。

`RowBinary`フォーマットと`RawBlob`フォーマットの比較については、次を参照してください：[Raw Formats Comparison](../RawBLOB.md/#raw-formats-comparison)

## 例の使い方 {#example-usage}

## フォーマット設定 {#format-settings}

<RowBinaryFormatSettings/>
