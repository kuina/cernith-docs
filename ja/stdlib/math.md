# std.math

`Float64` の数学関数と定数を提供します。`import std.math;` で読み込み、`math.sin(x: 1.0)` のように呼び出します。定数には `math.pi` のように括弧を付けずにアクセスします。

`Float64` には無限大も NaN もないので、定義域の外の入力は事前条件で禁じられ、違反する呼び出しは欠陥です。結果が `Float64` の範囲を超える演算も欠陥になります（[03 型](../language/03-types.md)）。`Float64` を使う例では、この桁あふれの証明義務が `E0508`（float-overflow）や `W0504`（証明不明）として現れやすいので、入力の範囲を `requires` で絞ります。

このモジュールの関数は、`Float64` の `+`、`-`、`*`、`/` と `sqrt` だけで計算されます。そのため、同じ入力には、どの環境でも同じビット列の結果が返ります。精度は[精度](#精度)の節にあります。絶対値、最大・最小、平方根、`Int` からの変換は [float](float.md) にあります。

## 定数

| 定数 | 値 | 意味 |
| --- | --- | --- |
| `math.pi` | `3.141592653589793` | 円周率 |
| `math.tau` | `6.283185307179586` | 2π |
| `math.half_pi` | `1.5707963267948966` | π/2 |
| `math.quarter_pi` | `0.7853981633974483` | π/4 |
| `math.e` | `2.718281828459045` | 自然対数の底 |
| `math.sqrt_2` | `1.4142135623730951` | 2 の平方根 |
| `math.ln_2` | `0.6931471805599453` | 2 の自然対数 |
| `math.ln_10` | `2.302585092994046` | 10 の自然対数 |

すべて `Float64` です。

## 関数一覧

| 関数 | 戻り値の型 | 意味 | 例 |
| --- | --- | --- | --- |
| `math.sin(x)` | `Float64` | 正弦（`x` はラジアン） | `math.sin(x: 1.0)` → `0.8414709848078965` |
| `math.cos(x)` | `Float64` | 余弦 | `math.cos(x: 0.0)` → `1.0` |
| `math.tan(x)` | `Float64` | 正接 | `math.tan(x: 0.0)` → `0.0` |
| `math.asin(x)` | `Float64` | 逆正弦（−π/2 から π/2） | `math.asin(x: 1.0)` → `1.5707963267948966` |
| `math.acos(x)` | `Float64` | 逆余弦（0 から π） | `math.acos(x: -1.0)` → `3.141592653589793` |
| `math.atan(x)` | `Float64` | 逆正接（−π/2 から π/2） | `math.atan(x: 0.0)` → `0.0` |
| `math.atan2(y, x)` | `Float64` | 点 (x, y) の偏角（−π から π） | `math.atan2(y: 1.0, x: -1.0)` → `2.356194490192345` |
| `math.hypot(x, y)` | `Float64` | 原点から (x, y) までの距離 | `math.hypot(x: 3.0, y: 4.0)` → `5.0` |
| `math.exp(x)` | `Float64` | e の `x` 乗 | `math.exp(x: 0.0)` → `1.0` |
| `math.ln(x)` | `Float64` | 自然対数 | `math.ln(x: 2.0)` → `0.6931471805599453` |
| `math.log2(x)` | `Float64` | 2 を底とする対数 | `math.log2(x: 8.0)` → `3.0` |
| `math.log10(x)` | `Float64` | 10 を底とする対数 | `math.log10(x: 1.0)` → `0.0` |
| `math.pow(base, exponent)` | `Float64` | `base` の `exponent` 乗 | `math.pow(base: 2.0, exponent: 10.0)` → `1024.0` |
| `math.floor(x)` | `Float64` | `x` 以下で最大の整数値 | `math.floor(x: -2.5)` → `-3.0` |
| `math.ceil(x)` | `Float64` | `x` 以上で最小の整数値 | `math.ceil(x: -2.5)` → `-2.0` |
| `math.trunc(x)` | `Float64` | 小数部分を捨てる（0 方向） | `math.trunc(x: -2.9)` → `-2.0` |
| `math.round(x)` | `Float64` | 最も近い整数値（半分は 0 から遠ざかる方向） | `math.round(x: -2.5)` → `-3.0` |
| `math.to_int(x)` | `Int` | 整数値の `Float64` を `Int` にする | `math.to_int(x: -7.0)` → `-7` |

## 定義域

表の定義域はすべて事前条件です。範囲外の入力を渡すと欠陥になります。

| 関数 | 事前条件 |
| --- | --- |
| `sin`、`cos`、`tan` | `x >= -500000000.0 and x <= 500000000.0` |
| `asin`、`acos` | `x >= -1.0 and x <= 1.0` |
| `exp` | `x <= 709.0` |
| `ln`、`log2`、`log10` | `x > 0.0` |
| `pow` | 下記 |
| `to_int` | `x == math.trunc(x: x) and x >= -9007199254740992.0 and x <= 9007199254740992.0` |
| `atan`、`atan2`、`hypot`、`floor`、`ceil`、`trunc`、`round` | なし |

`sin`、`cos`、`tan` で `x` の大きさに上限があるのは、角度を π/2 の範囲に収める計算に必要な π の桁数が、保持している精度では足りなくなるためです。`exp` の上限は、e の 710 乗が `Float64` の最大値を超えることに由来します。

定義域の中でも、結果が `Float64` の範囲を超えるなら欠陥です（`hypot` に、どちらも非常に大きい値を渡す場合など）。結果が範囲に収まる場合でも、計算の途中の値が範囲を超えれば欠陥です。`atan2(y: 1.0e300, x: 1.0e-10)` は、先に `y / x` を計算するので欠陥になります。同じ理由で、`acos` は絶対値がおよそ `5.6e-309` より小さい `x` に対して欠陥になります。逆に、`exp` の結果が最小の `Float64` より小さくなるとき（`exp(x: -800.0)` など）は、欠陥でなく `0.0` です。

## 精度

`floor`、`ceil`、`trunc`、`round` は正確な結果を返します。ほかの関数の誤差は、最下位桁の単位（ULP）で表すと次の範囲になります。正しく丸められた結果であることは保証しません。

| 関数 | 誤差 |
| --- | --- |
| `sin`、`cos`、`tan`、`atan`、`atan2`、`asin`、`acos`、`exp`、`ln`、`hypot` | 4 ULP 以内 |
| `log2`、`log10` | 6 ULP 以内 |
| `pow`（整数値の指数で、絶対値が 64 以下） | `abs(exponent)` ULP 以内。結果が `Float64` で正確に表せる値（2¹⁰、10³ など）なら、正確 |
| `pow`（それ以外の指数） | `4 + abs(exponent * ln(base))` ULP 以内 |

したがって、`==` で結果を比べられるのは、`sin(x: 0.0)` のような既知の値や、正確な関数の結果です。そうでない結果の比較には、許容誤差を付けた比較（下の例の `within`）を使います。

## 三角関数

```text
pub fn sin(x: Float64) -> Float64
pub fn cos(x: Float64) -> Float64
pub fn tan(x: Float64) -> Float64
```

`x` の単位はラジアンです。事前条件は `requires x >= -500000000.0 and x <= 500000000.0;` です。`sin` と `cos` の事後条件は `result >= -1.0000000000000002 and result <= 1.0000000000000002` で、結果が −1 から 1 の範囲を最下位桁の 1 単位分だけ超える可能性を含みます。

`tan` は、π/2 の奇数倍の近くでは、結果の絶対値が非常に大きくなります。

`cos(x: 1.5707963267948966)` は `0.0` でなく、`6.123233995736766e-17` に近い値です。`math.half_pi` は π/2 そのものでなく、それに最も近い `Float64` だからです。

## 逆三角関数

```text
pub fn asin(x: Float64) -> Float64
pub fn acos(x: Float64) -> Float64
pub fn atan(x: Float64) -> Float64
pub fn atan2(y: Float64, x: Float64) -> Float64
```

`asin` と `acos` の事前条件は `requires x >= -1.0 and x <= 1.0;` です。

`atan` は、正接が `x` になる角（ラジアン）です。事後条件は、`result >= -1.5707963267948968 and result <= 1.5707963267948968` です。

`atan2(y, x)` は、原点から点 (`x`, `y`) を見た角で、象限を区別します。`atan2(y: 1.0, x: -1.0)` は 3π/4、`atan2(y: -1.0, x: -1.0)` は −3π/4 です。`atan2(y: 0.0, x: 0.0)` は `0.0` です。事後条件は `result >= -3.1415926535897936 and result <= 3.1415926535897936` です。引数の順序は `y`、`x` です。

## hypot

```text
pub fn hypot(x: Float64, y: Float64) -> Float64
```

`x` と `y` の二乗和の平方根です。二乗を先に計算しないので、`x` や `y` の二乗が `Float64` の範囲を超えるときでも、結果が範囲に収まるなら計算できます。事後条件は `result >= 0.0` です。

## 指数と対数

```text
pub fn exp(x: Float64) -> Float64
pub fn ln(x: Float64) -> Float64
pub fn log2(x: Float64) -> Float64
pub fn log10(x: Float64) -> Float64
```

`exp` の事前条件は `requires x <= 709.0;`、事後条件は `result >= 0.0` です。`x` が非常に小さく、結果が最小の `Float64` より小さいときは `0.0` です。

`ln`、`log2`、`log10` の事前条件は `requires x > 0.0;` です。`0.0` や負の数は、対数が有限の値にならないので、欠陥です。`ln(x: 5.0e-324)`（最小の正の `Float64`）は `-744.4400719213812` です。

## pow

```text
pub fn pow(base: Float64, exponent: Float64) -> Float64
```

`base` の `exponent` 乗です。`pow(base: 0.0, exponent: 0.0)` は `1.0`、`exponent` が `0.0` なら常に `1.0` です。

事前条件は、次のどちらかです。

```text
requires (
  exponent == trunc(x: exponent)
    and exponent >= -2147483648.0
    and exponent <= 2147483648.0
    and (base != 0.0 or exponent >= 0.0)
)
  or (base > 0.0 and exponent * ln(x: base) <= 709.0);
```

- 指数が整数値で、絶対値が 2³¹ 以下。底が `0.0` のときは、指数は 0 以上。底は負でもかまいません。
- 底が正で、`exponent * ln(x: base)` が 709 以下。指数は小数でもかまいません。

底が正でないときは指数が整数値でなければならず、底が `0.0` のときは、さらに指数が 0 以上でなければなりません。そうでない呼び出しは欠陥です（`pow(base: -2.0, exponent: 0.5)`、`pow(base: 0.0, exponent: 0.5)`、`pow(base: 0.0, exponent: -1.0)`）。負の底の奇数乗は負、偶数乗は正です。

計算の方法は、指数によって異なります。絶対値が 64 以下の整数値の指数は、繰り返し二乗で計算します。それ以外は `exp(exponent * ln(base))` です。精度は[精度](#精度)の表のとおりです。

指数が 64 より大きい整数値のときは、`exp` で計算するので、結果が `Float64` の範囲を超えると欠陥です（`pow(base: 10.0, exponent: 400.0)`）。指数が小さな負の数で、結果が最小の `Float64` より小さいときは `0.0` です（`pow(base: 10.0, exponent: -400.0)`）。繰り返し二乗の途中で範囲を超えるときも欠陥です。

## 整数値にする関数

```text
pub fn floor(x: Float64) -> Float64
pub fn ceil(x: Float64) -> Float64
pub fn trunc(x: Float64) -> Float64
pub fn round(x: Float64) -> Float64
```

結果はどれも、整数値の `Float64` で、事前条件はありません。`Float64` がもともと整数値になる大きさ（2⁵² 以上）の値は、そのまま返ります。

| `x` | `floor` | `ceil` | `trunc` | `round` |
| ---: | ---: | ---: | ---: | ---: |
| 2.5 | 2.0 | 3.0 | 2.0 | 3.0 |
| -2.5 | -3.0 | -2.0 | -2.0 | -3.0 |
| 2.4 | 2.0 | 3.0 | 2.0 | 2.0 |
| -2.6 | -3.0 | -2.0 | -2.0 | -3.0 |
| 3.0 | 3.0 | 3.0 | 3.0 | 3.0 |

- `floor`：`x` 以下で最大の整数値（負の無限大の方向）。事後条件は `result <= x and result >= x - 1.0`。
- `ceil`：`x` 以上で最小の整数値（正の無限大の方向）。事後条件は `result >= x and result <= x + 1.0`。
- `trunc`：小数部分を捨てます（0 の方向）。
- `round`：最も近い整数値です。ちょうど半分（`2.5`、`-2.5`）のときは、**0 から遠ざかる方向**に丸めます（`2.5` は `3.0`、`-2.5` は `-3.0`）。偶数への丸めではありません。`0.49999999999999994`（0.5 の直前の `Float64`）は `0.0` です。

## to_int

```text
pub fn to_int(x: Float64) -> Int
```

整数値の `Float64` を、丸めずに `Int` に変換します。次の事前条件を満たさない呼び出しは欠陥になります。

```text
requires x == trunc(x: x)
  and x >= -9007199254740992.0
  and x <= 9007199254740992.0;
```

つまり、`x` は整数値で、絶対値が 2⁵³ 以下です。小数を含む値を `Int` にするには、先に `floor`、`ceil`、`trunc`、`round` のどれかで整数値にします。ただし、チェッカーはこれらの結果が整数値であることを証明できません（次の節を参照）。

## チェッカーが証明できない条件

`pow` と `to_int` の事前条件は、`trunc` を使って、引数が整数値であることを求めます。チェッカーは `trunc` の結果の性質を知らないので、次のように、リテラルの引数でも、事前条件を証明できず、`W0504`（証明不明）を報告します。

```cernith-error W0504
module guide.math_unproved;

import std.math;

pub fn twelve() -> Int {
  math.to_int(x: 12.0)
}
```

`pow` の呼び出しも同じです。この警告は、呼び出しが誤っているという意味ではありません。事前条件に違反する呼び出しは、欠陥です。

## 例

次の例では、`within` で許容誤差を考慮した比較を、`radians` で定数の使い方を示します。`square` と `to_float` には、`pow` と `to_int` の example を付けています。これらを通常の関数内で呼ぶと、上記の理由で検査時に警告が出るため、呼び出しは example 内にのみ記述しています。

```cernith
module guide.maths;

import std.float;
import std.math;

pub fn arc_cosine(x: Float64) -> Float64 {
  requires x >= -1.0 and x <= 1.0;

  math.acos(x: x)
}

example arc_cosine_has_its_known_values for arc_cosine {
  arc_cosine(x: 1.0) == 0.0
    and arc_cosine(x: 0.0) == math.half_pi
    and arc_cosine(x: -1.0) == math.pi
    and within(
      actual: arc_cosine(x: 0.5),
      expected: 1.0471975511965979,
      tolerance: 1.0e-14,
    )
    and defects(of: math.acos(x: 1.5))
    and defects(of: math.acos(x: -1.5))
}

pub fn arc_sine(x: Float64) -> Float64 {
  requires x >= -1.0 and x <= 1.0;

  math.asin(x: x)
}

example arc_sine_has_its_known_values for arc_sine {
  arc_sine(x: 0.0) == 0.0
    and arc_sine(x: 1.0) == math.half_pi
    and arc_sine(x: -1.0) == -math.half_pi
    and within(
      actual: arc_sine(x: 0.5),
      expected: 0.5235987755982989,
      tolerance: 1.0e-14,
    )
    and defects(of: math.asin(x: 1.5))
    and defects(of: math.asin(x: -1.5))
}

pub fn arc_tangent(x: Float64) -> Float64 {
  math.atan(x: x)
}

example arc_tangent_has_its_known_values for arc_tangent {
  arc_tangent(x: 0.0) == 0.0
    and within(
      actual: arc_tangent(x: 1.0),
      expected: math.quarter_pi,
      tolerance: 1.0e-15,
    )
    and within(
      actual: arc_tangent(x: -1.0),
      expected: -math.quarter_pi,
      tolerance: 1.0e-15,
    )
    and within(
      actual: arc_tangent(x: 1000000.0),
      expected: 1.5707953267948966,
      tolerance: 1.0e-15,
    )
}

pub fn bearing(y: Float64, x: Float64) -> Float64 {
  math.atan2(y: y, x: x)
}

example bearing_chooses_the_quadrant for bearing {
  bearing(y: 1.0, x: 1.0) == 0.7853981633974483
    and bearing(y: 1.0, x: -1.0) == 2.356194490192345
    and bearing(y: -1.0, x: -1.0) == -2.356194490192345
    and bearing(y: -1.0, x: 1.0) == -0.7853981633974483
    and bearing(y: 1.0, x: 0.0) == math.half_pi
    and bearing(y: -1.0, x: 0.0) == -math.half_pi
    and bearing(y: 0.0, x: 0.0) == 0.0
}

pub fn ceiling(x: Float64) -> Float64 {
  math.ceil(x: x)
}

example ceiling_rounds_toward_plus_infinity for ceiling {
  ceiling(x: 2.5) == 3.0
    and ceiling(x: -2.5) == -2.0
    and ceiling(x: 3.0) == 3.0
    and ceiling(x: 0.0) == 0.0
    and ceiling(x: 0.1) == 1.0
    and ceiling(x: 4503599627370497.0) == 4503599627370497.0
}

pub fn cosine(x: Float64) -> Float64 {
  requires x >= -500000000.0 and x <= 500000000.0;

  math.cos(x: x)
}

example cosine_has_its_known_values for cosine {
  cosine(x: 0.0) == 1.0
    and within(
      actual: cosine(x: 1.0),
      expected: 0.5403023058681398,
      tolerance: 1.0e-15,
    )
    and within(
      actual: cosine(x: 3.0),
      expected: -0.9899924966004454,
      tolerance: 1.0e-15,
    )
    and within(
      actual: cosine(x: math.half_pi),
      expected: 6.123233995736766e-17,
      tolerance: 1.0e-15,
    )
    and cosine(x: 500000000.0) <= 1.0000000000000002
    and defects(of: math.cos(x: 500000001.0))
    and defects(of: math.cos(x: -500000001.0))
}

pub fn distance(x: Float64, y: Float64) -> Float64 {
  math.hypot(x: x, y: y)
}

example distance_is_the_length_of_the_hypotenuse for distance {
  distance(x: 3.0, y: 4.0) == 5.0
    and distance(x: -3.0, y: 4.0) == 5.0
    and distance(x: 0.0, y: 0.0) == 0.0
    and distance(x: 1.0, y: 1.0) == math.sqrt_2
    and distance(x: 1.0e200, y: 1.0e200) > 1.4e200
    and defects(of: math.hypot(x: 1.7e308, y: 1.7e308))
}

pub fn exponential(x: Float64) -> Float64 {
  requires x <= 709.0;

  math.exp(x: x)
}

example exponential_has_its_known_values for exponential {
  exponential(x: 0.0) == 1.0
    and within(
      actual: exponential(x: 1.0),
      expected: math.e,
      tolerance: 1.0e-15,
    )
    and within(
      actual: exponential(x: -1.0),
      expected: 0.36787944117144233,
      tolerance: 1.0e-15,
    )
    and within(
      actual: exponential(x: 10.0),
      expected: 22026.465794806718,
      tolerance: 1.0e-10,
    )
    and exponential(x: 709.0) > 8.2e307
    and exponential(x: -745.0) > 0.0
    and exponential(x: -800.0) == 0.0
    and defects(of: math.exp(x: 710.0))
}

pub fn log_base_10(x: Float64) -> Float64 {
  requires x > 0.0;

  math.log10(x: x)
}

example log_base_10_of_powers_of_ten for log_base_10 {
  log_base_10(x: 1.0) == 0.0
    and within(
      actual: log_base_10(x: 1000.0),
      expected: 3.0,
      tolerance: 1.0e-14,
    )
    and within(
      actual: log_base_10(x: 2.0),
      expected: 0.3010299956639812,
      tolerance: 1.0e-15,
    )
    and defects(of: math.log10(x: 0.0))
    and defects(of: math.log10(x: -1.0))
}

pub fn log_base_2(x: Float64) -> Float64 {
  requires x > 0.0;

  math.log2(x: x)
}

example log_base_2_of_powers_of_two for log_base_2 {
  log_base_2(x: 1.0) == 0.0
    and log_base_2(x: 8.0) == 3.0
    and log_base_2(x: 0.25) == -2.0
    and within(
      actual: log_base_2(x: 10.0),
      expected: 3.321928094887362,
      tolerance: 1.0e-14,
    )
    and defects(of: math.log2(x: 0.0))
    and defects(of: math.log2(x: -1.0))
}

pub fn lower_whole(x: Float64) -> Float64 {
  math.floor(x: x)
}

example lower_whole_rounds_toward_minus_infinity for lower_whole {
  lower_whole(x: 2.5) == 2.0
    and lower_whole(x: -2.5) == -3.0
    and lower_whole(x: 3.0) == 3.0
    and lower_whole(x: -3.0) == -3.0
    and lower_whole(x: 0.0) == 0.0
    and lower_whole(x: 4503599627370497.0) == 4503599627370497.0
    and lower_whole(x: 0.49999999999999994) == 0.0
}

pub fn natural_log(x: Float64) -> Float64 {
  requires x > 0.0;

  math.ln(x: x)
}

example natural_log_has_its_known_values for natural_log {
  natural_log(x: 1.0) == 0.0
    and natural_log(x: 2.0) == 0.6931471805599453
    and natural_log(x: 10.0) == 2.302585092994046
    and natural_log(x: 0.5) == -0.6931471805599453
    and natural_log(x: math.e) == 1.0
    and natural_log(x: 1.0e-300) == -690.7755278982137
    and natural_log(x: 5.0e-324) == -744.4400719213812
    and defects(of: math.ln(x: 0.0))
    and defects(of: math.ln(x: -1.0))
}

pub fn nearest_whole(x: Float64) -> Float64 {
  math.round(x: x)
}

example nearest_whole_sends_halves_away_from_zero for nearest_whole {
  nearest_whole(x: 2.5) == 3.0
    and nearest_whole(x: -2.5) == -3.0
    and nearest_whole(x: 0.5) == 1.0
    and nearest_whole(x: -0.5) == -1.0
    and nearest_whole(x: 2.4) == 2.0
    and nearest_whole(x: -2.6) == -3.0
    and nearest_whole(x: 3.0) == 3.0
    and nearest_whole(x: 0.0) == 0.0
    and nearest_whole(x: 0.49999999999999994) == 0.0
}

pub fn radians(degrees: Float64) -> Float64 {
  requires degrees >= -1000000000.0 and degrees <= 1000000000.0;

  degrees * math.pi / 180.0
}

example radians_agrees_with_the_constants for radians {
  radians(degrees: 180.0) == math.pi
    and radians(degrees: 90.0) == math.half_pi
    and radians(degrees: 45.0) == math.quarter_pi
    and radians(degrees: 360.0) == math.tau
    and radians(degrees: 0.0) == 0.0
    and math.tau == 2.0 * math.pi
    and math.sqrt_2 == float.sqrt(value: 2.0)
    and math.ln_2 == math.ln(x: 2.0)
    and math.ln_10 == math.ln(x: 10.0)
    and within(actual: math.exp(x: 1.0), expected: math.e, tolerance: 1.0e-15)
}

pub fn sine(x: Float64) -> Float64 {
  requires x >= -500000000.0 and x <= 500000000.0;

  math.sin(x: x)
}

example sine_has_its_known_values for sine {
  sine(x: 0.0) == 0.0
    and sine(x: 1.0) == 0.8414709848078965
    and sine(x: -1.0) == -0.8414709848078965
    and sine(x: 3.0) == 0.1411200080598672
    and sine(x: 100.0) == -0.5063656411097588
    and sine(x: 500000000.0) <= 1.0000000000000002
    and defects(of: math.sin(x: 500000001.0))
    and defects(of: math.sin(x: -500000001.0))
}

pub fn square(base: Float64) -> Float64 {
  requires base >= -1.0e100 and base <= 1.0e100;

  base * base
}

example pow_agrees_with_known_powers for square {
  math.pow(base: 2.0, exponent: 10.0) == 1024.0
    and math.pow(base: 2.0, exponent: -2.0) == 0.25
    and math.pow(base: 10.0, exponent: 3.0) == 1000.0
    and math.pow(base: -2.0, exponent: 3.0) == -8.0
    and math.pow(base: -2.0, exponent: 2.0) == 4.0
    and math.pow(base: 0.0, exponent: 0.0) == 1.0
    and math.pow(base: 5.0, exponent: 0.0) == 1.0
    and math.pow(base: 0.0, exponent: 2.0) == 0.0
    and math.pow(base: 3.5, exponent: 2.0) == square(base: 3.5)
    and within(
      actual: math.pow(base: 2.0, exponent: 0.5),
      expected: math.sqrt_2,
      tolerance: 1.0e-15,
    )
    and math.pow(base: 10.0, exponent: -400.0) == 0.0
    and within(
      actual: math.pow(base: 10.0, exponent: 300.0) / 1.0e300,
      expected: 1.0,
      tolerance: 1.0e-12,
    )
    and defects(of: math.pow(base: 0.0, exponent: -1.0))
    and defects(of: math.pow(base: -2.0, exponent: 0.5))
    and defects(of: math.pow(base: 10.0, exponent: 400.0))
    and defects(of: math.pow(base: 10000000000.0, exponent: 40.0))
    and defects(of: math.pow(base: 2.0, exponent: 2147483649.0))
}

pub fn tangent(x: Float64) -> Float64 {
  requires x >= -500000000.0 and x <= 500000000.0;

  math.tan(x: x)
}

example tangent_has_its_known_values for tangent {
  tangent(x: 0.0) == 0.0
    and within(
      actual: tangent(x: 1.0),
      expected: 1.5574077246549023,
      tolerance: 1.0e-14,
    )
    and within(
      actual: tangent(x: -1.0),
      expected: -1.5574077246549023,
      tolerance: 1.0e-14,
    )
    and within(
      actual: tangent(x: math.quarter_pi),
      expected: 1.0,
      tolerance: 1.0e-14,
    )
    and defects(of: math.tan(x: 500000001.0))
    and defects(of: math.tan(x: -500000001.0))
}

pub fn to_float(count: Int) -> Float64 {
  requires count >= -9007199254740992 and count <= 9007199254740992;

  float.from_int(value: count)
}

example to_int_undoes_to_float for to_float {
  math.to_int(x: to_float(count: 12)) == 12
    and math.to_int(x: to_float(count: -7)) == -7
    and math.to_int(x: to_float(count: 0)) == 0
    and math.to_int(x: to_float(count: 255)) == 255
    and math.to_int(x: to_float(count: 65536)) == 65536
    and math.to_int(x: to_float(count: 4503599627370497)) == 4503599627370497
    and math.to_int(x: to_float(count: 9007199254740992)) == 9007199254740992
    and math.to_int(x: to_float(count: -9007199254740992)) == -9007199254740992
    and math.to_int(x: 12.0) == 12
    and defects(of: math.to_int(x: 1.5))
    and defects(of: math.to_int(x: -0.5))
    and defects(of: math.to_int(x: 9007199254740994.0))
    and defects(of: math.to_int(x: -9007199254740994.0))
}

pub fn toward_zero(x: Float64) -> Float64 {
  math.trunc(x: x)
}

example toward_zero_drops_the_fraction for toward_zero {
  toward_zero(x: 2.9) == 2.0
    and toward_zero(x: -2.9) == -2.0
    and toward_zero(x: 0.5) == 0.0
    and toward_zero(x: -0.5) == 0.0
    and toward_zero(x: 3.0) == 3.0
    and toward_zero(x: 0.0) == 0.0
}

pub fn within(actual: Float64, expected: Float64, tolerance: Float64) -> Bool {
  requires actual >= -1.0e100 and actual <= 1.0e100;
  requires expected >= -1.0e100 and expected <= 1.0e100;
  requires tolerance >= 0.0 and tolerance <= 1.0e100;

  float.abs(value: actual - expected) <= tolerance
}

example within_compares_with_a_tolerance for within {
  within(actual: 1.0, expected: 1.0, tolerance: 0.0)
    and within(actual: 1.0, expected: 1.05, tolerance: 0.1)
    and not within(actual: 1.0, expected: 1.5, tolerance: 0.1)
}
```

## 関連

- [float](float.md)：`abs`、`min`、`max`、`clamp`、`sqrt`、`from_int`、`parse`、`to_text`
- [int](int.md)：`Int` の関数
- [03 型](../language/03-types.md)：`Float64` と、桁あふれの扱い
- [10 検証](../language/10-verification.md)：証明義務と、証明不明（`W0504`）
