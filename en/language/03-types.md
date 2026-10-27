# Types

Cernith types are determined statically and are nominal. Two types with different names are different types even if they have the same structure. This chapter describes the built-in types. User-defined types are covered in [07 Records and variants](07-records-and-variants.md) and [08 Refined types and distinct types](08-refined-and-distinct-types.md).

## Built-in types

| Type | Meaning | Example values |
| --- | --- | --- |
| `Int` | An integer with no upper limit on the number of digits | `0`, `-7`, `12345678901234567890` |
| `Float64` | A finite IEEE 754 binary64 | `1.5`, `0.0`, `1.0e21` |
| `Bool` | A Boolean value | `true`, `false` |
| `String` | A sequence of Unicode scalar values | `"héllo"` |
| `Unit` | A type with exactly one value | `()` |
| `List<T>` | An immutable finite sequence | `[1, 2, 3]`, `[]` |
| `Option<T>` | A value is present or absent | `Some(value: x)`, `None` |
| `Result<T, E>` | Success or error | `Ok(value: x)`, `Err(error: e)` |
| `fn(label: A, ...) -> R` | A function | Lambda expressions, function names |
| `Bytes` | A sequence of integers from 0 to 255 (bytes) | Created with functions of `std.bytes` |
| `Floats` | A sequence of finite `Float64` values | Created with functions of `std.floats` |
| `Ints` | A sequence of 64-bit integers | Created with functions of `std.ints` |

`Bytes`, `Floats`, and `Ints` are sequences that are stored more compactly than a `List`. They are values, and `==` compares their elements. They cannot be compared by magnitude. There are no literals for them, and they are created and read with functions of the standard library ([bytes](../stdlib/bytes.md), [floats and ints](../stdlib/floats-ints.md)).

### Int

`Int` does not overflow. The result of addition, subtraction, and multiplication is always the exact integer.

For division, the rounding rule is chosen by name. Using `/` on `Int`s is error `E0307`. Write which of `div_floor`, `div_trunc`, and `div_euclid` of `std.int` to use (and the corresponding remainders `rem_floor`, `rem_trunc`, and `rem_euclid`) ([stdlib/int.md](../stdlib/int.md)). `/` can be used only on `Float64`s.

```cernith-error E0307
module guide.division;

pub fn half(value: Int) -> Int {
  value / 2
}
```

No single rounding rule is made the default, even though there are several, because the result changes when a negative number is divided. For example, dividing `-7` by `2` gives `-4` with rounding toward negative infinity (floor) and `-3` with truncation toward zero.

### Float64

`Float64` takes **only finite values**. There are no infinities and no NaN. An operation that would produce infinity or NaN (overflow, division by `0.0`, and so on) becomes a defect ([12 Errors and defects](12-errors-and-defects.md)). As a result, the comparisons `==` and `<` between `Float64`s form a total order over all values.

An operation that mixes `Int` and `Float64` is error `E0302`. Convert explicitly with `from_int` of `std.float`.

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

When `x` is close to the maximum value, `x * 10.0` exceeds the finite range, so `cernith check --verify` reports `E0508` (float-overflow) together with a counterexample input (plain `cernith check` does not examine proof obligations; see [10 Proof obligations and assurance levels](10-verification.md)). If the range of the input is narrowed with `requires`, the function is accepted.

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

`String` is a sequence of Unicode scalar values. `s.length` returns the number of scalar values, not the number of bytes or UTF-16 units. A single emoji has length 1. Functions of the standard library that take a position (such as `text.slice`) also count in scalar values. Strings are concatenated with `++`, not `+`.

`String`s can be compared with each other in the lexicographic order of code points.

There is no dedicated type for characters. A single character is a `String` of length 1.

### List

`List<T>` is immutable. To create a new sequence with an element added or replaced, use `++` or a function of the standard library ([stdlib/list.md](../stdlib/list.md)). `xs.length` is the number of elements (an `Int` that is zero or greater), and `xs[i]` is the element at index `i`. An out-of-range index is a defect.

`xs.length` and `s.length` are the only accessors of the built-in types that have the form of a field. All other operations are functions of modules. Notations such as `xs.is_empty` and `s.split(...)` do not exist.

```cernith-error E0506
module guide.builtins;

pub fn at(xs: List<Int>, i: Int) -> Int {
  xs[i]
}
```

Because `i` can be out of range, `cernith check --verify` reports `E0506` (index-out-of-range). If the condition that the index is within range is written in `requires`, the function is accepted. In that case, a call with an out-of-range index becomes a defect as a call that violates `requires`.

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

### Option and Result

`Option<T>` expresses in the type that “there may be no value,” and `Result<T, E>` expresses that “the operation may fail.” There is no `null`. The possibility that a value is absent appears in the type as `Option`, and the user handles both cases with `match` ([12 Errors and defects](12-errors-and-defects.md)).

### Function types

A function type is written `fn(label: type, ...) -> return type`, followed by `uses` and the effects if there are any ([11 Effects](11-effects.md)). The labels of the arguments are also part of the type ([06 Functions](06-functions.md)).

## Forms for writing a type

| Form | Meaning |
| --- | --- |
| `Name` | The name of a type (a type in the same module, or a type in the prelude) |
| `module.Name` | A type in another module. The alias of the imported module is prefixed |
| `Name<A, B>` | A generic type with type arguments passed |
| `fn(label: A) -> B` | A function type |

## Types that do not exist

| What does not exist | What to use instead |
| --- | --- |
| Tuples | Records (`record Pair { first: Int, second: Int }`) |
| Type aliases | A refined type or a `distinct` type |
| A character type | A `String` of length 1 |
| Fixed-width integer types (`Int32` and so on) | A refined type with a range (`Int where self >= 0 and self <= 255`), or `std.word32` and `std.word64` |
| `null` | `Option<T>` |

Writing a tuple is a syntax error (`E0001`), and declaring a type alias is error `E0314`. There are no aliases because, if the same type could be called by another name, the difference between types could not be seen from the name alone. Use a `distinct` type to distinguish meanings, and a refined type to express a range.

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

## Equality and ordering

`==` and `!=` can be used when the types of both operands support equality comparison. `<`, `<=`, `>`, and `>=` can be used only on types that have an order.

| Comparison | Types it can be used on |
| --- | --- |
| `==`, `!=` (bound `Eq`) | `Int`, `Float64`, `String`, `Bool`, `Unit`, and `List`, `Option`, `Result`, records, and variants whose elements satisfy these |
| `<` and so on (bound `Ord`) | `Int`, `Float64`, `String`, `Bool` |

Function types have neither. `List`, `Option`, `Result`, records, and variants have no magnitude comparison. Using one is error `E0316`. `Eq` and `Ord` are fixed: a program cannot implement them for its own types. Bounds are also used in generic functions ([06 Functions](06-functions.md)), and a program can declare bounds of its own, called interfaces ([22 Interfaces](22-interfaces.md)).

```cernith-error E0316
module guide.ordering;

pub fn before(first: List<Int>, second: List<Int>) -> Bool {
  first < second
}
```

## Related

- The typing rules for operators are in [04 Expressions](04-expressions.md).
- Compatibility between types (subtyping) is in [08 Refined types and distinct types](08-refined-and-distinct-types.md).
