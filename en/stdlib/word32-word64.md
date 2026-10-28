# std.word32 and std.word64

These modules provide wraparound arithmetic on unsigned 32-bit and 64-bit integers, or **words**. Use them for calculations modulo 2³² or 2⁶⁴, such as hashes, checksums, and encodings. Import them separately with `import std.word32;` and `import std.word64;`, then call functions such as `word32.add(first: a, second: b)` or `word64.add(first: a, second: b)`.

Both modules have the same function names and signatures; only the word width differs. Below, `N` denotes that width: 32 or 64.

## Words

A **word** is an `Int` from 0 to 2ᴺ − 1 inclusive. It is not a dedicated type; `is_word` expresses that an `Int` value lies in this range.

```text
pub spec fn is_word(value: Int) -> Bool
```

`word32.is_word(value: x)` is `x >= 0 and x < 4294967296`, and `word64.is_word(value: x)` is `x >= 0 and x < 18446744073709551616`. Because `is_word` is a `spec fn`, it can be called only in specifications (preconditions, postconditions, and loop invariants) and in tests (`example` and `property`). Calling it from ordinary code is error `E0331` ([09 Contracts](../language/09-contracts.md)).

- The functions for addition, subtraction, multiplication, and the bitwise operations make it a **precondition** that the arguments are words and a **postcondition** that the result is a word. A call that passes an `Int` that is not a word (a negative number or a number of 2ᴺ or more) is a defect.
- Every operation discards the bits above 2ᴺ. For example, the result of `add` is the remainder of the sum divided by 2ᴺ.
- `wrap` is what turns an `Int` that is not a word into a word.

Because `Int` operations do not overflow, word operations can be read as `Int` operations ([int](int.md)) combined with `wrap`. You may use `Int` operations on words, but the results can be 2ᴺ or more, or negative.

## Function list

| Function | Return type | Meaning | Precondition |
| --- | --- | --- | --- |
| `word32.wrap(value)` | `Int` | Turns an `Int` into a word (the nonnegative remainder after division by 2ᴺ) | None |
| `word32.add(first, second)` | `Int` | Sum (modulo 2ᴺ) | Both are words |
| `word32.sub(first, second)` | `Int` | Difference (modulo 2ᴺ) | Both are words |
| `word32.mul(first, second)` | `Int` | Product (modulo 2ᴺ) | Both are words |
| `word32.bit_and(first, second)` | `Int` | Bitwise AND | Both are words |
| `word32.bit_or(first, second)` | `Int` | Bitwise OR | Both are words |
| `word32.bit_xor(first, second)` | `Int` | Bitwise exclusive OR | Both are words |
| `word32.bit_not(value)` | `Int` | Inversion of all N bits | A word |
| `word32.shift_left(value, count)` | `Int` | Logical left shift | `value` is a word, and `count` is from 0 to N |
| `word32.shift_right(value, count)` | `Int` | Logical right shift | `value` is a word, and `count` is from 0 to N |
| `word32.rotate_left(value, count)` | `Int` | Left rotation | `value` is a word |
| `word32.rotate_right(value, count)` | `Int` | Right rotation | `value` is a word |
| `word32.is_word(value)` | `Bool` | `spec fn`. Whether the value is a word | None |

The table is written with `word32`, but `word64` is the same. The postcondition of every function other than `is_word` is `is_word(value: result)` (the result is a word).

## wrap

```text
pub fn wrap(value: Int) -> Int
```

Turns an `Int` into the nonnegative remainder after division by 2ᴺ. Negative numbers are read as two’s complement. That is, `wrap(value: -1)` is the largest word (all bits are 1). There is no precondition; any `Int` can be passed.

| Call | Result |
| --- | --- |
| `word32.wrap(value: 4294967296 + 5)` | `5` |
| `word32.wrap(value: -1)` | `4294967295` |
| `word32.wrap(value: -4294967296)` | `0` |
| `word64.wrap(value: -1)` | `18446744073709551615` |
| `word64.wrap(value: 18446744073709551616)` | `0` |

## add, sub, and mul

```text
pub fn add(first: Int, second: Int) -> Int
pub fn sub(first: Int, second: Int) -> Int
pub fn mul(first: Int, second: Int) -> Int
```

These compute the sum, difference, or product and then `wrap` it. A difference below 0 and a sum or product of 2ᴺ or more wrap around.

| Call | Result |
| --- | --- |
| `word32.add(first: 4294967295, second: 1)` | `0` |
| `word32.sub(first: 0, second: 1)` | `4294967295` |
| `word32.mul(first: 4294967295, second: 4294967295)` | `1` |
| `word64.add(first: 18446744073709551615, second: 1)` | `0` |
| `word64.sub(first: 0, second: 1)` | `18446744073709551615` |

`add` and `sub` are inverses of each other: `sub(first: add(first: a, second: b), second: b)` returns `a`.

## Bitwise operations

```text
pub fn bit_and(first: Int, second: Int) -> Int
pub fn bit_or(first: Int, second: Int) -> Int
pub fn bit_xor(first: Int, second: Int) -> Int
pub fn bit_not(value: Int) -> Int
```

These compute bit by bit, treating the arguments as N-bit words. `bit_not` inverts all N bits, so `word32.bit_not(value: x)` is `4294967295 - x` and `word64.bit_not(value: x)` is `18446744073709551615 - x`. This differs from `int.bit_not(value)` in [int](int.md), which is `-1 - value` and therefore gives a negative number.

## Shifts

```text
pub fn shift_left(value: Int, count: Int) -> Int
pub fn shift_right(value: Int, count: Int) -> Int
```

Both are **logical shifts**. `shift_left` discards the bits that overflow N bits and shifts in 0 at the low end. `shift_right` discards the low bits and shifts in 0 at the high end. Because words are unsigned, a right shift has no sign extension, and `shift_right` is the value obtained by dividing by 2 to the power of `count` and discarding the fractional part.

The preconditions are that `value` is a word and that `count >= 0 and count <= N`. When `count` is N, all the bits are shifted out, so the result is `0`. A call with `count` negative or greater than N is a defect (unlike `int.shift_left` in [int](int.md), whose upper limit on `count` is much larger).

| Call | Result |
| --- | --- |
| `word32.shift_left(value: 1, count: 3)` | `8` |
| `word32.shift_left(value: 4294967295, count: 1)` | `4294967294` |
| `word32.shift_left(value: 1, count: 32)` | `0` |
| `word32.shift_right(value: 8, count: 3)` | `1` |
| `word32.shift_right(value: 4294967295, count: 1)` | `2147483647` |
| `word32.shift_right(value: 4294967295, count: 32)` | `0` |

## Rotations

```text
pub fn rotate_left(value: Int, count: Int) -> Int
pub fn rotate_right(value: Int, count: Int) -> Int
```

A rotation feeds the bits that leave at one end back in at the other end. No bits are lost. **The rotation amount is taken as the remainder after division by N** (0 or more and less than N), so `count` has no restriction: negative numbers and numbers of N or more can be passed. With `rotate_left`, a negative `count` effectively rotates to the right. `rotate_right` is the opposite direction, and `rotate_right(value: v, count: c)` gives the same result as `rotate_left(value: v, count: -c)`.

The only precondition is that `value` is a word.

| Call | Result |
| --- | --- |
| `word32.rotate_left(value: 2147483648, count: 1)` | `1` |
| `word32.rotate_left(value: 3, count: 2)` | `12` |
| `word32.rotate_left(value: 1, count: 32)` | `1` |
| `word32.rotate_left(value: 1, count: 33)` | `2` |
| `word32.rotate_left(value: 1, count: -1)` | `2147483648` |
| `word32.rotate_right(value: 1, count: 1)` | `2147483648` |
| `word32.rotate_right(value: 12, count: 2)` | `3` |

## Example

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

Calling `is_word` from ordinary code is rejected.

```cernith-error E0331
module guide.words_in_code;

import std.word32;

pub fn check(value: Int) -> Bool {
  word32.is_word(value: value)
}
```

## Related

- [int](int.md): bitwise operations and shifts for `Int`, which does not overflow
- [09 Contracts](../language/09-contracts.md): `spec fn`
