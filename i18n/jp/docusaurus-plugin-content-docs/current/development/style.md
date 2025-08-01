---
description: 'ClickHouse C++ 開発のコーディングスタイルガイド'
sidebar_label: 'C++スタイルガイド'
sidebar_position: 70
slug: '/development/style'
title: 'C++スタイルガイド'
---

# C++ スタイルガイド

## 一般的な推奨事項 {#general-recommendations}

以下は推奨事項であり、要件ではありません。
コードを編集する場合は、既存のコードのフォーマットに従うことが理にかなっています。
コードスタイルは一貫性のために必要です。一貫性があると、コードを読みやすくし、検索もしやすくなります。
多くのルールには論理的な理由がない場合があります; それらは確立された慣行に従っています。

## フォーマット {#formatting}

**1.** フォーマットのほとんどは `clang-format` によって自動的に行われます。

**2.** インデントは4スペースです。タブが4スペースを追加するように開発環境を設定してください。

**3.** 開始および終了の波括弧は別の行に置かなければなりません。

```cpp
inline void readBoolText(bool & x, ReadBuffer & buf)
{
    char tmp = '0';
    readChar(tmp, buf);
    x = tmp != '0';
}
```

**4.** 関数本体が単一の `statement` の場合、それを1行に配置することができます。波括弧の周りにスペースを置きます（行の終わりのスペースを除く）。

```cpp
inline size_t mask() const                { return buf_size() - 1; }
inline size_t place(HashValue x) const    { return x & mask(); }
```

**5.** 関数の場合。括弧の周りにスペースを入れないでください。

```cpp
void reinsert(const Value & x)
```

```cpp
memcpy(&buf[place_value], &x, sizeof(x));
```

**6.** `if`、`for`、`while` およびその他の式では、開括弧の前にスペースを挿入します（関数呼び出しとは対照的に）。

```cpp
for (size_t i = 0; i < rows; i += storage.index_granularity)
```

**7.** バイナリ演算子（`+`、`-`、`*`、`/`、`%` など）および三項演算子 `?:` の周りにスペースを追加します。

```cpp
UInt16 year = (s[0] - '0') * 1000 + (s[1] - '0') * 100 + (s[2] - '0') * 10 + (s[3] - '0');
UInt8 month = (s[5] - '0') * 10 + (s[6] - '0');
UInt8 day = (s[8] - '0') * 10 + (s[9] - '0');
```

**8.** 行終止が入力される場合は、演算子を新しい行に入れ、その前にインデントを増やします。

```cpp
if (elapsed_ns)
    message << " ("
        << rows_read_on_server * 1000000000 / elapsed_ns << " rows/s., "
        << bytes_read_on_server * 1000.0 / elapsed_ns << " MB/s.) ";
```

**9.** 必要に応じて、行内の整列のためにスペースを使用できます。

```cpp
dst.ClickLogID         = click.LogID;
dst.ClickEventID       = click.EventID;
dst.ClickGoodEvent     = click.GoodEvent;
```

**10.** 演算子 `.`、`->` の周りにスペースを使わないでください。

必要な場合、演算子は次の行にラップされることがあります。この場合、その前のオフセットは増加します。

**11.** 単項演算子（`--`、`++`、`*`、`&` など）と引数との間にスペースを使用しないでください。

**12.** カンマの後にスペースを置きますが、その前には置きません。同じルールは `for` 式内のセミコロンにも適用されます。

**13.** `[]` 演算子の間にスペースを使用しないでください。

**14.** `template <...>` 式では、`template` と `<` の間にスペースを置き、`<` の後や `>` の前にはスペースを置かないでください。

```cpp
template <typename TKey, typename TValue>
struct AggregatedStatElement
{}
```

**15.** クラスおよび構造体内では、`public`、`private`、`protected` を `class/struct` と同じレベルに書き、残りのコードをインデントします。

```cpp
template <typename T>
class MultiVersion
{
public:
    /// 使用のためのオブジェクトのバージョン。shared_ptr がバージョンのライフタイムを管理します。
    using Version = std::shared_ptr<const T>;
    ...
}
```

**16.** ファイル全体で同じ `namespace` が使用され、他に重要なことがない場合、`namespace` 内にオフセットを必要としません。

**17.** `if`、`for`、`while`、または他の式のブロックが単一の `statement` で構成されている場合、波括弧はオプションです。代わりに `statement` を別の行に置きます。このルールは、ネストされた `if`、`for`、`while` でも有効です。

ただし、内部の `statement` に波括弧または `else` が含まれる場合、外部ブロックは波括弧で書かれるべきです。

```cpp
/// 書き込みを終了します。
for (auto & stream : streams)
    stream.second->finalize();
```

**18.** 行の末尾にスペースが存在しないようにします。

**19.** ソースファイルはUTF-8でエンコードされています。

**20.** 文字列リテラル内で非ASCII文字を使用できます。

```cpp
<< ", " << (timer.elapsed() / chunks_stats.hits) << " μsec/hit.";
```

**21.** 単一の行に複数の式を書かないでください。

**22.** 関数内のコードのセクションをグループ分けし、1つの空行を挟んで区切ります。

**23.** 関数、クラスなどを1つまたは2つの空行で分けます。

**24.** `const`（値に関連する）は型名の前に書かなければなりません。

```cpp
//正しい
const char * pos
const std::string & s
//間違い
char const * pos
```

**25.** ポインタまたは参照を宣言する場合、`*` と `&` シンボルの両側にスペースを挿入する必要があります。

```cpp
//正しい
const char * pos
//間違い
const char* pos
const char *pos
```

**26.** テンプレート型を使用する場合、最も単純なケースを除いて、`using` キーワードでエイリアスを作成します。

言い換えれば、テンプレートパラメータは `using` のみで指定され、コード内で繰り返されることはありません。

`using` は、関数内などローカルに宣言できます。

```cpp
//正しい
using FileStreams = std::map<std::string, std::shared_ptr<Stream>>;
FileStreams streams;
//間違い
std::map<std::string, std::shared_ptr<Stream>> streams;
```

**27.** 異なる型の複数の変数を1つの文で宣言しないでください。

```cpp
//間違い
int x, *y;
```

**28.** Cスタイルのキャストを使用しないでください。

```cpp
//間違い
std::cerr << (int)c <<; std::endl;
//正しい
std::cerr << static_cast<int>(c) << std::endl;
```

**29.** クラスおよび構造体内では、メンバーと関数をそれぞれ可視性スコープ内でグループ化します。

**30.** 小さなクラスや構造体の場合、メソッド宣言を実装から分ける必要はありません。

他のクラスや構造体の小さなメソッドにも同様が適用されます。

テンプレートクラスや構造体の場合、メソッド宣言を実装から分けないでください（さもなければ同じ翻訳単位内で定義する必要があります）。

**31.** 140文字で行をラップできますが、80文字ではありません。

**32.** 後置インクリメント/デクリメント演算子が必要でない場合は、常に前置演算子を使用してください。

```cpp
for (Names::const_iterator it = column_names.begin(); it != column_names.end(); ++it)
```

## コメント {#comments}

**1.** 非自明なコードの部分には必ずコメントを追加してください。

これは非常に重要です。コメントを書くことで、コードが不要であることや、設計が間違っていることに気付くかもしれません。

```cpp
/** 使用できるメモリの一部。
  * たとえば、internal_buffer が1MBで、ファイルから読み取るためにバッファに読み込まれたのがわずか10バイトの場合、
  * working_buffer のサイズはわずか10バイトになります
  * (working_buffer.end() は、読み取れる10バイトのすぐ後の位置を指します)。
  */
```

**2.** コメントは必要に応じて詳細にできます。

**3.** コメントは、その説明するコードの前に置いてください。まれに、コメントはコードの後、同じ行に置くこともできます。

```cpp
/** クエリを解析し、実行します。
*/
void executeQuery(
    ReadBuffer & istr, /// クエリを読み取る場所（および、INSERTの場合はデータ）
    WriteBuffer & ostr, /// 結果を書き込む場所
    Context & context, /// DB、テーブル、データ型、エンジン、関数、集約関数...
    BlockInputStreamPtr & query_plan, /// クエリがどのように実行されたかの説明がここに書かれる可能性があります
    QueryProcessingStage::Enum stage = QueryProcessingStage::Complete /// SELECTクエリを処理する段階まで
    )
```

**4.** コメントは英語のみで書くべきです。

**5.** ライブラリを書く場合、主要なヘッダーファイル内に詳細なコメントを含めて説明してください。

**6.** 追加情報を提供しないコメントは避けてください。特に、次のようなエンプティコメントを残さないでください：

```cpp
/*
* 手続き名：
* 元の手続き名：
* 著者：
* 作成日：
* 修正日：
* 修正著者：
* 元のファイル名：
* 目的：
* 意図：
* 指名：
* 使用されるクラス：
* 定数：
* ローカル変数：
* パラメータ：
* 作成日：
* 目的：
*/
```

この例は http://home.tamk.fi/~jaalto/course/coding-style/doc/unmaintainable-code/ から借用したものです。

**7.** 各ファイルの先頭に無意味なコメント（著者、作成日など）を書く必要はありません。

**8.** 単一行コメントは3つのスラッシュ `///` で始まり、複数行コメントは `/**` で始まります。これらのコメントは「ドキュメンテーション」と見なされます。

注：これらのコメントからドキュメントを生成するためにDoxygenを使用できます。しかし、Doxygenは一般的に使用されておらず、IDEでコードをナビゲートする方が便利です。

**9.** 複数行コメントの始まりと終わりには空行を含めるべきではありません（複数行コメントを閉じる行を除く）。

**10.** コードをコメントアウトする場合、基本的なコメントを使用し、「ドキュメンテーション」コメントは使用しないでください。

**11.** コミットする前にコメントアウトされた部分のコードは削除してください。

**12.** コメントやコードに不適切な言葉を使用しないでください。

**13.** 大文字を使用しないでください。過剰な句読点を使用しないでください。

```cpp
/// WHAT THE FAIL???
```

**14.** コメントを区切りとして使用しないでください。

```cpp
///******************************************************
```

**15.** コメント内で議論を開始する必要はありません。

```cpp
/// なぜこのことをしたのですか？
```

**16.** ブロックの最後に、それが何だったかを説明するコメントを書く必要はありません。

```cpp
/// for
```

## 名前 {#names}

**1.** 変数名とクラスメンバー名には小文字とアンダースコアを使用します。

```cpp
size_t max_block_size;
```

**2.** 関数（メソッド）の名前には小文字で始まるキャメルケースを使用します。

```cpp
std::string getName() const override { return "Memory"; }
```

**3.** クラス（構造体）の名前には大文字で始まるキャメルケースを使用します。インターフェースのプレフィックスはIを使用しません。

```cpp
class StorageMemory : public IStorage
```

**4.** `using` もクラスと同様に命名されます。

**5.** テンプレートタイプの名前：単純な場合は `T` を使用します； `T`、`U`； `T1`、`T2` を使用します。

より複雑な場合は、クラス名のルールに従うか、プレフィックス `T` を追加します。

```cpp
template <typename TKey, typename TValue>
struct AggregatedStatElement
```

**6.** テンプレート定数引数の名前：変数名のルールに従うか、単純なケースでは `N` を使用します。

```cpp
template <bool without_www>
struct ExtractDomain
```

**7.** 抽象クラス（インターフェース）の場合、`I` プレフィックスを追加できます。

```cpp
class IProcessor
```

**8.** 変数をローカルに使用する場合は、短い名前を使用できます。

他のすべてのケースでは、意味を説明する名前を使用してください。

```cpp
bool info_successfully_loaded = false;
```

**9.** `define` およびグローバル定数の名前は、アンダースコアを使用したALL_CAPSになります。

```cpp
#define MAX_SRC_TABLE_NAMES_TO_STORE 1000
```

**10.** ファイル名はその内容と同じスタイルを使用します。

ファイルが単一のクラスを含む場合、ファイルもクラスと同じ名前（CamelCase）にします。

ファイルが単一の関数を含む場合、ファイルも関数と同じ名前（camelCase）にします。

**11.** 名前に略語が含まれている場合は：

- 変数名の場合、略語は小文字を使用するべきです `mysql_connection` （ではなく `mySQL_connection`）。
- クラスおよび関数の名前の場合、略語の大文字を保持します `MySQLConnection` （ではなく `MySqlConnection`）。

**12.** クラスメンバーを初期化するためだけに使用されるコンストラクタ引数は、クラスメンバーと同じ名前を使用しましたが、末尾にアンダースコアを追加します。

```cpp
FileQueueProcessor(
    const std::string & path_,
    const std::string & prefix_,
    std::shared_ptr<FileHandler> handler_)
    : path(path_),
    prefix(prefix_),
    handler(handler_),
    log(&Logger::get("FileQueueProcessor"))
{
}
```

アンダースコアのサフィックスは、引数がコンストラクタ本体で使用されない場合は省略できます。

**13.** ローカル変数とクラスメンバーの名前に違いはありません（プレフィックスは不要）。

```cpp
timer (not m_timer)
```

**14.** `enum` 内の定数には、先頭が大文字のキャメルケースを使用します。ALL_CAPSも許可されています。`enum` がローカルでない場合は、`enum class` を使用します。

```cpp
enum class CompressionMethod
{
    QuickLZ = 0,
    LZ4     = 1,
};
```

**15.** すべての名前は英語で書かれなければなりません。ヘブライ語の単語の音訳を許可することはできません。

    not T_PAAMAYIM_NEKUDOTAYIM

**16.** 略語は、よく知られている場合（略語の意味をWikipediaや検索エンジンで簡単に見つけられる場合）は、許可されます。

    `AST`, `SQL`。

    許可されない略語： `NVDH`（いくつかのランダムな文字）。

不完全な単語は、短縮版が一般的に使用されている場合は許可されます。

略語の横に完全な名前がコメントに記載されている場合、略語を使用することもできます。

**17.** C++のソースコードを含むファイルは `.cpp` 拡張子を持つ必要があります。ヘッダーファイルは `.h` 拡張子を持つ必要があります。

## コードの書き方 {#how-to-write-code}

**1.** メモリ管理。

手動メモリ解放（`delete`）はライブラリコードでのみ使用できます。

ライブラリコードでは、`delete` 演算子はデストラクタ内でのみ使用できます。

アプリケーションコードでは、メモリはそれを所有するオブジェクトによって解放される必要があります。

例：

- 最も簡単な方法は、オブジェクトをスタックに置くか、別のクラスのメンバーにすることです。
- 小さなオブジェクトが大量にある場合は、コンテナを使用します。
- ヒープ内の少数のオブジェクトの自動解放には `shared_ptr/unique_ptr` を使用します。

**2.** リソース管理。

`RAII` を使用し、上記を参照してください。

**3.** エラーハンドリング。

例外を使用します。ほとんどの場合、例外をスローするだけで捕捉する必要はありません（`RAII` のため）。

オフラインデータ処理アプリケーションでは、例外をキャッチしないことが許可されていることがよくあります。

ユーザーリクエストを処理するサーバーでは、接続ハンドラーのトップレベルで例外をキャッチすることが通常十分です。

スレッド関数では、すべての例外をキャッチして保持し、`join` の後にメインスレッドで再スローする必要があります。

```cpp
/// まだ計算が行われていない場合、最初のブロックを同期的に計算します
if (!started)
{
    calculate();
    started = true;
}
else /// 計算が既に進行中の場合、結果を待ちます
    pool.wait();

if (exception)
    exception->rethrow();
```

例外を処理せずに隠すべきではありません。すべての例外を無視するだけでログに記録することは避けてください。

```cpp
//正しくない
catch (...) {}
```

一部の例外を無視する必要がある場合は、特定の例外に対してのみ行い、残りを再スローしてください。

```cpp
catch (const DB::Exception & e)
{
    if (e.code() == ErrorCodes::UNKNOWN_AGGREGATE_FUNCTION)
        return nullptr;
    else
        throw;
}
```

応答コードや `errno` を持つ関数を使用する場合は、常に結果を確認し、エラーが発生した場合は例外をスローしてください。

```cpp
if (0 != close(fd))
    throw ErrnoException(ErrorCodes::CANNOT_CLOSE_FILE, "Cannot close file {}", file_name);
```

コード内の不変条件をチェックするためにアサートを使用できます。

**4.** 例外タイプ。

アプリケーションコードでは、複雑な例外階層を使用する必要はありません。例外テキストはシステム管理者に理解可能でなければなりません。

**5.** デストラクタからの例外スロー。

これは推奨されていませんが、許可されています。

次のオプションを使用します：

- 例外が発生する可能性があるすべての作業を事前に行う関数（`done()` または `finalize()`）を作成します。その関数が呼び出された場合、後でデストラクタに例外がないようにします。
- ネットワークを介してメッセージを送信するようなあまりにも複雑なタスクは、クラスユーザーが破棄前に呼び出さなければならない別のメソッドに置くことができます。
- デストラクタに例外が発生した場合、隠すよりもログを記録する方が良いです（ロガーが使用可能な場合）。
- 簡単なアプリケーションでは、例外を処理するために `std::terminate`（C++11でデフォルトで `noexcept` の場合）に依存することが許可されます。

**6.** 匿名コードブロック。

特定の変数をローカルにするために、単一の関数内に別のコードブロックを作成し、そのブロックを出る際にデストラクタが呼び出されるようにできます。

```cpp
Block block = data.in->read();

{
    std::lock_guard<std::mutex> lock(mutex);
    data.ready = true;
    data.block = block;
}

ready_any.set();
```

**7.** マルチスレッド。

オフラインデータ処理プログラムでは：

- 単一のCPUコアで可能な限り最高のパフォーマンスを得るようにしてください。必要に応じてコードを並列化できます。

サーバーアプリケーションでは：

- 要求を処理するためにスレッドプールを使用します。この時点で、ユーザー空間のコンテキストスイッチを必要とするタスクはありませんでした。

フォークは並列化には使われません。

**8.** スレッドの同期。

異なるスレッドが異なるメモリセル（さらに良いのは異なるキャッシュライン）を使用し、スレッド同期を使用しないことが可能な場合がよくあります（`joinAll`を除く）。

同期が必要な場合は、ほとんどの場合、`lock_guard` の下でミューテックスを使用するのが十分です。

他のケースでは、システム同期原始を使用します。ビジーウェイトは使用しないでください。

原子的な操作は、最も単純なケースでのみ使用するべきです。

ロックフリーのデータ構造を実装しようとしないでください、専門分野でない限り。

**9.** ポインタ対参照。

ほとんどの場合、参照の方を好みます。

**10.** `const`。

定数参照、定数へのポインタ、`const_iterator`、および `const` メソッドを使用します。

`const` をデフォルトと見なし、必要な場合にのみ非 `const` を使用します。

値を渡す場合、`const` を使用するのは通常意味がありません。

**11.** unsigned。

必要な場合は `unsigned` を使用してください。

**12.** 数値型。

`UInt8`、`UInt16`、`UInt32`、`UInt64`、`Int8`、`Int16`、`Int32`、および `Int64`、さらに `size_t`、`ssize_t`、`ptrdiff_t` 型を使用します。

これらの型を `signed/unsigned long`、`long long`、`short`、`signed/unsigned char`、`char` の数字に使用しないでください。

**13.** 引数を渡す。

複雑な値は、移動される場合は値で渡し、`std::move`を使用します。ループ内で値を更新する場合は参照で渡します。

ヒープ内で作成されたオブジェクトの所有権を関数が取得する場合、引数の型を `shared_ptr` または `unique_ptr` にします。

**14.** 戻り値。

ほとんどの場合、単に `return` を使用します。`return std::move(res)` と書かないでください。

関数がヒープにオブジェクトを割り当ててそれを返す場合、`shared_ptr` または `unique_ptr` を使用します。

まれに（ループ内での値の更新）、引数を介して値を返す必要がある場合、この引数は参照であるべきです。

```cpp
using AggregateFunctionPtr = std::shared_ptr<IAggregateFunction>;

/** 名前から集約関数を作成します。
  */
class AggregateFunctionFactory
{
public:
    AggregateFunctionFactory();
    AggregateFunctionPtr get(const String & name, const DataTypes & argument_types) const;
```

**15.** `namespace`。

アプリケーションコードには、別々の `namespace` を使う必要はありません。

小さなライブラリにもこれを必要としません。

中程度から大規模なライブラリでは、すべてを `namespace` に配置してください。

ライブラリの `.h` ファイルでは、実装の詳細を隠すために `namespace detail` を使用できます。

`.cpp` ファイル内では、シンボルを隠すために `static` または匿名 `namespace` を使用できます。

また、`enum`のために `namespace` を使用して、対応する名前が外部の `namespace` に落ちないようにすることができます（ただし、`enum class` を使用する方が良いです）。

**16.** 遅延初期化。

初期化に引数が必要な場合、通常、デフォルトコンストラクタを書くべきではありません。

後で初期化を遅らせる必要がある場合は、無効なオブジェクトを作成するデフォルトコンストラクタを追加できます。あるいは、少数のオブジェクトの場合、`shared_ptr/unique_ptr` を使用できます。

```cpp
Loader(DB::Connection * connection_, const std::string & query, size_t max_block_size_);

/// 遅延初期化用
Loader() {}
```

**17.** 仮想関数。

クラスが多態的に使用されることを意図していない場合、関数を仮想にする必要はありません。これはデストラクタにも当てはまります。

**18.** エンコーディング。

どこでもUTF-8を使用します。`std::string` と `char *` を使用します。`std::wstring` と `wchar_t` を使用しないでください。

**19.** ロギング。

コード全体の例を参照してください。

コミットする前に、意味のないログとデバッグログ、およびその他のデバッグ出力をすべて削除してください。

サイクル内でのログは、Traceレベルでさえ避けるべきです。

ログは、どのログレベルでも読みやすくなければなりません。

ログは、基本的にアプリケーションコードでのみ使用されるべきです。

ログメッセージは英語で書く必要があります。

ログは、システム管理者にも理解できるようにpreferablyであるべきです。

ログに不適切な言葉を使用しないでください。

ログにはUTF-8エンコーディングを使用します。稀な場合には、ログに非ASCII文字を使用できます。

**20.** 入出力。

アプリケーションのパフォーマンスに重要な内部ループで `iostreams` を使用しないでください（`stringstream` は決して使用しないでください）。

代わりに `DB/IO` ライブラリを使用してください。

**21.** 日付と時刻。

`DateLUT` ライブラリを参照してください。

**22.** インクルード。

常にインクルードガードの代わりに `#pragma once` を使用してください。

**23.** using。

`using namespace` は使用しません。特定のもので `using` を使用できますが、クラスや関数内にローカルにしてください。

**24.** 不要な場合を除いて、関数の `trailing return type` を使用しないでください。

```cpp
auto f() -> void
```

**25.** 変数の宣言と初期化。

```cpp
//正しい方法
std::string s = "Hello";
std::string s{"Hello"};

//間違った方法
auto s = std::string{"Hello"};
```

**26.** 仮想関数については、基底クラスには `virtual` を書きますが、子孫クラスには `virtual` の代わりに `override` を書きます。

## C++の未使用機能 {#unused-features-of-c}

**1.** 仮想継承は使用されません。

**2.** 現代C++の便利な構文シュガーを持つ構文（例えば：

```cpp
//構文シュガーなしでの従来の方法
template <typename G, typename = std::enable_if_t<std::is_same<G, F>::value, void>> // std::enable_ifによるSFINAE、::valueの使用
std::pair<int, int> func(const E<G> & e) // 明示的に指定された戻り値の型
{
    if (elements.count(e)) // .count() メンバーシップテスト
    {
        // ...
    }

    elements.erase(
        std::remove_if(
            elements.begin(), elements.end(),
            [&](const auto x){
                return x == 1;
            }),
        elements.end()); // remove-eraseイディオム

    return std::make_pair(1, 2); // make_pair()を使用してペアを作成する
}

//構文シュガーあり（C++14/17/20）
template <typename G>
requires std::same_v<G, F> // C++20コンセプトによるSFINAE、C++14テンプレートエイリアスの使用
auto func(const E<G> & e) // auto戻り値の型(C++14)
{
    if (elements.contains(e)) // C++20 .contains メンバーシップテスト
    {
        // ...
    }

    elements.erase_if(
        elements,
        [&](const auto x){
            return x == 1;
        }); // C++20 std::erase_if

    return {1, 2}; // または: std::pair(1, 2)を返します; 初期化リストまたは値初期化(C++17)でペアを作成します。
}
```

## プラットフォーム {#platform}

**1.** 特定のプラットフォーム向けにコードを書きます。

ただし、他の条件が同じであれば、クロスプラットフォームまたはポータブルコードが優先されます。

**2.** 言語：C++20（利用可能な [C++20 機能](https://en.cppreference.com/w/cpp/compiler_support#C.2B.2B20_features) のリストを参照してください）。

**3.** コンパイラ： `clang`。執筆時点（2025年3月）では、コードはclangバージョン>=19でコンパイルされます。

標準ライブラリが使用されます（`libc++`）。 

**4.**OS：Linux Ubuntu、Preciseより古くはありません。

**5.**x86_64 CPUアーキテクチャ向けにコードが書かれています。

CPU命令セットは我々のサーバー間で最小限のサポートされているセットです。現在のところ、SSE 4.2です。

**6.** いくつかの例外を除いて `-Wall -Wextra -Werror -Weverything` コンパイルフラグを使用します。

**7.** スタティックリンクをすべてのライブラリで使用しますが、静的に接続するのが難しいライブラリはあります（`ldd` コマンドの出力を参照）。

**8.** コードはリリース設定で開発およびデバッグされます。

## ツール {#tools}

**1.** KDevelop は良いIDEです。

**2.** デバッグには `gdb`、`valgrind`（`memcheck`）、`strace`、`-fsanitize=...`、または `tcmalloc_minimal_debug` を使用してください。

**3.** プロファイリングには `Linux Perf`、`valgrind`（`callgrind`）、または `strace -cf` を使用します。

**4.** ソースはGitで管理されています。

**5.** アセンブリは `CMake` を使用します。

**6.** プログラムは `deb` パッケージを使用してリリースされます。

**7.** master へのコミットはビルドを壊してはいけません。

ただし、有効と見なされるのは選択されたリビジョンのみです。

**8.** コードがまだ部分的にしか準備されていなくても、できるだけ頻繁にコミットを行います。

この目的のためにブランチを使用してください。

`master` ブランチ内のコードがまだビルド可能でない場合は、`push` の前にビルドから除外します。数日以内に完成させるか、削除する必要があります。

**9.** 重要でない変更の場合、ブランチを利用してサーバーで公開します。

**10.** 未使用のコードはリポジトリから削除されます。

## ライブラリ {#libraries}

**1.** C++20標準ライブラリが使用されており（実験的な拡張も許可されます）、`boost` と `Poco` フレームワークも使用されています。

**2.** OSパッケージからのライブラリの使用は許可されていません。事前にインストールされたライブラリの使用も禁止されています。すべてのライブラリは、`contrib` ディレクトリ内のソースコードの形で配置され、ClickHouseで構築される必要があります。詳細については、[新しいサードパーティライブラリを追加するためのガイドライン](/development/contrib#adding-and-maintaining-third-party-libraries)を参照してください。

**3.** 既存で使用されているライブラリに優先権を与えます。

## 一般的な推奨事項 {#general-recommendations-1}

**1.** できるだけ少ないコードを書くようにしてください。

**2.** 最も単純な解決策を試してください。

**3.** コードがどのように機能するか、内部ループがどのように機能するかがわかるまでコードを記述しないでください。

**4.** 単純な場合は、クラスや構造体の代わりに `using` を利用してください。

**5.** 可能であれば、コピーコンストラクタ、代入演算子、デストラクタ（少なくとも1つの仮想関数が含まれている場合の仮想関数を除く）、ムーブコンストラクタやムーブ代入演算子を記述しないでください。言い換えれば、コンパイラが生成する関数は正しく機能する必要があります。`default` を使用できます。

**6.** コードの簡素化が奨励されています。可能な限りコードのサイズを削減してください。
