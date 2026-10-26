# std.word32 と std.word64

符号なしの 32 ビット・64 ビットの整数（**ワード**）を、折り返す演算（ラップアラウンド）で扱う 2 つのモジュールです。ハッシュ、チェックサム、符号化のように、2³² や 2⁶⁴ を法として計算するときに使います。`import std.word32;` と `import std.word64;` で別々に取り込み、`word32.add(first: a, second: b)`、`word64.add(first: a, second: b)` のように呼びます。

2 つのモジュールの関数は、名前も署名も同じで、違いは語長（32 か 64）だけです。以下では、語長を `N` と書き、`N` は 32 か 64 です。

## ワード

**ワード**は、0 以上 2ᴺ − 1 以下の `Int` です。専用の型ではなく、`Int` の値がこの範囲にあることを、`is_word` が表します。

```text
pub spec fn is_word(value: Int) -> Bool
```

`word32.is_word(value: x)` は `x >= 0 and x < 4294967296`、`word64.is_word(value: x)` は `x >= 0 and x < 18446744073709551616` です。`is_word` は `spec fn` なので、仕様（事前条件・事後条件・ループ不変条件）とテスト（`example` と `property`）の中でだけ呼べます。通常のコードから呼ぶと、エラー `E0331` です（[09 契約](../language/09-contracts.md)）。

- 加減乗とビット演算の関数は、引数がワードであることを**事前条件**にし、結果がワードであることを**事後条件**にします。ワードでない `Int`（負の数や 2ᴺ 以上の数）を渡す呼び出しは、欠陥です。
- どの演算も、2ᴺ を超えた上位のビットを捨てます。たとえば `add` の結果は、和を 2ᴺ で割った余りです。
- ワードでない `Int` をワードにするのは `wrap` です。

`Int` の演算は桁あふれしないので、ワードの演算は `Int` の演算（[int](int.md)）に `wrap` を組み合わせたものとして読めます。ワードに対して `Int` の演算を使っても構いませんが、その結果は 2ᴺ 以上や負になりえます。

## 関数一覧

| 関数 | 戻り値の型 | 意味 | 事前条件 |
| --- | --- | --- | --- |
| `word32.wrap(value)` | `Int` | `Int` をワードにする（2ᴺ で割った 0 以上の余り） | なし |
| `word32.add(first, second)` | `Int` | 和（2ᴺ を法とする） | 2 つともワード |
| `word32.sub(first, second)` | `Int` | 差（2ᴺ を法とする） | 2 つともワード |
| `word32.mul(first, second)` | `Int` | 積（2ᴺ を法とする） | 2 つともワード |
| `word32.bit_and(first, second)` | `Int` | ビットごとの論理積 | 2 つともワード |
| `word32.bit_or(first, second)` | `Int` | ビットごとの論理和 | 2 つともワード |
| `word32.bit_xor(first, second)` | `Int` | ビットごとの排他的論理和 | 2 つともワード |
| `word32.bit_not(value)` | `Int` | N 個のビットすべての反転 | ワード |
| `word32.shift_left(value, count)` | `Int` | 論理左シフト | `value` がワード、`count` は 0 以上 N 以下 |
| `word32.shift_right(value, count)` | `Int` | 論理右シフト | `value` がワード、`count` は 0 以上 N 以下 |
| `word32.rotate_left(value, count)` | `Int` | 左回転 | `value` がワード |
| `word32.rotate_right(value, count)` | `Int` | 右回転 | `value` がワード |
| `word32.is_word(value)` | `Bool` | `spec fn`。ワードか | なし |

表では `word32` と書きましたが、`word64` も同じです。事後条件は、`is_word` 以外のすべての関数で `is_word(value: result)`（結果はワード）です。

## wrap

```text
pub fn wrap(value: Int) -> Int
```

`Int` を、2ᴺ で割った 0 以上の余りにします。負の数は 2 の補数として読みます。つまり、`wrap(value: -1)` は、最大のワード（すべてのビットが 1）です。事前条件はなく、どんな `Int` でも渡せます。

| 呼び出し | 結果 |
| --- | --- |
| `word32.wrap(value: 4294967296 + 5)` | `5` |
| `word32.wrap(value: -1)` | `4294967295` |
| `word32.wrap(value: -4294967296)` | `0` |
| `word64.wrap(value: -1)` | `18446744073709551615` |
| `word64.wrap(value: 18446744073709551616)` | `0` |

## add、sub、mul

```text
pub fn add(first: Int, second: Int) -> Int
pub fn sub(first: Int, second: Int) -> Int
pub fn mul(first: Int, second: Int) -> Int
```

和、差、積を計算して、`wrap` します。結果が 0 未満になる差や、2ᴺ 以上になる和・積は、折り返します。

| 呼び出し | 結果 |
| --- | --- |
| `word32.add(first: 4294967295, second: 1)` | `0` |
| `word32.sub(first: 0, second: 1)` | `4294967295` |
| `word32.mul(first: 4294967295, second: 4294967295)` | `1` |
| `word64.add(first: 18446744073709551615, second: 1)` | `0` |
| `word64.sub(first: 0, second: 1)` | `18446744073709551615` |

`add` と `sub` は互いの逆で、`sub(first: add(first: a, second: b), second: b)` は `a` に戻ります。

## ビット演算

```text
pub fn bit_and(first: Int, second: Int) -> Int
pub fn bit_or(first: Int, second: Int) -> Int
pub fn bit_xor(first: Int, second: Int) -> Int
pub fn bit_not(value: Int) -> Int
```

N ビットのワードとして、ビットごとに計算します。`bit_not` は N 個のビットすべてを反転するので、`word32.bit_not(value: x)` は `4294967295 - x`、`word64.bit_not(value: x)` は `18446744073709551615 - x` です。[int](int.md) の `int.bit_not(value)` は `-1 - value` で、負の数になるのとは異なります。

## シフト

```text
pub fn shift_left(value: Int, count: Int) -> Int
pub fn shift_right(value: Int, count: Int) -> Int
```

どちらも**論理シフト**です。`shift_left` は、N ビットからあふれたビットを捨て、下位に 0 を入れます。`shift_right` は、下位のビットを捨て、上位に 0 を入れます。ワードは符号なしなので、右シフトに符号の拡張はなく、`shift_right` は、2 の `count` 乗で割って、小数点以下を切り捨てた値です。

事前条件は、`value` がワードであることと、`count >= 0 and count <= N` です。`count` が N のときは、すべてのビットが出ていくので、結果は `0` です。`count` が負、または N を超える呼び出しは欠陥です（[int](int.md) の `int.shift_left` は、`count` の上限がもっと大きいのと異なります）。

| 呼び出し | 結果 |
| --- | --- |
| `word32.shift_left(value: 1, count: 3)` | `8` |
| `word32.shift_left(value: 4294967295, count: 1)` | `4294967294` |
| `word32.shift_left(value: 1, count: 32)` | `0` |
| `word32.shift_right(value: 8, count: 3)` | `1` |
| `word32.shift_right(value: 4294967295, count: 1)` | `2147483647` |
| `word32.shift_right(value: 4294967295, count: 32)` | `0` |

## 回転

```text
pub fn rotate_left(value: Int, count: Int) -> Int
pub fn rotate_right(value: Int, count: Int) -> Int
```

回転は、一方から出ていくビットを、もう一方から入れます。ビットは失われません。**回転量は N で割った余り**（0 以上 N 未満）として扱われるので、`count` に制限はありません。負の数も、N 以上の数も渡せます。`rotate_left` で `count` が負なら、実質的に右へ回転します。`rotate_right` は逆向きで、`rotate_right(value: v, count: c)` は `rotate_left(value: v, count: -c)` と同じ結果です。

事前条件は、`value` がワードであることだけです。

| 呼び出し | 結果 |
| --- | --- |
| `word32.rotate_left(value: 2147483648, count: 1)` | `1` |
| `word32.rotate_left(value: 3, count: 2)` | `12` |
| `word32.rotate_left(value: 1, count: 32)` | `1` |
| `word32.rotate_left(value: 1, count: 33)` | `2` |
| `word32.rotate_left(value: 1, count: -1)` | `2147483648` |
| `word32.rotate_right(value: 1, count: 1)` | `2147483648` |
| `word32.rotate_right(value: 12, count: 2)` | `3` |

## 例

```cernith
module guide.words;

import std.word32;
import std.word64;

pub fn arithmetic32(first: Int, second: Int) -> List<Int> {
  requires word32.is_word(value: first) and word32.is_word(value: second);

  [
    word32.add(first: first, second: second),
    word32.sub(first: first, second: second),
    word32.mul(first: first, second: second),
  ]
}

example arithmetic32_wraps_around for arithmetic32 {
  arithmetic32(first: 4294967295, second: 1) == [0, 4294967294, 4294967295]
    and arithmetic32(first: 7, second: 8) == [15, 4294967295, 56]
    and arithmetic32(first: 4294967295, second: 4294967295)
      == [4294967294, 0, 1]
    and arithmetic32(first: 0, second: 1) == [1, 4294967295, 0]
    and arithmetic32(first: 0, second: 0) == [0, 0, 0]
    and arithmetic32(first: 2147483648, second: 2)
      == [2147483650, 2147483646, 0]
    and word32.sub(first: 9, second: 4) == 5
    and word32.mul(first: 6, second: 7) == 42
    and defects(of: word32.add(first: 4294967296, second: 0))
    and defects(of: word32.add(first: -1, second: 0))
    and defects(of: word32.sub(first: 0, second: 4294967296))
    and defects(of: word32.mul(first: 0, second: -1))
}

pub fn arithmetic64(first: Int, second: Int) -> List<Int> {
  requires word64.is_word(value: first) and word64.is_word(value: second);

  [
    word64.add(first: first, second: second),
    word64.sub(first: first, second: second),
    word64.mul(first: first, second: second),
  ]
}

example arithmetic64_wraps_around for arithmetic64 {
  arithmetic64(first: 18446744073709551615, second: 1)
    == [0, 18446744073709551614, 18446744073709551615]
    and arithmetic64(first: 7, second: 8) == [15, 18446744073709551615, 56]
    and arithmetic64(first: 18446744073709551615, second: 18446744073709551615)
      == [18446744073709551614, 0, 1]
    and arithmetic64(first: 0, second: 0) == [0, 0, 0]
    and arithmetic64(first: 9223372036854775808, second: 2)
      == [9223372036854775810, 9223372036854775806, 0]
    and defects(of: word64.add(first: 18446744073709551616, second: 0))
    and defects(of: word64.sub(first: -1, second: 0))
    and defects(of: word64.mul(first: 0, second: -1))
}

pub fn bitwise32(first: Int, second: Int) -> List<Int> {
  requires word32.is_word(value: first) and word32.is_word(value: second);

  [
    word32.bit_and(first: first, second: second),
    word32.bit_or(first: first, second: second),
    word32.bit_xor(first: first, second: second),
  ]
}

example bitwise32_works_on_32_bits for bitwise32 {
  bitwise32(first: 4294967295, second: 255) == [255, 4294967295, 4294967040]
    and bitwise32(first: 2147483648, second: 1) == [0, 2147483649, 2147483649]
    and bitwise32(first: 5, second: 3) == [1, 7, 6]
    and bitwise32(first: 4294967295, second: 4294967295)
      == [4294967295, 4294967295, 0]
    and bitwise32(first: 0, second: 0) == [0, 0, 0]
    and defects(of: word32.bit_and(first: 4294967296, second: 0))
    and defects(of: word32.bit_or(first: -1, second: 0))
    and defects(of: word32.bit_xor(first: 0, second: -1))
}

pub fn bitwise64(first: Int, second: Int) -> List<Int> {
  requires word64.is_word(value: first) and word64.is_word(value: second);

  [
    word64.bit_and(first: first, second: second),
    word64.bit_or(first: first, second: second),
    word64.bit_xor(first: first, second: second),
  ]
}

example bitwise64_works_on_64_bits for bitwise64 {
  bitwise64(first: 18446744073709551615, second: 255)
    == [255, 18446744073709551615, 18446744073709551360]
    and bitwise64(first: 9223372036854775808, second: 1)
      == [0, 9223372036854775809, 9223372036854775809]
    and bitwise64(first: 5, second: 3) == [1, 7, 6]
    and bitwise64(first: 0, second: 0) == [0, 0, 0]
    and defects(of: word64.bit_and(first: 18446744073709551616, second: 0))
    and defects(of: word64.bit_or(first: -1, second: 0))
    and defects(of: word64.bit_xor(first: 0, second: -1))
}

pub fn flipped(value: Int) -> List<Int> {
  requires word32.is_word(value: value) and word64.is_word(value: value);

  [word32.bit_not(value: value), word64.bit_not(value: value)]
}

example flipped_inverts_every_bit_of_the_word for flipped {
  flipped(value: 0) == [4294967295, 18446744073709551615]
    and flipped(value: 255) == [4294967295 - 255, 18446744073709551615 - 255]
    and flipped(value: 4294967295) == [0, 18446744073709551615 - 4294967295]
    and word32.bit_not(value: 4294967295) == 0
    and word64.bit_not(value: 18446744073709551615) == 0
    and defects(of: word32.bit_not(value: 4294967296))
    and defects(of: word32.bit_not(value: -1))
    and defects(of: word64.bit_not(value: 18446744073709551616))
}

pub fn rotations32(value: Int, count: Int) -> List<Int> {
  requires word32.is_word(value: value);

  [
    word32.rotate_left(value: value, count: count),
    word32.rotate_right(value: value, count: count),
  ]
}

example rotations32_take_the_count_modulo_32 for rotations32 {
  rotations32(value: 2147483648, count: 1) == [1, 1073741824]
    and rotations32(value: 3, count: 2) == [12, 3221225472]
    and rotations32(value: 1, count: 32) == [1, 1]
    and rotations32(value: 1, count: 33) == [2, 2147483648]
    and rotations32(value: 1, count: -1) == [2147483648, 2]
    and rotations32(value: 1, count: 0) == [1, 1]
    and rotations32(value: 0, count: 5) == [0, 0]
    and word32.rotate_right(value: 1, count: 1) == 2147483648
    and word32.rotate_right(value: 12, count: 2) == 3
    and defects(of: word32.rotate_left(value: 4294967296, count: 1))
    and defects(of: word32.rotate_right(value: -1, count: 1))
}

pub fn rotations64(value: Int, count: Int) -> List<Int> {
  requires word64.is_word(value: value);

  [
    word64.rotate_left(value: value, count: count),
    word64.rotate_right(value: value, count: count),
  ]
}

example rotations64_take_the_count_modulo_64 for rotations64 {
  rotations64(value: 9223372036854775808, count: 1) == [1, 4611686018427387904]
    and rotations64(value: 1, count: 64) == [1, 1]
    and rotations64(value: 1, count: 65) == [2, 9223372036854775808]
    and rotations64(value: 1, count: -1) == [9223372036854775808, 2]
    and rotations64(value: 0, count: 5) == [0, 0]
    and word64.rotate_right(value: 1, count: 1) == 9223372036854775808
    and defects(of: word64.rotate_left(value: 18446744073709551616, count: 1))
    and defects(of: word64.rotate_right(value: -1, count: 1))
}

pub fn shifts32(value: Int, count: Int) -> List<Int> {
  requires word32.is_word(value: value) and count >= 0 and count <= 32;

  [
    word32.shift_left(value: value, count: count),
    word32.shift_right(value: value, count: count),
  ]
}

example shifts32_are_logical for shifts32 {
  shifts32(value: 1, count: 3) == [8, 0]
    and shifts32(value: 8, count: 3) == [64, 1]
    and shifts32(value: 4294967295, count: 1) == [4294967294, 2147483647]
    and shifts32(value: 4294967295, count: 32) == [0, 0]
    and shifts32(value: 1, count: 32) == [0, 0]
    and shifts32(value: 5, count: 0) == [5, 5]
    and shifts32(value: 0, count: 7) == [0, 0]
    and defects(of: word32.shift_left(value: 1, count: 33))
    and defects(of: word32.shift_right(value: 1, count: 33))
    and defects(of: word32.shift_left(value: 1, count: -1))
    and defects(of: word32.shift_right(value: 4294967296, count: 1))
}

pub fn shifts64(value: Int, count: Int) -> List<Int> {
  requires word64.is_word(value: value) and count >= 0 and count <= 64;

  [
    word64.shift_left(value: value, count: count),
    word64.shift_right(value: value, count: count),
  ]
}

example shifts64_are_logical for shifts64 {
  shifts64(value: 1, count: 3) == [8, 0]
    and shifts64(value: 18446744073709551615, count: 1)
      == [18446744073709551614, 9223372036854775807]
    and shifts64(value: 18446744073709551615, count: 64) == [0, 0]
    and shifts64(value: 5, count: 0) == [5, 5]
    and defects(of: word64.shift_left(value: 1, count: 65))
    and defects(of: word64.shift_right(value: 1, count: 65))
    and defects(of: word64.shift_left(value: 1, count: -1))
}

pub fn wrapped(value: Int) -> List<Int> {
  [word32.wrap(value: value), word64.wrap(value: value)]
}

example wrapped_reads_a_number_modulo_two_to_the_n for wrapped {
  wrapped(value: 0) == [0, 0]
    and wrapped(value: -1) == [4294967295, 18446744073709551615]
    and wrapped(value: 4294967296) == [0, 4294967296]
    and wrapped(value: 4294967296 + 5) == [5, 4294967301]
    and wrapped(value: -4294967296) == [0, 18446744073709551616 - 4294967296]
    and wrapped(value: 18446744073709551616) == [0, 0]
    and wrapped(value: 18446744073709551616 + 5) == [5, 5]
    and word32.wrap(value: 1267650600228229401496703205383) == 7
    and word32.is_word(value: 0)
    and word32.is_word(value: 4294967295)
    and not word32.is_word(value: 4294967296)
    and not word32.is_word(value: -1)
    and word64.is_word(value: 4294967296)
    and word64.is_word(value: 18446744073709551615)
    and not word64.is_word(value: 18446744073709551616)
    and not word64.is_word(value: -1)
}
```

`is_word` を通常のコードから呼ぶと、拒否されます。

```cernith-error E0331
module guide.words_in_code;

import std.word32;

pub fn check(value: Int) -> Bool {
  word32.is_word(value: value)
}
```

## 関連

- [int](int.md)：桁あふれしない `Int` のビット演算とシフト
- [09 契約](../language/09-contracts.md)：`spec fn`
