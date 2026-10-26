# 字句

ソースファイルは、BOM なしの UTF-8 で書きます。字句解析器は、左から右へ、最長一致で記号を切り出します。空白は記号を区切るだけで、意味を持ちません。ただし、文字列リテラルの外のタブ文字と復帰文字（CRLF の改行）は字句の誤り（`E0001`）です。インデントには空白を、行末には LF を使います。構文は書式（改行やインデント）に依存しないので、どんな書式でも文法上は同じ構文木になります。書式の規則は別に定めています（[19 正準形](19-canonical-form.md)）。

字句の誤りは、すべてコード `E0001`（syntax-error）で報告されます。

## 識別子

| 種類 | 形 | 使う場所 |
| --- | --- | --- |
| 小文字の名前 | `[a-z][a-z0-9]*(_[a-z0-9]+)*` | 値、関数、フィールド、パラメータ、ラベル、モジュール、変数 |
| 大文字の名前 | `[A-Z][A-Za-z0-9]*` | 型、レコード、バリアントとそのケース、効果パラメータ |

小文字の名前は snake_case で、ASCII の英小文字と数字だけを使えます。先頭・末尾・連続の `_` は使えません。大文字の名前は PascalCase で、`_` を含められません。この区別によって、名前を見ただけで、それが値なのか型なのかが分かります。

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
- 浮動小数点数は、`.` と小数部の 1 桁以上が必須です。`1e5` や `1.` や `.5` はリテラルではありません。指数部は `e` または `E` のあとに、省略可能な符号と数字列を続けます。値が `Float64` の範囲を超えるリテラル（たとえば `1.0e400`）は `E0001` です。
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

コメントは `//` から行末までです。ブロックコメントはありません。コメントに意味はなく、プログラムの動作に影響しません。

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

`unit` は、今の版では何の機能も持たない予約語です。`self` は、型の `where` 節の中だけに書けます（[09 契約](09-contracts.md)）。`async` と `scope` はキーワードではありません。`async` は効果の名前（`uses async`）、`scope` は `std.async` の関数の名前として使われます（[18 並行処理](18-concurrency.md)）。

`result` もキーワードではありません。`ensures` 節の中でだけ、関数の戻り値を表します。それ以外の場所では、普通の名前です（局所変数を `result` と呼んでも構いません。ただし `ensures` 節の中の `result` は、常に戻り値です）。

## 記号

```text
( ) [ ] { } , : ; . .. ..= :: -> => = == != < <= > >= ++ + - * / ?
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

`hole(reason: "説明")` は、未完成の箇所を表す式です。どんな型の位置にも書けます。チェッカーは警告 `W0701`（hole-present）で知らせ、実行して到達すると欠陥になります（[12 エラーと欠陥](12-errors-and-defects.md)）。

プレリュードにあるのは、リテラル・`?`・`match`・演算子が直接参照する型だけです。`Map` や `Set` など、それ以外の型は標準ライブラリのモジュールから、モジュール名を付けて使います（`map.Map<String, Int>`）。

## 関連

- 識別子の命名規則に反したときの診断は、[20 診断コード](20-diagnostics.md) の `E0001` にあります。
- 型の一覧は [03 型](03-types.md) にあります。
