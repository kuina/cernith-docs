# std.float

`Float64` を扱う基本の関数のモジュールです。`import std.float;` と書き、`float.abs(value: x)` のように呼びます。`Int` からの変換、文字列との変換、平方根はこのモジュールにあります。三角関数、対数、べき乗、`floor` や `round` は [math](math.md) です。

`Float64` は有限の値だけを取り、無限大も NaN もありません。そのため、結果が有限の範囲を超える演算は欠陥になります。チェッカーは、そうなりうる箇所を `E0508`（float-overflow）で報告します（[03 型](../language/03-types.md)）。

## 関数一覧

| 関数 | 戻り値の型 | 意味 | 例 |
| --- | --- | --- | --- |
| `float.abs(value)` | `Float64` | 絶対値 | `float.abs(value: -2.5)` → `2.5` |
| `float.min(first, second)` | `Float64` | 小さいほう | `float.min(first: 1.5, second: 2.5)` → `1.5` |
| `float.max(first, second)` | `Float64` | 大きいほう | `float.max(first: 1.5, second: 2.5)` → `2.5` |
| `float.clamp(value, low, high)` | `Float64` | `low` 以上 `high` 以下に収める | `float.clamp(value: 7.5, low: 0.0, high: 1.0)` → `1.0` |
| `float.sqrt(value)` | `Float64` | 平方根 | `float.sqrt(value: 9.0)` → `3.0` |
| `float.from_int(value)` | `Float64` | `Int` を `Float64` にする | `float.from_int(value: 3)` → `3.0` |
| `float.parse(source)` | `Option<Float64>` | 10 進の文字列を読む | `float.parse(source: "1.5E-7")` → `Some(value: 1.5e-7)` |
| `float.to_text(value)` | `String` | 文字列にする | `float.to_text(value: 100.0)` → `"100.0"` |

## 関数の詳細

### float.abs、float.min、float.max、float.clamp

```text
pub fn abs(value: Float64) -> Float64
pub fn min(first: Float64, second: Float64) -> Float64
pub fn max(first: Float64, second: Float64) -> Float64
pub fn clamp(value: Float64, low: Float64, high: Float64) -> Float64
```

`abs` の結果は 0.0 以上で、`value` か `0.0 - value` のどちらかです。`min` と `max` は、引数のどちらかを返します。

`clamp` は、`value` が `low` 以上 `high` 以下ならそのまま、外ならば近いほうの端を返します。事前条件は `requires low <= high;` で、`low > high` の呼び出しは欠陥です。

### float.sqrt

```text
pub fn sqrt(value: Float64) -> Float64
```

平方根です。結果は、真の平方根に最も近い `Float64` です。`sqrt(value: 2.0)` は `1.4142135623730951` です。

事前条件は `requires value >= 0.0;` です。負の数の呼び出しは欠陥です（`Float64` には、負の数の平方根を表す値がありません）。事後条件は `result >= 0.0` です。

### float.from_int

```text
pub fn from_int(value: Int) -> Float64
```

`Int` を、同じ値の `Float64` にします。`Int` と `Float64` を混ぜた演算はエラー `E0302` なので、`Int` を `Float64` の式で使うには、この関数で変換します。

事前条件は `requires value >= -9007199254740992 and value <= 9007199254740992;` です。この範囲（−2⁵³ から 2⁵³）の整数は、`Float64` で正確に表せます。範囲の外の呼び出しは欠陥です。

### float.parse

```text
pub fn parse(source: String) -> Option<Float64>
```

10 進の数を読みます。受け付ける書式は、次のとおりです。

```text
-?digits[.digits][(e|E)[+-]digits]
```

- 先頭の `-` は省略できます。`+` は書けません。
- `digits` は 1 文字以上の `0` から `9` です。`.` の前にも後にも、1 文字以上の数字が必要です（`"1."` と `".5"` は `None`）。
- 指数は `e` か `E` に、省略できる符号と 1 文字以上の数字が続きます（`"1e3"`、`"1.5E-7"`、`"1e+5"`）。
- 整数部の前の `0` は許され、結果に影響しません（`"007.50"` は `7.5`）。
- 空白、区切り、`inf`、`NaN` は書けません。

書式に合わない文字列は `None` です。書式に合っていても、数を `d.ddd × 10^n`（`d` は 1 から 9 の数字で始まる）の形に書いたときの `n` が 308 を超える数（`1e309`、`10e308` など）は `None` です。逆に小さいほうでは、`0` と区別できないほど小さい数（`"1e-9999"` など）は、`None` でなく `0.0` になります。読んだ結果の `0` は常に `0.0` で、`"-0"` も `0.0` です。

結果は、書かれた 10 進の数に最も近い `Float64` です（`"0.1"` は `0.1`）。`n` が 308 以下でも、`Float64` の最大値（`1.7976931348623157e308`）を超える数（`1.9e308` など）は、`Float64` で表せません。そのような文字列を渡したときの結果は規定されていません。

| 呼び出し | 結果 |
| --- | --- |
| `float.parse(source: "1.5")` | `Some(value: 1.5)` |
| `float.parse(source: "-2.25")` | `Some(value: -2.25)` |
| `float.parse(source: "100")` | `Some(value: 100.0)` |
| `float.parse(source: "1e3")` | `Some(value: 1000.0)` |
| `float.parse(source: "1e-9999")` | `Some(value: 0.0)` |
| `float.parse(source: "1e309")` | `None` |
| `float.parse(source: "1.")` | `None` |
| `float.parse(source: ".5")` | `None` |
| `float.parse(source: "1e")` | `None` |
| `float.parse(source: "")` | `None` |

`None` を返すのは、書式に合わない入力が、欠陥でなく普通の結果だからです。

### float.to_text

```text
pub fn to_text(value: Float64) -> String
```

`Float64` を文字列にします。書き方は決まっています。

- 桁は、同じ `Float64` に読み戻せる最も短いものです（`0.1 + 0.2` は `"0.30000000000000004"`、`0.1` は `"0.1"`）。
- 値の 10 進の指数（最上位の桁の位。`123.0` なら 2、`0.001` なら −3）が −5 から 15 のときは、位取りで書きます。小数点は必ず付き、整数でも `.0` が付きます（`"100.0"`、`"0.001"`、`"1000000000000000.0"`）。
- それ以外のときは、`d.ddde+XX` の形です。仮数部は 1 桁目の後に小数点が付き、桁が 1 つなら `.0` が付きます。指数には符号が必ず付き、先頭の 0 はありません（`"1.0e+16"`、`"1.5e-7"`、`"1.5e+20"`）。
- 負の数は先頭に `-` が付きます。`0.0` は `"0.0"` です。

| 呼び出し | 結果 |
| --- | --- |
| `float.to_text(value: 1.5)` | `"1.5"` |
| `float.to_text(value: -2.25)` | `"-2.25"` |
| `float.to_text(value: 0.0)` | `"0.0"` |
| `float.to_text(value: 100.0)` | `"100.0"` |
| `float.to_text(value: 0.00001)` | `"0.00001"` |
| `float.to_text(value: 0.000001)` | `"1.0e-6"` |
| `float.to_text(value: 1000000000000000.0)` | `"1000000000000000.0"` |
| `float.to_text(value: 10000000000000000.0)` | `"1.0e+16"` |
| `float.to_text(value: 150000000000000000000.0)` | `"1.5e+20"` |

`to_text` が返す文字列を `parse` に渡すと、元の値に戻ります。

## 例

```cernith
module guide.floats;

import std.float;

pub fn as_float(count: Int) -> Float64 {
  requires count >= -9007199254740992 and count <= 9007199254740992;

  float.from_int(value: count)
}

example as_float_is_exact_in_range for as_float {
  as_float(count: 3) == 3.0
    and as_float(count: -2) == -2.0
    and as_float(count: 0) == 0.0
    and as_float(count: 9007199254740992) == 9007199254740992.0
    and as_float(count: -9007199254740992) == -9007199254740992.0
    and defects(of: float.from_int(value: 9007199254740993))
    and defects(of: float.from_int(value: -9007199254740993))
}

pub fn larger_and_smaller(first: Float64, second: Float64) -> List<Float64> {
  [
    float.max(first: first, second: second),
    float.min(first: first, second: second),
  ]
}

example larger_and_smaller_pick_one_of_the_two for larger_and_smaller {
  larger_and_smaller(first: 1.5, second: 2.5) == [2.5, 1.5]
    and larger_and_smaller(first: 2.5, second: 1.5) == [2.5, 1.5]
    and larger_and_smaller(first: -1.0, second: -1.0) == [-1.0, -1.0]
    and larger_and_smaller(first: 0.0, second: -3.5) == [0.0, -3.5]
}

pub fn limited(value: Float64, low: Float64, high: Float64) -> Float64 {
  requires low <= high;

  float.clamp(value: value, low: low, high: high)
}

example limited_inside_and_outside for limited {
  limited(value: 0.5, low: 0.0, high: 1.0) == 0.5
    and limited(value: -3.0, low: 0.0, high: 1.0) == 0.0
    and limited(value: 7.5, low: 0.0, high: 1.0) == 1.0
    and limited(value: 0.0, low: 0.0, high: 1.0) == 0.0
    and limited(value: 1.0, low: 0.0, high: 1.0) == 1.0
    and limited(value: 2.0, low: 2.0, high: 2.0) == 2.0
    and defects(of: float.clamp(value: 0.5, low: 1.0, high: 0.0))
}

pub fn magnitude(value: Float64) -> Float64 {
  float.abs(value: value)
}

example magnitude_of_both_signs for magnitude {
  magnitude(value: -2.5) == 2.5
    and magnitude(value: 2.5) == 2.5
    and magnitude(value: 0.0) == 0.0
}

pub fn read_float(source: String) -> Option<Float64> {
  float.parse(source: source)
}

example read_float_reads_the_nearest_float for read_float {
  read_float(source: "1.5") == Some(value: 1.5)
    and read_float(source: "-2.25") == Some(value: -2.25)
    and read_float(source: "100") == Some(value: 100.0)
    and read_float(source: "0.1") == Some(value: 0.1)
    and read_float(source: "1e3") == Some(value: 1000.0)
    and read_float(source: "1.5E-7") == Some(value: 1.5e-7)
    and read_float(source: "1e+5") == Some(value: 100000.0)
    and read_float(source: "007.50") == Some(value: 7.5)
    and read_float(source: "0") == Some(value: 0.0)
    and read_float(source: "-0") == Some(value: 0.0)
    and read_float(source: "1e308") == Some(value: 1.0e308)
    and read_float(source: "1e-9999") == Some(value: 0.0)
}

example read_float_rejects_everything_else for read_float {
  read_float(source: "") == None
    and read_float(source: "-") == None
    and read_float(source: "1.") == None
    and read_float(source: ".5") == None
    and read_float(source: "1e") == None
    and read_float(source: "1e+") == None
    and read_float(source: "1x") == None
    and read_float(source: "+1") == None
    and read_float(source: " 1") == None
    and read_float(source: "1.5.2") == None
    and read_float(source: "1_000") == None
    and read_float(source: "inf") == None
    and read_float(source: "NaN") == None
    and read_float(source: "1e309") == None
    and read_float(source: "10e308") == None
}

pub fn root(value: Float64) -> Float64 {
  requires value >= 0.0;

  float.sqrt(value: value)
}

example root_is_the_nearest_float for root {
  root(value: 9.0) == 3.0
    and root(value: 0.0) == 0.0
    and root(value: 2.0) == 1.4142135623730951
    and root(value: 0.25) == 0.5
    and defects(of: float.sqrt(value: -1.0))
}

pub fn write_float(value: Float64) -> String {
  float.to_text(value: value)
}

example write_float_uses_the_shortest_digits for write_float {
  write_float(value: 1.5) == "1.5"
    and write_float(value: -2.25) == "-2.25"
    and write_float(value: 100.0) == "100.0"
    and write_float(value: 0.001) == "0.001"
    and write_float(value: 0.1) == "0.1"
    and write_float(value: 0.0) == "0.0"
    and write_float(value: 123456789.125) == "123456789.125"
    and write_float(value: 1000000000000000.0) == "1000000000000000.0"
    and write_float(value: 10000000000000000.0) == "1.0e+16"
    and write_float(value: 150000000000000000000.0) == "1.5e+20"
    and write_float(value: 0.00001) == "0.00001"
    and write_float(value: 0.000001) == "1.0e-6"
    and write_float(value: 1.5e-7) == "1.5e-7"
    and write_float(value: 5.0e-324) == "5.0e-324"
    and write_float(value: 1.7976931348623157e308) == "1.7976931348623157e+308"
}
```

## 関連

- [math](math.md)：三角関数、指数・対数、べき乗、`floor`・`ceil`・`round`・`trunc`、`to_int`
- [int](int.md)：`Int` の関数
- [03 型](../language/03-types.md)：`Float64` と、桁あふれの扱い
