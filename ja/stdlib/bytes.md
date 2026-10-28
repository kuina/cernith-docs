# std.bytes

`Bytes`（0 から 255 の整数の列）の作成・読み出し・加工を行うモジュールです。`Bytes` はプレリュードの型で、専用のリテラルはありません（[03 型](../language/03-types.md)）。`import std.bytes;` で読み込み、`bytes.length(bytes: data)` のように呼び出します。モジュールの別名と引数ラベルは、どちらも `bytes` ですが、役割は異なります。

`Bytes` は不変です。`==` では長さと各要素を比較でき、大小比較はできません。`bytes.set` や `bytes.fill` は元の列を書き換えず、変更を反映した新しい列を返します。位置は 0 から数えます。

すべての関数は純粋で、効果を持ちません。事前条件（`requires`）を満たさない呼び出しは欠陥です（[12 エラーと欠陥](../language/12-errors-and-defects.md)）。外部から来るデータ（文字列、リスト）を `Bytes` に変える関数（`from_list`、`from_hex`、`from_base64`）と、`Bytes` を文字列に戻す `to_text` は、変換できない入力に対して欠陥ではなく `None` を返します。

## 関数一覧

| 関数 | 戻り値の型 | 意味 |
| --- | --- | --- |
| `bytes.empty()` | `Bytes` | 長さ 0 の列 |
| `bytes.new(count, fill)` | `Bytes` | 値 `fill` を `count` 個並べた列 |
| `bytes.from_list(items)` | `Option<Bytes>` | `List<Int>` から作る。0 から 255 以外の要素があれば `None` |
| `bytes.to_list(bytes)` | `List<Int>` | 要素のリスト |
| `bytes.length(bytes)` | `Int` | バイト数 |
| `bytes.get(bytes, index)` | `Int` | `index` 番目のバイト |
| `bytes.set(bytes, index, value)` | `Bytes` | `index` 番目を `value` にした新しい列 |
| `bytes.fill(bytes, from, to, value)` | `Bytes` | 範囲を `value` で埋めた新しい列 |
| `bytes.slice(bytes, from, to)` | `Bytes` | 位置 `from` 以上 `to` 未満の部分 |
| `bytes.concat(first, second)` | `Bytes` | 2 つをつなげた列 |
| `bytes.concat_all(parts)` | `Bytes` | リストの列をすべてつなげた列 |
| `bytes.find(bytes, pattern)` | `Option<Int>` | `pattern` が最初に現れる位置 |
| `bytes.index_of(bytes, value)` | `Option<Int>` | バイト `value` が最初に現れる位置 |
| `bytes.starts_with(bytes, prefix)` | `Bool` | `prefix` で始まるなら `true` |
| `bytes.read_uint(bytes, at, width, big_endian)` | `Int` | 位置 `at` から `width` バイトを符号なし整数として読む |
| `bytes.read_int(bytes, at, width, big_endian)` | `Int` | 同じ範囲を 2 の補数の符号つき整数として読む |
| `bytes.uint_bytes(value, width, big_endian)` | `List<Int>` | 符号なし整数を `width` 個のバイトにする |
| `bytes.int_bytes(value, width, big_endian)` | `List<Int>` | 符号つき整数を 2 の補数の `width` 個のバイトにする |
| `bytes.from_text(text)` | `Bytes` | 文字列の UTF-8 |
| `bytes.to_text(bytes)` | `Option<String>` | UTF-8 として文字列に戻す。不正なら `None` |
| `bytes.to_hex(bytes)` | `String` | 16 進数の文字列 |
| `bytes.from_hex(text)` | `Option<Bytes>` | 16 進数の文字列から作る。不正なら `None` |
| `bytes.to_base64(bytes)` | `String` | Base64 の文字列 |
| `bytes.from_base64(text)` | `Option<Bytes>` | Base64 の文字列から作る。不正なら `None` |

以降の例の `→` の右は、呼び出しの値です。`→ 欠陥` は、その呼び出しが事前条件に違反して欠陥になることを表します。例では、`Bytes` を `[1, 2]` のようにリストで書きますが、これは「`bytes.from_list(items: [1, 2])` で作った `Bytes`」の略記です。これらは、末尾の「例」の `example` で確かめてあります。

## 作る、読む

### bytes.empty

```text
pub fn empty() -> Bytes
```

長さ 0 の列です。

- `bytes.length(bytes: bytes.empty())` → `0`

### bytes.new

```text
pub fn new(count: Int, fill: Int) -> Bytes
```

値 `fill` を `count` 個並べた列を作ります。

事前条件：`0 <= count and count <= 1073741823 and fill >= 0 and fill <= 255`。負の個数、1073741823（2^30 - 1）を超える個数、0 から 255 の外の `fill` は欠陥です。

事後条件：結果の長さは `count` です。

- `bytes.to_list(bytes: bytes.new(count: 3, fill: 7))` → `[7, 7, 7]`
- `bytes.length(bytes: bytes.new(count: 0, fill: 0))` → `0`
- `bytes.new(count: -1, fill: 0)` → 欠陥
- `bytes.new(count: 1073741824, fill: 0)` → 欠陥
- `bytes.new(count: 1, fill: 256)` → 欠陥

### bytes.from_list と bytes.to_list

```text
pub fn from_list(items: List<Int>) -> Option<Bytes>
pub fn to_list(bytes: Bytes) -> List<Int>
```

`from_list` は、`List<Int>` を `Bytes` にします。すべての要素が 0 から 255 のときは `Some`、1 つでも範囲の外（256 以上、負）なら `None` です。空のリストは `Some` で、空の列になります。`to_list` は逆で、要素を `List<Int>` にします。結果の長さは `bytes.length` と同じで、要素は 0 から 255 です。

`from_list` が `Option` を返すのは、`List<Int>` にバイトの範囲外の整数が含まれる可能性があるためです。呼び出し側で事前に除外する必要はなく、`None` として処理できます。

- `bytes.from_list(items: [0, 255])` → `Some(value: ...)`（要素が `[0, 255]` の `Bytes`）
- `bytes.from_list(items: [])` → `Some(value: bytes.empty())`
- `bytes.from_list(items: [256])` → `None`
- `bytes.from_list(items: [-1])` → `None`
- `bytes.to_list(bytes: bytes.from_text(text: "A"))` → `[65]`

### bytes.length

```text
pub fn length(bytes: Bytes) -> Int
```

バイト数を返します。`bytes.from_text` で `String` から作った場合、文字数とは一致しないことがあります。

- `bytes.length(bytes: bytes.from_text(text: "é"))` → `2`

### bytes.get

```text
pub fn get(bytes: Bytes, index: Int) -> Int
```

`index` 番目のバイトを、0 から 255 の `Int` で返します。

事前条件：`index >= 0 and index < bytes.length(bytes: bytes)`。範囲の外の位置は欠陥です。

- `bytes.get(bytes: bytes.from_text(text: "ab"), index: 1)` → `98`
- `bytes.get(bytes: bytes.from_text(text: "a"), index: 1)` → 欠陥
- `bytes.get(bytes: bytes.from_text(text: "a"), index: -1)` → 欠陥

## 加工

### bytes.set

```text
pub fn set(bytes: Bytes, index: Int, value: Int) -> Bytes
```

`index` 番目を `value` にした、新しい列を返します。元の列は変わりません。

事前条件：`index >= 0 and index < bytes.length(bytes: bytes)`、かつ `value >= 0 and value <= 255`。位置が範囲の外、または `value` が 0 から 255 の外なら欠陥です。

事後条件：結果の長さは元の列と同じです。

変数に入れた列を更新するときは、`data = bytes.set(bytes: data, index: i, value: v);` のように、結果を同じ変数に代入します（[05 文と束縛](../language/05-statements.md)）。

- `bytes.to_list(bytes: bytes.set(bytes: bytes.new(count: 3, fill: 0), index: 1, value: 9))` → `[0, 9, 0]`
- `bytes.set(bytes: bytes.new(count: 1, fill: 0), index: 1, value: 0)` → 欠陥
- `bytes.set(bytes: bytes.new(count: 1, fill: 0), index: 0, value: 256)` → 欠陥
- `bytes.set(bytes: bytes.new(count: 1, fill: 0), index: 0, value: -1)` → 欠陥

元の列が変わらないことは、`set_keeps_the_original` の example で確認しています。

### bytes.fill

```text
pub fn fill(bytes: Bytes, from: Int, to: Int, value: Int) -> Bytes
```

位置 `from` から `to` の手前までを `value` にした、新しい列を返します。`from` は含み、`to` は含みません。

事前条件：`0 <= from and from <= to and to <= bytes.length(bytes: bytes)`、かつ `value >= 0 and value <= 255`。

事後条件：結果の長さは元の列と同じです。

境界条件：`from == to` なら、何も変わりません。

- `bytes.to_list(bytes: bytes.fill(bytes: bytes.new(count: 4, fill: 1), from: 1, to: 3, value: 9))` → `[1, 9, 9, 1]`
- `bytes.to_list(bytes: bytes.fill(bytes: bytes.new(count: 2, fill: 1), from: 1, to: 1, value: 9))` → `[1, 1]`
- `bytes.fill(bytes: bytes.new(count: 1, fill: 0), from: 0, to: 2, value: 0)` → 欠陥
- `bytes.fill(bytes: bytes.new(count: 1, fill: 0), from: 0, to: 1, value: 256)` → 欠陥

### bytes.slice

```text
pub fn slice(bytes: Bytes, from: Int, to: Int) -> Bytes
```

位置 `from` から `to` の手前までの部分を返します。`from` は含み、`to` は含みません。

事前条件：`0 <= from and from <= to and to <= bytes.length(bytes: bytes)`。

事後条件：結果の長さは `to - from` です。

- `bytes.to_list(bytes: bytes.slice(bytes: bytes.from_text(text: "abcd"), from: 1, to: 3))` → `[98, 99]`
- `bytes.length(bytes: bytes.slice(bytes: bytes.from_text(text: "ab"), from: 2, to: 2))` → `0`
- `bytes.slice(bytes: bytes.from_text(text: "a"), from: 1, to: 0)` → 欠陥
- `bytes.slice(bytes: bytes.from_text(text: "a"), from: 0, to: 2)` → 欠陥

### bytes.concat と bytes.concat_all

```text
pub fn concat(first: Bytes, second: Bytes) -> Bytes
pub fn concat_all(parts: List<Bytes>) -> Bytes
```

`concat` は 2 つの列を、`concat_all` はリストのすべての列を、順につなげます。`concat` の結果の長さは、2 つの長さの和です。事前条件はありません。`concat_all` に空のリストを渡すと、空の列になります。

- `bytes.to_list(bytes: bytes.concat(first: bytes.from_text(text: "a"), second: bytes.from_text(text: "bc")))` → `[97, 98, 99]`
- `bytes.to_list(bytes: bytes.concat_all(parts: [bytes.from_text(text: "a"), bytes.empty(), bytes.from_text(text: "bc")]))` → `[97, 98, 99]`
- `bytes.length(bytes: bytes.concat_all(parts: []))` → `0`

## 検索

### bytes.find、bytes.index_of、bytes.starts_with

```text
pub fn find(bytes: Bytes, pattern: Bytes) -> Option<Int>
pub fn index_of(bytes: Bytes, value: Int) -> Option<Int>
pub fn starts_with(bytes: Bytes, prefix: Bytes) -> Bool
```

`find` は、列 `pattern` が最初に現れる位置を返します。現れなければ `None` です。`index_of` は、1 つのバイト `value` が最初に現れる位置を返します。現れなければ `None` です。`starts_with` は、`bytes` が `prefix` で始まるなら `true` です。事前条件はありません。

境界条件：`pattern` や `prefix` が空の列なら、`bytes` が空でも、`find` は `Some(value: 0)`、`starts_with` は `true` です。`pattern` が `bytes` より長ければ `None` です。`index_of` に 0 から 255 の外の `value` を渡しても欠陥ではなく、どのバイトとも等しくないので `None` です。

- `bytes.find(bytes: bytes.from_text(text: "abcabc"), pattern: bytes.from_text(text: "ca"))` → `Some(value: 2)`
- `bytes.find(bytes: bytes.from_text(text: "abc"), pattern: bytes.from_text(text: "x"))` → `None`
- `bytes.find(bytes: bytes.from_text(text: "abc"), pattern: bytes.empty())` → `Some(value: 0)`
- `bytes.find(bytes: bytes.empty(), pattern: bytes.empty())` → `Some(value: 0)`
- `bytes.find(bytes: bytes.from_text(text: "a"), pattern: bytes.from_text(text: "ab"))` → `None`
- `bytes.index_of(bytes: bytes.from_text(text: "abca"), value: 99)` → `Some(value: 2)`
- `bytes.index_of(bytes: bytes.from_text(text: "ab"), value: 0)` → `None`
- `bytes.index_of(bytes: bytes.from_text(text: "ab"), value: 256)` → `None`
- `bytes.index_of(bytes: bytes.empty(), value: 0)` → `None`
- `bytes.starts_with(bytes: bytes.from_text(text: "abc"), prefix: bytes.from_text(text: "ab"))` → `true`
- `bytes.starts_with(bytes: bytes.from_text(text: "abc"), prefix: bytes.from_text(text: "bc"))` → `false`
- `bytes.starts_with(bytes: bytes.empty(), prefix: bytes.empty())` → `true`

## 整数の読み書き

複数のバイトで整数を表すときの幅とバイト順は、引数で指定します。

- `width`：バイト数。1 以上 8 以下です。
- `big_endian`：`true` ならバイト順は上位のバイトが先（ビッグエンディアン）、`false` なら下位のバイトが先（リトルエンディアン）です。

符号つきの整数は 2 の補数で表します。`width` バイトで表せる範囲は、次のとおりです。

| 種類 | 範囲 | `width: 1` | `width: 8` |
| --- | --- | --- | --- |
| 符号なし | 0 以上 256^width 未満 | 0 から 255 | 0 から 18446744073709551615 |
| 符号つき | -128 × 256^(width-1) 以上 128 × 256^(width-1) 未満 | -128 から 127 | -9223372036854775808 から 9223372036854775807 |

### bytes.read_uint

```text
pub fn read_uint(bytes: Bytes, at: Int, width: Int, big_endian: Bool) -> Int
```

位置 `at` から `width` バイトを、符号なし整数として読みます。

事前条件：`width >= 1 and width <= 8 and at >= 0 and at + width <= bytes.length(bytes: bytes)`。幅が範囲の外、または読む範囲が列の外にはみ出すと欠陥です。

事後条件：結果は 0 以上 256^width 未満です。

バイト列 `[1, 2, 3, 4]` を幅 4 で読むと、`big_endian: true` では `1 × 256^3 + 2 × 256^2 + 3 × 256 + 4 = 16909060`、`big_endian: false` では `4 × 256^3 + 3 × 256^2 + 2 × 256 + 1 = 67305985` です。

- `bytes.read_uint(bytes: [1, 2, 3, 4], at: 0, width: 4, big_endian: true)` → `16909060`
- `bytes.read_uint(bytes: [1, 2, 3, 4], at: 0, width: 4, big_endian: false)` → `67305985`
- `bytes.read_uint(bytes: [1, 2, 3], at: 1, width: 2, big_endian: true)` → `515`
- `bytes.read_uint(bytes: [9, 255, 255, 255, 255, 255, 255, 255, 255], at: 1, width: 8, big_endian: true)` → `18446744073709551615`
- `bytes.read_uint(bytes: [1, 2, 3, 4, 5, 6, 7, 8], at: 0, width: 8, big_endian: false)` → `578437695752307201`
- `bytes.read_uint(bytes: [1, 2], at: 1, width: 2, big_endian: true)` → 欠陥
- `bytes.read_uint(bytes: [1, 2], at: -1, width: 1, big_endian: true)` → 欠陥
- `bytes.read_uint(bytes: [1, 2], at: 0, width: 0, big_endian: true)` → 欠陥

### bytes.read_int

```text
pub fn read_int(bytes: Bytes, at: Int, width: Int, big_endian: Bool) -> Int
```

`read_uint` と同じ範囲を、2 の補数の符号つき整数として読みます。最上位のビットが 1 なら負です。事前条件は `read_uint` と同じです。

- `bytes.read_int(bytes: [254, 255], at: 0, width: 2, big_endian: false)` → `-2`
- `bytes.read_int(bytes: [255, 254], at: 0, width: 2, big_endian: true)` → `-2`
- `bytes.read_int(bytes: [128], at: 0, width: 1, big_endian: true)` → `-128`
- `bytes.read_int(bytes: [127], at: 0, width: 1, big_endian: true)` → `127`
- `bytes.read_int(bytes: [128, 0, 0, 0, 0, 0, 0, 0], at: 0, width: 8, big_endian: true)` → `-9223372036854775808`
- `bytes.read_int(bytes: [1, 2], at: 0, width: 9, big_endian: true)` → 欠陥

### bytes.uint_bytes

```text
pub fn uint_bytes(value: Int, width: Int, big_endian: Bool) -> List<Int>
```

符号なし整数 `value` を、`width` 個のバイトのリストにします。`read_uint` の逆です。

事前条件：`width >= 1 and width <= 8 and value >= 0 and value < 256^width`。負の値や、`width` バイトに入らない値は欠陥です。

事後条件：結果の長さは `width` で、各要素は 0 から 255 です。

`Bytes` にするには、`bytes.from_list` に渡します（事前条件を満たすなら、常に `Some` です）。

- `bytes.uint_bytes(value: 16909060, width: 4, big_endian: true)` → `[1, 2, 3, 4]`
- `bytes.uint_bytes(value: 16909060, width: 4, big_endian: false)` → `[4, 3, 2, 1]`
- `bytes.uint_bytes(value: 258, width: 2, big_endian: false)` → `[2, 1]`
- `bytes.uint_bytes(value: 0, width: 3, big_endian: true)` → `[0, 0, 0]`
- `bytes.uint_bytes(value: 18446744073709551615, width: 8, big_endian: true)` → `[255, 255, 255, 255, 255, 255, 255, 255]`
- `bytes.uint_bytes(value: 256, width: 1, big_endian: true)` → 欠陥
- `bytes.uint_bytes(value: -1, width: 1, big_endian: true)` → 欠陥
- `bytes.uint_bytes(value: 0, width: 9, big_endian: true)` → 欠陥

### bytes.int_bytes

```text
pub fn int_bytes(value: Int, width: Int, big_endian: Bool) -> List<Int>
```

符号つき整数 `value` を、2 の補数の `width` 個のバイトのリストにします。`read_int` の逆です。

事前条件：`width >= 1 and width <= 8`、かつ `value` が `-128 × 256^(width-1)` 以上 `128 × 256^(width-1)` 未満。範囲の外の値は欠陥です。

事後条件：結果の長さは `width` です。

- `bytes.int_bytes(value: -2, width: 2, big_endian: false)` → `[254, 255]`
- `bytes.int_bytes(value: -2, width: 2, big_endian: true)` → `[255, 254]`
- `bytes.int_bytes(value: 127, width: 1, big_endian: true)` → `[127]`
- `bytes.int_bytes(value: -128, width: 1, big_endian: true)` → `[128]`
- `bytes.int_bytes(value: -1, width: 3, big_endian: false)` → `[255, 255, 255]`
- `bytes.int_bytes(value: -9223372036854775808, width: 8, big_endian: true)` → `[128, 0, 0, 0, 0, 0, 0, 0]`
- `bytes.int_bytes(value: 128, width: 1, big_endian: true)` → 欠陥
- `bytes.int_bytes(value: -129, width: 1, big_endian: true)` → 欠陥
- `bytes.int_bytes(value: 0, width: 0, big_endian: true)` → 欠陥

## 文字列との変換

### bytes.from_text と bytes.to_text

```text
pub fn from_text(text: String) -> Bytes
pub fn to_text(bytes: Bytes) -> Option<String>
```

`from_text` は、文字列を UTF-8 のバイト列にします。どんな `String` でも変換できます。スカラー値によって 1 から 4 バイトになるので、結果の長さは文字数と一致するとは限りません。

`to_text` は、バイト列を UTF-8 として読み、文字列にします。UTF-8 として正しくないときは `None` です。次のものは正しくないと見なされます。

- 先頭のバイトが不正なもの（192、193、245 以上など）
- 続きのバイトが足りないもの、続きのバイトでないバイトが続くもの、先頭のバイトを伴わない続きのバイト
- 必要より長い形で書かれたもの（`[192, 128]`、`[224, 128, 128]` など）
- サロゲート（U+D800 から U+DFFF）を表すもの
- U+10FFFF を超える値を表すもの

空の列は `Some(value: "")` です。バイト 0 は U+0000 として読まれ、バイト順マーク（`[239, 187, 191]`、U+FEFF）も取り除かれずに、文字として残ります。`to_text(from_text(s))` は、どんな `s` でも `Some(value: s)` です。

`to_text` が `Option` を返すのは、すべてのバイト列が Unicode スカラー値の列として正しいわけではないためです。不正な UTF-8 は置換文字などで補わず、`None` として返します。

- `bytes.to_list(bytes: bytes.from_text(text: "aé€😀"))` → `[97, 195, 169, 226, 130, 172, 240, 159, 152, 128]`
- `bytes.to_list(bytes: bytes.from_text(text: ""))` → `[]`
- `bytes.to_text(bytes: bytes.from_text(text: "aé€😀"))` → `Some(value: "aé€😀")`
- `bytes.to_text(bytes: bytes.empty())` → `Some(value: "")`
- `bytes.to_text(bytes: [0])` → `Some(value: "\u{0}")`
- `bytes.to_text(bytes: [239, 187, 191])` → `Some(value: "\u{feff}")`
- `bytes.to_text(bytes: [244, 143, 191, 191])` → `Some(value: "\u{10ffff}")`
- `bytes.to_text(bytes: [255])` → `None`
- `bytes.to_text(bytes: [97, 128])` → `None`
- `bytes.to_text(bytes: [195])` → `None`
- `bytes.to_text(bytes: [226, 130])` → `None`
- `bytes.to_text(bytes: [192, 128])` → `None`
- `bytes.to_text(bytes: [224, 128, 128])` → `None`
- `bytes.to_text(bytes: [240, 128, 128, 128])` → `None`
- `bytes.to_text(bytes: [237, 160, 128])` → `None`
- `bytes.to_text(bytes: [244, 144, 128, 128])` → `None`
- `bytes.to_text(bytes: [245, 128, 128, 128])` → `None`

### bytes.to_hex と bytes.from_hex

```text
pub fn to_hex(bytes: Bytes) -> String
pub fn from_hex(text: String) -> Option<Bytes>
```

`to_hex` は、各バイトを 2 桁の小文字の 16 進数にして連結します。結果の長さは、バイト数の 2 倍です。

`from_hex` は、その逆です。2 桁ずつ読み、数字 `0` から `9`、英字 `a` から `f` と `A` から `F` を受け付けます。次の入力は `None` です。

- 長さが奇数
- 16 進数の数字でない文字を含む（`0x` の接頭辞、空白、区切り文字、ASCII でない文字も含む）

空の文字列は `Some(value: bytes.empty())` です。

- `bytes.to_hex(bytes: [0, 255, 122])` → `"00ff7a"`
- `bytes.to_hex(bytes: bytes.empty())` → `""`
- `bytes.from_hex(text: "00ff7A")` → `Some(value: ...)`（要素が `[0, 255, 122]` の `Bytes`）
- `bytes.from_hex(text: "FF0a")` → `Some(value: ...)`（要素が `[255, 10]` の `Bytes`）
- `bytes.from_hex(text: "")` → `Some(value: bytes.empty())`
- `bytes.from_hex(text: "0")` → `None`
- `bytes.from_hex(text: "abc")` → `None`
- `bytes.from_hex(text: "zz")` → `None`
- `bytes.from_hex(text: "0x")` → `None`
- `bytes.from_hex(text: "ab cd")` → `None`
- `bytes.from_hex(text: "éé")` → `None`

### bytes.to_base64 と bytes.from_base64

```text
pub fn to_base64(bytes: Bytes) -> String
pub fn from_base64(text: String) -> Option<Bytes>
```

標準の Base64（文字は `A` から `Z`、`a` から `z`、`0` から `9`、`+`、`/`）です。3 バイトを 4 文字にし、最後の端数は `=` で 4 文字に揃えます。

`to_base64` は、常に `=` でパディングした形式で出力します。

`from_base64` は、**正規の形だけ**を受け付けます。次の入力は `None` です。

- 長さが 4 の倍数でない（パディングの `=` を省略した形式を含む）
- Base64 の文字でも `=` でもない文字を含む（空白、改行、URL 向けの `-` と `_` を含む）
- `=` が、最後の 4 文字の末尾の 1 つか 2 つ以外の位置にある
- パディング部分の未使用ビットが 0 でない（`to_base64` が出力しない形式）

空の文字列は `Some(value: bytes.empty())` です。

- `bytes.to_base64(bytes: bytes.from_text(text: "hi"))` → `"aGk="`
- `bytes.to_base64(bytes: bytes.from_text(text: "h"))` → `"aA=="`
- `bytes.to_base64(bytes: bytes.from_text(text: "hij"))` → `"aGlq"`
- `bytes.to_base64(bytes: bytes.empty())` → `""`
- `bytes.from_base64(text: "aGk=")` → `Some(value: bytes.from_text(text: "hi"))`
- `bytes.from_base64(text: "aA==")` → `Some(value: bytes.from_text(text: "h"))`
- `bytes.from_base64(text: "aGlq")` → `Some(value: bytes.from_text(text: "hij"))`
- `bytes.from_base64(text: "+/+/")` → `Some(value: ...)`（要素が `[251, 255, 191]` の `Bytes`）
- `bytes.from_base64(text: "")` → `Some(value: bytes.empty())`
- `bytes.from_base64(text: "aGk")` → `None`
- `bytes.from_base64(text: "a")` → `None`
- `bytes.from_base64(text: "aG=k")` → `None`
- `bytes.from_base64(text: "aGl=")` → `None`
- `bytes.from_base64(text: "aB==")` → `None`
- `bytes.from_base64(text: "====")` → `None`
- `bytes.from_base64(text: "aGk=aGlq")` → `None`
- `bytes.from_base64(text: "aGk=aGk=")` → `None`
- `bytes.from_base64(text: "!!!!")` → `None`
- `bytes.from_base64(text: "a-_=")` → `None`
- `bytes.from_base64(text: "aGk=\n")` → `None`

## 例

関数ごとの例です。事前条件に違反する呼び出しは、`defects(of: ...)` で、欠陥になることを確かめます（[16 テスト](../language/16-testing.md)）。`example` は、同じモジュールの関数に付ける決まりなので、各モジュールに小さな関数を置いて、それに付けています。`bytes_of` は、リストから `Bytes` を作る補助の関数です。

作る、読む、加工する、検索する例です。

```cernith
module guide.bytes_basics;

import std.bytes;

pub fn bytes_of(items: List<Int>) -> Bytes {
  match bytes.from_list(items: items) {
    Some(value: found) => found,
    None => bytes.empty(),
  }
}

example bytes_of_builds_from_a_list for bytes_of {
  bytes.to_list(bytes: bytes_of(items: [1, 2])) == [1, 2]
    and bytes.length(bytes: bytes_of(items: [])) == 0
}

example concat_all_joins_every_part for bytes_of {
  bytes.to_list(
    bytes: bytes.concat_all(
      parts: [
        bytes.from_text(text: "a"),
        bytes.empty(),
        bytes.from_text(text: "bc"),
      ],
    ),
  )
    == [97, 98, 99]
    and bytes.length(bytes: bytes.concat_all(parts: [])) == 0
}

example concat_joins_two_sequences for bytes_of {
  bytes.to_list(
    bytes: bytes.concat(
      first: bytes.from_text(text: "a"),
      second: bytes.from_text(text: "bc"),
    ),
  )
    == [97, 98, 99]
}

example empty_has_no_bytes for bytes_of {
  bytes.length(bytes: bytes.empty()) == 0
}

example fill_out_of_range_is_a_defect for bytes_of {
  defects(
    of: bytes.fill(
      bytes: bytes.new(count: 1, fill: 0),
      from: 0,
      to: 2,
      value: 0,
    ),
  )
    and defects(
      of: bytes.fill(
        bytes: bytes.new(count: 1, fill: 0),
        from: 0,
        to: 1,
        value: 256,
      ),
    )
}

example fill_sets_a_range for bytes_of {
  bytes.to_list(
    bytes: bytes.fill(
      bytes: bytes.new(count: 4, fill: 1),
      from: 1,
      to: 3,
      value: 9,
    ),
  )
    == [1, 9, 9, 1]
    and bytes.to_list(
      bytes: bytes.fill(
        bytes: bytes.new(count: 2, fill: 1),
        from: 1,
        to: 1,
        value: 9,
      ),
    )
      == [1, 1]
}

example find_gives_the_first_position for bytes_of {
  bytes.find(
    bytes: bytes.from_text(text: "abcabc"),
    pattern: bytes.from_text(text: "ca"),
  )
    == Some(value: 2)
    and bytes.find(
      bytes: bytes.from_text(text: "abc"),
      pattern: bytes.from_text(text: "x"),
    )
      == None
    and bytes.find(bytes: bytes.from_text(text: "abc"), pattern: bytes.empty())
      == Some(value: 0)
    and bytes.find(bytes: bytes.empty(), pattern: bytes.empty())
      == Some(value: 0)
    and bytes.find(
      bytes: bytes.from_text(text: "a"),
      pattern: bytes.from_text(text: "ab"),
    )
      == None
}

example from_list_accepts_only_byte_values for bytes_of {
  bytes.from_list(items: [0, 255]) == Some(value: bytes_of(items: [0, 255]))
    and bytes.from_list(items: []) == Some(value: bytes.empty())
    and bytes.from_list(items: [256]) == None
    and bytes.from_list(items: [-1]) == None
}

example get_out_of_range_is_a_defect for bytes_of {
  defects(of: bytes.get(bytes: bytes.from_text(text: "a"), index: 1))
    and defects(of: bytes.get(bytes: bytes.from_text(text: "a"), index: -1))
}

example get_reads_one_byte for bytes_of {
  bytes.get(bytes: bytes.from_text(text: "ab"), index: 1) == 98
}

example index_of_gives_the_first_position for bytes_of {
  bytes.index_of(bytes: bytes.from_text(text: "abca"), value: 99)
    == Some(value: 2)
    and bytes.index_of(bytes: bytes.from_text(text: "ab"), value: 0) == None
    and bytes.index_of(bytes: bytes.from_text(text: "ab"), value: 256) == None
    and bytes.index_of(bytes: bytes.empty(), value: 0) == None
}

example length_counts_bytes_not_characters for bytes_of {
  bytes.length(bytes: bytes.from_text(text: "é")) == 2
}

example new_makes_equal_bytes for bytes_of {
  bytes.to_list(bytes: bytes.new(count: 3, fill: 7)) == [7, 7, 7]
    and bytes.length(bytes: bytes.new(count: 0, fill: 0)) == 0
}

example new_out_of_range_is_a_defect for bytes_of {
  defects(of: bytes.new(count: -1, fill: 0))
    and defects(of: bytes.new(count: 1073741824, fill: 0))
    and defects(of: bytes.new(count: 1, fill: 256))
}

example set_out_of_range_is_a_defect for bytes_of {
  defects(
    of: bytes.set(bytes: bytes.new(count: 1, fill: 0), index: 1, value: 0),
  )
    and defects(
      of: bytes.set(bytes: bytes.new(count: 1, fill: 0), index: 0, value: 256),
    )
    and defects(
      of: bytes.set(bytes: bytes.new(count: 1, fill: 0), index: 0, value: -1),
    )
}

example slice_out_of_range_is_a_defect for bytes_of {
  defects(of: bytes.slice(bytes: bytes.from_text(text: "a"), from: 1, to: 0))
    and defects(
      of: bytes.slice(bytes: bytes.from_text(text: "a"), from: 0, to: 2),
    )
}

example slice_takes_a_range for bytes_of {
  bytes.to_list(
    bytes: bytes.slice(bytes: bytes.from_text(text: "abcd"), from: 1, to: 3),
  )
    == [98, 99]
    and bytes.length(
      bytes: bytes.slice(bytes: bytes.from_text(text: "ab"), from: 2, to: 2),
    )
      == 0
}

example starts_with_compares_the_front for bytes_of {
  bytes.starts_with(
    bytes: bytes.from_text(text: "abc"),
    prefix: bytes.from_text(text: "ab"),
  )
    and not bytes.starts_with(
      bytes: bytes.from_text(text: "abc"),
      prefix: bytes.from_text(text: "bc"),
    )
    and bytes.starts_with(bytes: bytes.empty(), prefix: bytes.empty())
}

example to_list_gives_the_values for bytes_of {
  bytes.to_list(bytes: bytes.from_text(text: "A")) == [65]
}

pub fn set_keeps_the_original() -> Bool {
  let before = bytes.new(count: 3, fill: 0);
  let after = bytes.set(bytes: before, index: 1, value: 9);
  bytes.to_list(bytes: before) == [0, 0, 0]
    and bytes.to_list(bytes: after) == [0, 9, 0]
}

example set_gives_a_new_sequence for set_keeps_the_original {
  set_keeps_the_original()
}
```

整数の読み書きの例です。`encode_header` は、1 バイトのバージョン番号と 2 バイトの長さ（ビッグエンディアン）からなるヘッダーを作り、`read_length` はその長さを読み戻します。

```cernith
module guide.bytes_numbers;

import std.bytes;

pub fn bytes_of(items: List<Int>) -> Bytes {
  match bytes.from_list(items: items) {
    Some(value: found) => found,
    None => bytes.empty(),
  }
}

pub fn encode_header(version: Int, length: Int) -> List<Int> {
  requires version >= 0 and version <= 255;
  requires length >= 0 and length <= 65535;

  bytes.uint_bytes(value: version, width: 1, big_endian: true)
    ++ bytes.uint_bytes(value: length, width: 2, big_endian: true)
}

example encode_header_cases for encode_header {
  encode_header(version: 1, length: 258) == [1, 1, 2]
    and encode_header(version: 0, length: 0) == [0, 0, 0]
}

example encode_header_out_of_range_is_a_defect for encode_header {
  defects(of: encode_header(version: 256, length: 0))
    and defects(of: encode_header(version: 0, length: 65536))
}

example int_bytes_out_of_range_is_a_defect for encode_header {
  defects(of: bytes.int_bytes(value: 128, width: 1, big_endian: true))
    and defects(of: bytes.int_bytes(value: -129, width: 1, big_endian: true))
    and defects(of: bytes.int_bytes(value: 0, width: 0, big_endian: true))
}

example int_bytes_writes_twos_complement for encode_header {
  bytes.int_bytes(value: -2, width: 2, big_endian: false) == [254, 255]
    and bytes.int_bytes(value: -2, width: 2, big_endian: true) == [255, 254]
    and bytes.int_bytes(value: 127, width: 1, big_endian: true) == [127]
    and bytes.int_bytes(value: -128, width: 1, big_endian: true) == [128]
    and bytes.int_bytes(value: -1, width: 3, big_endian: false)
      == [255, 255, 255]
    and bytes.int_bytes(value: -9223372036854775808, width: 8, big_endian: true)
      == [128, 0, 0, 0, 0, 0, 0, 0]
}

example uint_bytes_out_of_range_is_a_defect for encode_header {
  defects(of: bytes.uint_bytes(value: 256, width: 1, big_endian: true))
    and defects(of: bytes.uint_bytes(value: -1, width: 1, big_endian: true))
    and defects(of: bytes.uint_bytes(value: 0, width: 9, big_endian: true))
}

example uint_bytes_writes_either_order for encode_header {
  bytes.uint_bytes(value: 16909060, width: 4, big_endian: true) == [1, 2, 3, 4]
    and bytes.uint_bytes(value: 16909060, width: 4, big_endian: false)
      == [4, 3, 2, 1]
    and bytes.uint_bytes(value: 258, width: 2, big_endian: false) == [2, 1]
    and bytes.uint_bytes(value: 0, width: 3, big_endian: true) == [0, 0, 0]
    and bytes.uint_bytes(
      value: 18446744073709551615,
      width: 8,
      big_endian: true,
    )
      == [255, 255, 255, 255, 255, 255, 255, 255]
}

pub fn read_length(frame: Bytes) -> Int {
  requires bytes.length(bytes: frame) >= 3;

  bytes.read_uint(bytes: frame, at: 1, width: 2, big_endian: true)
}

example read_int_reads_twos_complement for read_length {
  bytes.read_int(
    bytes: bytes_of(items: [254, 255]),
    at: 0,
    width: 2,
    big_endian: false,
  )
    == -2
    and bytes.read_int(
      bytes: bytes_of(items: [255, 254]),
      at: 0,
      width: 2,
      big_endian: true,
    )
      == -2
    and bytes.read_int(
      bytes: bytes_of(items: [128]),
      at: 0,
      width: 1,
      big_endian: true,
    )
      == -128
    and bytes.read_int(
      bytes: bytes_of(items: [127]),
      at: 0,
      width: 1,
      big_endian: true,
    )
      == 127
    and bytes.read_int(
      bytes: bytes_of(items: [128, 0, 0, 0, 0, 0, 0, 0]),
      at: 0,
      width: 8,
      big_endian: true,
    )
      == -9223372036854775808
}

example read_int_with_a_bad_width_is_a_defect for read_length {
  defects(
    of: bytes.read_int(
      bytes: bytes_of(items: [1, 2]),
      at: 0,
      width: 9,
      big_endian: true,
    ),
  )
}

example read_length_reads_the_header for read_length {
  read_length(frame: bytes_of(items: [1, 1, 2])) == 258
    and read_length(
      frame: bytes_of(items: encode_header(version: 7, length: 65535)),
    )
      == 65535
}

example read_uint_out_of_range_is_a_defect for read_length {
  defects(
    of: bytes.read_uint(
      bytes: bytes_of(items: [1, 2]),
      at: 1,
      width: 2,
      big_endian: true,
    ),
  )
    and defects(
      of: bytes.read_uint(
        bytes: bytes_of(items: [1, 2]),
        at: -1,
        width: 1,
        big_endian: true,
      ),
    )
    and defects(
      of: bytes.read_uint(
        bytes: bytes_of(items: [1, 2]),
        at: 0,
        width: 0,
        big_endian: true,
      ),
    )
}

example read_uint_reads_either_order for read_length {
  bytes.read_uint(
    bytes: bytes_of(items: [1, 2, 3, 4]),
    at: 0,
    width: 4,
    big_endian: true,
  )
    == 16909060
    and bytes.read_uint(
      bytes: bytes_of(items: [1, 2, 3, 4]),
      at: 0,
      width: 4,
      big_endian: false,
    )
      == 67305985
    and bytes.read_uint(
      bytes: bytes_of(items: [1, 2, 3]),
      at: 1,
      width: 2,
      big_endian: true,
    )
      == 515
    and bytes.read_uint(
      bytes: bytes_of(items: [9, 255, 255, 255, 255, 255, 255, 255, 255]),
      at: 1,
      width: 8,
      big_endian: true,
    )
      == 18446744073709551615
    and bytes.read_uint(
      bytes: bytes_of(items: [1, 2, 3, 4, 5, 6, 7, 8]),
      at: 0,
      width: 8,
      big_endian: false,
    )
      == 578437695752307201
}
```

文字列との変換の例です。

```cernith
module guide.bytes_text;

import std.bytes;

pub fn bytes_of(items: List<Int>) -> Bytes {
  match bytes.from_list(items: items) {
    Some(value: found) => found,
    None => bytes.empty(),
  }
}

pub fn fingerprint(text: String) -> String {
  bytes.to_hex(bytes: bytes.from_text(text: text))
}

example fingerprint_is_the_hex_of_utf_8 for fingerprint {
  fingerprint(text: "hi") == "6869"
    and fingerprint(text: "é") == "c3a9"
    and fingerprint(text: "") == ""
}

example from_base64_reads_canonical_text_only for fingerprint {
  bytes.from_base64(text: "aGk=") == Some(value: bytes.from_text(text: "hi"))
    and bytes.from_base64(text: "aA==")
      == Some(value: bytes.from_text(text: "h"))
    and bytes.from_base64(text: "aGlq")
      == Some(value: bytes.from_text(text: "hij"))
    and bytes.from_base64(text: "+/+/")
      == Some(value: bytes_of(items: [251, 255, 191]))
    and bytes.from_base64(text: "") == Some(value: bytes.empty())
}

example from_base64_refuses_everything_else for fingerprint {
  bytes.from_base64(text: "aGk") == None
    and bytes.from_base64(text: "a") == None
    and bytes.from_base64(text: "aG=k") == None
    and bytes.from_base64(text: "aGl=") == None
    and bytes.from_base64(text: "aB==") == None
    and bytes.from_base64(text: "====") == None
    and bytes.from_base64(text: "aGk=aGlq") == None
    and bytes.from_base64(text: "aGk=aGk=") == None
    and bytes.from_base64(text: "!!!!") == None
    and bytes.from_base64(text: "a-_=") == None
    and bytes.from_base64(text: "aGk=\n") == None
}

example from_hex_reads_both_cases_and_refuses_the_rest for fingerprint {
  bytes.from_hex(text: "00ff7A") == Some(value: bytes_of(items: [0, 255, 122]))
    and bytes.from_hex(text: "FF0a") == Some(value: bytes_of(items: [255, 10]))
    and bytes.from_hex(text: "") == Some(value: bytes.empty())
    and bytes.from_hex(text: "0") == None
    and bytes.from_hex(text: "abc") == None
    and bytes.from_hex(text: "zz") == None
    and bytes.from_hex(text: "0x") == None
    and bytes.from_hex(text: "ab cd") == None
    and bytes.from_hex(text: "éé") == None
}

example from_text_writes_utf_8 for fingerprint {
  bytes.to_list(bytes: bytes.from_text(text: "aé€😀"))
    == [97, 195, 169, 226, 130, 172, 240, 159, 152, 128]
    and bytes.to_list(bytes: bytes.from_text(text: "")) == []
}

example to_base64_pads_to_four_characters for fingerprint {
  bytes.to_base64(bytes: bytes.from_text(text: "hi")) == "aGk="
    and bytes.to_base64(bytes: bytes.from_text(text: "h")) == "aA=="
    and bytes.to_base64(bytes: bytes.from_text(text: "hij")) == "aGlq"
    and bytes.to_base64(bytes: bytes.empty()) == ""
}

example to_hex_writes_two_lower_case_digits_per_byte for fingerprint {
  bytes.to_hex(bytes: bytes_of(items: [0, 255, 122])) == "00ff7a"
    and bytes.to_hex(bytes: bytes.empty()) == ""
}

example to_text_reads_utf_8 for fingerprint {
  bytes.to_text(bytes: bytes.from_text(text: "aé€😀")) == Some(value: "aé€😀")
    and bytes.to_text(bytes: bytes.empty()) == Some(value: "")
    and bytes.to_text(bytes: bytes_of(items: [0])) == Some(value: "\u{0}")
    and bytes.to_text(bytes: bytes_of(items: [239, 187, 191]))
      == Some(value: "\u{feff}")
    and bytes.to_text(bytes: bytes_of(items: [244, 143, 191, 191]))
      == Some(value: "􏿿")
}

example to_text_refuses_what_is_not_utf_8 for fingerprint {
  bytes.to_text(bytes: bytes_of(items: [255])) == None
    and bytes.to_text(bytes: bytes_of(items: [97, 128])) == None
    and bytes.to_text(bytes: bytes_of(items: [195])) == None
    and bytes.to_text(bytes: bytes_of(items: [226, 130])) == None
    and bytes.to_text(bytes: bytes_of(items: [192, 128])) == None
    and bytes.to_text(bytes: bytes_of(items: [224, 128, 128])) == None
    and bytes.to_text(bytes: bytes_of(items: [240, 128, 128, 128])) == None
    and bytes.to_text(bytes: bytes_of(items: [237, 160, 128])) == None
    and bytes.to_text(bytes: bytes_of(items: [244, 144, 128, 128])) == None
    and bytes.to_text(bytes: bytes_of(items: [245, 128, 128, 128])) == None
}
```

## 関連

- [03 型](../language/03-types.md)：`Bytes` は `List` よりコンパクトに格納される列
- [12 エラーと欠陥](../language/12-errors-and-defects.md)：事前条件の違反は欠陥
- [text](text.md)：`String` の関数（`String` は Unicode スカラー値の列）
- [floats と ints](floats-ints.md)：`Floats` と `Ints`
- [word32 と word64](word32-word64.md)：固定幅の整数の演算
