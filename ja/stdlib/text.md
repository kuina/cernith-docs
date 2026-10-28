# std.text

`String` の操作を提供するモジュールです。`import std.text;` で読み込み、`text.split(of: ..., separator: ...)` のように呼び出します。分割結果を表すレコード型 `text.SplitPair` も定義されています。

`String` は Unicode スカラー値の列です（[03 型](../language/03-types.md)）。このモジュールの位置、長さ、個数は、すべてスカラー値で数えます。バイト数でも UTF-16 の単位数でも、人が 1 文字と感じる単位（書記素）でもありません。たとえば `"a😀b"` の長さは 3 で、`text.index_of(of: "a😀b", part: "b")` は `Some(value: 2)` です。結合文字を使った `"e\u{301}"`（`e` と U+0301）は、見た目が 1 文字でも長さ 2 です。

文字を表す専用の型はありません。1 文字は長さ 1 の `String` です。このモジュールの「文字」は、すべて長さ 1 の `String` を指します。

すべての関数は純粋で、効果を持ちません。事前条件（`requires`）を満たさない呼び出しは欠陥です（[12 エラーと欠陥](../language/12-errors-and-defects.md)）。事前条件を持つのは、`code_point`、`from_code_point`、`slice`、`count`、`split`、`split_once`、`replace`、`repeat`、`pad_start`、`pad_end` です。ほかの関数は、どんな引数でも欠陥になりません。

## 関数一覧

| 関数 | 戻り値の型 | 意味 |
| --- | --- | --- |
| `text.is_empty(of)` | `Bool` | 長さ 0 なら `true` |
| `text.chars(of)` | `List<String>` | スカラー値ごとに分けた、長さ 1 の文字列のリスト |
| `text.code_point(of)` | `Int` | 長さ 1 の文字列のコードポイント |
| `text.from_code_point(value)` | `String` | コードポイントから長さ 1 の文字列を作る |
| `text.slice(of, from, to)` | `String` | 位置 `from` 以上 `to` 未満の部分 |
| `text.index_of(of, part)` | `Option<Int>` | `part` が最初に現れる位置 |
| `text.contains(of, part)` | `Bool` | `part` を含むなら `true` |
| `text.starts_with(of, prefix)` | `Bool` | `prefix` で始まるなら `true` |
| `text.ends_with(of, suffix)` | `Bool` | `suffix` で終わるなら `true` |
| `text.count(of, part)` | `Int` | `part` が重ならずに現れる回数 |
| `text.split(of, separator)` | `List<String>` | `separator` で区切った部分のリスト |
| `text.split_once(of, separator)` | `Option<SplitPair>` | 最初の `separator` の前と後の 2 つに分ける |
| `text.join(parts, separator)` | `String` | `separator` を挟んで連結する |
| `text.lines(of)` | `List<String>` | 行のリスト |
| `text.replace(of, target, replacement)` | `String` | `target` をすべて `replacement` に置き換える |
| `text.repeat(of, count)` | `String` | `count` 回繰り返す |
| `text.pad_start(of, width, fill)` | `String` | 長さ `width` になるまで左に `fill` を足す |
| `text.pad_end(of, width, fill)` | `String` | 長さ `width` になるまで右に `fill` を足す |
| `text.trim(of)` | `String` | 両端の空白を取り除く |
| `text.trim_start(of)` | `String` | 先頭の空白を取り除く |
| `text.trim_end(of)` | `String` | 末尾の空白を取り除く |
| `text.is_space(of)` | `Bool` | 空白 1 文字なら `true` |
| `text.to_lower(of)` | `String` | ASCII の英大文字を小文字にする |
| `text.to_upper(of)` | `String` | ASCII の英小文字を大文字にする |

型は次の 1 つです。

```text
pub record SplitPair {
  before: String,
  after: String,
}
```

`text.split_once` の結果を表す型で、区切りの前（`before`）と後（`after`）を保持します。Cernith にはタプルがないため、2 つの値をレコードにまとめています（[03 型](../language/03-types.md)）。

以降の例の `→` の右は、呼び出しの値です。`→ 欠陥` は、その呼び出しが事前条件に違反して欠陥になることを表します。これらは、末尾の「例」の `example` で確かめてあります。

## 長さと文字

### text.is_empty

```text
pub fn is_empty(of: String) -> Bool
```

長さが 0 の文字列（`""`）のときだけ `true` です。空白だけの文字列は空ではありません。

- `text.is_empty(of: "")` → `true`
- `text.is_empty(of: " ")` → `false`

### text.chars

```text
pub fn chars(of: String) -> List<String>
```

文字列をスカラー値ごとに分けます。結果の長さは `of.length` で、`i` 番目の要素は `text.slice(of: of, from: i, to: i + 1)` です。`text.join(parts: text.chars(of: of), separator: "")` は `of` に戻ります。

- `text.chars(of: "ab")` → `["a", "b"]`
- `text.chars(of: "")` → `[]`
- `text.chars(of: "a😀")` → `["a", "😀"]`
- `text.chars(of: "e\u{301}").length` → `2`

### text.code_point

```text
pub fn code_point(of: String) -> Int
```

長さ 1 の文字列の Unicode コードポイントを `Int` で返します。

事前条件：`of.length == 1`。長さが 0 や 2 以上の文字列を渡すと欠陥です。

事後条件：結果は 0 以上 1114111（U+10FFFF）以下で、サロゲート（55296 から 57343、U+D800 から U+DFFF）ではありません。`String` はスカラー値の列なので、サロゲートの文字列はそもそも作れません。

- `text.code_point(of: "A")` → `65`
- `text.code_point(of: "😀")` → `128512`
- `text.code_point(of: "\n")` → `10`
- `text.code_point(of: "")` → 欠陥
- `text.code_point(of: "ab")` → 欠陥

### text.from_code_point

```text
pub fn from_code_point(value: Int) -> String
```

コードポイントから、長さ 1 の文字列を作ります。`text.code_point` の逆です。

事前条件：`value >= 0 and value <= 1114111 and not (value >= 55296 and value <= 57343)`。範囲の外の値と、サロゲートの値を渡すと欠陥です。

事後条件：結果の長さは 1 です。

スカラー値でない値は `String` に含められないため、この関数では呼び出し側に範囲の保証を求めます。そのため、`Option` を返す代わりに事前条件を設けています。

- `text.from_code_point(value: 97)` → `"a"`
- `text.from_code_point(value: 128512)` → `"😀"`
- `text.from_code_point(value: 0).length` → `1`
- `text.from_code_point(value: 55295).length` → `1`（サロゲートの直前）
- `text.from_code_point(value: 57344).length` → `1`（サロゲートの直後）
- `text.from_code_point(value: 1114111).length` → `1`（最大）
- `text.from_code_point(value: 55296)` → 欠陥
- `text.from_code_point(value: 57343)` → 欠陥
- `text.from_code_point(value: 1114112)` → 欠陥
- `text.from_code_point(value: -1)` → 欠陥

## 部分文字列と検索

### text.slice

```text
pub fn slice(of: String, from: Int, to: Int) -> String
```

位置 `from` から `to` の手前までの部分を返します。位置は 0 から数えたスカラー値の個数で、`from` は含み、`to` は含みません。

事前条件：`0 <= from and from <= to and to <= of.length`。負の位置や、末尾を超える位置は欠陥です。後ろから数える位置はありません。

事後条件：結果の長さは `to - from` です。

- `text.slice(of: "a😀bc", from: 1, to: 3)` → `"😀b"`
- `text.slice(of: "abc", from: 0, to: 3)` → `"abc"`
- `text.slice(of: "abc", from: 0, to: 0)` → `""`
- `text.slice(of: "abc", from: 3, to: 3)` → `""`
- `text.slice(of: "abc", from: 2, to: 1)` → 欠陥
- `text.slice(of: "abc", from: -1, to: 2)` → 欠陥
- `text.slice(of: "abc", from: 1, to: 4)` → 欠陥

### text.index_of

```text
pub fn index_of(of: String, part: String) -> Option<Int>
```

`part` が最初に現れる位置を `Some(value: 位置)` で返します。現れなければ `None` です。位置はスカラー値で数えます。

境界条件：`part` が空の文字列なら、`of` が空でも `Some(value: 0)` です。`part` が `of` より長ければ `None` です。

事後条件：`Some(value: position)` のとき、`position >= 0`、`position + part.length <= of.length`、`text.slice(of: of, from: position, to: position + part.length) == part` です。

見つからない場合も通常の結果として扱い、`-1` などの番兵値ではなく `Option` で表します。

- `text.index_of(of: "banana", part: "an")` → `Some(value: 1)`
- `text.index_of(of: "banana", part: "x")` → `None`
- `text.index_of(of: "a😀b", part: "b")` → `Some(value: 2)`
- `text.index_of(of: "abc", part: "")` → `Some(value: 0)`
- `text.index_of(of: "", part: "")` → `Some(value: 0)`
- `text.index_of(of: "a", part: "ab")` → `None`

### text.contains、text.starts_with、text.ends_with

```text
pub fn contains(of: String, part: String) -> Bool
pub fn starts_with(of: String, prefix: String) -> Bool
pub fn ends_with(of: String, suffix: String) -> Bool
```

それぞれ、`part` を含む、`prefix` で始まる、`suffix` で終わる、のときに `true` です。`contains` は `text.index_of` が `Some` を返すことと同じです。`starts_with` は `index_of` が `Some(value: 0)` を返すことと同じです。

境界条件：`part`、`prefix`、`suffix` が空の文字列なら、`of` が何であっても（空でも）`true` です。引数が `of` より長ければ `false` です。

- `text.contains(of: "banana", part: "nan")` → `true`
- `text.contains(of: "banana", part: "nab")` → `false`
- `text.contains(of: "abc", part: "")` → `true`
- `text.contains(of: "", part: "a")` → `false`
- `text.starts_with(of: "report.txt", prefix: "rep")` → `true`
- `text.starts_with(of: "report.txt", prefix: "ort")` → `false`
- `text.starts_with(of: "a", prefix: "")` → `true`
- `text.starts_with(of: "a", prefix: "ab")` → `false`
- `text.ends_with(of: "report.txt", suffix: ".txt")` → `true`
- `text.ends_with(of: "report.txt", suffix: ".md")` → `false`
- `text.ends_with(of: "a", suffix: "")` → `true`
- `text.ends_with(of: "a", suffix: "ba")` → `false`

### text.count

```text
pub fn count(of: String, part: String) -> Int
```

`of` に `part` が現れる回数を返します。左から検索し、一致した部分の直後から再開するため、重なり合う一致は数えません。

事前条件：`not text.is_empty(of: part)`。空の `part` を渡すと欠陥です（空の文字列は、どの位置にも現れるため、回数が決まりません）。

事後条件：結果は 0 以上です。

- `text.count(of: "a-b-c", part: "-")` → `2`
- `text.count(of: "aaa", part: "aa")` → `1`
- `text.count(of: "aaaa", part: "aa")` → `2`
- `text.count(of: "abc", part: "x")` → `0`
- `text.count(of: "", part: "a")` → `0`
- `text.count(of: "a😀a", part: "a")` → `2`
- `text.count(of: "abc", part: "")` → 欠陥

## 分割と結合

### text.split

```text
pub fn split(of: String, separator: String) -> List<String>
```

`separator` で分割した文字列のリストを返します。左から検索し、一致した部分の直後から再開するため、重なり合う一致は区切りとして使いません。区切りの前後が空の場合も、空文字列として結果に残ります。

事前条件：`not text.is_empty(of: separator)`。空の `separator` を渡すと欠陥です。1 文字ずつに分けたいときは `text.chars` を使います。

事後条件：結果の長さは 1 以上で、`text.join(parts: result, separator: separator)` は `of` に戻ります。

境界条件：`of` が空の文字列なら、結果は空の文字列 1 つのリスト（`[""]`）です。空のリストにはなりません。区切りが連続すれば、その間に空の文字列が入ります。区切りが先頭や末尾にあれば、先頭や末尾に空の文字列が入ります。

- `text.split(of: "a,b,,c", separator: ",")` → `["a", "b", "", "c"]`
- `text.split(of: "abc", separator: ",")` → `["abc"]`
- `text.split(of: "", separator: ",")` → `[""]`
- `text.split(of: ",", separator: ",")` → `["", ""]`
- `text.split(of: ",,", separator: ",")` → `["", "", ""]`
- `text.split(of: "a,", separator: ",")` → `["a", ""]`
- `text.split(of: "aaa", separator: "aa")` → `["", "a"]`
- `text.split(of: "a--b---c", separator: "--")` → `["a", "b", "-c"]`
- `text.split(of: "abc", separator: "")` → 欠陥

### text.split_once

```text
pub fn split_once(of: String, separator: String) -> Option<SplitPair>
```

最初に現れる `separator` の前と後で、2 つに分けます。`Some(value: SplitPair(before: ..., after: ...))` を返し、`separator` が現れなければ `None` です。2 つ目以降の `separator` は、`after` の中にそのまま残ります。

事前条件：`not text.is_empty(of: separator)`。空の `separator` を渡すと欠陥です。

境界条件：`before` と `after` は、空の文字列になりえます。`of` が空の文字列なら `None` です。

`key=value` のように、最初の区切りで 2 つに分けたいときに使います。値の中に `=` が含まれていても、`after` にそのまま残ります。

- `text.split_once(of: "k=v=w", separator: "=")` → `Some(value: text.SplitPair(before: "k", after: "v=w"))`
- `text.split_once(of: "kv", separator: "=")` → `None`
- `text.split_once(of: "", separator: "=")` → `None`
- `text.split_once(of: "=", separator: "=")` → `Some(value: text.SplitPair(before: "", after: ""))`
- `text.split_once(of: "k=", separator: "=")` → `Some(value: text.SplitPair(before: "k", after: ""))`
- `text.split_once(of: "a==b", separator: "==")` → `Some(value: text.SplitPair(before: "a", after: "b"))`
- `text.split_once(of: "abc", separator: "")` → 欠陥

### text.join

```text
pub fn join(parts: List<String>, separator: String) -> String
```

`parts` の要素を、間に `separator` を挟んで連結します。`separator` は要素の間にだけ入り、先頭と末尾には付きません。`text.split` の逆です。事前条件はありません。

境界条件：`parts` が空のリストなら `""`、要素が 1 つなら、その要素です。`separator` は空の文字列でもかまいません。

- `text.join(parts: ["a", "b", "c"], separator: ", ")` → `"a, b, c"`
- `text.join(parts: [], separator: ",")` → `""`
- `text.join(parts: ["x"], separator: ",")` → `"x"`
- `text.join(parts: ["", ""], separator: "-")` → `"-"`
- `text.join(parts: ["a", "b"], separator: "")` → `"ab"`

### text.lines

```text
pub fn lines(of: String) -> List<String>
```

文字列を、次の 3 段階で行に分割します。

1. `"\n"` で `text.split` する。
2. 最後の部分が空の文字列なら、その 1 つを取り除く。
3. 各部分の末尾に `"\r"` があれば、その 1 つを取り除く。

最後の行が `"\n"` で終わっていても、その後ろに空の行はできません。手順 3 は各行の末尾だけを見るので、`"\n"` が続かない最後の行の末尾の `"\r"` も取り除かれます。取り除くのは 1 つだけで、行の途中の `"\r"` は残ります。

`text.split` と違って、空の文字列の行は 0 個です（`text.lines(of: "")` は `[]`）。

- `text.lines(of: "a\nb\n")` → `["a", "b"]`
- `text.lines(of: "a\r\nb")` → `["a", "b"]`
- `text.lines(of: "a")` → `["a"]`
- `text.lines(of: "")` → `[]`
- `text.lines(of: "\n")` → `[""]`
- `text.lines(of: "\n\n")` → `["", ""]`
- `text.lines(of: "a\n\nb")` → `["a", "", "b"]`
- `text.lines(of: "a\n\n")` → `["a", ""]`
- `text.lines(of: "a\r")` → `["a"]`
- `text.lines(of: "\r")` → `[""]`
- `text.lines(of: "a\rb")` → `["a\rb"]`
- `text.lines(of: "a\r\r\n")` → `["a\r"]`
- `text.lines(of: "a\n\r")` → `["a", ""]`
- `text.lines(of: " ")` → `[" "]`

### text.replace

```text
pub fn replace(of: String, target: String, replacement: String) -> String
```

`of` 内の `target` をすべて `replacement` に置き換えます。`text.split` と同様に、左から順に重ならない一致だけを対象とします。置換後の文字列を再び検索することはありません。`target` がなければ `of` をそのまま返します。

事前条件：`not text.is_empty(of: target)`。空の `target` を渡すと欠陥です。

- `text.replace(of: "a-b-c", target: "-", replacement: "+")` → `"a+b+c"`
- `text.replace(of: "aaa", target: "aa", replacement: "b")` → `"ba"`
- `text.replace(of: "aaaa", target: "aa", replacement: "")` → `""`
- `text.replace(of: "aaa", target: "a", replacement: "aa")` → `"aaaaaa"`
- `text.replace(of: "abc", target: "x", replacement: "y")` → `"abc"`
- `text.replace(of: "", target: "a", replacement: "b")` → `""`
- `text.replace(of: "abc", target: "", replacement: "x")` → 欠陥

## 整形

### text.repeat

```text
pub fn repeat(of: String, count: Int) -> String
```

`of` を `count` 回連結します。

事前条件：`count >= 0`。負の回数は欠陥です。

事後条件：結果の長さは `of.length * count` です。

- `text.repeat(of: "ab", count: 3)` → `"ababab"`
- `text.repeat(of: "ab", count: 0)` → `""`
- `text.repeat(of: "", count: 5)` → `""`
- `text.repeat(of: "a", count: -1)` → 欠陥

### text.pad_start と text.pad_end

```text
pub fn pad_start(of: String, width: Int, fill: String) -> String
pub fn pad_end(of: String, width: Int, fill: String) -> String
```

`of` の長さが `width` になるまで、`fill` を `pad_start` は左に、`pad_end` は右に足します。`of` の長さがすでに `width` 以上なら、`of` のままです。切り詰めることはありません。`width` が負でもかまいません（`of` のままです）。

事前条件：`fill.length == 1`。長さ 0 や 2 以上の `fill` は欠陥です。

事後条件：`of.length >= width` なら結果は `of` で、`of.length < width` なら結果の長さは `width` です。

- `text.pad_start(of: "7", width: 3, fill: "0")` → `"007"`
- `text.pad_start(of: "1234", width: 3, fill: "0")` → `"1234"`
- `text.pad_start(of: "ab", width: 2, fill: "0")` → `"ab"`
- `text.pad_start(of: "", width: 2, fill: "0")` → `"00"`
- `text.pad_start(of: "ab", width: -3, fill: "0")` → `"ab"`
- `text.pad_start(of: "ab", width: 5, fill: "😀")` → `"😀😀😀ab"`
- `text.pad_start(of: "a", width: 3, fill: "")` → 欠陥
- `text.pad_end(of: "ab", width: 5, fill: ".")` → `"ab..."`
- `text.pad_end(of: "abcdef", width: 3, fill: ".")` → `"abcdef"`
- `text.pad_end(of: "", width: 0, fill: ".")` → `""`
- `text.pad_end(of: "a", width: 3, fill: "ab")` → 欠陥

### text.trim、text.trim_start、text.trim_end

```text
pub fn trim(of: String) -> String
pub fn trim_start(of: String) -> String
pub fn trim_end(of: String) -> String
```

`trim` は両端、`trim_start` は先頭、`trim_end` は末尾の空白を取り除きます。文字列の途中の空白は残ります。空白だけの文字列は `""` になります。

取り除く空白は、次の 4 つの文字だけです。`text.is_space` が `true` を返す文字と同じです。

| 文字 | コードポイント |
| --- | --- |
| 空白（スペース） | U+0020 |
| タブ `\t` | U+0009 |
| 改行 `\n` | U+000A |
| 復帰 `\r` | U+000D |

垂直タブ（U+000B）、改ページ（U+000C）、ノーブレークスペース（U+00A0）、全角スペース（U+3000）など、そのほかの Unicode の空白は取り除きません。

- `text.trim(of: "  a b \t\n")` → `"a b"`
- `text.trim(of: "abc")` → `"abc"`
- `text.trim(of: "   ")` → `""`
- `text.trim(of: "")` → `""`
- `text.trim(of: "\u{a0}a\u{a0}")` → `"\u{a0}a\u{a0}"`
- `text.trim(of: "\u{b}a")` → `"\u{b}a"`
- `text.trim_start(of: " \n a b  ")` → `"a b  "`
- `text.trim_start(of: "   ")` → `""`
- `text.trim_start(of: "x")` → `"x"`
- `text.trim_end(of: "  a b \n")` → `"  a b"`
- `text.trim_end(of: "   ")` → `""`
- `text.trim_end(of: "x")` → `"x"`

### text.is_space

```text
pub fn is_space(of: String) -> Bool
```

`of` が、空白 1 文字（`" "`、`"\t"`、`"\n"`、`"\r"` のどれか）のときだけ `true` です。空の文字列や、2 文字以上の文字列は `false` です。`trim`、`trim_start`、`trim_end` が取り除く文字の基準です。

- `text.is_space(of: " ")` → `true`
- `text.is_space(of: "\t")` → `true`
- `text.is_space(of: "\n")` → `true`
- `text.is_space(of: "\r")` → `true`
- `text.is_space(of: "a")` → `false`
- `text.is_space(of: "")` → `false`
- `text.is_space(of: "  ")` → `false`
- `text.is_space(of: "\u{3000}")` → `false`

### text.to_lower と text.to_upper

```text
pub fn to_lower(of: String) -> String
pub fn to_upper(of: String) -> String
```

`to_lower` は ASCII の英大文字 `A` から `Z` を小文字にし、`to_upper` は ASCII の英小文字 `a` から `z` を大文字にします。それ以外の文字は、変わりません。ロケールの指定も、Unicode の大文字・小文字の対応もありません。`É` と `é`、全角の `Ａ` と `ａ`、`ß` などは、そのままです。結果の長さは `of` と同じです。

大文字・小文字を区別せずに ASCII の文字列を比べるには、両方を `to_lower` にしてから `==` で比べます。

- `text.to_lower(of: "Hello, WORLD 42")` → `"hello, world 42"`
- `text.to_lower(of: "É")` → `"É"`
- `text.to_lower(of: "ＡÉß")` → `"ＡÉß"`
- `text.to_lower(of: "")` → `""`
- `text.to_upper(of: "Hello, world 42")` → `"HELLO, WORLD 42"`
- `text.to_upper(of: "é")` → `"é"`
- `text.to_upper(of: "ß")` → `"ß"`
- `text.to_upper(of: "")` → `""`

## 例

関数ごとの例です。事前条件に違反する呼び出しは、`defects(of: ...)` で、欠陥になることを確かめます（[16 テスト](../language/16-testing.md)）。`example` は、同じモジュールの関数に付ける決まりなので、各モジュールに小さな関数を置いて、それに付けています。

位置と検索の例です。

```cernith
module guide.text_positions;

import std.text;

pub fn last_character(of: String) -> Option<String> {
  if of.length == 0 {
    None
  } else {
    Some(value: text.slice(of: of, from: of.length - 1, to: of.length))
  }
}

example chars_split_into_code_points for last_character {
  text.chars(of: "ab") == ["a", "b"]
    and text.chars(of: "") == []
    and text.chars(of: "a😀") == ["a", "😀"]
    and text.chars(of: "é").length == 2
}

example code_point_names_a_character for last_character {
  text.code_point(of: "A") == 65
    and text.code_point(of: "😀") == 128512
    and text.code_point(of: "\n") == 10
}

example code_point_of_the_wrong_length_is_a_defect for last_character {
  defects(of: text.code_point(of: ""))
    and defects(of: text.code_point(of: "ab"))
}

example contains_present_and_absent for last_character {
  text.contains(of: "banana", part: "nan")
    and not text.contains(of: "banana", part: "nab")
    and text.contains(of: "abc", part: "")
    and not text.contains(of: "", part: "a")
}

example count_counts_separate_occurrences for last_character {
  text.count(of: "a-b-c", part: "-") == 2
    and text.count(of: "aaa", part: "aa") == 1
    and text.count(of: "aaaa", part: "aa") == 2
    and text.count(of: "abc", part: "x") == 0
    and text.count(of: "", part: "a") == 0
    and text.count(of: "a😀a", part: "a") == 2
}

example count_of_an_empty_part_is_a_defect for last_character {
  defects(of: text.count(of: "abc", part: ""))
}

example ends_with_cases for last_character {
  text.ends_with(of: "report.txt", suffix: ".txt")
    and not text.ends_with(of: "report.txt", suffix: ".md")
    and text.ends_with(of: "a", suffix: "")
    and not text.ends_with(of: "a", suffix: "ba")
}

example from_code_point_makes_a_character for last_character {
  text.from_code_point(value: 97) == "a"
    and text.from_code_point(value: 128512) == "😀"
    and text.from_code_point(value: 0).length == 1
    and text.from_code_point(value: 55295).length == 1
    and text.from_code_point(value: 57344).length == 1
    and text.from_code_point(value: 1114111).length == 1
}

example from_code_point_outside_the_scalar_values_is_a_defect for last_character {
  defects(of: text.from_code_point(value: 55296))
    and defects(of: text.from_code_point(value: 57343))
    and defects(of: text.from_code_point(value: 1114112))
    and defects(of: text.from_code_point(value: -1))
}

example index_of_first_occurrence for last_character {
  text.index_of(of: "banana", part: "an") == Some(value: 1)
    and text.index_of(of: "banana", part: "x") == None
    and text.index_of(of: "a😀b", part: "b") == Some(value: 2)
    and text.index_of(of: "abc", part: "") == Some(value: 0)
    and text.index_of(of: "", part: "") == Some(value: 0)
    and text.index_of(of: "a", part: "ab") == None
}

example is_empty_cases for last_character {
  text.is_empty(of: "") and not text.is_empty(of: " ")
}

example last_character_cases for last_character {
  last_character(of: "abc") == Some(value: "c")
    and last_character(of: "a😀") == Some(value: "😀")
    and last_character(of: "") == None
}

example slice_counts_code_points for last_character {
  text.slice(of: "a😀bc", from: 1, to: 3) == "😀b"
    and text.slice(of: "abc", from: 0, to: 3) == "abc"
    and text.slice(of: "abc", from: 0, to: 0) == ""
    and text.slice(of: "abc", from: 3, to: 3) == ""
}

example slice_out_of_range_is_a_defect for last_character {
  defects(of: text.slice(of: "abc", from: 2, to: 1))
    and defects(of: text.slice(of: "abc", from: -1, to: 2))
    and defects(of: text.slice(of: "abc", from: 1, to: 4))
}

example starts_with_cases for last_character {
  text.starts_with(of: "report.txt", prefix: "rep")
    and not text.starts_with(of: "report.txt", prefix: "ort")
    and text.starts_with(of: "a", prefix: "")
    and not text.starts_with(of: "a", prefix: "ab")
}
```

分割、結合、置き換えの例です。

```cernith
module guide.text_split;

import std.text;

pub fn parse_setting(line: String) -> Option<text.SplitPair> {
  text.split_once(of: text.trim(of: line), separator: "=")
}

example join_with_separator for parse_setting {
  text.join(parts: ["a", "b", "c"], separator: ", ") == "a, b, c"
    and text.join(parts: [], separator: ",") == ""
    and text.join(parts: ["x"], separator: ",") == "x"
    and text.join(parts: ["", ""], separator: "-") == "-"
    and text.join(parts: ["a", "b"], separator: "") == "ab"
}

example lines_drop_the_last_line_end for parse_setting {
  text.lines(of: "a\nb\n") == ["a", "b"]
    and text.lines(of: "a\r\nb") == ["a", "b"]
    and text.lines(of: "a") == ["a"]
    and text.lines(of: "") == []
    and text.lines(of: " ") == [" "]
}

example lines_edge_cases for parse_setting {
  text.lines(of: "\n") == [""]
    and text.lines(of: "\n\n") == ["", ""]
    and text.lines(of: "a\n\nb") == ["a", "", "b"]
    and text.lines(of: "a\n\n") == ["a", ""]
    and text.lines(of: "a\r") == ["a"]
    and text.lines(of: "\r") == [""]
    and text.lines(of: "a\rb") == ["a\rb"]
    and text.lines(of: "a\r\r\n") == ["a\r"]
    and text.lines(of: "a\n\r") == ["a", ""]
}

example parse_setting_reads_key_and_value for parse_setting {
  parse_setting(line: "  mode = fast \n")
    == Some(value: text.SplitPair(before: "mode ", after: " fast"))
    and parse_setting(line: "key=a=b")
      == Some(value: text.SplitPair(before: "key", after: "a=b"))
    and parse_setting(line: "novalue") == None
}

example replace_every_occurrence for parse_setting {
  text.replace(of: "a-b-c", target: "-", replacement: "+") == "a+b+c"
    and text.replace(of: "aaa", target: "aa", replacement: "b") == "ba"
    and text.replace(of: "aaaa", target: "aa", replacement: "") == ""
    and text.replace(of: "aaa", target: "a", replacement: "aa") == "aaaaaa"
    and text.replace(of: "abc", target: "x", replacement: "y") == "abc"
    and text.replace(of: "", target: "a", replacement: "b") == ""
}

example replace_of_an_empty_target_is_a_defect for parse_setting {
  defects(of: text.replace(of: "abc", target: "", replacement: "x"))
}

example split_cases for parse_setting {
  text.split(of: "a,b,,c", separator: ",") == ["a", "b", "", "c"]
    and text.split(of: "abc", separator: ",") == ["abc"]
    and text.split(of: "", separator: ",") == [""]
    and text.split(of: ",", separator: ",") == ["", ""]
    and text.split(of: ",,", separator: ",") == ["", "", ""]
    and text.split(of: "a,", separator: ",") == ["a", ""]
    and text.split(of: "aaa", separator: "aa") == ["", "a"]
    and text.split(of: "a--b---c", separator: "--") == ["a", "b", "-c"]
}

example split_of_an_empty_separator_is_a_defect for parse_setting {
  defects(of: text.split(of: "abc", separator: ""))
}

example split_once_at_the_first_separator for parse_setting {
  text.split_once(of: "k=v=w", separator: "=")
    == Some(value: text.SplitPair(before: "k", after: "v=w"))
    and text.split_once(of: "kv", separator: "=") == None
    and text.split_once(of: "", separator: "=") == None
    and text.split_once(of: "=", separator: "=")
      == Some(value: text.SplitPair(before: "", after: ""))
    and text.split_once(of: "k=", separator: "=")
      == Some(value: text.SplitPair(before: "k", after: ""))
    and text.split_once(of: "a==b", separator: "==")
      == Some(value: text.SplitPair(before: "a", after: "b"))
}

example split_once_of_an_empty_separator_is_a_defect for parse_setting {
  defects(of: text.split_once(of: "abc", separator: ""))
}
```

整形の例です。

```cernith
module guide.text_format;

import std.text;

pub fn column(name: String, width: Int) -> String {
  text.pad_end(
    of: text.to_upper(of: text.trim(of: name)),
    width: width,
    fill: ".",
  )
}

example column_cases for column {
  column(name: " id ", width: 5) == "ID..."
    and column(name: "name", width: 2) == "NAME"
}

example is_space_names_ascii_whitespace for column {
  text.is_space(of: " ")
    and text.is_space(of: "\t")
    and text.is_space(of: "\n")
    and text.is_space(of: "\r")
    and not text.is_space(of: "a")
    and not text.is_space(of: "")
    and not text.is_space(of: "  ")
    and not text.is_space(of: "　")
}

example pad_end_fills_on_the_right for column {
  text.pad_end(of: "ab", width: 5, fill: ".") == "ab..."
    and text.pad_end(of: "abcdef", width: 3, fill: ".") == "abcdef"
    and text.pad_end(of: "", width: 0, fill: ".") == ""
}

example pad_end_with_a_bad_fill_is_a_defect for column {
  defects(of: text.pad_end(of: "a", width: 3, fill: "ab"))
}

example pad_start_fills_on_the_left for column {
  text.pad_start(of: "7", width: 3, fill: "0") == "007"
    and text.pad_start(of: "1234", width: 3, fill: "0") == "1234"
    and text.pad_start(of: "ab", width: 2, fill: "0") == "ab"
    and text.pad_start(of: "", width: 2, fill: "0") == "00"
    and text.pad_start(of: "ab", width: -3, fill: "0") == "ab"
    and text.pad_start(of: "ab", width: 5, fill: "😀") == "😀😀😀ab"
}

example pad_start_with_a_bad_fill_is_a_defect for column {
  defects(of: text.pad_start(of: "a", width: 3, fill: ""))
}

example repeat_a_negative_number_of_times_is_a_defect for column {
  defects(of: text.repeat(of: "a", count: -1))
}

example repeat_copies for column {
  text.repeat(of: "ab", count: 3) == "ababab"
    and text.repeat(of: "ab", count: 0) == ""
    and text.repeat(of: "", count: 5) == ""
}

example to_lower_changes_ascii_letters for column {
  text.to_lower(of: "Hello, WORLD 42") == "hello, world 42"
    and text.to_lower(of: "É") == "É"
    and text.to_lower(of: "ＡÉß") == "ＡÉß"
    and text.to_lower(of: "") == ""
}

example to_upper_changes_ascii_letters for column {
  text.to_upper(of: "Hello, world 42") == "HELLO, WORLD 42"
    and text.to_upper(of: "é") == "é"
    and text.to_upper(of: "ß") == "ß"
    and text.to_upper(of: "") == ""
}

example trim_end_removes_trailing_space for column {
  text.trim_end(of: "  a b \n") == "  a b"
    and text.trim_end(of: "   ") == ""
    and text.trim_end(of: "x") == "x"
}

example trim_removes_ascii_whitespace for column {
  text.trim(of: "  a b \t\n") == "a b"
    and text.trim(of: "abc") == "abc"
    and text.trim(of: "   ") == ""
    and text.trim(of: "") == ""
    and text.trim(of: " a ") == " a "
    and text.trim(of: "\u{b}a") == "\u{b}a"
}

example trim_start_removes_leading_space for column {
  text.trim_start(of: " \n a b  ") == "a b  "
    and text.trim_start(of: "   ") == ""
    and text.trim_start(of: "x") == "x"
}
```

## 関連

- [03 型](../language/03-types.md)：`String` は Unicode スカラー値の列
- [12 エラーと欠陥](../language/12-errors-and-defects.md)：事前条件の違反は欠陥
- [list](list.md)：`List<T>` の関数
- [bytes](bytes.md)：`String` と UTF-8 のバイト列の変換（`bytes.from_text`、`bytes.to_text`）
