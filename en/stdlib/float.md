# std.float

The module of basic functions for `Float64`. Write `import std.float;` and call a function as in `float.abs(value: x)`. Conversion from `Int`, conversion to and from strings, and the square root are in this module. Trigonometric functions, logarithms, powers, and `floor` and `round` are in [math](math.md).

`Float64` takes only finite values, with no infinity and no NaN. For that reason, an operation whose result goes beyond the finite range is a defect. The checker reports the places where this can happen with `E0508` (float-overflow) ([03 Types](../language/03-types.md)).

## Function list

| Function | Return type | Meaning | Example |
| --- | --- | --- | --- |
| `float.abs(value)` | `Float64` | Absolute value | `float.abs(value: -2.5)` → `2.5` |
| `float.min(first, second)` | `Float64` | The smaller one | `float.min(first: 1.5, second: 2.5)` → `1.5` |
| `float.max(first, second)` | `Float64` | The larger one | `float.max(first: 1.5, second: 2.5)` → `2.5` |
| `float.clamp(value, low, high)` | `Float64` | Restrict to the range from `low` to `high` | `float.clamp(value: 7.5, low: 0.0, high: 1.0)` → `1.0` |
| `float.sqrt(value)` | `Float64` | Square root | `float.sqrt(value: 9.0)` → `3.0` |
| `float.from_int(value)` | `Float64` | Convert an `Int` to a `Float64` | `float.from_int(value: 3)` → `3.0` |
| `float.parse(source)` | `Option<Float64>` | Read a decimal string | `float.parse(source: "1.5E-7")` → `Some(value: 1.5e-7)` |
| `float.to_text(value)` | `String` | Convert to a string | `float.to_text(value: 100.0)` → `"100.0"` |

## Function details

### float.abs, float.min, float.max, float.clamp

```text
pub fn abs(value: Float64) -> Float64
pub fn min(first: Float64, second: Float64) -> Float64
pub fn max(first: Float64, second: Float64) -> Float64
pub fn clamp(value: Float64, low: Float64, high: Float64) -> Float64
```

The result of `abs` is 0.0 or greater, and is either `value` or `0.0 - value`. `min` and `max` return one of the two arguments.

`clamp` returns `value` unchanged if it is between `low` and `high` inclusive, and otherwise the nearer end. The precondition is `requires low <= high;`, and a call with `low > high` is a defect.

### float.sqrt

```text
pub fn sqrt(value: Float64) -> Float64
```

The square root. The result is the `Float64` nearest to the true square root. `sqrt(value: 2.0)` is `1.4142135623730951`.

The precondition is `requires value >= 0.0;`. A call with a negative number is a defect (`Float64` has no value that represents the square root of a negative number). The postcondition is `result >= 0.0`.

### float.from_int

```text
pub fn from_int(value: Int) -> Float64
```

Converts an `Int` to a `Float64` of the same value. An operation that mixes `Int` and `Float64` is error `E0302`, so to use an `Int` in a `Float64` expression, convert it with this function.

The precondition is `requires value >= -9007199254740992 and value <= 9007199254740992;`. Integers in this range (−2⁵³ to 2⁵³) can be represented exactly as a `Float64`. A call outside the range is a defect.

### float.parse

```text
pub fn parse(source: String) -> Option<Float64>
```

Reads a decimal number. The accepted format is as follows.

```text
-?digits[.digits][(e|E)[+-]digits]
```

- The leading `-` is optional. `+` cannot be written.
- `digits` is one or more characters from `0` to `9`. One or more digits are required both before and after the `.` (`"1."` and `".5"` give `None`).
- The exponent is `e` or `E` followed by an optional sign and one or more digits (`"1e3"`, `"1.5E-7"`, `"1e+5"`).
- A `0` before the integer part is allowed and does not affect the result (`"007.50"` is `7.5`).
- Spaces, separators, `inf`, and `NaN` cannot be written.

A string that does not match the format gives `None`. Even when the format matches, a number gives `None` if, when it is written in the form `d.ddd × 10^n` (where `d` is a digit from 1 to 9), `n` exceeds 308 (`1e309`, `10e308`, and so on). In the other direction, a number so small that it cannot be distinguished from `0` (such as `"1e-9999"`) gives `0.0`, not `None`. A `0` that is read is always `0.0`, and `"-0"` is also `0.0`.

The result is the `Float64` nearest to the written decimal number (`"0.1"` is `0.1`). Even when `n` is 308 or less, a number that exceeds the maximum `Float64` value (`1.7976931348623157e308`), such as `1.9e308`, cannot be represented as a `Float64`. The result of passing such a string is not specified.

| Call | Result |
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

The function returns `None` because input that does not match the format is an ordinary result, not a defect.

### float.to_text

```text
pub fn to_text(value: Float64) -> String
```

Converts a `Float64` to a string. The way it is written is fixed.

- The digits are the shortest ones that read back to the same `Float64` (`0.1 + 0.2` is `"0.30000000000000004"`, and `0.1` is `"0.1"`).
- When the decimal exponent of the value (the place of the most significant digit; 2 for `123.0`, −3 for `0.001`) is from −5 to 15, the value is written in positional notation. A decimal point is always included, and an integer gets `.0` (`"100.0"`, `"0.001"`, `"1000000000000000.0"`).
- Otherwise, the form is `d.ddde+XX`. The mantissa has a decimal point after the first digit, and if there is only one digit it gets `.0`. The exponent always has a sign and no leading zeros (`"1.0e+16"`, `"1.5e-7"`, `"1.5e+20"`).
- A negative number gets a leading `-`. `0.0` is `"0.0"`.

| Call | Result |
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

Passing the string returned by `to_text` to `parse` gives back the original value.

## Examples

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

## Related

- [math](math.md): trigonometric functions, exponentials and logarithms, powers, `floor`, `ceil`, `round`, `trunc`, and `to_int`
- [int](int.md): functions on `Int`
- [03 Types](../language/03-types.md): `Float64` and how overflow is handled
