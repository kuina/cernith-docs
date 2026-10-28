# 式

式には、リテラル、名前、呼び出し、フィールドの読み出し、添字、演算子、`if`、`match`、関数式（ラムダ）、リスト、`?` があります。この章では、`?` を除いて文法と型付けの規則を説明します。`?` は [12 エラーと欠陥](12-errors-and-defects.md) で扱います。

## 呼び出しとラベル

関数の呼び出しでは、**すべての引数にラベルを付け、宣言と同じ順序で並べます**。ラベルは、パラメータの名前です。

```text
関数名(ラベル1: 式1, ラベル2: 式2, ...)
```

ラベルには関数内で使うパラメータ名をそのまま使うため、呼び出し側のコードからも各引数の役割が分かります。

```cernith
module guide.labels;

pub fn area(width: Int, height: Int) -> Int {
  width * height
}

example area_of_3_by_4 for area {
  area(width: 3, height: 4) == 12
}
```

ラベルを省くとエラー `E0304`（missing-argument-label）、ラベルが宣言と一致しない（名前が違う、足りない、余計、順序が違う）とエラー `E0310`（argument-labels-mismatch）です。

```cernith-error E0304
module guide.labels;

pub fn area(width: Int, height: Int) -> Int {
  width * height
}

pub fn use_area() -> Int {
  area(3, 4)
}
```

```cernith-error E0310
module guide.labels;

pub fn area(width: Int, height: Int) -> Int {
  width * height
}

pub fn use_area() -> Int {
  area(height: 4, width: 3)
}
```

同じラベルの規則が、関数のほか、レコードやバリアントのケースの生成（`Rect(width: 3, height: 4)`）、`Type.with(...)`、パターンにも適用されます（[07 レコードとバリアント](07-records-and-variants.md)）。ラベルが不要な呼び出しは、添字 `xs[i]` だけです。

関数型の値を介して呼び出すときのラベルは、**呼び出し側から見える型**のラベルです（[06 関数](06-functions.md)）。

## フィールド、添字、長さ

| 式 | 意味 |
| --- | --- |
| `r.name` | レコード `r` のフィールド `name` |
| `xs[i]` | リスト `xs` の `i` 番目（先頭は 0）。範囲外は欠陥 |
| `xs.length`、`s.length` | `List` や `String` の長さ（0 以上の `Int`） |
| `f()` に続く `.name`、`[i]` | 呼び出し結果にも同じ記法が続けられる（`make().x`） |

## 演算子

| 演算子 | オペランド | 結果 | 備考 |
| --- | --- | --- | --- |
| `+`、`-`、`*` | 2 つの `Int`、または 2 つの `Float64` | 同じ型 | `Int` と `Float64` の混在は `E0302` |
| `/` | 2 つの `Float64` | `Float64` | `Int` どうしは `E0307` |
| 単項 `-` | `Int` または `Float64` | 同じ型 | |
| `==`、`!=` | 等価比較できる同じ型の 2 値 | `Bool` | 境界 `Eq`（[03 型](03-types.md)） |
| `<`、`<=`、`>`、`>=` | 順序のある同じ型の 2 値 | `Bool` | 境界 `Ord` |
| `and`、`or`、`not`、`implies` | `Bool` | `Bool` | `and`、`or`、`implies` は短絡評価 |
| `++` | 2 つの `String`、または同じ型の 2 つの `List` | 同じ型 | 連結 |

`+` で `String` や `List` を連結することはできません。連結は `++` です。

### 優先順位

優先順位が高い順に並べています。同じ行の演算子は同じ優先順位です。

| 強さ | 演算子 | 結合 |
| --- | --- | --- |
| 1（強い） | 後置：`.name`、`(...)`、`[i]`、`?` | 左 |
| 2 | 単項 `-`、`await` | |
| 3 | `*`、`/` | 左 |
| 4 | `+`、`-` | 左 |
| 5 | `++` | 左 |
| 6 | `==`、`!=`、`<`、`<=`、`>`、`>=` | 連鎖不可 |
| 7 | `not` | 前置 |
| 8 | `and` | 左 |
| 9 | `or` | 左 |
| 10（弱い） | `implies` | 右 |

`await` のオペランドの直後に書いた `?` は `await` に付きます。`await e?` は `(await e)?` です（[18 並行処理](18-concurrency.md)）。比較演算子は連鎖できません。`a < b < c` は構文エラー（`E0001`）です。`not` は比較よりも弱いので、`not x == y` は `not (x == y)` と読まれます。`and` は `or` より強いので、`a and b or c` は `(a and b) or c` と読まれます（正準形では、この括弧が補われます。[19 正準形](19-canonical-form.md)）。

```cernith
module guide.operators;

pub fn arithmetic() -> Int {
  1 + 2 * 3 - 4
}

example arithmetic_is_3 for arithmetic {
  arithmetic() == 3
}

pub fn concatenate() -> List<Int> {
  [1] ++ [2, 3] ++ []
}

example concatenate_joins for concatenate {
  concatenate() == [1, 2, 3]
}

pub fn logic(a: Bool, b: Bool, c: Bool) -> Bool {
  (a and b) or (c and not a == b)
}

example logic_follows_precedence for logic {
  logic(a: true, b: true, c: false) and not logic(a: true, b: false, c: false)
}
```

```cernith-error E0001
module guide.operators;

pub fn between(a: Int, b: Int, c: Int) -> Bool {
  a < b < c
}
```

### 短絡評価

`a and b` は `a` が偽なら、`a or b` は `a` が真なら、`a implies b` は `a` が偽なら、`b` を評価しません。そのため、`and` と `implies` の右辺では左辺が真であることを、`or` の右辺では左辺が偽であることを前提にできます。検証（`cernith check --verify`）でも同じ前提が使われるので、左辺で条件を確認してから右辺で添字にアクセスするように書けます。

```cernith
module guide.guards;

pub fn first_is_positive(xs: List<Int>) -> Bool {
  xs.length > 0 and xs[0] > 0
}

example empty_list_is_not_positive for first_is_positive {
  not first_is_positive(xs: []) and first_is_positive(xs: [1])
}
```

## if 式と match 式

`if` と `match` は式です。値を持ち、ブロックの最後に置くとそのブロックの値になります。

```text
if 条件 { 式 } else { 式 }
if 条件 { 式 } else if 条件 { 式 } else { 式 }
match 式 {
  パターン => 式,
  パターン => { ブロック },
}
```

- 条件は `Bool` で、括弧は付けません。
- 各分岐の値は、サブタイプを考慮したうえで互換性のある型でなければなりません。型が合わないとエラー `E0301` になります（`Int` と `Float64` なら `E0302`、`distinct` 型が関わるなら `E0303`）。
- `else` のない `if` は、型が `Unit` です。値を返す分岐を、`else` なしでは書けません。
- `match` の各ケースは `パターン => 式,` または `パターン => { ブロック },` と書き、**最後のケースにもカンマを付けます**。`match` では、判別する値が取りうるすべての場合を網羅する必要があります（[07 レコードとバリアント](07-records-and-variants.md)）。

`else` のない `if` の値を使おうとすると、型が合わずにエラーになります。

```cernith-error E0301
module guide.branches;

pub fn maybe(flag: Bool) -> Int {
  let value = if flag { 1 };
  value
}
```

```cernith
module guide.branches;

pub fn describe(sign: Int) -> String {
  if sign < 0 {
    "negative"
  } else if sign == 0 {
    "zero"
  } else {
    "positive"
  }
}

example describes_signs for describe {
  describe(sign: -3) == "negative"
    and describe(sign: 0) == "zero"
    and describe(sign: 8) == "positive"
}

pub fn or_zero(value: Option<Int>) -> Int {
  match value {
    Some(value: number) => number,
    None => 0,
  }
}

example or_zero_defaults for or_zero {
  or_zero(value: Some(value: 5)) == 5 and or_zero(value: None) == 0
}
```

`if`、`match`、`for`、`while` が式として書けるのは、**式の先頭**です。演算子のオペランドや、後ろに `.name`・`[i]` を続ける基の式としては、括弧なしに書けません。`1 + if ...` は構文エラー（`E0001`）で、括弧で囲めば書けます。リストの要素・呼び出しの引数・フィールドの値の位置では、括弧は不要です。

```cernith-error E0001
module guide.branches;

pub fn plus(flag: Bool) -> Int {
  1 + if flag { 1 } else { 2 }
}
```

```cernith
module guide.branches;

pub fn plus(flag: Bool) -> Int {
  1
    + (
      if flag {
        1
      } else {
        2
      }
    )
}

example plus_adds_the_branch for plus {
  plus(flag: true) == 2 and plus(flag: false) == 3
}
```

この規則があるので、「`if` 文の次の行が `-x` で始まる」場合と「`if` 式から `x` を引く」場合を、文法上取り違えることがありません。

## リスト

`[a, b, c]` は、`List` を作ります。要素の型は、揃っている必要があります（サブタイプを考慮した上位の型になります）。空のリスト `[]` は、型が文脈から決まらないと使えません（`var xs: List<Int> = [];` のように、型を与えます。[06 関数](06-functions.md)の型推論）。

`a..b`（`a` 以上 `b` 未満）と `a..=b`（`a` 以上 `b` 以下）の範囲は、`for` ループと量化子（`forall`、`exists`）の中にだけ書けます。値としては使えません。

## 関数式（ラムダ）

```text
fn(ラベル: 型, ...) -> 戻り値の型 { 本体 }
```

引数と戻り値の型は省略できません。本体は通常の関数と同じ形で書き、効果を使う場合は `uses` を付けます。関数式から、それを囲む関数の `var` を読み書きすることはできません（エラー `E0322`）。詳しくは [06 関数](06-functions.md) を参照してください。

## 関連

- 文と束縛：[05 文と束縛](05-statements.md)
- 関数の定義と呼び出し：[06 関数](06-functions.md)
- 演算子の文法：[21 文法](21-grammar.md)
