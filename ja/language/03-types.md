# 型

Cernith は静的型付けの言語で、型を名前で区別します。名前が異なれば、構造が同じでも別の型として扱われます。この章では組み込み型を説明します。ユーザー定義型については、[07 レコードとバリアント](07-records-and-variants.md) と [08 制約付き型と distinct 型](08-refined-and-distinct-types.md) を参照してください。

## 組み込み型

| 型 | 意味 | 値の例 |
| --- | --- | --- |
| `Int` | 桁数に上限のない整数 | `0`、`-7`、`12345678901234567890` |
| `Float64` | 有限の IEEE 754 binary64 | `1.5`、`0.0`、`1.0e21` |
| `Bool` | 真偽値 | `true`、`false` |
| `String` | Unicode スカラー値の列 | `"héllo"` |
| `Unit` | 値を 1 つだけ持つ型 | `()` |
| `List<T>` | 変更できない有限の列 | `[1, 2, 3]`、`[]` |
| `Option<T>` | 値があるか、ないか | `Some(value: x)`、`None` |
| `Result<T, E>` | 成功か、エラーか | `Ok(value: x)`、`Err(error: e)` |
| `fn(ラベル: A, ...) -> R` | 関数 | ラムダ式、関数名 |
| `Bytes` | 0 から 255 の整数（バイト）の列 | `std.bytes` の関数で作る |
| `Floats` | 有限の `Float64` の列 | `std.floats` の関数で作る |
| `Ints` | 64 ビット整数の列 | `std.ints` の関数で作る |

`Bytes`、`Floats`、`Ints` は、`List` よりもコンパクトに格納される列です。いずれも値として扱われ、`==` では要素どうしが比較されます。大小比較はできません。専用のリテラルはなく、作成や読み出しには標準ライブラリの関数を使います（[bytes](../stdlib/bytes.md)、[floats と ints](../stdlib/floats-ints.md)）。

### Int

`Int` は桁あふれしません。足し算・引き算・掛け算の結果は、常に正確な整数です。

整数の割り算では、丸め方を明示的に選びます。`Int` どうしに `/` を使うとエラー `E0307` になります。代わりに `std.int` の `div_floor`、`div_trunc`、`div_euclid` を使います。対応する剰余の関数は `rem_floor`、`rem_trunc`、`rem_euclid` です（[stdlib/int.md](../stdlib/int.md)）。`/` は `Float64` どうしにのみ使えます。

```cernith-error E0307
module guide.division;

pub fn half(value: Int) -> Int {
  value / 2
}
```

既定の丸め方がないのは、負の数を含む割り算では丸め方によって結果が変わるためです。たとえば `-7` を `2` で割ると、負の無限大に向けて丸める場合（floor）は `-4`、0 に向けて切り捨てる場合は `-3` になります。

### Float64

`Float64` は、**有限の値だけ**を取ります。無限大と NaN はありません。無限大や NaN になる演算（桁あふれ、`0.0` による割り算など）は、欠陥になります（[12 エラーと欠陥](12-errors-and-defects.md)）。そのため、`Float64` どうしの比較 `==`、`<` は、すべての値について全順序になります。

`Int` と `Float64` を混ぜた演算は、エラー `E0302` です。`std.float` の `from_int` で明示的に変換します。

```cernith-error E0302
module guide.mixing;

pub fn scale(count: Int, factor: Float64) -> Float64 {
  count * factor
}
```

```cernith-error E0508
module guide.floats;

pub fn scale_up(x: Float64) -> Float64 {
  x * 10.0
}
```

`x` が最大値に近いと `x * 10.0` は有限の範囲を超えるので、`cernith check --verify` は反例の入力を添えて `E0508`（float-overflow）を報告します（`--verify` なしの `cernith check` は証明義務を調べません。[10 証明義務と保証レベル](10-verification.md)）。入力の範囲を `requires` で絞れば、受理されます。

```cernith
module guide.floats;

import std.float;

pub fn scale_up(x: Float64) -> Float64 {
  requires x >= -1.0e300 and x <= 1.0e300;

  x * 10.0
}

example scale_up_multiplies_by_ten for scale_up {
  scale_up(x: 2.0) == 20.0
}

pub fn to_float(count: Int) -> Float64 {
  requires count >= -1000 and count <= 1000;

  float.from_int(value: count)
}

example to_float_converts for to_float {
  to_float(count: 3) == 3.0
}
```

### String

`String` は Unicode スカラー値の列です。`s.length` は、バイト数や UTF-16 の単位数ではなく、スカラー値の個数を返します。絵文字 1 つは長さ 1 です。位置を取る標準ライブラリの関数（`text.slice` など）も、スカラー値の個数で数えます。`+` ではなく `++` で連結します。

`String` どうしは、コードポイントの辞書式順序で比較できます。

文字を表す専用の型はありません。1 文字は、長さ 1 の `String` です。

### List

`List<T>` は変更できません。要素を加えたり置き換えたりした新しい列を作るには、`++`、または標準ライブラリの関数（[stdlib/list.md](../stdlib/list.md)）を使います。`xs.length` は要素数（0 以上の `Int`）、`xs[i]` は `i` 番目の要素です。範囲外の添字は欠陥です。

組み込み型でフィールドの記法を使えるのは、`xs.length` と `s.length` だけです。そのほかの操作にはモジュールの関数を使います。`xs.is_empty` や `s.split(...)` といった記法はありません。

```cernith-error E0506
module guide.builtins;

pub fn at(xs: List<Int>, i: Int) -> Int {
  xs[i]
}
```

`i` が範囲の外になりうるので、`cernith check --verify` は `E0506`（index-out-of-range）を報告します。添字が範囲内であることを `requires` に書くと、受理されます。このとき、範囲外の呼び出しは、`requires` に違反する呼び出しとして欠陥になります。

```cernith
module guide.builtins;

pub fn at(xs: List<Int>, i: Int) -> Int {
  requires i >= 0 and i < xs.length;

  xs[i]
}

example at_out_of_range_is_a_defect for at {
  defects(of: at(xs: [1, 2], i: 2))
}

example at_second for at {
  at(xs: [1, 2], i: 1) == 2
}

pub fn length_of(s: String) -> Int {
  s.length
}

example length_counts_scalar_values for length_of {
  length_of(s: "héllo") == 5 and length_of(s: "😀") == 1
}
```

### Option と Result

`Option<T>` は値がない可能性を、`Result<T, E>` は処理が失敗する可能性を表します。`null` はありません。値がない可能性は `Option` で型に明示し、`match` で両方の場合を扱います（[12 エラーと欠陥](12-errors-and-defects.md)）。

### 関数型

関数型は `fn(ラベル: 型, ...) -> 戻り値の型` と書き、効果があれば `uses` 以下に続けます（[11 効果](11-effects.md)）。引数のラベルも型の一部です（[06 関数](06-functions.md)）。

## 型を書く形

| 形 | 意味 |
| --- | --- |
| `Name` | 型の名前（同じモジュールの型や、プレリュードの型） |
| `module.Name` | 他のモジュールの型。`import` したモジュールの別名を前置する |
| `Name<A, B>` | 型引数を渡したジェネリックな型 |
| `fn(label: A) -> B` | 関数型 |

## ない型

| ないもの | 代わりに使うもの |
| --- | --- |
| タプル | レコード（`record Pair { first: Int, second: Int }`） |
| 型の別名 | 制約付き型か `distinct` 型 |
| 文字型 | 長さ 1 の `String` |
| 固定長の整数型（`Int32` など） | 範囲を持つ制約付き型（`Int where self >= 0 and self <= 255`）、または `std.word32` と `std.word64` |
| `null` | `Option<T>` |

タプルを書くと構文エラー（`E0001`）に、型の別名を宣言するとエラー `E0314` になります。別名がないのは、同じ型を別の名前で呼べると、名前を見ただけでは型の違いが分からなくなるからです。意味を区別したいなら `distinct` 型を、範囲を表したいなら制約付き型を使います。

```cernith-error E0314
module guide.alias;

pub type UserId = Int;
```

```cernith-error E0001
module guide.tuple;

pub fn pair() -> Int {
  let p = (1, 2);
  1
}
```

## 等価と順序

`==` と `!=` は、両辺の型が等価比較できるときに使えます。`<`、`<=`、`>`、`>=` は、順序がある型だけに使えます。

| 比較 | 使える型 |
| --- | --- |
| `==`、`!=`（境界 `Eq`） | `Int`、`Float64`、`String`、`Bool`、`Unit`、および要素がこれらを満たす `List`、`Option`、`Result`、レコード、バリアント |
| `<` など（境界 `Ord`） | `Int`、`Float64`、`String`、`Bool` |

関数型は、等価比較にも大小比較にも対応していません。`List`、`Option`、`Result`、レコード、バリアントも大小比較には対応せず、使うとエラー `E0316` になります。`Eq` と `Ord` は組み込みの境界であり、ユーザーが実装することはできません。境界はジェネリック関数でも使います（[06 関数](06-functions.md)）。独自の境界は、インターフェースとして宣言できます（[22 インターフェース](22-interfaces.md)）。

```cernith-error E0316
module guide.ordering;

pub fn before(first: List<Int>, second: List<Int>) -> Bool {
  first < second
}
```

## 関連

- 演算子の型付け規則は [04 式](04-expressions.md) にあります。
- 型どうしの互換性（サブタイプ）は [08 制約付き型と distinct 型](08-refined-and-distinct-types.md) にあります。
