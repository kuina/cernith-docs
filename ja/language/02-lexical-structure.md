# 字句

ソースファイルは BOM なしの UTF-8 で書きます。字句解析器は左から右へ読み進め、最長一致でトークンを切り出します。空白はトークンの区切りとしてのみ使われます。ただし、文字列リテラルの外のタブ文字と復帰文字（CRLF の改行を含む）は字句エラー（`E0001`）になります。インデントには空白を、改行には LF を使ってください。この範囲であれば、改行やインデントの違いは構文木に影響しません。書式の規則は別に定められています（[19 正準形](19-canonical-form.md)）。

字句の誤りは、すべてコード `E0001`（syntax-error）で報告されます。

## 識別子

| 種類 | 形 | 使う場所 |
| --- | --- | --- |
| 小文字の名前 | `[a-z][a-z0-9]*(_[a-z0-9]+)*` | 値、関数、フィールド、パラメータ、ラベル、モジュール、変数 |
| 大文字の名前 | `[A-Z][A-Za-z0-9]*` | 型、レコード、バリアントとそのケース、効果パラメータ |

小文字で始まる名前は snake_case で書き、ASCII の英小文字と数字を使います。`_` は単語の区切りに使えますが、先頭や末尾に置いたり、連続させたりすることはできません。大文字で始まる名前は PascalCase で書き、`_` は使えません。この区別により、名前から値と型を見分けられます。

```cernith-error E0001
module guide.names;

pub fn My_fn() -> Int {
  1
}
```

キーワード（後述）は名前に使えません。

```cernith-error E0001
module guide.names;

pub fn f(unit: Int) -> Int {
  unit
}
```

名前 `_`（アンダースコア 1 つ）は、パターンとループ変数で「使わない値」を表します（[05 文と束縛](05-statements.md)、[07 レコードとバリアント](07-records-and-variants.md)）。

## リテラル

| 種類 | 書き方 | 例 |
| --- | --- | --- |
| 整数（`Int`） | `0`、または `0` 以外で始まる 10 進数字列 | `0`、`42`、`12345678901234567890` |
| 浮動小数点数（`Float64`） | 整数部、`.`、1 桁以上の小数部、省略可能な指数部 | `1.5`、`0.25`、`1.0e21`、`2.5E-3` |
| 文字列（`String`） | `"` で囲む | `"héllo"` |
| 真偽値（`Bool`） | `true`、`false` | |
| ユニット（`Unit`） | `()` | |

- 整数は 10 進数だけです。桁区切り、先頭の `0`、接尾辞は使えません。負の数は、単項の `-` を付けて書きます（`-7`）。`Int` の桁数に上限はありません。
- 浮動小数点数のリテラルには、`.` と 1 桁以上の小数部が必要です。`1e5`、`1.`、`.5` はリテラルとして認められません。指数部には `e` または `E` に続けて、省略可能な符号と数字列を書きます。値が `Float64` の範囲を超えるリテラル（たとえば `1.0e400`）は、エラー `E0001` になります。
- 文字列の中では、`\\`、`\"`、`\n`、`\t`、`\r`、`\u{16 進数}` のエスケープが使えます。ほかの `\` の使い方は誤りです。文字列リテラルの中に、改行をそのまま書くことはできません（改行は `\n`）。それ以外の Unicode スカラー値は、そのまま書けます。

```cernith
module guide.literals;

pub fn answer() -> Int {
  42
}

example answer_is_42 for answer {
  answer() == 42
}

pub fn big() -> Int {
  12345678901234567890
}

example big_is_big for big {
  big() == 12345678901234567890
}

pub fn greeting() -> String {
  "tab:\t quote:\" backslash:\\ héllo"
}

example greeting_length for greeting {
  greeting().length == 31
}

pub fn nothing() -> Unit {
  ()
}

example nothing_is_unit for nothing {
  nothing() == ()
}

pub fn ratio() -> Float64 {
  1500.0
}

example ratio_is_1500 for ratio {
  ratio() == 1500.0
}
```

次の例は、いずれも字句の誤りとして拒否されます。

```cernith-error E0001
module guide.literals;

pub fn f() -> Float64 {
  1e5
}
```

```cernith-error E0001
module guide.literals;

pub fn f() -> Int {
  007
}
```

```cernith-error E0001
module guide.literals;

pub fn f() -> String {
  "a\qb"
}
```

> 補足：浮動小数点数は有限の値だけを扱います。無限大や NaN になる演算は欠陥です（[03 型](03-types.md)）。

## コメント

コメントは `//` から行末までです。ブロックコメントはありません。コメントはプログラムの動作に影響しません。

コメントを書ける位置は、定義・`import`・文・節・フィールド・バリアントのケース・`match` のケースの直前と、ブロックの末尾、ファイルの末尾です。式の途中やパラメータリストの途中には書けません。コード行の末尾に書いたコメントは、正準形では次に続く要素の直前の行へ移されます（[19 正準形](19-canonical-form.md)）。

```cernith
module guide.comments;

// 2 倍にする。
pub fn double(value: Int) -> Int {
  // 加算で書く。
  value + value
}

example doubles for double {
  double(value: 4) == 8
}
```

## キーワード

次の語はキーワードで、名前には使えません。

```text
module import as pub type distinct where record variant const fn spec
foreign unit uses requires ensures decreases invariant let var return
discard assert if else match for in while example property and or not
implies forall exists true false self effect resource release using
await capability manifest
```

`unit` は単位を宣言します（[23 単位](23-units-of-measure.md)）。`interface`、`impl`、`law`、`derive` はキーワードではなく、項目やインターフェースのメンバーを記述する位置、または `impl` ヘッダーの末尾でのみ特別な意味を持ちます。`self` は、型の `where` 節の中でのみ使えます（[09 契約](09-contracts.md)）。`async` と `scope` もキーワードではありません。`async` は効果の名前（`uses async`）、`scope` は `std.async` の関数名として使われます（[18 並行処理](18-concurrency.md)）。

`result` もキーワードではありません。`ensures` 節の中でだけ、関数の戻り値を表します。それ以外の場所では、普通の名前です（局所変数を `result` と呼んでも構いません。ただし `ensures` 節の中の `result` は、常に戻り値です）。

## 記号

```text
( ) [ ] { } , : ; . .. ..= :: -> => = == != < <= > >= ++ + - * / ? | ^
```

`>` が型引数を閉じ、直後に `=` が続くと、`>=` と読まれます。`var xs: List<Int> = [];` のように、`>` と `=` の間に空白を置いてください。正準形では、常に空白が入ります。

```cernith-error E0001
module guide.wart;

pub fn f() -> Int {
  var xs: List<Int>= [];
  xs.length
}
```

## プレリュード

次の名前は、`import` なしで常に使えます。キーワードではありません。

| 種類 | 名前 |
| --- | --- |
| 型 | `Int`、`Float64`、`Bool`、`String`、`Unit`、`List`、`Option`、`Result`、`Bytes`、`Floats`、`Ints` |
| バリアントのケース | `Ok`、`Err`、`Some`、`None` |
| 組み込み関数 | `hole` |

`hole(reason: "説明")` は、未完成のコードを示す式です。どの型の値が求められる場所にも書けます。チェッカーは警告 `W0701`（hole-present）を出し、実行時にこの式に到達すると欠陥になります（[12 エラーと欠陥](12-errors-and-defects.md)）。

プレリュードにあるのは、リテラル・`?`・`match`・演算子が直接参照する型だけです。`Map` や `Set` など、それ以外の型は標準ライブラリのモジュールから、モジュール名を付けて使います（`map.Map<String, Int>`）。

## 関連

- 識別子の命名規則に反したときの診断は、[20 診断コード](20-diagnostics.md) の `E0001` にあります。
- 型の一覧は [03 型](03-types.md) にあります。
