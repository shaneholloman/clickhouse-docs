---
'description': '字符串替换的函数的文档'
'sidebar_label': '字符串替换'
'sidebar_position': 150
'slug': '/sql-reference/functions/string-replace-functions'
'title': '字符串替换的函数'
---


# 字符串替换函数

[一般字符串函数](string-functions.md) 和 [字符串搜索函数](string-search-functions.md) 分别进行描述。

## overlay {#overlay}

用另一个字符串 `replace` 替换字符串 `input` 的一部分，从 1 开始的索引 `offset` 开始。

**语法**

```sql
overlay(s, replace, offset[, length])
```

**参数**

- `s`: 字符串类型 [String](../data-types/string.md)。
- `replace`: 字符串类型 [String](../data-types/string.md)。
- `offset`: 整数类型 [Int](../data-types/int-uint.md)（以 1 为基准）。如果 `offset` 为负，则从字符串 `s` 的末尾计数。
- `length`: 可选。整数类型 [Int](../data-types/int-uint.md)。`length` 指定要替换的输入字符串 `s` 中片段的长度。如果未指定 `length`，从 `s` 中移除的字节数等于 `replace` 的长度；否则将移除 `length` 字节。

**返回值**

- 一个 [String](../data-types/string.md) 数据类型的值。

**示例**

```sql
SELECT overlay('My father is from Mexico.', 'mother', 4) AS res;
```

结果：

```text
┌─res──────────────────────┐
│ My mother is from Mexico.│
└──────────────────────────┘
```

```sql
SELECT overlay('My father is from Mexico.', 'dad', 4, 6) AS res;
```

结果：

```text
┌─res───────────────────┐
│ My dad is from Mexico.│
└───────────────────────┘
```

## overlayUTF8 {#overlayutf8}

用另一个字符串 `replace` 替换字符串 `input` 的一部分，从 1 开始的索引 `offset` 开始。

假设字符串包含有效的 UTF-8 编码文本。如果违反此假设，则不会抛出异常，结果是未定义的。

**语法**

```sql
overlayUTF8(s, replace, offset[, length])
```

**参数**

- `s`: 字符串类型 [String](../data-types/string.md)。
- `replace`: 字符串类型 [String](../data-types/string.md)。
- `offset`: 整数类型 [Int](../data-types/int-uint.md)（以 1 为基准）。如果 `offset` 为负，则从输入字符串 `s` 的末尾计数。
- `length`: 可选。整数类型 [Int](../data-types/int-uint.md)。`length` 指定要替换的输入字符串 `s` 中片段的长度。如果未指定 `length`，从 `s` 中移除的字符数等于 `replace` 的长度；否则将移除 `length` 个字符。

**返回值**

- 一个 [String](../data-types/string.md) 数据类型的值。

**示例**

```sql
SELECT overlay('Mein Vater ist aus Österreich.', 'der Türkei', 20) AS res;
```

结果：

```text
┌─res───────────────────────────┐
│ Mein Vater ist aus der Türkei.│
└───────────────────────────────┘
```

## replaceOne {#replaceone}

替换 `haystack` 中的子字符串 `pattern` 的第一个出现为 `replacement` 字符串。

**语法**

```sql
replaceOne(haystack, pattern, replacement)
```

## replaceAll {#replaceall}

将 `haystack` 中所有出现的子字符串 `pattern` 替换为 `replacement` 字符串。

**语法**

```sql
replaceAll(haystack, pattern, replacement)
```

别名：`replace`。

## replaceRegexpOne {#replaceregexpone}

将符合正则表达式 `pattern`（使用 [re2 语法](https://github.com/google/re2/wiki/Syntax)）匹配的子字符串在 `haystack` 中的第一个出现替换为 `replacement` 字符串。

`replacement` 可以包含替换 `\0-\9`。
替换 `\1-\9` 对应于第 1 到第 9 个捕获组（子匹配），替换 `\0` 对应于整个匹配。

要在 `pattern` 或 `replacement` 字符串中使用逐字 `\` 字符，请使用 `\` 进行转义。
还要注意字符串字面量需要额外的转义。

**语法**

```sql
replaceRegexpOne(haystack, pattern, replacement)
```

**示例**

将 ISO 日期转换为美国格式：

```sql
SELECT DISTINCT
    EventDate,
    replaceRegexpOne(toString(EventDate), '(\\d{4})-(\\d{2})-(\\d{2})', '\\2/\\3/\\1') AS res
FROM test.hits
LIMIT 7
FORMAT TabSeparated
```

结果：

```text
2014-03-17      03/17/2014
2014-03-18      03/18/2014
2014-03-19      03/19/2014
2014-03-20      03/20/2014
2014-03-21      03/21/2014
2014-03-22      03/22/2014
2014-03-23      03/23/2014
```

将字符串复制十次：

```sql
SELECT replaceRegexpOne('Hello, World!', '.*', '\\0\\0\\0\\0\\0\\0\\0\\0\\0\\0') AS res
```

结果：

```text
┌─res────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Hello, World!Hello, World!Hello, World!Hello, World!Hello, World!Hello, World!Hello, World!Hello, World!Hello, World!Hello, World! │
└────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

## replaceRegexpAll {#replaceregexpall}

类似于 `replaceRegexpOne`，但替换模式的所有出现。

别名：`REGEXP_REPLACE`。

**示例**

```sql
SELECT replaceRegexpAll('Hello, World!', '.', '\\0\\0') AS res
```

结果：

```text
┌─res────────────────────────┐
│ HHeelllloo,,  WWoorrlldd!! │
└────────────────────────────┘
```

作为例外，如果正则表达式在空子字符串上工作，则替换不得多于一次，例如：

```sql
SELECT replaceRegexpAll('Hello, World!', '^', 'here: ') AS res
```

结果：

```text
┌─res─────────────────┐
│ here: Hello, World! │
└─────────────────────┘
```

## regexpQuoteMeta {#regexpquotemeta}

在这些在正则表达式中具有特殊含义的字符之前添加反斜杠： `\0`, `\\`, `|`, `(`, `)`, `^`, `$`, `.`, `[`, `]`, `?`, `*`, `+`, `{`, `:`, `-`。

此实现与 re2::RE2::QuoteMeta 略有不同。它将零字节转义为 `\0` 而不是 `\x00`，并且仅转义所需的字符。
有关更多信息，请参见 [RE2](https://github.com/google/re2/blob/master/re2/re2.cc#L473)

**语法**

```sql
regexpQuoteMeta(s)
```

## format {#format}

使用参数中列出的值（字符串、整数等）格式化 `pattern` 字符串，类似于 Python 中的格式化。模式字符串可以包含用大括号 `{}` 包围的替换字段。未包含在大括号中的任何内容都被视为文字文本，并原样复制到输出中。文字大括号字符可以通过两个大括号进行转义：`{{ '{{' }}` 和 `{{ '}}' }}`。字段名称可以是数字（从零开始）或者为空（则隐式赋予单调增加的数字）。

**语法**

```sql
format(pattern, s0, s1, ...)
```

**示例**

```sql
SELECT format('{1} {0} {1}', 'World', 'Hello')
```

```result
┌─format('{1} {0} {1}', 'World', 'Hello')─┐
│ Hello World Hello                       │
└─────────────────────────────────────────┘
```

使用隐式数字：

```sql
SELECT format('{} {}', 'Hello', 'World')
```

```result
┌─format('{} {}', 'Hello', 'World')─┐
│ Hello World                       │
└───────────────────────────────────┘
```

## translate {#translate}

使用 `from` 和 `to` 字符串定义的字符一对一映射来替换字符串 `s` 中的字符。
`from` 和 `to` 必须是常量 ASCII 字符串。
如果 `from` 和 `to` 的大小相等，则 `s` 中的 `first` 的第 1 个字符的每个出现都被替换为 `to` 的第 1 个字符，`first` 的第 2 个字符在 `s` 中的出现被替换为 `to` 的第 2 个字符，依此类推。
如果 `from` 包含的字符比 `to` 多，则从 `s` 中删除 `from` 末尾所有没有对应字符在 `to` 中的字符。
`s` 中的非 ASCII 字符不会被此函数修改。

**语法**

```sql
translate(s, from, to)
```

**示例**

```sql
SELECT translate('Hello, World!', 'delor', 'DELOR') AS res
```

结果：

```text
┌─res───────────┐
│ HELLO, WORLD! │
└───────────────┘
```

`from` 和 `to` 参数具有不同的长度：

```sql
SELECT translate('clickhouse', 'clickhouse', 'CLICK') AS res
```

结果：

```text
┌─res───┐
│ CLICK │
└───────┘
```

## translateUTF8 {#translateutf8}

类似于 [translate](#translate)，但假设 `s`、`from` 和 `to` 是 UTF-8 编码字符串。

**语法**

```sql
translateUTF8(s, from, to)
```

**参数**

- `s`: 字符串类型 [String](../data-types/string.md)。
- `from`: 字符串类型 [String](../data-types/string.md)。
- `to`: 字符串类型 [String](../data-types/string.md)。

**返回值**

- 一个 [String](../data-types/string.md) 数据类型的值。

**示例**

查询：

```sql
SELECT translateUTF8('Münchener Straße', 'üß', 'us') AS res;
```

```response
┌─res──────────────┐
│ Munchener Strase │
└──────────────────┘
```

## printf {#printf}

`printf` 函数格式化给定字符串，与 C++ 中的 printf 函数类似，使用参数中列出的值（字符串、整数、浮点数等）。格式字符串可以包含以 `%` 字符开头的格式说明符。未包含在 `%` 和后随格式说明符中的任何内容都被视为文字文本，并原样复制到输出中。文字 `%` 字符可以通过 `%%` 进行转义。

**语法**

```sql
printf(format, arg1, arg2, ...)
```

**示例**

查询：

```sql
select printf('%%%s %s %d', 'Hello', 'World', 2024);
```

```response
┌─printf('%%%s %s %d', 'Hello', 'World', 2024)─┐
│ %Hello World 2024                            │
└──────────────────────────────────────────────┘
```

<!-- 
The inner content of the tags below are replaced at doc framework build time with 
docs generated from system.functions. Please do not modify or remove the tags.
See: https://github.com/ClickHouse/clickhouse-docs/blob/main/contribute/autogenerated-documentation-from-source.md
-->

<!--AUTOGENERATED_START-->
<!--AUTOGENERATED_END-->
