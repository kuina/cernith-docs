# std.int

The module of functions for `Int` (integers with no limit on the number of digits). Write `import std.int;` and call a function as in `int.abs(value: x)`.

The `+`, `-`, and `*` operators on `Int` never overflow. The division operator `/` can be used only on `Float64` values, so division and remainder on `Int` are written with `div_*` and `rem_*` from this module, choosing how the result is rounded ([03 Types](../language/03-types.md)). Bitwise operations and shifts are also functions of this module. Operations that overflow (wrap around) at 32 or 64 bits are in [word32 and word64](word32-word64.md).

## Function list

| Function | Return type | Meaning | Example |
| --- | --- | --- | --- |
| `int.abs(value)` | `Int` | Absolute value | `int.abs(value: -7)` → `7` |
| `int.sign(value)` | `Int` | `1` if positive, `-1` if negative, `0` if `0` | `int.sign(value: -40)` → `-1` |
| `int.min(first, second)` | `Int` | The smaller one | `int.min(first: 3, second: 9)` → `3` |
| `int.max(first, second)` | `Int` | The larger one | `int.max(first: 3, second: 9)` → `9` |
| `int.clamp(value, low, high)` | `Int` | Restrict to the range from `low` to `high` | `int.clamp(value: 12, low: 0, high: 9)` → `9` |
| `int.div_floor(dividend, divisor)` | `Int` | Quotient (rounded toward negative infinity) | `int.div_floor(dividend: -7, divisor: 2)` → `-4` |
| `int.div_trunc(dividend, divisor)` | `Int` | Quotient (rounded toward zero) | `int.div_trunc(dividend: -7, divisor: 2)` → `-3` |
| `int.div_euclid(dividend, divisor)` | `Int` | Quotient (rounded so that the remainder is 0 or greater) | `int.div_euclid(dividend: 7, divisor: -2)` → `-3` |
| `int.rem_floor(dividend, divisor)` | `Int` | Remainder (pairs with `div_floor`) | `int.rem_floor(dividend: -7, divisor: 2)` → `1` |
| `int.rem_trunc(dividend, divisor)` | `Int` | Remainder (pairs with `div_trunc`) | `int.rem_trunc(dividend: -7, divisor: 2)` → `-1` |
| `int.rem_euclid(dividend, divisor)` | `Int` | Remainder (pairs with `div_euclid`) | `int.rem_euclid(dividend: 7, divisor: -2)` → `1` |
| `int.gcd(first, second)` | `Int` | Greatest common divisor | `int.gcd(first: -12, second: 18)` → `6` |
| `int.pow(base, exponent)` | `Int` | Power (exponentiation) | `int.pow(base: 2, exponent: 100)` → `1267650600228229401496703205376` |
| `int.bit_and(first, second)` | `Int` | Bitwise AND | `int.bit_and(first: 12, second: 10)` → `8` |
| `int.bit_or(first, second)` | `Int` | Bitwise OR | `int.bit_or(first: 12, second: 10)` → `14` |
| `int.bit_xor(first, second)` | `Int` | Bitwise exclusive OR | `int.bit_xor(first: 12, second: 10)` → `6` |
| `int.bit_not(value)` | `Int` | Bit inversion (`-1 - value`) | `int.bit_not(value: 5)` → `-6` |
| `int.shift_left(value, count)` | `Int` | Shift left by `count` bits | `int.shift_left(value: 3, count: 4)` → `48` |
| `int.shift_right(value, count)` | `Int` | Shift right by `count` bits | `int.shift_right(value: -49, count: 4)` → `-4` |
| `int.test_bit(value, index)` | `Bool` | Whether bit number `index` (the lowest bit is 0) is 1 | `int.test_bit(value: 5, index: 2)` → `true` |
| `int.low_bits(value, count)` | `Int` | Extract the lowest `count` bits as a number that is 0 or greater | `int.low_bits(value: -1, count: 8)` → `255` |
| `int.bit_count(value)` | `Int` | Number of 1 bits | `int.bit_count(value: 255)` → `8` |
| `int.bit_length(value)` | `Int` | Number of bits needed to write the value | `int.bit_length(value: 256)` → `9` |
| `int.parse(text)` | `Option<Int>` | Read a decimal string | `int.parse(text: "-7")` → `Some(value: -7)` |
| `int.to_text(value)` | `String` | Convert to a decimal string | `int.to_text(value: -12)` → `"-12"` |
| `int.power_of(base, exponent)` | `Int` | A `spec fn`. Exponentiation for use in specifications | |

The argument labels are as written in the `(...)` of the list.

## Division and remainder

Division of `Int` values can round the quotient in several ways, so you choose one of `div_floor`, `div_trunc`, and `div_euclid`. When the division is exact, all three give the same value. When it is not, the quotient is the exact quotient (a fraction) rounded to an integer as follows.

- `div_floor`: toward negative infinity (the floor function).
- `div_trunc`: toward zero (discarding the fractional part).
- `div_euclid`: in the direction that makes the remainder 0 or greater. If `divisor` is positive, this is the same as `div_floor`; if it is negative, it is the ceiling (toward positive infinity).

The corresponding `rem_floor`, `rem_trunc`, and `rem_euclid` are the remainders that pair with each quotient, and the following equation always holds.

```text
divisor * div_X(dividend, divisor) + rem_X(dividend, divisor) == dividend
```

The sign and magnitude of the remainder are determined by how the paired quotient is rounded.

| Function | Range of the result |
| --- | --- |
| `rem_floor` | The sign is the same as that of `divisor` (or 0). If `divisor` is positive, 0 or greater and less than `divisor`; if negative, greater than `divisor` and 0 or less |
| `rem_trunc` | The sign is the same as that of `dividend` (or 0). The absolute value is less than `abs(divisor)` |
| `rem_euclid` | Always 0 or greater and less than `abs(divisor)` |

The results for each combination of signs of `dividend` and `divisor` are as follows.

| `dividend` | `divisor` | `div_floor` | `rem_floor` | `div_trunc` | `rem_trunc` | `div_euclid` | `rem_euclid` |
| ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 7 | 2 | 3 | 1 | 3 | 1 | 3 | 1 |
| -7 | 2 | -4 | 1 | -3 | -1 | -4 | 1 |
| 7 | -2 | -4 | -1 | -3 | 1 | -3 | 1 |
| -7 | -2 | 3 | -1 | 3 | -1 | 4 | 1 |

When `dividend` and `divisor` are both positive, the three ways of rounding make no difference. The difference appears when a negative number is involved. Which one to choose depends on what you want to do.

- When you want a position within a cycle (the hands of a clock, wrapping around an array index), use `rem_euclid`. It returns a value that is 0 or greater regardless of the sign of `divisor`.
- When you count “how many intervals fit,” use `div_floor`. Even when `dividend` is on the negative side, the intervals line up with the same width.
- When you want the same result as dividing the absolute values and then restoring the sign, use `div_trunc` and `rem_trunc`.

The operator `/` is not available for `Int`, and the rounding is chosen by name like this, because the result of dividing negative numbers changes depending on how it is rounded.

The precondition is common to the three quotients and the three remainders: `requires divisor != 0;`. A call where `divisor` is `0` is a defect.

The postconditions are on the remainders. `rem_euclid` has `result >= 0 and result < abs(value: divisor)`, `rem_trunc` has `result > 0 - abs(value: divisor) and result < abs(value: divisor)`, and for `rem_floor` the range depends on the sign of `divisor`, as in the table above.

### Signatures

```text
pub fn div_floor(dividend: Int, divisor: Int) -> Int
pub fn div_trunc(dividend: Int, divisor: Int) -> Int
pub fn div_euclid(dividend: Int, divisor: Int) -> Int
pub fn rem_floor(dividend: Int, divisor: Int) -> Int
pub fn rem_trunc(dividend: Int, divisor: Int) -> Int
pub fn rem_euclid(dividend: Int, divisor: Int) -> Int
```

## Bitwise operations and shifts

`bit_and`, `bit_or`, `bit_xor`, `bit_not`, `shift_left`, `shift_right`, `test_bit`, and `low_bits` read an `Int` as an **infinite-width two’s complement** number. A nonnegative number is a bit string with 0 continuing endlessly in the upper positions. A negative number is a bit string with 1 continuing endlessly above the highest 0. For example, `-1` is the number whose bits are all 1, and `-8` is the number whose lowest three bits are 0 and whose bits above them are all 1.

Because of this way of reading, there is no limit on the number of digits and the result is always exact. There is no wrapping around and no 64-bit limit.

- `bit_not(value)` is `-1 - value`. `bit_not(value: 5)` is `-6`, and `bit_not(value: -1)` is `0`.
- `bit_and(first: -1, second: x)` is `x`, because `-1` has all bits set to 1.
- The bits of a negative number are read from the lowest position, so that `bit_and(first: -8, second: 15)` is `8` (the lowest four bits of `-8` are `1000`).

| Call | Result |
| --- | --- |
| `int.bit_and(first: 12, second: 10)` | `8` |
| `int.bit_and(first: -8, second: 15)` | `8` |
| `int.bit_and(first: -6, second: -3)` | `-8` |
| `int.bit_or(first: 12, second: 10)` | `14` |
| `int.bit_or(first: -8, second: 3)` | `-5` |
| `int.bit_xor(first: 12, second: 10)` | `6` |
| `int.bit_xor(first: -1, second: 5)` | `-6` |

The main points of the postconditions are as follows. For `bit_and`, if either operand is 0 or greater, the result is 0 or greater and does not exceed that operand. For `bit_or`, if both are 0 or greater, the result is greater than or equal to both operands; if either is negative, the result is negative. For `bit_xor`, if both are 0 or greater, the result is 0 or greater.

### Shifts

`shift_left(value, count)` is `value` multiplied by 2 to the power `count`, and `shift_right(value, count)` is `value` divided by 2 to the power `count` and **rounded toward negative infinity** (equal to `div_floor(dividend: value, divisor: 2^count)`). When a negative number is shifted right, 1 bits enter from the top. For that reason, `shift_right(value: -49, count: 4)` is `-4`, not `-3`, and `shift_right(value: -5, count: 100)` is `-1`, not `0`.

```text
pub fn shift_left(value: Int, count: Int) -> Int
pub fn shift_right(value: Int, count: Int) -> Int
```

The precondition of both is `requires count >= 0 and count <= 16777216;`. A call where the shift amount is negative or exceeds 16,777,216 (2 to the 24th power) is a defect. The upper limit exists to prevent `shift_left` from requesting a number larger than the memory available to the program.

The main points of the postconditions are as follows. If `count == 0`, the result equals `value`. The result of `shift_left` is greater than or equal to `value` if `value` is 0 or greater, and less than or equal to `value` if it is negative. The result of `shift_right` is on the same side of zero as `value` (if `value` is 0 or greater, the result is between 0 and `value`; if negative, the result is negative and greater than or equal to `value`), and if `value > 0 and count > 0`, it is smaller than `value`.

### test_bit and low_bits

```text
pub fn test_bit(value: Int, index: Int) -> Bool
pub fn low_bits(value: Int, count: Int) -> Int
```

`test_bit` returns whether bit number `index` of `value` (the lowest bit is 0) is 1. A negative number has infinitely many 1 bits, so `test_bit(value: -1, index: 200)` is `true`. The precondition is `requires index >= 0 and index <= 16777216;`.

`low_bits` returns the lowest `count` bits as a number that is 0 or greater. This is the same as the nonnegative remainder (`rem_euclid`) of dividing `value` by 2 to the power `count`, and a negative number is read as two’s complement. `low_bits(value: -1, count: 8)` is `255`. The precondition is `requires count >= 0 and count <= 16777216;`, and the postconditions are `result >= 0` and, if `count == 0`, `result == 0`.

### bit_count and bit_length

```text
pub fn bit_count(value: Int) -> Int
pub fn bit_length(value: Int) -> Int
```

Both accept only numbers that are 0 or greater (`requires value >= 0;`; passing a negative number is a defect). `bit_count` is the number of 1 bits, and `bit_length` is the number of bits needed to write the value: the smallest `n` such that `value < 2^n`, or `0` if the value is `0`. `bit_length(value: 255)` is `8`, and `bit_length(value: 256)` is `9`.

## Basic functions for numbers

```text
pub fn abs(value: Int) -> Int
pub fn sign(value: Int) -> Int
pub fn min(first: Int, second: Int) -> Int
pub fn max(first: Int, second: Int) -> Int
pub fn clamp(value: Int, low: Int, high: Int) -> Int
pub fn gcd(first: Int, second: Int) -> Int
```

The result of `abs` is 0 or greater, and is either `value` or `0 - value`. The result of `sign` is one of `-1`, `0`, and `1`, and `sign(value) * abs(value)` equals `value`. `min` and `max` return one of the two arguments.

`clamp` returns `value` unchanged if it is between `low` and `high` inclusive, and otherwise the nearer end. The precondition is `requires low <= high;`, and a call with `low > high` is a defect.

`gcd` is the greatest common divisor. The signs of the arguments are ignored, and the result is always 0 or greater. `gcd(first: 0, second: 5)` is `5`, and `gcd(first: 0, second: 0)` is `0` (this is the only case where the result is 0). When the result is greater than 0, both arguments are divisible by the result.

## pow

```text
pub fn pow(base: Int, exponent: Int) -> Int
```

`base` raised to the power `exponent`. The result is an `Int`, so it does not overflow, and `pow(base: 2, exponent: 100)` is `1267650600228229401496703205376`. `pow(base: 0, exponent: 0)` is `1`, and so is any call where `exponent` is `0`.

The precondition is `requires exponent >= 0;`. A call where `exponent` is negative is a defect. The postcondition is `ensures result == power_of(base: base, exponent: exponent);`.

### power_of

```text
pub spec fn power_of(base: Int, exponent: Int) -> Int
```

`power_of` is a `spec fn`, a function for specifications that gives `pow` its meaning. `power_of(base, 0)` is defined as `1`, and `power_of(base, n)` as `base * power_of(base, n - 1)`. The precondition is `requires exponent >= 0;`.

A `spec fn` can be called only inside preconditions, postconditions, loop invariants, `example`, and `property`. Calling it from ordinary code is error `E0331` (spec-function-in-code) ([09 Contracts](../language/09-contracts.md)). The power to use at run time is `pow`. Writing `power_of` in the contract of a function that calls `pow` lets the contract state the value of the result as a specification.

## parse

```text
pub fn parse(text: String) -> Option<Int>
```

Reads a decimal integer. The only format accepted is **an optional `-` followed by one or more decimal digits (`0` to `9`)**. There is no limit on the number of digits. A `0` before the digits is allowed and does not affect the result that is read (`"007"` is `7`, `"-007"` is `-7`, and `"-0"` is `0`).

All of the following strings give `None`.

| String | Reason |
| --- | --- |
| `""` | There are no digits |
| `"-"` | There are no digits |
| `"+5"` | `+` cannot be written |
| `" 5"`, `"5 "` | Spaces cannot be written |
| `"1.5"`, `"1e3"` | A decimal point and an exponent cannot be written |
| `"1_000"` | Separators cannot be written |
| `"0x10"` | Hexadecimal cannot be written |
| `"５"` (a full-width digit) | Only the characters `0` to `9` are accepted |

A string that is not in integer format gives `None`, not a defect. A string written by `to_text` can be read back to the original value by `parse`.

## to_text

```text
pub fn to_text(value: Int) -> String
```

Converts to a decimal string. A negative number gets a leading `-`. `0` is `"0"`, and no `+`, leading `0`, or separator is added. `to_text(value: -12)` is `"-12"`, and `to_text(value: 123456789012345678901234567890)` is `"123456789012345678901234567890"`.

## Examples

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

Calling `power_of` from ordinary code is rejected.

```cernith-error E0331
module guide.spec_in_code;

import std.int;

pub fn cube(base: Int) -> Int {
  int.power_of(base: base, exponent: 3)
}
```

## Related

- [word32 and word64](word32-word64.md): operations that wrap around at 32 or 64 bits
- [float](float.md): functions on `Float64` (`from_int` converts from `Int`)
- [03 Types](../language/03-types.md): `Int` and `Float64`
- [09 Contracts](../language/09-contracts.md): `spec fn`
