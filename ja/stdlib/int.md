# std.int

`Int`（桁数に上限のない整数）を扱う関数のモジュールです。`import std.int;` と書き、`int.abs(value: x)` のように呼びます。

`Int` の `+`、`-`、`*` は桁あふれしません。割り算の演算子 `/` は `Float64` どうしにだけ使えるので、`Int` の割り算と剰余は、このモジュールの `div_*` と `rem_*` で、丸め方を選んで書きます（[03 型](../language/03-types.md)）。ビット演算とシフトも、このモジュールの関数です。32 ビット・64 ビットで桁あふれする（折り返す）演算は、[word32 と word64](word32-word64.md) にあります。

## 関数一覧

| 関数 | 戻り値の型 | 意味 | 例 |
| --- | --- | --- | --- |
| `int.abs(value)` | `Int` | 絶対値 | `int.abs(value: -7)` → `7` |
| `int.sign(value)` | `Int` | 正なら `1`、負なら `-1`、`0` なら `0` | `int.sign(value: -40)` → `-1` |
| `int.min(first, second)` | `Int` | 小さいほう | `int.min(first: 3, second: 9)` → `3` |
| `int.max(first, second)` | `Int` | 大きいほう | `int.max(first: 3, second: 9)` → `9` |
| `int.clamp(value, low, high)` | `Int` | `low` 以上 `high` 以下に収める | `int.clamp(value: 12, low: 0, high: 9)` → `9` |
| `int.div_floor(dividend, divisor)` | `Int` | 商（負の無限大方向に丸める） | `int.div_floor(dividend: -7, divisor: 2)` → `-4` |
| `int.div_trunc(dividend, divisor)` | `Int` | 商（0 方向に丸める） | `int.div_trunc(dividend: -7, divisor: 2)` → `-3` |
| `int.div_euclid(dividend, divisor)` | `Int` | 商（剰余が 0 以上になるように丸める） | `int.div_euclid(dividend: 7, divisor: -2)` → `-3` |
| `int.rem_floor(dividend, divisor)` | `Int` | 剰余（`div_floor` と組） | `int.rem_floor(dividend: -7, divisor: 2)` → `1` |
| `int.rem_trunc(dividend, divisor)` | `Int` | 剰余（`div_trunc` と組） | `int.rem_trunc(dividend: -7, divisor: 2)` → `-1` |
| `int.rem_euclid(dividend, divisor)` | `Int` | 剰余（`div_euclid` と組） | `int.rem_euclid(dividend: 7, divisor: -2)` → `1` |
| `int.gcd(first, second)` | `Int` | 最大公約数 | `int.gcd(first: -12, second: 18)` → `6` |
| `int.pow(base, exponent)` | `Int` | べき乗 | `int.pow(base: 2, exponent: 100)` → `1267650600228229401496703205376` |
| `int.bit_and(first, second)` | `Int` | ビットごとの論理積 | `int.bit_and(first: 12, second: 10)` → `8` |
| `int.bit_or(first, second)` | `Int` | ビットごとの論理和 | `int.bit_or(first: 12, second: 10)` → `14` |
| `int.bit_xor(first, second)` | `Int` | ビットごとの排他的論理和 | `int.bit_xor(first: 12, second: 10)` → `6` |
| `int.bit_not(value)` | `Int` | ビットの反転（`-1 - value`） | `int.bit_not(value: 5)` → `-6` |
| `int.shift_left(value, count)` | `Int` | `count` ビット左へ | `int.shift_left(value: 3, count: 4)` → `48` |
| `int.shift_right(value, count)` | `Int` | `count` ビット右へ | `int.shift_right(value: -49, count: 4)` → `-4` |
| `int.test_bit(value, index)` | `Bool` | `index` ビット目（最下位が 0）が 1 か | `int.test_bit(value: 5, index: 2)` → `true` |
| `int.low_bits(value, count)` | `Int` | 下位 `count` ビットを 0 以上の数として取り出す | `int.low_bits(value: -1, count: 8)` → `255` |
| `int.bit_count(value)` | `Int` | 1 のビットの数 | `int.bit_count(value: 255)` → `8` |
| `int.bit_length(value)` | `Int` | 書くのに必要なビット数 | `int.bit_length(value: 256)` → `9` |
| `int.parse(text)` | `Option<Int>` | 10 進の文字列を読む | `int.parse(text: "-7")` → `Some(value: -7)` |
| `int.to_text(value)` | `String` | 10 進の文字列にする | `int.to_text(value: -12)` → `"-12"` |
| `int.power_of(base, exponent)` | `Int` | `spec fn`。仕様の中で使うべき乗 | |

引数のラベルは、一覧の `(...)` に書いたとおりです。

## 割り算と剰余

`Int` どうしの割り算は、商の丸め方が複数あるので、`div_floor`、`div_trunc`、`div_euclid` のどれかを選びます。割り切れるときは、3 つとも同じ値です。割り切れないとき、商は、正確な商（分数）を次のように整数へ丸めた値になります。

- `div_floor`：負の無限大の方向へ（床関数）。
- `div_trunc`：0 の方向へ（小数部分の切り捨て）。
- `div_euclid`：剰余が 0 以上になる方向へ。`divisor` が正なら `div_floor` と同じ、負なら天井（正の無限大の方向）です。

対応する `rem_floor`、`rem_trunc`、`rem_euclid` は、それぞれの商と組になる剰余で、次の式が常に成り立ちます。

```text
divisor * div_X(dividend, divisor) + rem_X(dividend, divisor) == dividend
```

剰余の符号と大きさは、組になる商の丸め方で決まります。

| 関数 | 結果の範囲 |
| --- | --- |
| `rem_floor` | 符号は `divisor` と同じ（または 0）。`divisor` が正なら 0 以上 `divisor` 未満、負なら `divisor` より大きく 0 以下 |
| `rem_trunc` | 符号は `dividend` と同じ（または 0）。絶対値は `abs(divisor)` 未満 |
| `rem_euclid` | 常に 0 以上 `abs(divisor)` 未満 |

`dividend` と `divisor` の符号を変えたときの結果は、次のとおりです。

| `dividend` | `divisor` | `div_floor` | `rem_floor` | `div_trunc` | `rem_trunc` | `div_euclid` | `rem_euclid` |
| ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 7 | 2 | 3 | 1 | 3 | 1 | 3 | 1 |
| -7 | 2 | -4 | 1 | -3 | -1 | -4 | 1 |
| 7 | -2 | -4 | -1 | -3 | 1 | -3 | 1 |
| -7 | -2 | 3 | -1 | 3 | -1 | 4 | 1 |

`dividend` と `divisor` がどちらも正なら、3 つの丸め方に違いはありません。違いが出るのは、負の数が混ざるときです。どれを選ぶかは、何をしたいかで決まります。

- 周期の中の位置（時計の針、配列の添字の巻き戻し）が欲しいときは、`rem_euclid` です。`divisor` の符号にかかわらず、0 以上の値が返ります。
- 「何個ぶんの区切りか」を数えるときは `div_floor` です。`dividend` が負の側にあっても、区切りが同じ幅で並びます。
- 絶対値で割ってから符号を戻す計算と同じにしたいときは、`div_trunc` と `rem_trunc` です。

`Int` の演算子 `/` が使えず、このように名前で選ぶのは、負の数を割ったときの結果が丸め方によって変わるからです。

事前条件は 3 つの商と 3 つの剰余に共通で、`requires divisor != 0;` です。`divisor` が `0` の呼び出しは欠陥です。

事後条件は、剰余にあります。`rem_euclid` は `result >= 0 and result < abs(value: divisor)`、`rem_trunc` は `result > 0 - abs(value: divisor) and result < abs(value: divisor)`、`rem_floor` は上の表のとおり、`divisor` の符号で範囲が決まります。

### 署名

```text
pub fn div_floor(dividend: Int, divisor: Int) -> Int
pub fn div_trunc(dividend: Int, divisor: Int) -> Int
pub fn div_euclid(dividend: Int, divisor: Int) -> Int
pub fn rem_floor(dividend: Int, divisor: Int) -> Int
pub fn rem_trunc(dividend: Int, divisor: Int) -> Int
pub fn rem_euclid(dividend: Int, divisor: Int) -> Int
```

## ビット演算とシフト

`bit_and`、`bit_or`、`bit_xor`、`bit_not`、`shift_left`、`shift_right`、`test_bit`、`low_bits` は、`Int` を**無限桁の 2 の補数**として読みます。非負の数は、上位に 0 が限りなく続くビット列です。負の数は、最も上位の 0 より上に、1 が限りなく続くビット列です。たとえば `-1` は、すべてのビットが 1 の数、`-8` は、下位 3 ビットが 0 で、それより上がすべて 1 の数です。

この読み方のため、桁数に上限がなく、結果は常に正確です。折り返しも、64 ビットの上限もありません。

- `bit_not(value)` は `-1 - value` です。`bit_not(value: 5)` は `-6`、`bit_not(value: -1)` は `0` です。
- `bit_and(first: -1, second: x)` は `x` です。`-1` はすべてのビットが 1 だからです。
- 負の数のビットは、`bit_and(first: -8, second: 15)` が `8` になるように、下位から見て読みます（`-8` の下位 4 ビットは `1000`）。

| 呼び出し | 結果 |
| --- | --- |
| `int.bit_and(first: 12, second: 10)` | `8` |
| `int.bit_and(first: -8, second: 15)` | `8` |
| `int.bit_and(first: -6, second: -3)` | `-8` |
| `int.bit_or(first: 12, second: 10)` | `14` |
| `int.bit_or(first: -8, second: 3)` | `-5` |
| `int.bit_xor(first: 12, second: 10)` | `6` |
| `int.bit_xor(first: -1, second: 5)` | `-6` |

事後条件の要点は次のとおりです。`bit_and` は、どちらかのオペランドが 0 以上なら、結果は 0 以上で、そのオペランド以下です。`bit_or` は、両方が 0 以上なら、結果はどちらのオペランド以上です。どちらかが負なら、結果は負です。`bit_xor` は、両方が 0 以上なら、結果は 0 以上です。

### シフト

`shift_left(value, count)` は `value` に 2 の `count` 乗を掛けた値、`shift_right(value, count)` は `value` を 2 の `count` 乗で割って**負の無限大の方向に丸めた**値です（`div_floor(dividend: value, divisor: 2^count)` に等しい）。負の数を右へシフトすると、上位から 1 が入ります。そのため、`shift_right(value: -49, count: 4)` は `-3` でなく `-4` で、`shift_right(value: -5, count: 100)` は `0` でなく `-1` です。

```text
pub fn shift_left(value: Int, count: Int) -> Int
pub fn shift_right(value: Int, count: Int) -> Int
```

事前条件は、どちらも `requires count >= 0 and count <= 16777216;` です。シフト量が負、または 16,777,216（2 の 24 乗）を超える呼び出しは欠陥です。上限があるのは、`shift_left` が、プログラムの使えるメモリを超える大きさの数を要求することを防ぐためです。

事後条件の要点は、`count == 0` なら結果は `value` と等しいこと、`shift_left` の結果は `value` が 0 以上なら `value` 以上、負なら `value` 以下であること、`shift_right` の結果は `value` と同じ符号側にあり（0 以上なら 0 以上 `value` 以下、負なら負で `value` 以上）、`value > 0 and count > 0` なら `value` より小さいことです。

### test_bit と low_bits

```text
pub fn test_bit(value: Int, index: Int) -> Bool
pub fn low_bits(value: Int, count: Int) -> Int
```

`test_bit` は、`value` の `index` ビット目（最下位が 0）が 1 かどうかを返します。負の数には 1 のビットが限りなくあるので、`test_bit(value: -1, index: 200)` は `true` です。事前条件は `requires index >= 0 and index <= 16777216;` です。

`low_bits` は、下位 `count` ビットを 0 以上の数として返します。`value` を 2 の `count` 乗で割った 0 以上の剰余（`rem_euclid`）と同じで、負の数は 2 の補数として読みます。`low_bits(value: -1, count: 8)` は `255` です。事前条件は `requires count >= 0 and count <= 16777216;`、事後条件は `result >= 0`、`count == 0` なら `result == 0` です。

### bit_count と bit_length

```text
pub fn bit_count(value: Int) -> Int
pub fn bit_length(value: Int) -> Int
```

どちらも 0 以上の数だけを受け取ります（`requires value >= 0;`。負の数を渡すのは欠陥です）。`bit_count` は 1 のビットの数、`bit_length` は書くのに必要なビット数です。`value < 2^n` となる最小の `n` で、`0` なら `0` です。`bit_length(value: 255)` は `8`、`bit_length(value: 256)` は `9` です。

## 数を扱う基本の関数

```text
pub fn abs(value: Int) -> Int
pub fn sign(value: Int) -> Int
pub fn min(first: Int, second: Int) -> Int
pub fn max(first: Int, second: Int) -> Int
pub fn clamp(value: Int, low: Int, high: Int) -> Int
pub fn gcd(first: Int, second: Int) -> Int
```

`abs` の結果は 0 以上で、`value` か `0 - value` のどちらかです。`sign` の結果は `-1`、`0`、`1` のどれかで、`sign(value) * abs(value)` は `value` に等しくなります。`min` と `max` は、引数のどちらかを返します。

`clamp` は、`value` が `low` 以上 `high` 以下ならそのまま、外ならば近いほうの端を返します。事前条件は `requires low <= high;` で、`low > high` の呼び出しは欠陥です。

`gcd` は最大公約数です。引数の符号は無視され、結果は常に 0 以上です。`gcd(first: 0, second: 5)` は `5` で、`gcd(first: 0, second: 0)` は `0` です（この場合だけ、結果が 0 です）。結果が 0 より大きいとき、どちらの引数も結果で割り切れます。

## pow

```text
pub fn pow(base: Int, exponent: Int) -> Int
```

`base` の `exponent` 乗です。結果は `Int` なので桁あふれせず、`pow(base: 2, exponent: 100)` は `1267650600228229401496703205376` です。`pow(base: 0, exponent: 0)` と、`exponent` が `0` のときは `1` です。

事前条件は `requires exponent >= 0;` です。`exponent` が負の呼び出しは欠陥です。事後条件は `ensures result == power_of(base: base, exponent: exponent);` です。

### power_of

```text
pub spec fn power_of(base: Int, exponent: Int) -> Int
```

`power_of` は `spec fn` で、`pow` の意味を与える仕様用の関数です。`power_of(base, 0)` は `1`、`power_of(base, n)` は `base * power_of(base, n - 1)` と定義されています。事前条件は `requires exponent >= 0;` です。

`spec fn` は、事前条件・事後条件・ループ不変条件や、`example`・`property` の中だけで呼べます。通常のコードから呼ぶと、エラー `E0331`（spec-function-in-code）です（[09 契約](../language/09-contracts.md)）。実行時に使うべき乗は `pow` です。`pow` を呼ぶ関数の契約に `power_of` を書くと、結果の値を仕様として述べられます。

## parse

```text
pub fn parse(text: String) -> Option<Int>
```

10 進の整数を読みます。受け付ける書式は、**省略できる `-` に、1 文字以上の 10 進数字（`0` から `9`）が続く**ものだけです。桁数に上限はありません。数字の前の `0` は許され、読んだ結果には影響しません（`"007"` は `7`、`"-007"` は `-7`、`"-0"` は `0`）。

次の文字列は、すべて `None` です。

| 文字列 | 理由 |
| --- | --- |
| `""` | 数字がない |
| `"-"` | 数字がない |
| `"+5"` | `+` は書けない |
| `" 5"`、`"5 "` | 空白は書けない |
| `"1.5"`、`"1e3"` | 小数点・指数は書けない |
| `"1_000"` | 区切りは書けない |
| `"0x10"` | 16 進は書けない |
| `"５"`（全角の数字） | `0` から `9` の文字だけ |

整数の書式でない文字列は、欠陥でなく `None` になります。`to_text` で書いた文字列は、`parse` で元の値に読み戻せます。

## to_text

```text
pub fn to_text(value: Int) -> String
```

10 進の文字列にします。負の数は先頭に `-` が付きます。`0` は `"0"` で、`+` や先頭の `0`、区切りは付きません。`to_text(value: -12)` は `"-12"`、`to_text(value: 123456789012345678901234567890)` は `"123456789012345678901234567890"` です。

## 例

```cernith
module guide.integers;

import std.int;

pub fn bit_at(value: Int, index: Int) -> Bool {
  requires index >= 0 and index <= 16777216;

  int.test_bit(value: value, index: index)
}

example bit_at_reads_one_bit for bit_at {
  bit_at(value: 5, index: 0)
    and not bit_at(value: 5, index: 1)
    and bit_at(value: 5, index: 2)
    and not bit_at(value: 5, index: 3)
    and bit_at(value: -1, index: 200)
    and not bit_at(value: 0, index: 0)
    and defects(of: int.test_bit(value: 5, index: -1))
    and defects(of: int.test_bit(value: 5, index: 16777217))
}

pub fn bit_operations(first: Int, second: Int) -> List<Int> {
  [
    int.bit_and(first: first, second: second),
    int.bit_or(first: first, second: second),
    int.bit_xor(first: first, second: second),
  ]
}

example bit_operations_read_negative_numbers_in_twos_complement for bit_operations {
  bit_operations(first: 12, second: 10) == [8, 14, 6]
    and bit_operations(first: -8, second: 15) == [8, -1, -9]
    and bit_operations(first: -6, second: -3) == [-8, -1, 7]
    and bit_operations(first: -1, second: 12345) == [12345, -1, -12346]
    and bit_operations(first: 0, second: 0) == [0, 0, 0]
    and int.bit_and(first: -8, second: 3) == 0
    and int.bit_or(first: -8, second: 3) == -5
    and int.bit_xor(first: -1, second: 5) == -6
    and int.bit_and(first: 340282366920938463463374607431768211455, second: 255)
      == 255
}

pub fn bit_totals(value: Int) -> List<Int> {
  requires value >= 0;

  [int.bit_count(value: value), int.bit_length(value: value)]
}

example bit_totals_count_ones_and_width for bit_totals {
  bit_totals(value: 255) == [8, 8]
    and bit_totals(value: 256) == [1, 9]
    and bit_totals(value: 0) == [0, 0]
    and bit_totals(value: 1) == [1, 1]
    and int.bit_count(value: 1267650600228229401496703205375) == 100
    and defects(of: int.bit_count(value: -1))
    and defects(of: int.bit_length(value: -1))
}

pub fn bounded(value: Int, low: Int, high: Int) -> Int {
  requires low <= high;

  int.clamp(value: value, low: low, high: high)
}

example bounded_inside_and_outside for bounded {
  bounded(value: 5, low: 0, high: 9) == 5
    and bounded(value: -3, low: 0, high: 9) == 0
    and bounded(value: 12, low: 0, high: 9) == 9
    and bounded(value: 0, low: 0, high: 9) == 0
    and bounded(value: 9, low: 0, high: 9) == 9
    and bounded(value: 4, low: 4, high: 4) == 4
    and defects(of: int.clamp(value: 5, low: 9, high: 0))
}

pub fn euclid_pair(dividend: Int, divisor: Int) -> List<Int> {
  requires divisor != 0;

  [
    int.div_euclid(dividend: dividend, divisor: divisor),
    int.rem_euclid(dividend: dividend, divisor: divisor),
  ]
}

example euclid_pair_keeps_the_remainder_non_negative for euclid_pair {
  euclid_pair(dividend: 7, divisor: 2) == [3, 1]
    and euclid_pair(dividend: -7, divisor: 2) == [-4, 1]
    and euclid_pair(dividend: 7, divisor: -2) == [-3, 1]
    and euclid_pair(dividend: -7, divisor: -2) == [4, 1]
    and euclid_pair(dividend: 0, divisor: 5) == [0, 0]
    and euclid_pair(dividend: -6, divisor: 3) == [-2, 0]
    and defects(of: int.div_euclid(dividend: 1, divisor: 0))
    and defects(of: int.rem_euclid(dividend: 1, divisor: 0))
}

pub fn flip(value: Int) -> Int {
  int.bit_not(value: value)
}

example flip_is_minus_one_minus_the_value for flip {
  flip(value: 0) == -1 and flip(value: 5) == -6 and flip(value: -1) == 0
}

pub fn floor_pair(dividend: Int, divisor: Int) -> List<Int> {
  requires divisor != 0;

  [
    int.div_floor(dividend: dividend, divisor: divisor),
    int.rem_floor(dividend: dividend, divisor: divisor),
  ]
}

example floor_pair_rounds_toward_negative_infinity for floor_pair {
  floor_pair(dividend: 7, divisor: 2) == [3, 1]
    and floor_pair(dividend: -7, divisor: 2) == [-4, 1]
    and floor_pair(dividend: 7, divisor: -2) == [-4, -1]
    and floor_pair(dividend: -7, divisor: -2) == [3, -1]
    and floor_pair(dividend: 0, divisor: -5) == [0, 0]
    and floor_pair(dividend: -6, divisor: 3) == [-2, 0]
    and defects(of: int.div_floor(dividend: 1, divisor: 0))
    and defects(of: int.rem_floor(dividend: 1, divisor: 0))
}

pub fn greatest_common_divisor(first: Int, second: Int) -> Int {
  int.gcd(first: first, second: second)
}

example greatest_common_divisor_ignores_signs for greatest_common_divisor {
  greatest_common_divisor(first: 12, second: 18) == 6
    and greatest_common_divisor(first: -12, second: 18) == 6
    and greatest_common_divisor(first: 0, second: 5) == 5
    and greatest_common_divisor(first: 5, second: 0) == 5
    and greatest_common_divisor(first: 0, second: 0) == 0
    and greatest_common_divisor(first: 7, second: 13) == 1
}

pub fn larger_and_smaller(first: Int, second: Int) -> List<Int> {
  [int.max(first: first, second: second), int.min(first: first, second: second)]
}

example larger_and_smaller_pick_one_of_the_two for larger_and_smaller {
  larger_and_smaller(first: 3, second: 9) == [9, 3]
    and larger_and_smaller(first: 9, second: 3) == [9, 3]
    and larger_and_smaller(first: -2, second: -2) == [-2, -2]
    and larger_and_smaller(first: -5, second: 0) == [0, -5]
}

pub fn low_part(value: Int, count: Int) -> Int {
  requires count >= 0 and count <= 16777216;

  int.low_bits(value: value, count: count)
}

example low_part_keeps_the_low_bits for low_part {
  low_part(value: 300, count: 8) == 44
    and low_part(value: -1, count: 8) == 255
    and low_part(value: -256, count: 8) == 0
    and low_part(value: 5, count: 0) == 0
    and int.low_bits(value: -1, count: 100)
      == int.pow(base: 2, exponent: 100) - 1
    and defects(of: int.low_bits(value: 5, count: -1))
    and defects(of: int.low_bits(value: 5, count: 16777217))
}

pub fn magnitude_and_sign(value: Int) -> List<Int> {
  [int.abs(value: value), int.sign(value: value)]
}

example magnitude_and_sign_of_three_cases for magnitude_and_sign {
  magnitude_and_sign(value: -7) == [7, -1]
    and magnitude_and_sign(value: 7) == [7, 1]
    and magnitude_and_sign(value: 0) == [0, 0]
    and magnitude_and_sign(value: -40) == [40, -1]
}

pub fn power(base: Int, exponent: Int) -> Int {
  requires exponent >= 0;
  ensures result == int.power_of(base: base, exponent: exponent);

  int.pow(base: base, exponent: exponent)
}

example power_of_small_and_large_numbers for power {
  power(base: 2, exponent: 10) == 1024
    and power(base: -3, exponent: 3) == -27
    and power(base: -2, exponent: 2) == 4
    and power(base: 5, exponent: 0) == 1
    and power(base: 0, exponent: 0) == 1
    and power(base: 0, exponent: 5) == 0
    and power(base: 2, exponent: 100) == 1267650600228229401496703205376
    and int.power_of(base: 2, exponent: 10) == 1024
    and defects(of: int.pow(base: 2, exponent: -1))
}

pub fn read_int(text: String) -> Option<Int> {
  int.parse(text: text)
}

example read_int_accepts_an_optional_minus_and_digits for read_int {
  read_int(text: "42") == Some(value: 42)
    and read_int(text: "-7") == Some(value: -7)
    and read_int(text: "007") == Some(value: 7)
    and read_int(text: "-007") == Some(value: -7)
    and read_int(text: "-0") == Some(value: 0)
    and read_int(text: "123456789012345678901234567890")
      == Some(value: 123456789012345678901234567890)
}

example read_int_rejects_everything_else for read_int {
  read_int(text: "") == None
    and read_int(text: "-") == None
    and read_int(text: "x") == None
    and read_int(text: "+5") == None
    and read_int(text: " 5") == None
    and read_int(text: "5 ") == None
    and read_int(text: "1.5") == None
    and read_int(text: "1e3") == None
    and read_int(text: "1_000") == None
    and read_int(text: "0x10") == None
    and read_int(text: "５") == None
}

pub fn shifted(value: Int, count: Int) -> List<Int> {
  requires count >= 0 and count <= 16777216;

  [
    int.shift_left(value: value, count: count),
    int.shift_right(value: value, count: count),
  ]
}

example shifted_multiplies_and_divides_by_a_power_of_two for shifted {
  shifted(value: 3, count: 4) == [48, 0]
    and shifted(value: -3, count: 4) == [-48, -1]
    and shifted(value: 48, count: 4) == [768, 3]
    and shifted(value: 49, count: 4) == [784, 3]
    and shifted(value: -48, count: 4) == [-768, -3]
    and shifted(value: -49, count: 4) == [-784, -4]
    and shifted(value: 7, count: 0) == [7, 7]
    and shifted(value: 5, count: 100) == [6338253001141147007483516026880, 0]
    and shifted(value: -5, count: 100) == [-6338253001141147007483516026880, -1]
    and int.shift_left(value: 1, count: 100) == 1267650600228229401496703205376
    and defects(of: int.shift_left(value: 1, count: -1))
    and defects(of: int.shift_left(value: 1, count: 16777217))
    and defects(of: int.shift_right(value: 1, count: -1))
    and defects(of: int.shift_right(value: 1, count: 16777217))
}

pub fn trunc_pair(dividend: Int, divisor: Int) -> List<Int> {
  requires divisor != 0;

  [
    int.div_trunc(dividend: dividend, divisor: divisor),
    int.rem_trunc(dividend: dividend, divisor: divisor),
  ]
}

example trunc_pair_rounds_toward_zero for trunc_pair {
  trunc_pair(dividend: 7, divisor: 2) == [3, 1]
    and trunc_pair(dividend: -7, divisor: 2) == [-3, -1]
    and trunc_pair(dividend: 7, divisor: -2) == [-3, 1]
    and trunc_pair(dividend: -7, divisor: -2) == [3, -1]
    and trunc_pair(dividend: 0, divisor: 5) == [0, 0]
    and trunc_pair(dividend: -6, divisor: 3) == [-2, 0]
    and defects(of: int.div_trunc(dividend: 1, divisor: 0))
    and defects(of: int.rem_trunc(dividend: 1, divisor: 0))
}

pub fn write_int(value: Int) -> String {
  int.to_text(value: value)
}

example write_int_is_plain_decimal for write_int {
  write_int(value: 0) == "0"
    and write_int(value: -12) == "-12"
    and write_int(value: 1000000) == "1000000"
    and write_int(value: 123456789012345678901234567890)
      == "123456789012345678901234567890"
}
```

`power_of` を通常のコードから呼ぶと、拒否されます。

```cernith-error E0331
module guide.spec_in_code;

import std.int;

pub fn cube(base: Int) -> Int {
  int.power_of(base: base, exponent: 3)
}
```

## 関連

- [word32 と word64](word32-word64.md)：32 ビット・64 ビットで折り返す演算
- [float](float.md)：`Float64` の関数（`from_int` は `Int` から変換する）
- [03 型](../language/03-types.md)：`Int` と `Float64`
- [09 契約](../language/09-contracts.md)：`spec fn`
