# std.math

This module provides mathematical functions and constants for `Float64`. Import it with `import std.math;` and call functions such as `math.sin(x: 1.0)`. Refer to constants without parentheses, as in `math.pi`.

`Float64` has no infinity and no NaN, so input outside the domain is forbidden by a precondition, and a call that violates it is a defect. An operation whose result goes beyond the range of `Float64` is also a defect ([03 Types](../language/03-types.md)). In examples that use `Float64`, the proof obligation for this overflow tends to appear as `E0508` (float-overflow) or `W0504` (proof unknown), so narrow the range of the input with `requires`.

The functions in this module are computed using only `+`, `-`, `*`, and `/` on `Float64` and `sqrt`. For that reason, the same input gives a result with the same bit pattern in every environment. The accuracy is described in the [Accuracy](#accuracy) section. Absolute value, maximum and minimum, the square root, and conversion from `Int` are in [float](float.md).

## Constants

| Constant | Value | Meaning |
| --- | --- | --- |
| `math.pi` | `3.141592653589793` | The ratio of a circle’s circumference to its diameter |
| `math.tau` | `6.283185307179586` | 2π |
| `math.half_pi` | `1.5707963267948966` | π/2 |
| `math.quarter_pi` | `0.7853981633974483` | π/4 |
| `math.e` | `2.718281828459045` | The base of the natural logarithm |
| `math.sqrt_2` | `1.4142135623730951` | The square root of 2 |
| `math.ln_2` | `0.6931471805599453` | The natural logarithm of 2 |
| `math.ln_10` | `2.302585092994046` | The natural logarithm of 10 |

All of them are `Float64`.

## Function list

| Function | Return type | Meaning | Example |
| --- | --- | --- | --- |
| `math.sin(x)` | `Float64` | Sine (`x` is in radians) | `math.sin(x: 1.0)` → `0.8414709848078965` |
| `math.cos(x)` | `Float64` | Cosine | `math.cos(x: 0.0)` → `1.0` |
| `math.tan(x)` | `Float64` | Tangent | `math.tan(x: 0.0)` → `0.0` |
| `math.asin(x)` | `Float64` | Inverse sine (−π/2 to π/2) | `math.asin(x: 1.0)` → `1.5707963267948966` |
| `math.acos(x)` | `Float64` | Inverse cosine (0 to π) | `math.acos(x: -1.0)` → `3.141592653589793` |
| `math.atan(x)` | `Float64` | Inverse tangent (−π/2 to π/2) | `math.atan(x: 0.0)` → `0.0` |
| `math.atan2(y, x)` | `Float64` | Angle of the point (x, y) (−π to π) | `math.atan2(y: 1.0, x: -1.0)` → `2.356194490192345` |
| `math.hypot(x, y)` | `Float64` | Distance from the origin to (x, y) | `math.hypot(x: 3.0, y: 4.0)` → `5.0` |
| `math.exp(x)` | `Float64` | e raised to the power `x` | `math.exp(x: 0.0)` → `1.0` |
| `math.ln(x)` | `Float64` | Natural logarithm | `math.ln(x: 2.0)` → `0.6931471805599453` |
| `math.log2(x)` | `Float64` | Logarithm to base 2 | `math.log2(x: 8.0)` → `3.0` |
| `math.log10(x)` | `Float64` | Logarithm to base 10 | `math.log10(x: 1.0)` → `0.0` |
| `math.pow(base, exponent)` | `Float64` | `base` raised to the power `exponent` | `math.pow(base: 2.0, exponent: 10.0)` → `1024.0` |
| `math.floor(x)` | `Float64` | The largest integer value that is less than or equal to `x` | `math.floor(x: -2.5)` → `-3.0` |
| `math.ceil(x)` | `Float64` | The smallest integer value that is greater than or equal to `x` | `math.ceil(x: -2.5)` → `-2.0` |
| `math.trunc(x)` | `Float64` | Discard the fractional part (toward zero) | `math.trunc(x: -2.9)` → `-2.0` |
| `math.round(x)` | `Float64` | The nearest integer value (halves go away from zero) | `math.round(x: -2.5)` → `-3.0` |
| `math.to_int(x)` | `Int` | Convert an integer-valued `Float64` to an `Int` | `math.to_int(x: -7.0)` → `-7` |

## Domain

The domains are all preconditions. A call that passes input outside the domain is a defect.

| Function | Precondition |
| --- | --- |
| `sin`, `cos`, `tan` | `x >= -500000000.0 and x <= 500000000.0` |
| `asin`, `acos` | `x >= -1.0 and x <= 1.0` |
| `exp` | `x <= 709.0` |
| `ln`, `log2`, `log10` | `x > 0.0` |
| `pow` | See below |
| `to_int` | `x == math.trunc(x: x) and x >= -9007199254740992.0 and x <= 9007199254740992.0` |
| `atan`, `atan2`, `hypot`, `floor`, `ceil`, `trunc`, `round` | None |

`sin`, `cos`, and `tan` limit the magnitude of `x` because their angle-reduction calculation needs more digits of π for larger inputs than the module stores. The upper bound for `exp` reflects the fact that e raised to 710 exceeds the maximum `Float64` value.

Even inside the domain, it is a defect if the result goes beyond the range of `Float64` (for example, when `hypot` is given two very large values). It is also a defect if an intermediate value of the computation goes beyond the range even though the result would fit: `atan2(y: 1.0e300, x: 1.0e-10)` is a defect because `y / x` is computed first, and for the same reason `acos` is a defect for an `x` whose magnitude is below about `5.6e-309`. Conversely, when the result of `exp` is smaller than the smallest `Float64` (as in `exp(x: -800.0)`), the result is `0.0`, not a defect.

## Accuracy

`floor`, `ceil`, `trunc`, and `round` are exact. For the other functions, the error relative to the true value, expressed in units in the last place (ULP), falls within the following ranges. The results are not correctly rounded.

| Function | Error |
| --- | --- |
| `sin`, `cos`, `tan`, `atan`, `atan2`, `asin`, `acos`, `exp`, `ln`, `hypot` | Within 4 ULP |
| `log2`, `log10` | Within 6 ULP |
| `pow` (integer-valued exponent with an absolute value of 64 or less) | Within `abs(exponent)` ULP. Exact if the result is a value that `Float64` can represent exactly (2¹⁰, 10³, and so on) |
| `pow` (any other exponent) | Within `4 + abs(exponent * ln(base))` ULP |

Therefore, results can be compared with `==` only for known values such as `sin(x: 0.0)` and for the results of exact functions. To compare other results, use a comparison with a tolerance (`within` in the example below).

## Trigonometric functions

```text
pub fn sin(x: Float64) -> Float64
pub fn cos(x: Float64) -> Float64
pub fn tan(x: Float64) -> Float64
```

`x` is in radians. The precondition is `requires x >= -500000000.0 and x <= 500000000.0;`. The postcondition of `sin` and `cos` is `result >= -1.0000000000000002 and result <= 1.0000000000000002`, which shows that the result may fall outside the range from −1 to 1 by one unit in the last place.

Near odd multiples of π/2, the absolute value of the result of `tan` becomes very large.

`cos(x: 1.5707963267948966)` is not `0.0` but a value close to `6.123233995736766e-17`, because `math.half_pi` is not π/2 itself but the `Float64` nearest to it.

## Inverse trigonometric functions

```text
pub fn asin(x: Float64) -> Float64
pub fn acos(x: Float64) -> Float64
pub fn atan(x: Float64) -> Float64
pub fn atan2(y: Float64, x: Float64) -> Float64
```

The precondition of `asin` and `acos` is `requires x >= -1.0 and x <= 1.0;`.

`atan` is the angle (in radians) whose tangent is `x`. The postcondition is `result >= -1.5707963267948968 and result <= 1.5707963267948968`.

`atan2(y, x)` is the angle of the point (`x`, `y`) as seen from the origin, and it distinguishes the quadrants. `atan2(y: 1.0, x: -1.0)` is 3π/4, and `atan2(y: -1.0, x: -1.0)` is −3π/4. `atan2(y: 0.0, x: 0.0)` is `0.0`. The postcondition is `result >= -3.1415926535897936 and result <= 3.1415926535897936`. The order of the arguments is `y`, then `x`.

## hypot

```text
pub fn hypot(x: Float64, y: Float64) -> Float64
```

The square root of the sum of the squares of `x` and `y`. The squares are not computed first, so even when the square of `x` or `y` goes beyond the range of `Float64`, the computation succeeds as long as the result fits in the range. The postcondition is `result >= 0.0`.

## Exponentials and logarithms

```text
pub fn exp(x: Float64) -> Float64
pub fn ln(x: Float64) -> Float64
pub fn log2(x: Float64) -> Float64
pub fn log10(x: Float64) -> Float64
```

The precondition of `exp` is `requires x <= 709.0;`, and the postcondition is `result >= 0.0`. When `x` is very small and the result is smaller than the smallest `Float64`, the result is `0.0`.

The precondition of `ln`, `log2`, and `log10` is `requires x > 0.0;`. `0.0` and negative numbers are defects, because their logarithms are not finite values. `ln(x: 5.0e-324)` (the smallest positive `Float64`) is `-744.4400719213812`.

## pow

```text
pub fn pow(base: Float64, exponent: Float64) -> Float64
```

`base` raised to the power `exponent`. `pow(base: 0.0, exponent: 0.0)` is `1.0`, and the result is always `1.0` when `exponent` is `0.0`.

The precondition is one of the following two.

```text
requires (
  exponent == trunc(x: exponent)
    and exponent >= -2147483648.0
    and exponent <= 2147483648.0
    and (base != 0.0 or exponent >= 0.0)
)
  or (base > 0.0 and exponent * ln(x: base) <= 709.0);
```

- The exponent is an integer value with an absolute value of 2³¹ or less. When the base is `0.0`, the exponent is 0 or greater. The base may be negative.
- The base is positive and `exponent * ln(x: base)` is 709 or less. The exponent may be fractional.

The rule is that when the base is not positive the exponent must be an integer value, and when the base is `0.0` the exponent must also be 0 or greater; a call that does not follow it is a defect (`pow(base: -2.0, exponent: 0.5)`, `pow(base: 0.0, exponent: 0.5)`, and `pow(base: 0.0, exponent: -1.0)`). An odd power of a negative base is negative, and an even power is positive.

The method of computation differs by exponent. An integer-valued exponent with an absolute value of 64 or less is computed by repeated squaring. Everything else is `exp(exponent * ln(base))`. The accuracy is as in the table in [Accuracy](#accuracy).

When the exponent is an integer value greater than 64, the computation uses `exp`, so it is a defect if the result goes beyond the range of `Float64` (`pow(base: 10.0, exponent: 400.0)`). When the exponent is a small negative number and the result is smaller than the smallest `Float64`, the result is `0.0` (`pow(base: 10.0, exponent: -400.0)`). It is also a defect if the range is exceeded partway through the repeated squaring.

## Functions that produce integer values

```text
pub fn floor(x: Float64) -> Float64
pub fn ceil(x: Float64) -> Float64
pub fn trunc(x: Float64) -> Float64
pub fn round(x: Float64) -> Float64
```

The result of each is an integer-valued `Float64`, and there is no precondition. A value of a magnitude at which a `Float64` is already an integer value (2⁵² or more) is returned unchanged.

| `x` | `floor` | `ceil` | `trunc` | `round` |
| ---: | ---: | ---: | ---: | ---: |
| 2.5 | 2.0 | 3.0 | 2.0 | 3.0 |
| -2.5 | -3.0 | -2.0 | -2.0 | -3.0 |
| 2.4 | 2.0 | 3.0 | 2.0 | 2.0 |
| -2.6 | -3.0 | -2.0 | -2.0 | -3.0 |
| 3.0 | 3.0 | 3.0 | 3.0 | 3.0 |

- `floor`: the largest integer value that is less than or equal to `x` (toward negative infinity). The postcondition is `result <= x and result >= x - 1.0`.
- `ceil`: the smallest integer value that is greater than or equal to `x` (toward positive infinity). The postcondition is `result >= x and result <= x + 1.0`.
- `trunc`: discards the fractional part (toward zero).
- `round`: the nearest integer value. In the case of exactly one half (`2.5`, `-2.5`), it rounds **away from zero** (`2.5` becomes `3.0`, and `-2.5` becomes `-3.0`). It is not rounding to even. `0.49999999999999994` (the `Float64` just below 0.5) gives `0.0`.

## to_int

```text
pub fn to_int(x: Float64) -> Int
```

Converts an integer-valued `Float64` to an `Int` without rounding. Calls that violate the following precondition cause a defect.

```text
requires x == trunc(x: x)
  and x >= -9007199254740992.0
  and x <= 9007199254740992.0;
```

That is, `x` is an integer value with an absolute value of 2⁵³ or less. To convert a value that has a fractional part to an `Int`, first make it an integer value with one of `floor`, `ceil`, `trunc`, and `round`. However, the checker cannot prove that the results of these are integer values (see the next section).

## Conditions the checker cannot prove

The preconditions of `pow` and `to_int` use `trunc` to require that the argument be an integer value. The checker does not know the properties of the result of `trunc`, so, as shown below, it cannot prove the precondition even for a literal argument, and it reports `W0504` (proof unknown).

```cernith-error W0504
module guide.math_unproved;

import std.math;

pub fn twelve() -> Int {
  math.to_int(x: 12.0)
}
```

The same applies to calls of `pow`. This warning does not mean that the call is wrong. A call that violates the precondition is a defect.

## Examples

In the following example, `within` is a comparison with a tolerance, `radians` shows the constants, and `square` and `to_float` are the places to hold the examples for `pow` and `to_int`. A function that calls `pow` or `to_int` cannot be checked without a warning, for the reason above, so these calls are written only inside `example`.

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

## Related

- [float](float.md): `abs`, `min`, `max`, `clamp`, `sqrt`, `from_int`, `parse`, and `to_text`
- [int](int.md): functions on `Int`
- [03 Types](../language/03-types.md): `Float64` and how overflow is handled
- [10 Proof obligations and assurance levels](../language/10-verification.md): proof obligations and unknown proofs (`W0504`)
