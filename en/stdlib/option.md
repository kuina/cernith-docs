# std.option

This module provides functions on `Option<T>`. `Option<T>` is a type that represents either the presence of a value (`Some(value: x)`) or its absence (`None`), and it is in the prelude. To use the functions of this module, write `import std.option;` and call them as in `option.map(...)`.

The basic way to take a value out is `match` ([07 Records and variants](../language/07-records-and-variants.md)). The functions of this module are for writing common `match` patterns more briefly.

## Function list

| Function | Return type | Meaning |
| --- | --- | --- |
| `option.is_some(option)` | `Bool` | `true` if it is `Some` |
| `option.is_none(option)` | `Bool` | `true` if it is `None` |
| `option.unwrap_or(option, default)` | `T` | `x` for `Some(value: x)`, and `default` for `None` |
| `option.map(option, apply)` | `Option<U>` | `Some(value: apply(value: x))` for `Some(value: x)`, and `None` for `None` |
| `option.and_then(option, apply)` | `Option<U>` | `apply(value: x)` for `Some(value: x)`, and `None` for `None` |
| `option.ok_or(option, error)` | `Result<T, Fail>` | `Ok(value: x)` for `Some(value: x)`, and `Err(error: error)` for `None` |

The functions that take `apply` (`map` and `and_then`) are effect-polymorphic. If `apply` is pure, the call is also pure, and if `apply` is `uses io`, the caller also declares `uses io` ([11 Effects](../language/11-effects.md)).

## Function details

### option.map

```text
pub fn map<T, U, effect E>(
  option: Option<T>,
  apply: fn(value: T) -> U uses E,
) -> Option<U> uses E
```

Applies `apply` to the contents of a `Some`. For `None`, `apply` is not called. The label of the argument of `apply` is `value`.

### option.and_then

```text
pub fn and_then<T, U, effect E>(
  option: Option<T>,
  apply: fn(value: T) -> Option<U> uses E,
) -> Option<U> uses E
```

Use it when `apply` itself returns an `Option`. Where `map` would give `Option<Option<U>>`, this gives `Option<U>` without nesting. If `apply` returns `None`, the result is `None`.

### option.unwrap_or

```text
pub fn unwrap_or<T>(option: Option<T>, default: T) -> T
```

Unwraps an `Option` by supplying the value to use for `None`. `default` is evaluated even in the `Some` case (Cernith evaluates arguments first).

### option.ok_or

```text
pub fn ok_or<T, Fail>(option: Option<T>, error: Fail) -> Result<T, Fail>
```

Converts an `Option` to a `Result`. It maps `None` to the specified error value. The function that converts a `Result` to an `Option` is `result.ok` ([result](result.md)).

### option.is_some and option.is_none

```text
pub fn is_some<T>(option: Option<T>) -> Bool
pub fn is_none<T>(option: Option<T>) -> Bool
```

Use them when the contents are not needed and you only want to know whether there is a value. A comparison with `==` (`option == None`) can be written only when `T` supports equality comparison, but these two can be used regardless of `T`.

## Example

```cernith
module guide.options;

import std.int;
import std.option;

pub fn even_or_none(value: Int) -> Option<Int> {
  if int.rem_euclid(dividend: value, divisor: 2) == 0 {
    Some(value: value)
  } else {
    None
  }
}

example and_then_chains for even_or_none {
  option.and_then(
    option: Some(value: 4),
    apply: fn(value: Int) -> Option<Int> {
      even_or_none(value: value + 2)
    },
  )
    == Some(value: 6)
    and option.and_then(
      option: Some(value: 3),
      apply: fn(value: Int) -> Option<Int> {
        even_or_none(value: value + 2)
      },
    )
      == None
}

example is_some_and_is_none for even_or_none {
  option.is_some(option: even_or_none(value: 4))
    and option.is_none(option: even_or_none(value: 3))
}

example map_applies_to_some for even_or_none {
  option.map(
    option: even_or_none(value: 4),
    apply: fn(value: Int) -> Int {
      value + 1
    },
  )
    == Some(value: 5)
    and option.map(
      option: even_or_none(value: 3),
      apply: fn(value: Int) -> Int {
        value + 1
      },
    )
      == None
}

example ok_or_gives_a_result for even_or_none {
  option.ok_or(option: even_or_none(value: 4), error: "odd") == Ok(value: 4)
    and option.ok_or(option: even_or_none(value: 3), error: "odd")
      == Err(error: "odd")
}

example unwrap_or_gives_the_default for even_or_none {
  option.unwrap_or(option: even_or_none(value: 4), default: 0) == 4
    and option.unwrap_or(option: even_or_none(value: 3), default: 0) == 0
}
```

## Related

- [result](result.md): functions on `Result<T, E>`
- [06 Functions](../language/06-functions.md): effect polymorphism
