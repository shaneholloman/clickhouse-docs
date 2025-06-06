---
'description': '哈希函数的文档'
'sidebar_label': 'Hash'
'sidebar_position': 85
'slug': '/sql-reference/functions/hash-functions'
'title': '哈希函数'
---



# Hash Functions

哈希函数可用于元素的确定性伪随机洗牌。

Simhash 是一种哈希函数，它为相近（相似）的参数返回相近的哈希值。

大多数哈希函数接受任意数量的任意类型的参数。

:::note
NULL 的哈希值是 NULL。要获得 Nullable 列的非 NULL 哈希值，请将其包装在元组中：
```sql
SELECT cityHash64(tuple(NULL))
```
:::

:::note
要计算整个表的内容的哈希值，请使用 `sum(cityHash64(tuple(*)))` (或其他哈希函数)。`tuple` 确保包含 NULL 值的行不会被跳过。`sum` 确保行的顺序无关紧要。
:::
## halfMD5 {#halfmd5}

[将所有输入参数解释为字符串](/sql-reference/functions/type-conversion-functions#reinterpretasstring)并计算每个参数的 [MD5](https://en.wikipedia.org/wiki/MD5) 哈希值。然后将哈希值组合，取结果字符串哈希的前 8 字节，并将其解释为大端字节顺序的 `UInt64`。

```sql
halfMD5(par1, ...)
```

该函数相对较慢（每个处理器核心每秒处理 500 万个短字符串）。
考虑使用 [sipHash64](#siphash64) 函数代替。

**Arguments**

该函数接受可变数量的输入参数。参数可以是任何 [支持的数据类型](../data-types/index.md)。对于某些数据类型，即使参数的类型不同（不同大小的整数，带有相同数据的命名和未命名的 `Tuple`，以及具有相同数据的 `Map` 和相应的 `Array(Tuple(key, value))` 类型），其哈希函数的计算值可能是相同的。

**Returned Value**

一个 [UInt64](../data-types/int-uint.md) 数据类型的哈希值。

**Example**

```sql
SELECT halfMD5(array('e','x','a'), 'mple', 10, toDateTime('2019-06-15 23:00:00')) AS halfMD5hash, toTypeName(halfMD5hash) AS type;
```

```response
┌────────halfMD5hash─┬─type───┐
│ 186182704141653334 │ UInt64 │
└────────────────────┴────────┘
```
## MD4 {#md4}

计算字符串的 MD4，并将结果字节集作为 FixedString(16) 返回。
## MD5 {#md5}

计算字符串的 MD5，并将结果字节集作为 FixedString(16) 返回。
如果您不特别需要 MD5，但需要一个体面的 128 位加密哈希，请使用 'sipHash128' 函数代替。
如果您想要与 md5sum 工具输出相同的结果，请使用 lower(hex(MD5(s)))。
## RIPEMD160 {#ripemd160}

生成 [RIPEMD-160](https://en.wikipedia.org/wiki/RIPEMD) 哈希值。

**Syntax**

```sql
RIPEMD160(input)
```

**Parameters**

- `input`: 输入字符串。 [字符串](../data-types/string.md)

**Returned value**

- 160 位 `RIPEMD-160` 哈希值，类型为 [FixedString(20)](../data-types/fixedstring.md)。

**Example**

使用 [hex](../functions/encoding-functions.md/#hex) 函数将结果表示为十六进制编码字符串。

查询：

```sql
SELECT HEX(RIPEMD160('The quick brown fox jumps over the lazy dog'));
```

```response
┌─HEX(RIPEMD160('The quick brown fox jumps over the lazy dog'))─┐
│ 37F332F68DB77BD9D7EDD4969571AD671CF9DD3B                      │
└───────────────────────────────────────────────────────────────┘
```
## sipHash64 {#siphash64}

生成 64 位 [SipHash](https://en.wikipedia.org/wiki/SipHash) 哈希值。

```sql
sipHash64(par1,...)
```

这是一个加密哈希函数。它的速度至少是 [MD5](#md5) 哈希函数的三倍。

该函数 [将所有输入参数解释为字符串](/sql-reference/functions/type-conversion-functions#reinterpretasstring) 并计算每个参数的哈希值。然后通过以下算法组合哈希：

1. 第一个和第二个哈希值连接到一个数组中并进行哈希。
2. 先前计算的哈希值和第三个输入参数的哈希值以类似方式进行哈希。
3. 对原始输入的所有剩余哈希值重复此计算。

**Arguments**

该函数接受可变数量的任何 [支持的数据类型](../data-types/index.md) 的输入参数。

**Returned Value**

一个 [UInt64](../data-types/int-uint.md) 数据类型的哈希值。

请注意，对于不同参数类型的相同输入值，计算得到的哈希值可能相等。这影响了例如不同大小的整数类型，带有相同数据的命名和未命名的 `Tuple`，`Map` 以及相应的具有相同数据的 `Array(Tuple(key, value))` 类型。

**Example**

```sql
SELECT sipHash64(array('e','x','a'), 'mple', 10, toDateTime('2019-06-15 23:00:00')) AS SipHash, toTypeName(SipHash) AS type;
```

```response
┌──────────────SipHash─┬─type───┐
│ 11400366955626497465 │ UInt64 │
└──────────────────────┴────────┘
```
## sipHash64Keyed {#siphash64keyed}

与 [sipHash64](#siphash64) 相同，但另外接收一个显式的密钥参数，而不是使用固定密钥。

**Syntax**

```sql
sipHash64Keyed((k0, k1), par1,...)
```

**Arguments**

与 [sipHash64](#siphash64) 相同，但第一个参数是表示密钥的两个 UInt64 值的元组。

**Returned value**

一个 [UInt64](../data-types/int-uint.md) 数据类型的哈希值。

**Example**

查询：

```sql
SELECT sipHash64Keyed((506097522914230528, 1084818905618843912), array('e','x','a'), 'mple', 10, toDateTime('2019-06-15 23:00:00')) AS SipHash, toTypeName(SipHash) AS type;
```

```response
┌─────────────SipHash─┬─type───┐
│ 8017656310194184311 │ UInt64 │
└─────────────────────┴────────┘
```
## sipHash128 {#siphash128}

与 [sipHash64](#siphash64) 类似，但生成一个 128 位哈希值，即最终的异或折叠状态进行到 128 位。

:::note
这个 128 位变体与参考实现不同，并且它更弱。
这个版本存在的原因是，在编写时，没有官方的 128 位 SipHash 扩展。
新的项目应该使用 [sipHash128Reference](#siphash128reference)。
:::

**Syntax**

```sql
sipHash128(par1,...)
```

**Arguments**

与 [sipHash64](#siphash64) 相同。

**Returned value**

一个 128 位 `SipHash` 哈希值，类型为 [FixedString(16)](../data-types/fixedstring.md)。

**Example**

查询：

```sql
SELECT hex(sipHash128('foo', '\x01', 3));
```

结果：

```response
┌─hex(sipHash128('foo', '', 3))────┐
│ 9DE516A64A414D4B1B609415E4523F24 │
└──────────────────────────────────┘
```
## sipHash128Keyed {#siphash128keyed}

与 [sipHash128](#siphash128) 相同，但另外接收一个显式的密钥参数，而不是使用固定密钥。

:::note
这个 128 位变体与参考实现不同，并且它更弱。
这个版本存在的原因是，在编写时，没有官方的 128 位 SipHash 扩展。
新的项目应该使用 [sipHash128ReferenceKeyed](#siphash128referencekeyed)。
:::

**Syntax**

```sql
sipHash128Keyed((k0, k1), par1,...)
```

**Arguments**

与 [sipHash128](#siphash128) 相同，但第一个参数是表示密钥的两个 UInt64 值的元组。

**Returned value**

一个 128 位 `SipHash` 哈希值，类型为 [FixedString(16)](../data-types/fixedstring.md)。

**Example**

查询：

```sql
SELECT hex(sipHash128Keyed((506097522914230528, 1084818905618843912),'foo', '\x01', 3));
```

结果：

```response
┌─hex(sipHash128Keyed((506097522914230528, 1084818905618843912), 'foo', '', 3))─┐
│ B8467F65C8B4CFD9A5F8BD733917D9BF                                              │
└───────────────────────────────────────────────────────────────────────────────┘
```
## sipHash128Reference {#siphash128reference}

与 [sipHash128](#siphash128) 相同，但实现了来自 SipHash 原始作者的 128 位算法。

**Syntax**

```sql
sipHash128Reference(par1,...)
```

**Arguments**

与 [sipHash128](#siphash128) 相同。

**Returned value**

一个 128 位 `SipHash` 哈希值，类型为 [FixedString(16)](../data-types/fixedstring.md)。

**Example**

查询：

```sql
SELECT hex(sipHash128Reference('foo', '\x01', 3));
```

结果：

```response
┌─hex(sipHash128Reference('foo', '', 3))─┐
│ 4D1BE1A22D7F5933C0873E1698426260       │
└────────────────────────────────────────┘
```
## sipHash128ReferenceKeyed {#siphash128referencekeyed}

与 [sipHash128Reference](#siphash128reference) 相同，但另外接收一个显式的密钥参数，而不是使用固定密钥。

**Syntax**

```sql
sipHash128ReferenceKeyed((k0, k1), par1,...)
```

**Arguments**

与 [sipHash128Reference](#siphash128reference) 相同，但第一个参数是表示密钥的两个 UInt64 值的元组。

**Returned value**

一个 128 位 `SipHash` 哈希值，类型为 [FixedString(16)](../data-types/fixedstring.md)。

**Example**

查询：

```sql
SELECT hex(sipHash128ReferenceKeyed((506097522914230528, 1084818905618843912),'foo', '\x01', 3));
```

结果：

```response
┌─hex(sipHash128ReferenceKeyed((506097522914230528, 1084818905618843912), 'foo', '', 3))─┐
│ 630133C9722DC08646156B8130C4CDC8                                                       │
└────────────────────────────────────────────────────────────────────────────────────────┘
```
## cityHash64 {#cityhash64}

生成 64 位 [CityHash](https://github.com/google/cityhash) 哈希值。

```sql
cityHash64(par1,...)
```

这是一个快速的非加密哈希函数。它使用 CityHash 算法处理字符串参数，并为其他数据类型的参数使用实现特定的快速非加密哈希函数。该函数使用 CityHash 组合器来获得最终结果。

请注意，Google 在将 CityHash 添加到 ClickHouse 之后更改了 CityHash 的算法。换句话说，ClickHouse 的 cityHash64 和 Google 的上游 CityHash 现在生成不同的结果。ClickHouse 的 cityHash64 对应于 CityHash v1.0.2。

**Arguments**

该函数接受可变数量的输入参数。参数可以是任何 [支持的数据类型](../data-types/index.md)。对于某些数据类型，即使参数的类型不同（不同大小的整数，带有相同数据的命名和未命名的 `Tuple`，`Map` 以及相应的具有相同数据的 `Array(Tuple(key, value))` 类型），其哈希函数的计算值可能是相同的。

**Returned Value**

一个 [UInt64](../data-types/int-uint.md) 数据类型的哈希值。

**Examples**

调用示例：

```sql
SELECT cityHash64(array('e','x','a'), 'mple', 10, toDateTime('2019-06-15 23:00:00')) AS CityHash, toTypeName(CityHash) AS type;
```

```response
┌─────────────CityHash─┬─type───┐
│ 12072650598913549138 │ UInt64 │
└──────────────────────┴────────┘
```

以下示例展示了如何以行序列顺序计算整个表的校验和：

```sql
SELECT groupBitXor(cityHash64(*)) FROM table
```
## intHash32 {#inthash32}

从任何类型的整数计算一个 32 位哈希码。
这是一个相对快速的非加密哈希函数，适用于数字的平均质量。

**Syntax**

```sql
intHash32(int)
```

**Arguments**

- `int` — 要哈希的整数。 [(U)Int*](../data-types/int-uint.md)。

**Returned value**

- 32 位哈希码。 [UInt32](../data-types/int-uint.md)。

**Example**

查询：

```sql
SELECT intHash32(42);
```

结果：

```response
┌─intHash32(42)─┐
│    1228623923 │
└───────────────┘
```
## intHash64 {#inthash64}

从任何类型的整数计算一个 64 位哈希码。
这是一个相对快速的非加密哈希函数，适用于数字的平均质量。
它的速度比 [intHash32](#inthash32) 快。

**Syntax**

```sql
intHash64(int)
```

**Arguments**

- `int` — 要哈希的整数。 [(U)Int*](../data-types/int-uint.md)。

**Returned value**

- 64 位哈希码。 [UInt64](../data-types/int-uint.md)。

**Example**

查询：

```sql
SELECT intHash64(42);
```

结果：

```response
┌────────intHash64(42)─┐
│ 11490350930367293593 │
└──────────────────────┘
```
## SHA1, SHA224, SHA256, SHA512, SHA512_256 {#sha1-sha224-sha256-sha512-sha512_256}

计算字符串的 SHA-1, SHA-224, SHA-256, SHA-512, SHA-512-256 哈希，并将结果字节集作为 [FixedString](../data-types/fixedstring.md) 返回。

**Syntax**

```sql
SHA1('s')
...
SHA512('s')
```

该函数操作相对较慢（SHA-1 每个处理器核心每秒处理约 500 万个短字符串，而 SHA-224 和 SHA-256 每秒约处理 220 万个）。
我们建议仅在需要特定哈希函数并且无法选择其他时使用此函数。
即使在这些情况下，我们也建议在将值插入表时离线应用该函数并预计算值，而不是在 `SELECT` 查询中应用。

**Arguments**

- `s` — 用于 SHA 哈希计算的输入字符串。 [字符串](../data-types/string.md)。

**Returned value**

- SHA 哈希以未编码的十六进制 FixedString 形式返回。SHA-1 返回为 FixedString(20)，SHA-224 返回为 FixedString(28)，SHA-256 返回为 FixedString(32)，SHA-512 返回为 FixedString(64)。 [FixedString](../data-types/fixedstring.md)。

**Example**

使用 [hex](../functions/encoding-functions.md/#hex) 函数将结果表示为十六进制编码字符串。

查询：

```sql
SELECT hex(SHA1('abc'));
```

结果：

```response
┌─hex(SHA1('abc'))─────────────────────────┐
│ A9993E364706816ABA3E25717850C26C9CD0D89D │
└──────────────────────────────────────────┘
```
## BLAKE3 {#blake3}

计算 BLAKE3 哈希字符串，并将结果字节集作为 [FixedString](../data-types/fixedstring.md) 返回。

**Syntax**

```sql
BLAKE3('s')
```

该加密哈希函数集成在 ClickHouse 中，使用 BLAKE3 Rust 库。该函数相当快速，性能大约比 SHA-2 快两倍，同时生成与 SHA-256 相同长度的哈希。

**Arguments**

- s - BLAKE3 哈希计算的输入字符串。 [字符串](../data-types/string.md)。

**Return value**

- BLAKE3 哈希，类型为 FixedString(32) 的字节数组。 [FixedString](../data-types/fixedstring.md)。

**Example**

使用函数 [hex](../functions/encoding-functions.md/#hex) 将结果表示为十六进制编码字符串。

查询：
```sql
SELECT hex(BLAKE3('ABC'))
```

结果：
```sql
┌─hex(BLAKE3('ABC'))───────────────────────────────────────────────┐
│ D1717274597CF0289694F75D96D444B992A096F1AFD8E7BBFA6EBB1D360FEDFC │
└──────────────────────────────────────────────────────────────────┘
```
## URLHash(url\[, N\]) {#urlhashurl-n}

一种快速、质量良好的非加密哈希函数，用于通过某种类型的规范化从 URL 中获得字符串。
`URLHash(s)` – 从字符串计算哈希，末尾去掉一个斜杠 `/`、`?` 或 `#`，如果存在的话。
`URLHash(s, N)` – 从字符串计算哈希，直到 URL 层次结构中的 N 级别，末尾去掉一个斜杠 `/`、`?` 或 `#`，如果存在的话。
层次结构与 URLHierarchy 中的相同。
## farmFingerprint64 {#farmfingerprint64}
## farmHash64 {#farmhash64}

生成 64 位 [FarmHash](https://github.com/google/farmhash) 或指纹值。`farmFingerprint64` 更适合于稳定且可移植的值。

```sql
farmFingerprint64(par1, ...)
farmHash64(par1, ...)
```

这些函数分别使用从所有 [可用方法](https://github.com/google/farmhash/blob/master/src/farmhash.h) 中的 `Fingerprint64` 和 `Hash64` 方法。

**Arguments**

该函数接受可变数量的输入参数。参数可以是任何 [支持的数据类型](../data-types/index.md)。对于某些数据类型，即使参数的类型不同（不同大小的整数，带有相同数据的命名和未命名的 `Tuple`，`Map` 以及相应的具有相同数据的 `Array(Tuple(key, value))` 类型），其哈希函数的计算值可能是相同的。

**Returned Value**

一个 [UInt64](../data-types/int-uint.md) 数据类型的哈希值。

**Example**

```sql
SELECT farmHash64(array('e','x','a'), 'mple', 10, toDateTime('2019-06-15 23:00:00')) AS FarmHash, toTypeName(FarmHash) AS type;
```

```response
┌─────────────FarmHash─┬─type───┐
│ 17790458267262532859 │ UInt64 │
└──────────────────────┴────────┘
```
## javaHash {#javahash}

从 [字符串](http://hg.openjdk.java.net/jdk8u/jdk8u/jdk/file/478a4add975b/src/share/classes/java/lang/String.java#l1452),
[字节](https://hg.openjdk.java.net/jdk8u/jdk8u/jdk/file/478a4add975b/src/share/classes/java/lang/Byte.java#l405),
[短整型](https://hg.openjdk.java.net/jdk8u/jdk8u/jdk/file/478a4add975b/src/share/classes/java/lang/Short.java#l410),
[整型](https://hg.openjdk.java.net/jdk8u/jdk8u/jdk/file/478a4add975b/src/share/classes/java/lang/Integer.java#l959),
[长整型](https://hg.openjdk.java.net/jdk8u/jdk8u/jdk/file/478a4add975b/src/share/classes/java/lang/Long.java#l1060) 计算 JavaHash。
这个哈希函数既不快速也没有很好的质量。使用它的唯一原因是在另一个系统中已经使用了该算法，并且您必须计算完全相同的结果。

请注意，Java 仅支持计算带符号整数哈希，因此如果要计算无符号整数哈希，必须将其转换为适当的带符号 ClickHouse 类型。

**Syntax**

```sql
SELECT javaHash('')
```

**Returned value**

一个 `Int32` 数据类型的哈希值。

**Example**

查询：

```sql
SELECT javaHash(toInt32(123));
```

结果：

```response
┌─javaHash(toInt32(123))─┐
│               123      │
└────────────────────────┘
```

查询：

```sql
SELECT javaHash('Hello, world!');
```

结果：

```response
┌─javaHash('Hello, world!')─┐
│               -1880044555 │
└───────────────────────────┘
```
## javaHashUTF16LE {#javahashutf16le}

从字符串计算 [JavaHash](http://hg.openjdk.java.net/jdk8u/jdk8u/jdk/file/478a4add975b/src/share/classes/java/lang/String.java#l1452)，假设它包含 UTF-16LE 编码的字节。

**Syntax**

```sql
javaHashUTF16LE(stringUtf16le)
```

**Arguments**

- `stringUtf16le` — UTF-16LE 编码的字符串。

**Returned value**

一个 `Int32` 数据类型的哈希值。

**Example**

正确的查询，包含 UTF-16LE 编码的字符串。

查询：

```sql
SELECT javaHashUTF16LE(convertCharset('test', 'utf-8', 'utf-16le'));
```

结果：

```response
┌─javaHashUTF16LE(convertCharset('test', 'utf-8', 'utf-16le'))─┐
│                                                      3556498 │
└──────────────────────────────────────────────────────────────┘
```
## hiveHash {#hivehash}

计算字符串的 `HiveHash`。

```sql
SELECT hiveHash('')
```

这只是 [JavaHash](#javahash)，其符号位为零。该函数用于 [Apache Hive](https://en.wikipedia.org/wiki/Apache_Hive) 版本 3.0 之前。这个哈希函数既不快速也没有很好的质量。使用它的唯一原因是在另一个系统中已经使用了该算法，并且您必须计算完全相同的结果。

**Returned value**

- `hiveHash` 哈希值。 [Int32](../data-types/int-uint.md)。

**Example**

查询：

```sql
SELECT hiveHash('Hello, world!');
```

结果：

```response
┌─hiveHash('Hello, world!')─┐
│                 267439093 │
└───────────────────────────┘
```
## metroHash64 {#metrohash64}

生成 64 位 [MetroHash](http://www.jandrewrogers.com/2015/05/27/metrohash/) 哈希值。

```sql
metroHash64(par1, ...)
```

**Arguments**

该函数接受可变数量的输入参数。参数可以是任何 [支持的数据类型](../data-types/index.md)。对于某些数据类型，即使参数的类型不同（不同大小的整数，带有相同数据的命名和未命名的 `Tuple`，`Map` 以及相应的具有相同数据的 `Array(Tuple(key, value))` 类型），其哈希函数的计算值可能是相同的。

**Returned Value**

一个 [UInt64](../data-types/int-uint.md) 数据类型的哈希值。

**Example**

```sql
SELECT metroHash64(array('e','x','a'), 'mple', 10, toDateTime('2019-06-15 23:00:00')) AS MetroHash, toTypeName(MetroHash) AS type;
```

```response
┌────────────MetroHash─┬─type───┐
│ 14235658766382344533 │ UInt64 │
└──────────────────────┴────────┘
```
## jumpConsistentHash {#jumpconsistenthash}

从一个 UInt64 计算 JumpConsistentHash。
接受两个参数：一个 UInt64 类型的键和桶的数量。返回 Int32。
有关更多信息，请参见链接：[JumpConsistentHash](https://arxiv.org/pdf/1406.2294.pdf)
## kostikConsistentHash {#kostikconsistenthash}

由 Konstantin 'kostik' Oblakov 提供的 O(1) 时间和空间一致哈希算法。以前称为 `yandexConsistentHash`。

**Syntax**

```sql
kostikConsistentHash(input, n)
```

别名：`yandexConsistentHash` （保留用于向后兼容）。

**Parameters**

- `input`: 一个 UInt64 类型的键 [UInt64](../data-types/int-uint.md)。
- `n`: 桶的数量。 [UInt16](../data-types/int-uint.md)。

**Returned value**

- 一个 [UInt16](../data-types/int-uint.md) 数据类型的哈希值。

**Implementation details**

当 n &lt;= 32768 时，它效率较高。

**Example**

查询：

```sql
SELECT kostikConsistentHash(16045690984833335023, 2);
```

```response
┌─kostikConsistentHash(16045690984833335023, 2)─┐
│                                             1 │
└───────────────────────────────────────────────┘
```
## murmurHash2_32, murmurHash2_64 {#murmurhash2_32-murmurhash2_64}

生成 [MurmurHash2](https://github.com/aappleby/smhasher) 哈希值。

```sql
murmurHash2_32(par1, ...)
murmurHash2_64(par1, ...)
```

**Arguments**

这两个函数接受可变数量的输入参数。参数可以是任何 [支持的数据类型](../data-types/index.md)。对于某些数据类型，即使参数的类型不同（不同大小的整数，带有相同数据的命名和未命名的 `Tuple`，`Map` 以及相应的具有相同数据的 `Array(Tuple(key, value))` 类型），其哈希函数的计算值可能是相同的。

**Returned Value**

- `murmurHash2_32` 函数返回类型为 [UInt32](../data-types/int-uint.md) 的哈希值。
- `murmurHash2_64` 函数返回类型为 [UInt64](../data-types/int-uint.md) 的哈希值。

**Example**

```sql
SELECT murmurHash2_64(array('e','x','a'), 'mple', 10, toDateTime('2019-06-15 23:00:00')) AS MurmurHash2, toTypeName(MurmurHash2) AS type;
```

```response
┌──────────MurmurHash2─┬─type───┐
│ 11832096901709403633 │ UInt64 │
└──────────────────────┴────────┘
```
## gccMurmurHash {#gccmurmurhash}

使用与 [gcc](https://github.com/gcc-mirror/gcc/blob/41d6b10e96a1de98e90a7c0378437c3255814b16/libstdc%2B%2B-v3/include/bits/functional_hash.h#L191) 相同的哈希种子计算 64 位 [MurmurHash2](https://github.com/aappleby/smhasher) 哈希值。它在 Clang 和 GCC 构建之间是可移植的。

**Syntax**

```sql
gccMurmurHash(par1, ...)
```

**Arguments**

- `par1, ...` — 可变数量的参数，可以是任何 [支持的数据类型](/sql-reference/data-types)。

**Returned value**

- 计算得到的哈希值。 [UInt64](../data-types/int-uint.md)。

**Example**

查询：

```sql
SELECT
    gccMurmurHash(1, 2, 3) AS res1,
    gccMurmurHash(('a', [1, 2, 3], 4, (4, ['foo', 'bar'], 1, (1, 2)))) AS res2
```

结果：

```response
┌─────────────────res1─┬────────────────res2─┐
│ 12384823029245979431 │ 1188926775431157506 │
└──────────────────────┴─────────────────────┘
```
## kafkaMurmurHash {#kafkamurmurhash}

使用与 [Kafka](https://github.com/apache/kafka/blob/461c5cfe056db0951d9b74f5adc45973670404d7/clients/src/main/java/org/apache/kafka/common/utils/Utils.java#L482) 相同的哈希种子计算 32 位 [MurmurHash2](https://github.com/aappleby/smhasher) 哈希值，并且不使用最高位，以便与 [Default Partitioner](https://github.com/apache/kafka/blob/139f7709bd3f5926901a21e55043388728ccca78/clients/src/main/java/org/apache/kafka/clients/producer/internals/BuiltInPartitioner.java#L328) 兼容。

**Syntax**

```sql
MurmurHash(par1, ...)
```

**Arguments**

- `par1, ...` — 可变数量的参数，可以是任何 [支持的数据类型](/sql-reference/data-types)。

**Returned value**

- 计算得到的哈希值。 [UInt32](../data-types/int-uint.md)。

**Example**

查询：

```sql
SELECT
    kafkaMurmurHash('foobar') AS res1,
    kafkaMurmurHash(array('e','x','a'), 'mple', 10, toDateTime('2019-06-15 23:00:00')) AS res2
```

结果：

```response
┌───────res1─┬─────res2─┐
│ 1357151166 │ 85479775 │
└────────────┴──────────┘
```
## murmurHash3_32, murmurHash3_64 {#murmurhash3_32-murmurhash3_64}

生成 [MurmurHash3](https://github.com/aappleby/smhasher) 哈希值。

```sql
murmurHash3_32(par1, ...)
murmurHash3_64(par1, ...)
```

**Arguments**

这两个函数接受可变数量的输入参数。参数可以是任何 [支持的数据类型](../data-types/index.md)。对于某些数据类型，即使参数的类型不同（不同大小的整数，带有相同数据的命名和未命名的 `Tuple`，`Map` 以及相应的具有相同数据的 `Array(Tuple(key, value))` 类型），其哈希函数的计算值可能是相同的。

**Returned Value**

- `murmurHash3_32` 函数返回类型为 [UInt32](../data-types/int-uint.md) 的哈希值。
- `murmurHash3_64` 函数返回类型为 [UInt64](../data-types/int-uint.md) 的哈希值。

**Example**

```sql
SELECT murmurHash3_32(array('e','x','a'), 'mple', 10, toDateTime('2019-06-15 23:00:00')) AS MurmurHash3, toTypeName(MurmurHash3) AS type;
```

```response
┌─MurmurHash3─┬─type───┐
│     2152717 │ UInt32 │
└─────────────┴────────┘
```
## murmurHash3_128 {#murmurhash3_128}

生成 128 位 [MurmurHash3](https://github.com/aappleby/smhasher) 哈希值。

**Syntax**

```sql
murmurHash3_128(expr)
```

**Arguments**

- `expr` — 一组 [表达式](/sql-reference/syntax#expressions)。 [字符串](../data-types/string.md)。

**Returned value**

一个 128 位 `MurmurHash3` 哈希值。 [FixedString(16)](../data-types/fixedstring.md)。

**Example**

查询：

```sql
SELECT hex(murmurHash3_128('foo', 'foo', 'foo'));
```

结果：

```response
┌─hex(murmurHash3_128('foo', 'foo', 'foo'))─┐
│ F8F7AD9B6CD4CF117A71E277E2EC2931          │
└───────────────────────────────────────────┘
```
## xxh3 {#xxh3}

生成 64 位 [xxh3](https://github.com/Cyan4973/xxHash) 哈希值。

**Syntax**

```sql
xxh3(expr)
```

**Arguments**

- `expr` — 一组 [表达式](/sql-reference/syntax#expressions)，可以是任意数据类型。

**Returned value**

一个 64 位 `xxh3` 哈希值。 [UInt64](../data-types/int-uint.md)。

**Example**

查询：

```sql
SELECT xxh3('Hello', 'world')
```

结果：

```response
┌─xxh3('Hello', 'world')─┐
│    5607458076371731292 │
└────────────────────────┘
```
## xxHash32, xxHash64 {#xxhash32-xxhash64}

从字符串计算 `xxHash`。提供 32 位和 64 位两种形式。

```sql
SELECT xxHash32('')

OR

SELECT xxHash64('')
```

**Returned value**

- 哈希值。 [UInt32/64](../data-types/int-uint.md)。

:::note
返回类型为 `xxHash32` 的将为 `UInt32`，而 `xxHash64` 将为 `UInt64`。
:::

**Example**

查询：

```sql
SELECT xxHash32('Hello, world!');
```

结果：

```response
┌─xxHash32('Hello, world!')─┐
│                 834093149 │
└───────────────────────────┘
```

**See Also**

- [xxHash](http://cyan4973.github.io/xxHash/).
## ngramSimHash {#ngramsimhash}

将 ASCII 字符串分成 n-grams，大小为 `ngramsize` 符号，并返回 n-gram `simhash`。区分大小写。

可以用于检测半重复字符串，配合 [bitHammingDistance](../functions/bit-functions.md/#bithammingdistance) 使用。计算的两个字符串的 `simhashes` 的 [Hamming Distance](https://en.wikipedia.org/wiki/Hamming_distance) 越小，这两 string 越可能相同。

**Syntax**

```sql
ngramSimHash(string[, ngramsize])
```

**Arguments**

- `string` — 字符串。 [字符串](../data-types/string.md)。
- `ngramsize` — n-gram 的大小。可选。可能值：从 `1` 到 `25` 的任意数字。默认值：`3`。 [UInt8](../data-types/int-uint.md)。

**Returned value**

- 哈希值。 [UInt64](../data-types/int-uint.md)。

**Example**

查询：

```sql
SELECT ngramSimHash('ClickHouse') AS Hash;
```

结果：

```response
┌───────Hash─┐
│ 1627567969 │
└────────────┘
```
## ngramSimHashCaseInsensitive {#ngramsimhashcaseinsensitive}

将 ASCII 字符串分成 n-grams，大小为 `ngramsize` 符号，并返回 n-gram `simhash`。不区分大小写。

可以用于检测半重复字符串，配合 [bitHammingDistance](../functions/bit-functions.md/#bithammingdistance) 使用。计算的两个字符串的 `simhashes` 的 [Hamming Distance](https://en.wikipedia.org/wiki/Hamming_distance) 越小，这两 string 越可能相同。

**Syntax**

```sql
ngramSimHashCaseInsensitive(string[, ngramsize])
```

**Arguments**

- `string` — 字符串。 [字符串](../data-types/string.md)。
- `ngramsize` — n-gram 的大小。可选。可能值：从 `1` 到 `25` 的任意数字。默认值：`3`。 [UInt8](../data-types/int-uint.md)。

**Returned value**

- 哈希值。 [UInt64](../data-types/int-uint.md)。

**Example**

查询：

```sql
SELECT ngramSimHashCaseInsensitive('ClickHouse') AS Hash;
```

结果：

```response
┌──────Hash─┐
│ 562180645 │
└───────────┘
```
## ngramSimHashUTF8 {#ngramsimhashutf8}

将 UTF-8 字符串分成 n-grams，大小为 `ngramsize` 符号，并返回 n-gram `simhash`。区分大小写。

可以用于检测半重复字符串，配合 [bitHammingDistance](../functions/bit-functions.md/#bithammingdistance) 使用。计算的两个字符串的 `simhashes` 的 [Hamming Distance](https://en.wikipedia.org/wiki/Hamming_distance) 越小，这两 string 越可能相同。

**Syntax**

```sql
ngramSimHashUTF8(string[, ngramsize])
```

**Arguments**

- `string` — 字符串。 [字符串](../data-types/string.md)。
- `ngramsize` — n-gram 的大小。可选。可能值：从 `1` 到 `25` 的任意数字。默认值：`3`。 [UInt8](../data-types/int-uint.md)。

**Returned value**

- 哈希值。 [UInt64](../data-types/int-uint.md)。

**Example**

查询：

```sql
SELECT ngramSimHashUTF8('ClickHouse') AS Hash;
```

结果：

```response
┌───────Hash─┐
│ 1628157797 │
└────────────┘
```
## ngramSimHashCaseInsensitiveUTF8 {#ngramsimhashcaseinsensitiveutf8}

将 UTF-8 字符串分成 n-grams，大小为 `ngramsize` 符号，并返回 n-gram `simhash`。不区分大小写。

可以用于检测半重复字符串，配合 [bitHammingDistance](../functions/bit-functions.md/#bithammingdistance) 使用。计算的两个字符串的 `simhashes` 的 [Hamming Distance](https://en.wikipedia.org/wiki/Hamming_distance) 越小，这两 string 越可能相同。

**Syntax**

```sql
ngramSimHashCaseInsensitiveUTF8(string[, ngramsize])
```

**Arguments**

- `string` — 字符串。 [字符串](../data-types/string.md)。
- `ngramsize` — n-gram 的大小。可选。可能值：从 `1` 到 `25` 的任意数字。默认值：`3`。 [UInt8](../data-types/int-uint.md)。

**Returned value**

- 哈希值。 [UInt64](../data-types/int-uint.md)。

**Example**

查询：

```sql
SELECT ngramSimHashCaseInsensitiveUTF8('ClickHouse') AS Hash;
```

结果：

```response
┌───────Hash─┐
│ 1636742693 │
└────────────┘
```
## wordShingleSimHash {#wordshinglesimhash}

将 ASCII 字符串分成部分（shingles），大小为 `shinglesize` 个单词，并返回单词 shingle 的 `simhash`。区分大小写。

可以用于检测半重复字符串，配合 [bitHammingDistance](../functions/bit-functions.md/#bithammingdistance) 使用。计算的两个字符串的 `simhashes` 的 [Hamming Distance](https://en.wikipedia.org/wiki/Hamming_distance) 越小，这两 string 越可能相同。

**Syntax**

```sql
wordShingleSimHash(string[, shinglesize])
```

**Arguments**

- `string` — 字符串。 [字符串](../data-types/string.md)。
- `shinglesize` — 单词 shingle 的大小。可选。可能值：从 `1` 到 `25` 的任意数字。默认值：`3`。 [UInt8](../data-types/int-uint.md)。

**Returned value**

- 哈希值。 [UInt64](../data-types/int-uint.md)。

**Example**

查询：

```sql
SELECT wordShingleSimHash('ClickHouse® is a column-oriented database management system (DBMS) for online analytical processing of queries (OLAP).') AS Hash;
```

结果：

```response
┌───────Hash─┐
│ 2328277067 │
└────────────┘
```
## wordShingleSimHashCaseInsensitive {#wordshinglesimhashcaseinsensitive}

将 ASCII 字符串分成部分（shingles），大小为 `shinglesize` 个单词，并返回单词 shingle 的 `simhash`。不区分大小写。

可以用于检测半重复字符串，配合 [bitHammingDistance](../functions/bit-functions.md/#bithammingdistance) 使用。计算的两个字符串的 `simhashes` 的 [Hamming Distance](https://en.wikipedia.org/wiki/Hamming_distance) 越小，这两 string 越可能相同。

**Syntax**

```sql
wordShingleSimHashCaseInsensitive(string[, shinglesize])
```

**Arguments**

- `string` — 字符串。 [字符串](../data-types/string.md)。
- `shinglesize` — 单词 shingle 的大小。可选。可能值：从 `1` 到 `25` 的任意数字。默认值：`3`。 [UInt8](../data-types/int-uint.md)。

**Returned value**

- 哈希值。 [UInt64](../data-types/int-uint.md)。

**Example**

查询：

```sql
SELECT wordShingleSimHashCaseInsensitive('ClickHouse® is a column-oriented database management system (DBMS) for online analytical processing of queries (OLAP).') AS Hash;
```

结果：

```response
┌───────Hash─┐
│ 2194812424 │
└────────────┘
```
## wordShingleSimHashUTF8 {#wordshinglesimhashutf8}

将 UTF-8 字符串分成部分（shingles），大小为 `shinglesize` 个单词，并返回单词 shingle 的 `simhash`。区分大小写。

可以用于检测半重复字符串，配合 [bitHammingDistance](../functions/bit-functions.md/#bithammingdistance) 使用。计算的两个字符串的 `simhashes` 的 [Hamming Distance](https://en.wikipedia.org/wiki/Hamming_distance) 越小，这两 string 越可能相同。

**Syntax**

```sql
wordShingleSimHashUTF8(string[, shinglesize])
```

**Arguments**

- `string` — 字符串。 [字符串](../data-types/string.md)。
- `shinglesize` — 单词 shingle 的大小。可选。可能值：从 `1` 到 `25` 的任意数字。默认值：`3`。 [UInt8](../data-types/int-uint.md)。

**Returned value**

- 哈希值。 [UInt64](../data-types/int-uint.md)。

**Example**

查询：

```sql
SELECT wordShingleSimHashUTF8('ClickHouse® is a column-oriented database management system (DBMS) for online analytical processing of queries (OLAP).') AS Hash;
```

结果：

```response
┌───────Hash─┐
│ 2328277067 │
└────────────┘
```
## wordShingleSimHashCaseInsensitiveUTF8 {#wordshinglesimhashcaseinsensitiveutf8}

将 UTF-8 字符串分成部分（shingles），大小为 `shinglesize` 个单词，并返回单词 shingle 的 `simhash`。不区分大小写。

可以用于检测半重复字符串，配合 [bitHammingDistance](../functions/bit-functions.md/#bithammingdistance) 使用。计算的两个字符串的 `simhashes` 的 [Hamming Distance](https://en.wikipedia.org/wiki/Hamming_distance) 越小，这两 string 越可能相同。

**Syntax**

```sql
wordShingleSimHashCaseInsensitiveUTF8(string[, shinglesize])
```

**Arguments**

- `string` — 字符串。 [字符串](../data-types/string.md)。
- `shinglesize` — 单词 shingle 的大小。可选。可能值：从 `1` 到 `25` 的任意数字。默认值：`3`。 [UInt8](../data-types/int-uint.md)。

**Returned value**

- 哈希值。 [UInt64](../data-types/int-uint.md)。

**Example**

查询：

```sql
SELECT wordShingleSimHashCaseInsensitiveUTF8('ClickHouse® is a column-oriented database management system (DBMS) for online analytical processing of queries (OLAP).') AS Hash;
```

结果：

```response
┌───────Hash─┐
│ 2194812424 │
└────────────┘
```
## wyHash64 {#wyhash64}

生成 64 位 [wyHash64](https://github.com/wangyi-fudan/wyhash) 哈希值。

**Syntax**

```sql
wyHash64(string)
```

**Arguments**

- `string` — 字符串。 [字符串](../data-types/string.md)。

**Returned value**

- 哈希值。 [UInt64](../data-types/int-uint.md)。

**Example**

查询：

```sql
SELECT wyHash64('ClickHouse') AS Hash;
```

结果：

```response
┌─────────────────Hash─┐
│ 12336419557878201794 │
└──────────────────────┘
```
## ngramMinHash {#ngramminhash}

将 ASCII 字符串分成 n-grams，大小为 `ngramsize` 符号，并计算每个 n-gram 的哈希值。使用 `hashnum` 最小哈希计算最小哈希，使用 `hashnum` 最大哈希计算最大哈希。返回包含这些哈希的元组。区分大小写。

可以用于检测半重复字符串，配合 [tupleHammingDistance](../functions/tuple-functions.md/#tuplehammingdistance) 使用。对于两个字符串：如果返回的哈希之一在两个字符串中是相同的，我们认为这两个字符串是相同的。

**Syntax**

```sql
ngramMinHash(string[, ngramsize, hashnum])
```

**Arguments**

- `string` — 字符串。 [字符串](../data-types/string.md)。
- `ngramsize` — n-gram 的大小。可选。可能值：从 `1` 到 `25` 的任意数字。默认值：`3`。 [UInt8](../data-types/int-uint.md)。
- `hashnum` — 用于计算结果的最小和最大哈希的数量。可选。可能值：从 `1` 到 `25` 的任意数字。默认值：`6`。 [UInt8](../data-types/int-uint.md)。

**Returned value**

- 包含两个哈希的元组 — 最小哈希和最大哈希。 [Tuple](../data-types/tuple.md)([UInt64](../data-types/int-uint.md), [UInt64](../data-types/int-uint.md))。

**Example**

查询：

```sql
SELECT ngramMinHash('ClickHouse') AS Tuple;
```

结果：

```response
┌─Tuple──────────────────────────────────────┐
│ (18333312859352735453,9054248444481805918) │
└────────────────────────────────────────────┘
```

## ngramMinHashCaseInsensitive {#ngramminhashcaseinsensitive}

将 ASCII 字符串拆分为 `ngramsize` 符号的 n-grams，并计算每个 n-gram 的哈希值。使用 `hashnum` 最小哈希计算最小哈希，并使用 `hashnum` 最大哈希计算最大哈希。返回一个包含这些哈希的元组。区分大小写。

可用于检测半重复字符串，结合使用 [tupleHammingDistance](../functions/tuple-functions.md/#tuplehammingdistance)。对于两个字符串：如果返回的哈希值之一在这两个字符串中相同，我们认为这两个字符串是相同的。

**语法**

```sql
ngramMinHashCaseInsensitive(string[, ngramsize, hashnum])
```

**参数**

- `string` — 字符串。 [String](../data-types/string.md)。
- `ngramsize` — n-gram 的大小。可选。可能的值：从 `1` 到 `25` 的任意数字。默认值：`3`。 [UInt8](../data-types/int-uint.md)。
- `hashnum` — 用于计算结果的最小和最大哈希的数量。可选。可能的值：从 `1` 到 `25` 的任意数字。默认值：`6`。 [UInt8](../data-types/int-uint.md)。

**返回值**

- 包含两个哈希的元组 — 最小和最大哈希。 [Tuple](../data-types/tuple.md)([UInt64](../data-types/int-uint.md), [UInt64](../data-types/int-uint.md))。

**示例**

查询：

```sql
SELECT ngramMinHashCaseInsensitive('ClickHouse') AS Tuple;
```

结果：

```response
┌─Tuple──────────────────────────────────────┐
│ (2106263556442004574,13203602793651726206) │
└────────────────────────────────────────────┘
```
## ngramMinHashUTF8 {#ngramminhashutf8}

将 UTF-8 字符串拆分为 `ngramsize` 符号的 n-grams，并计算每个 n-gram 的哈希值。使用 `hashnum` 最小哈希计算最小哈希，并使用 `hashnum` 最大哈希计算最大哈希。返回一个包含这些哈希的元组。区分大小写。

可用于检测半重复字符串，结合使用 [tupleHammingDistance](../functions/tuple-functions.md/#tuplehammingdistance)。对于两个字符串：如果返回的哈希值之一在这两个字符串中相同，我们认为这两个字符串是相同的。

**语法**

```sql
ngramMinHashUTF8(string[, ngramsize, hashnum])
```

**参数**

- `string` — 字符串。 [String](../data-types/string.md)。
- `ngramsize` — n-gram 的大小。可选。可能的值：从 `1` 到 `25` 的任意数字。默认值：`3`。 [UInt8](../data-types/int-uint.md)。
- `hashnum` — 用于计算结果的最小和最大哈希的数量。可选。可能的值：从 `1` 到 `25` 的任意数字。默认值：`6`。 [UInt8](../data-types/int-uint.md)。

**返回值**

- 包含两个哈希的元组 — 最小和最大哈希。 [Tuple](../data-types/tuple.md)([UInt64](../data-types/int-uint.md), [UInt64](../data-types/int-uint.md))。

**示例**

查询：

```sql
SELECT ngramMinHashUTF8('ClickHouse') AS Tuple;
```

结果：

```response
┌─Tuple──────────────────────────────────────┐
│ (18333312859352735453,6742163577938632877) │
└────────────────────────────────────────────┘
```
## ngramMinHashCaseInsensitiveUTF8 {#ngramminhashcaseinsensitiveutf8}

将 UTF-8 字符串拆分为 `ngramsize` 符号的 n-grams，并计算每个 n-gram 的哈希值。使用 `hashnum` 最小哈希计算最小哈希，并使用 `hashnum` 最大哈希计算最大哈希。返回一个包含这些哈希的元组。区分大小写。

可用于检测半重复字符串，结合使用 [tupleHammingDistance](../functions/tuple-functions.md/#tuplehammingdistance)。对于两个字符串：如果返回的哈希值之一在这两个字符串中相同，我们认为这两个字符串是相同的。

**语法**

```sql
ngramMinHashCaseInsensitiveUTF8(string [, ngramsize, hashnum])
```

**参数**

- `string` — 字符串。 [String](../data-types/string.md)。
- `ngramsize` — n-gram 的大小。可选。可能的值：从 `1` 到 `25` 的任意数字。默认值：`3`。 [UInt8](../data-types/int-uint.md)。
- `hashnum` — 用于计算结果的最小和最大哈希的数量。可选。可能的值：从 `1` 到 `25` 的任意数字。默认值：`6`。 [UInt8](../data-types/int-uint.md)。

**返回值**

- 包含两个哈希的元组 — 最小和最大哈希。 [Tuple](../data-types/tuple.md)([UInt64](../data-types/int-uint.md), [UInt64](../data-types/int-uint.md))。

**示例**

查询：

```sql
SELECT ngramMinHashCaseInsensitiveUTF8('ClickHouse') AS Tuple;
```

结果：

```response
┌─Tuple───────────────────────────────────────┐
│ (12493625717655877135,13203602793651726206) │
└─────────────────────────────────────────────┘
```
## ngramMinHashArg {#ngramminhasharg}

将 ASCII 字符串拆分为 `ngramsize` 符号的 n-grams，并返回通过相同输入计算的 [ngramMinHash](#ngramminhash) 函数得到的具有最小和最大哈希的 n-grams。区分大小写。

**语法**

```sql
ngramMinHashArg(string[, ngramsize, hashnum])
```

**参数**

- `string` — 字符串。 [String](../data-types/string.md)。
- `ngramsize` — n-gram 的大小。可选。可能的值：从 `1` 到 `25` 的任意数字。默认值：`3`。 [UInt8](../data-types/int-uint.md)。
- `hashnum` — 用于计算结果的最小和最大哈希的数量。可选。可能的值：从 `1` 到 `25` 的任意数字。默认值：`6`。 [UInt8](../data-types/int-uint.md)。

**返回值**

- 包含两个元组的元组，每个元组包含 `hashnum` 个 n-grams。 [Tuple](../data-types/tuple.md)([Tuple](../data-types/tuple.md)([String](../data-types/string.md)), [Tuple](../data-types/tuple.md)([String](../data-types/string.md)))。

**示例**

查询：

```sql
SELECT ngramMinHashArg('ClickHouse') AS Tuple;
```

结果：

```response
┌─Tuple─────────────────────────────────────────────────────────────────────────┐
│ (('ous','ick','lic','Hou','kHo','use'),('Hou','lic','ick','ous','ckH','Cli')) │
└───────────────────────────────────────────────────────────────────────────────┘
```
## ngramMinHashArgCaseInsensitive {#ngramminhashargcaseinsensitive}

将 ASCII 字符串拆分为 `ngramsize` 符号的 n-grams，并返回通过相同输入计算的 [ngramMinHashCaseInsensitive](#ngramminhashcaseinsensitive) 函数得到的具有最小和最大哈希的 n-grams。区分大小写。

**语法**

```sql
ngramMinHashArgCaseInsensitive(string[, ngramsize, hashnum])
```

**参数**

- `string` — 字符串。 [String](../data-types/string.md)。
- `ngramsize` — n-gram 的大小。可选。可能的值：从 `1` 到 `25` 的任意数字。默认值：`3`。 [UInt8](../data-types/int-uint.md)。
- `hashnum` — 用于计算结果的最小和最大哈希的数量。可选。可能的值：从 `1` 到 `25` 的任意数字。默认值：`6`。 [UInt8](../data-types/int-uint.md)。

**返回值**

- 包含两个元组的元组，每个元组包含 `hashnum` 个 n-grams。 [Tuple](../data-types/tuple.md)([Tuple](../data-types/tuple.md)([String](../data-types/string.md)), [Tuple](../data-types/tuple.md)([String](../data-types/string.md)))。

**示例**

查询：

```sql
SELECT ngramMinHashArgCaseInsensitive('ClickHouse') AS Tuple;
```

结果：

```response
┌─Tuple─────────────────────────────────────────────────────────────────────────┐
│ (('ous','ick','lic','kHo','use','Cli'),('kHo','lic','ick','ous','ckH','Hou')) │
└───────────────────────────────────────────────────────────────────────────────┘
```
## ngramMinHashArgUTF8 {#ngramminhashargutf8}

将 UTF-8 字符串拆分为 `ngramsize` 符号的 n-grams，并返回通过相同输入计算的 [ngramMinHashUTF8](#ngramminhashutf8) 函数得到的具有最小和最大哈希的 n-grams。区分大小写。

**语法**

```sql
ngramMinHashArgUTF8(string[, ngramsize, hashnum])
```

**参数**

- `string` — 字符串。 [String](../data-types/string.md)。
- `ngramsize` — n-gram 的大小。可选。可能的值：从 `1` 到 `25` 的任意数字。默认值：`3`。 [UInt8](../data-types/int-uint.md)。
- `hashnum` — 用于计算结果的最小和最大哈希的数量。可选。可能的值：从 `1` 到 `25` 的任意数字。默认值：`6`。 [UInt8](../data-types/int-uint.md)。

**返回值**

- 包含两个元组的元组，每个元组包含 `hashnum` 个 n-grams。 [Tuple](../data-types/tuple.md)([Tuple](../data-types/tuple.md)([String](../data-types/string.md)), [Tuple](../data-types/tuple.md)([String](../data-types/string.md)))。

**示例**

查询：

```sql
SELECT ngramMinHashArgUTF8('ClickHouse') AS Tuple;
```

结果：

```response
┌─Tuple─────────────────────────────────────────────────────────────────────────┐
│ (('ous','ick','lic','Hou','kHo','use'),('kHo','Hou','lic','ick','ous','ckH')) │
└───────────────────────────────────────────────────────────────────────────────┘
```
## ngramMinHashArgCaseInsensitiveUTF8 {#ngramminhashargcaseinsensitiveutf8}

将 UTF-8 字符串拆分为 `ngramsize` 符号的 n-grams，并返回通过相同输入计算的 [ngramMinHashCaseInsensitiveUTF8](#ngramminhashcaseinsensitiveutf8) 函数得到的具有最小和最大哈希的 n-grams。区分大小写。

**语法**

```sql
ngramMinHashArgCaseInsensitiveUTF8(string[, ngramsize, hashnum])
```

**参数**

- `string` — 字符串。 [String](../data-types/string.md)。
- `ngramsize` — n-gram 的大小。可选。可能的值：从 `1` 到 `25` 的任意数字。默认值：`3`。 [UInt8](../data-types/int-uint.md)。
- `hashnum` — 用于计算结果的最小和最大哈希的数量。可选。可能的值：从 `1` 到 `25` 的任意数字。默认值：`6`。 [UInt8](../data-types/int-uint.md)。

**返回值**

- 包含两个元组的元组，每个元组包含 `hashnum` 个 n-grams。 [Tuple](../data-types/tuple.md)([Tuple](../data-types/tuple.md)([String](../data-types/string.md)), [Tuple](../data-types/tuple.md)([String](../data-types/string.md)))。

**示例**

查询：

```sql
SELECT ngramMinHashArgCaseInsensitiveUTF8('ClickHouse') AS Tuple;
```

结果：

```response
┌─Tuple─────────────────────────────────────────────────────────────────────────┐
│ (('ckH','ous','ick','lic','kHo','use'),('kHo','lic','ick','ous','ckH','Hou')) │
└───────────────────────────────────────────────────────────────────────────────┘
```
## wordShingleMinHash {#wordshingleminhash}

将 ASCII 字符串拆分为 `shinglesize` 个单词的部分（shingles），并计算每个单词 shingle 的哈希值。使用 `hashnum` 最小哈希计算最小哈希，并使用 `hashnum` 最大哈希计算最大哈希。返回一个包含这些哈希的元组。区分大小写。

可用于检测半重复字符串，结合使用 [tupleHammingDistance](../functions/tuple-functions.md/#tuplehammingdistance)。对于两个字符串：如果返回的哈希值之一在这两个字符串中相同，我们认为这两个字符串是相同的。

**语法**

```sql
wordShingleMinHash(string[, shinglesize, hashnum])
```

**参数**

- `string` — 字符串。 [String](../data-types/string.md)。
- `shinglesize` — 单词 shingle 的大小。可选。可能的值：从 `1` 到 `25` 的任意数字。默认值：`3`。 [UInt8](../data-types/int-uint.md)。
- `hashnum` — 用于计算结果的最小和最大哈希的数量。可选。可能的值：从 `1` 到 `25` 的任意数字。默认值：`6`。 [UInt8](../data-types/int-uint.md)。

**返回值**

- 包含两个哈希的元组 — 最小和最大哈希。 [Tuple](../data-types/tuple.md)([UInt64](../data-types/int-uint.md), [UInt64](../data-types/int-uint.md))。

**示例**

查询：

```sql
SELECT wordShingleMinHash('ClickHouse® is a column-oriented database management system (DBMS) for online analytical processing of queries (OLAP).') AS Tuple;
```

结果：

```response
┌─Tuple──────────────────────────────────────┐
│ (16452112859864147620,5844417301642981317) │
└────────────────────────────────────────────┘
```
## wordShingleMinHashCaseInsensitive {#wordshingleminhashcaseinsensitive}

将 ASCII 字符串拆分为 `shinglesize` 个单词的部分（shingles），并计算每个单词 shingle 的哈希值。使用 `hashnum` 最小哈希计算最小哈希，并使用 `hashnum` 最大哈希计算最大哈希。返回一个包含这些哈希的元组。区分大小写。

可用于检测半重复字符串，结合使用 [tupleHammingDistance](../functions/tuple-functions.md/#tuplehammingdistance)。对于两个字符串：如果返回的哈希值之一在这两个字符串中相同，我们认为这两个字符串是相同的。

**语法**

```sql
wordShingleMinHashCaseInsensitive(string[, shinglesize, hashnum])
```

**参数**

- `string` — 字符串。 [String](../data-types/string.md)。
- `shinglesize` — 单词 shingle 的大小。可选。可能的值：从 `1` 到 `25` 的任意数字。默认值：`3`。 [UInt8](../data-types/int-uint.md)。
- `hashnum` — 用于计算结果的最小和最大哈希的数量。可选。可能的值：从 `1` 到 `25` 的任意数字。默认值：`6`。 [UInt8](../data-types/int-uint.md)。

**返回值**

- 包含两个哈希的元组 — 最小和最大哈希。 [Tuple](../data-types/tuple.md)([UInt64](../data-types/int-uint.md), [UInt64](../data-types/int-uint.md))。

**示例**

查询：

```sql
SELECT wordShingleMinHashCaseInsensitive('ClickHouse® is a column-oriented database management system (DBMS) for online analytical processing of queries (OLAP).') AS Tuple;
```

结果：

```response
┌─Tuple─────────────────────────────────────┐
│ (3065874883688416519,1634050779997673240) │
└───────────────────────────────────────────┘
```
## wordShingleMinHashUTF8 {#wordshingleminhashutf8}

将 UTF-8 字符串拆分为 `shinglesize` 个单词的部分（shingles），并计算每个单词 shingle 的哈希值。使用 `hashnum` 最小哈希计算最小哈希，并使用 `hashnum` 最大哈希计算最大哈希。返回一个包含这些哈希的元组。区分大小写。

可用于检测半重复字符串，结合使用 [tupleHammingDistance](../functions/tuple-functions.md/#tuplehammingdistance)。对于两个字符串：如果返回的哈希值之一在这两个字符串中相同，我们认为这两个字符串是相同的。

**语法**

```sql
wordShingleMinHashUTF8(string[, shinglesize, hashnum])
```

**参数**

- `string` — 字符串。 [String](../data-types/string.md)。
- `shinglesize` — 单词 shingle 的大小。可选。可能的值：从 `1` 到 `25` 的任意数字。默认值：`3`。 [UInt8](../data-types/int-uint.md)。
- `hashnum` — 用于计算结果的最小和最大哈希的数量。可选。可能的值：从 `1` 到 `25` 的任意数字。默认值：`6`。 [UInt8](../data-types/int-uint.md)。

**返回值**

- 包含两个哈希的元组 — 最小和最大哈希。 [Tuple](../data-types/tuple.md)([UInt64](../data-types/int-uint.md), [UInt64](../data-types/int-uint.md))。

**示例**

查询：

```sql
SELECT wordShingleMinHashUTF8('ClickHouse® is a column-oriented database management system (DBMS) for online analytical processing of queries (OLAP).') AS Tuple;
```

结果：

```response
┌─Tuple──────────────────────────────────────┐
│ (16452112859864147620,5844417301642981317) │
└────────────────────────────────────────────┘
```
## wordShingleMinHashCaseInsensitiveUTF8 {#wordshingleminhashcaseinsensitiveutf8}

将 UTF-8 字符串拆分为 `shinglesize` 个单词的部分（shingles），并计算每个单词 shingle 的哈希值。使用 `hashnum` 最小哈希计算最小哈希，并使用 `hashnum` 最大哈希计算最大哈希。返回一个包含这些哈希的元组。区分大小写。

可用于检测半重复字符串，结合使用 [tupleHammingDistance](../functions/tuple-functions.md/#tuplehammingdistance)。对于两个字符串：如果返回的哈希值之一在这两个字符串中相同，我们认为这两个字符串是相同的。

**语法**

```sql
wordShingleMinHashCaseInsensitiveUTF8(string[, shinglesize, hashnum])
```

**参数**

- `string` — 字符串。 [String](../data-types/string.md)。
- `shinglesize` — 单词 shingle 的大小。可选。可能的值：从 `1` 到 `25` 的任意数字。默认值：`3`。 [UInt8](../data-types/int-uint.md)。
- `hashnum` — 用于计算结果的最小和最大哈希的数量。可选。可能的值：从 `1` 到 `25` 的任意数字。默认值：`6`。 [UInt8](../data-types/int-uint.md)。

**返回值**

- 包含两个哈希的元组 — 最小和最大哈希。 [Tuple](../data-types/tuple.md)([UInt64](../data-types/int-uint.md), [UInt64](../data-types/int-uint.md))。

**示例**

查询：

```sql
SELECT wordShingleMinHashCaseInsensitiveUTF8('ClickHouse® is a column-oriented database management system (DBMS) for online analytical processing of queries (OLAP).') AS Tuple;
```

结果：

```response
┌─Tuple─────────────────────────────────────┐
│ (3065874883688416519,1634050779997673240) │
└───────────────────────────────────────────┘
```
## wordShingleMinHashArg {#wordshingleminhasharg}

将 ASCII 字符串拆分为 `shinglesize` 个单词部分（shingles），并返回通过相同输入计算的 [wordshingleMinHash](#wordshingleminhash) 函数得到的具有最小和最大单词哈希的 shingles。区分大小写。

**语法**

```sql
wordShingleMinHashArg(string[, shinglesize, hashnum])
```

**参数**

- `string` — 字符串。 [String](../data-types/string.md)。
- `shinglesize` — 单词 shingle 的大小。可选。可能的值：从 `1` 到 `25` 的任意数字。默认值：`3`。 [UInt8](../data-types/int-uint.md)。
- `hashnum` — 用于计算结果的最小和最大哈希的数量。可选。可能的值：从 `1` 到 `25` 的任意数字。默认值：`6`。 [UInt8](../data-types/int-uint.md)。

**返回值**

- 包含两个元组的元组，每个元组包含 `hashnum` 个单词 shingles。 [Tuple](../data-types/tuple.md)([Tuple](../data-types/tuple.md)([String](../data-types/string.md)), [Tuple](../data-types/tuple.md)([String](../data-types/string.md)))。

**示例**

查询：

```sql
SELECT wordShingleMinHashArg('ClickHouse® is a column-oriented database management system (DBMS) for online analytical processing of queries (OLAP).', 1, 3) AS Tuple;
```

结果：

```response
┌─Tuple─────────────────────────────────────────────────────────────────┐
│ (('OLAP','database','analytical'),('online','oriented','processing')) │
└───────────────────────────────────────────────────────────────────────┘
```
## wordShingleMinHashArgCaseInsensitive {#wordshingleminhashargcaseinsensitive}

将 ASCII 字符串拆分为 `shinglesize` 个单词部分（shingles），并返回通过相同输入计算的 [wordShingleMinHashCaseInsensitive](#wordshingleminhashcaseinsensitive) 函数得到的具有最小和最大单词哈希的 shingles。区分大小写。

**语法**

```sql
wordShingleMinHashArgCaseInsensitive(string[, shinglesize, hashnum])
```

**参数**

- `string` — 字符串。 [String](../data-types/string.md)。
- `shinglesize` — 单词 shingle 的大小。可选。可能的值：从 `1` 到 `25` 的任意数字。默认值：`3`。 [UInt8](../data-types/int-uint.md)。
- `hashnum` — 用于计算结果的最小和最大哈希的数量。可选。可能的值：从 `1` 到 `25` 的任意数字。默认值：`6`。 [UInt8](../data-types/int-uint.md)。

**返回值**

- 包含两个元组的元组，每个元组包含 `hashnum` 个单词 shingles。 [Tuple](../data-types/tuple.md)([Tuple](../data-types/tuple.md)([String](../data-types/string.md)), [Tuple](../data-types/tuple.md)([String](../data-types/string.md)))。

**示例**

查询：

```sql
SELECT wordShingleMinHashArgCaseInsensitive('ClickHouse® is a column-oriented database management system (DBMS) for online analytical processing of queries (OLAP).', 1, 3) AS Tuple;
```

结果：

```response
┌─Tuple──────────────────────────────────────────────────────────────────┐
│ (('queries','database','analytical'),('oriented','processing','DBMS')) │
└────────────────────────────────────────────────────────────────────────┘
```
## wordShingleMinHashArgUTF8 {#wordshingleminhashargutf8}

将 UTF-8 字符串拆分为 `shinglesize` 个单词部分（shingles），并返回通过相同输入计算的 [wordShingleMinHashUTF8](#wordshingleminhashutf8) 函数得到的具有最小和最大单词哈希的 shingles。区分大小写。

**语法**

```sql
wordShingleMinHashArgUTF8(string[, shinglesize, hashnum])
```

**参数**

- `string` — 字符串。 [String](../data-types/string.md)。
- `shinglesize` — 单词 shingle 的大小。可选。可能的值：从 `1` 到 `25` 的任意数字。默认值：`3`。 [UInt8](../data-types/int-uint.md)。
- `hashnum` — 用于计算结果的最小和最大哈希的数量。可选。可能的值：从 `1` 到 `25` 的任意数字。默认值：`6`。 [UInt8](../data-types/int-uint.md)。

**返回值**

- 包含两个元组的元组，每个元组包含 `hashnum` 个单词 shingles。 [Tuple](../data-types/tuple.md)([Tuple](../data-types/tuple.md)([String](../data-types/string.md)), [Tuple](../data-types/tuple.md)([String](../data-types/string.md)))。

**示例**

查询：

```sql
SELECT wordShingleMinHashArgUTF8('ClickHouse® is a column-oriented database management system (DBMS) for online analytical processing of queries (OLAP).', 1, 3) AS Tuple;
```

结果：

```response
┌─Tuple─────────────────────────────────────────────────────────────────┐
│ (('OLAP','database','analytical'),('online','oriented','processing')) │
└───────────────────────────────────────────────────────────────────────┘
```
## wordShingleMinHashArgCaseInsensitiveUTF8 {#wordshingleminhashargcaseinsensitiveutf8}

将 UTF-8 字符串拆分为 `shinglesize` 个单词部分（shingles），并返回通过相同输入计算的 [wordShingleMinHashCaseInsensitiveUTF8](#wordshingleminhashcaseinsensitiveutf8) 函数得到的具有最小和最大单词哈希的 shingles。区分大小写。

**语法**

```sql
wordShingleMinHashArgCaseInsensitiveUTF8(string[, shinglesize, hashnum])
```

**参数**

- `string` — 字符串。 [String](../data-types/string.md)。
- `shinglesize` — 单词 shingle 的大小。可选。可能的值：从 `1` 到 `25` 的任意数字。默认值：`3`。 [UInt8](../data-types/int-uint.md)。
- `hashnum` — 用于计算结果的最小和最大哈希的数量。可选。可能的值：从 `1` 到 `25` 的任意数字。默认值：`6`。 [UInt8](../data-types/int-uint.md)。

**返回值**

- 包含两个元组的元组，每个元组包含 `hashnum` 个单词 shingles。 [Tuple](../data-types/tuple.md)([Tuple](../data-types/tuple.md)([String](../data-types/string.md)), [Tuple](../data-types/tuple.md)([String](../data-types/string.md)))。

**示例**

查询：

```sql
SELECT wordShingleMinHashArgCaseInsensitiveUTF8('ClickHouse® is a column-oriented database management system (DBMS) for online analytical processing of queries (OLAP).', 1, 3) AS Tuple;
```

结果：

```response
┌─Tuple──────────────────────────────────────────────────────────────────┐
│ (('queries','database','analytical'),('oriented','processing','DBMS')) │
└────────────────────────────────────────────────────────────────────────┘
```
## sqidEncode {#sqidencode}

将数字编码为 [Sqid](https://sqids.org/)，即类似 YouTube 的 ID 字符串。
输出字母表为 `abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789`。
请勿将此函数用于哈希 - 生成的 ID 可以解码回原始数字。

**语法**

```sql
sqidEncode(number1, ...)
```

别名：`sqid`

**参数**

- 任意数量的 UInt8、UInt16、UInt32 或 UInt64 数字。

**返回值**

一个 sqid [String](../data-types/string.md)。

**示例**

```sql
SELECT sqidEncode(1, 2, 3, 4, 5);
```

```response
┌─sqidEncode(1, 2, 3, 4, 5)─┐
│ gXHfJ1C6dN                │
└───────────────────────────┘
```
## sqidDecode {#sqiddecode}

将 [Sqid](https://sqids.org/) 解码回其原始数字。
如果输入字符串不是有效的 sqid，则返回一个空数组。

**语法**

```sql
sqidDecode(sqid)
```

**参数**

- 一个 sqid - [String](../data-types/string.md)

**返回值**

将 sqid 转换为数字 [Array(UInt64)](../data-types/array.md)。

**示例**

```sql
SELECT sqidDecode('gXHfJ1C6dN');
```

```response
┌─sqidDecode('gXHfJ1C6dN')─┐
│ [1,2,3,4,5]              │
└──────────────────────────┘
```
## keccak256 {#keccak256}

计算 Keccak-256 哈希字符串，并将结果字节集作为 [FixedString](../data-types/fixedstring.md) 返回。

**语法**

```sql
keccak256('s')
```

此加密哈希函数在 [EVM-based blockchains](https://ethereum.github.io/yellowpaper/paper.pdf) 中被广泛使用。

**参数**

- s - 用于 Keccak-256 哈希计算的输入字符串。 [String](../data-types/string.md)。

**返回值**

- Keccak-256 哈希作为 FixedString(32) 类型的字节数组。 [FixedString](../data-types/fixedstring.md)。

**示例**

使用函数 [hex](../functions/encoding-functions.md/#hex) 将结果格式化为十六进制编码字符串。

查询：
```sql
select hex(keccak256('hello'))
```

结果：
```sql
   ┌─hex(keccak256('hello'))──────────────────────────────────────────┐
1. │ 1C8AFF950685C2ED4BC3174F3472287B56D9517B9C948127319A09A7A36DEAC8 │
   └──────────────────────────────────────────────────────────────────┘
```

<!-- 
The inner content of the tags below are replaced at doc framework build time with 
docs generated from system.functions. Please do not modify or remove the tags.
See: https://github.com/ClickHouse/clickhouse-docs/blob/main/contribute/autogenerated-documentation-from-source.md
-->

<!--AUTOGENERATED_START-->
<!--AUTOGENERATED_END-->
