# std.result

This module provides operations on the prelude type `Result<T, Fail>`, which represents success (`Ok(value: x)`) or failure (`Err(error: e)`). Import it with `import std.result;` and call functions such as `result.map(...)`. For similar operations on `Option<T>`, see [option](option.md).

Use `match` to handle a `Result` directly ([07 Records and variants](../language/07-records-and-variants.md)), or `?` to return an `Err` to the caller ([12 Errors and defects](../language/12-errors-and-defects.md)). This module offers shorter forms for common matching patterns.

The label of every argument that receives a `Result` is `outcome` (in the `option` module, it is `option`).

## Function list

| Function | Return type | Meaning |
| --- | --- | --- |
| `result.is_ok(outcome)` | `Bool` | `true` if it is `Ok` |
| `result.is_err(outcome)` | `Bool` | `true` if it is `Err` |
| `result.unwrap_or(outcome, default)` | `T` | `x` for `Ok(value: x)`, and `default` for `Err` |
| `result.map(outcome, apply)` | `Result<U, Fail>` | `Ok(value: apply(value: x))` for `Ok(value: x)`, and `Err` unchanged |
| `result.map_error(outcome, apply)` | `Result<T, Other>` | `Err(error: apply(error: e))` for `Err(error: e)`, and `Ok` unchanged |
| `result.and_then(outcome, apply)` | `Result<U, Fail>` | `apply(value: x)` for `Ok(value: x)`, and `Err` unchanged |
| `result.ok(outcome)` | `Option<T>` | `Some(value: x)` for `Ok(value: x)`, and `None` for `Err` |

The functions that take `apply` (`map`, `map_error`, and `and_then`) are effect-polymorphic. If `apply` is pure, the call is also pure, and if `apply` is `uses io`, the caller also declares `uses io` ([11 Effects](../language/11-effects.md)).

The functions of this module have no preconditions. Whatever `Result` is passed, the function itself does not become a defect (except when the evaluation of `apply` or `default` is a defect). There is also no function that turns an `Err` into a defect and extracts the contents. To handle the `Err` case, supply a default with `unwrap_or`, or write a `match`.

## Function details

### result.map

```text
pub fn map<T, U, Fail, effect E>(
  outcome: Result<T, Fail>,
  apply: fn(value: T) -> U uses E,
) -> Result<U, Fail> uses E
```

Applies `apply` to the contents of an `Ok`. For an `Err`, `apply` is not called and the error value is returned as it is. The label of the argument of `apply` is `value`.

### result.map_error

```text
pub fn map_error<T, Fail, Other, effect E>(
  outcome: Result<T, Fail>,
  apply: fn(error: Fail) -> Other uses E,
) -> Result<T, Other> uses E
```

Applies `apply` to the contents of an `Err` and changes the error type from `Fail` to `Other`. For an `Ok`, `apply` is not called. The label of the argument of `apply` is `error`. It is a function that exists only for `Result`, with no counterpart in `option.map`.

### result.and_then

```text
pub fn and_then<T, U, Fail, effect E>(
  outcome: Result<T, Fail>,
  apply: fn(value: T) -> Result<U, Fail> uses E,
) -> Result<U, Fail> uses E
```

Use it when `apply` itself returns a `Result`. Where `map` would give `Result<Result<U, Fail>, Fail>`, this gives `Result<U, Fail>` without nesting. When steps that can fail are written in sequence, evaluation stops at the first `Err`, and that `Err` is the result. The later steps are not called.

The error types of `outcome` and of the result of `apply` must be the same `Fail`. If they differ, it is type error `E0301`.

```cernith-error E0301
module guide.result_types;

import std.result;

pub fn lookup(code: Result<Int, Int>) -> Result<Int, String> {
  result.and_then(
    outcome: code,
    apply: fn(value: Int) -> Result<Int, String> {
      Ok(value: value)
    },
  )
}
```

To align the error types, use `result.map_error`. Pass `outcome` through `result.map_error` to make it a `Result<Int, String>` before passing it on (`lookup` in `guide.results`). `?` also works only when the error types are equal.

> Note: `let half = halve(value: value)?; halve(value: half)` gives the same result as `result.and_then(outcome: halve(value: value), apply: ...)` (`quarter` in `guide.results`).

### result.unwrap_or

```text
pub fn unwrap_or<T, Fail>(outcome: Result<T, Fail>, default: T) -> T
```

Unwraps a `Result` by supplying the value to use for `Err`. The error value is discarded. `default` is evaluated even in the `Ok` case (Cernith evaluates arguments first). If `default` is an expression whose evaluation is a defect, it is a defect even when an `Ok` is passed.

### result.ok

```text
pub fn ok<T, Fail>(outcome: Result<T, Fail>) -> Option<T>
```

Converts a `Result` to an `Option`. It turns `Ok(value: x)` into `Some(value: x)` and `Err` into `None`, and the error value is discarded. The function that converts an `Option` to a `Result` is `option.ok_or` ([option](option.md)).

### result.is_ok and result.is_err

```text
pub fn is_ok<T, Fail>(outcome: Result<T, Fail>) -> Bool
pub fn is_err<T, Fail>(outcome: Result<T, Fail>) -> Bool
```

Use them when the contents are not needed and you only want to know whether it succeeded or failed. For the same `outcome`, the two results are always opposite. A comparison with `==` (`outcome == Ok(value: 1)`) can be written only when `T` and `Fail` support equality comparison, but these two can be used regardless of `T` and `Fail`.

## Example

```cernith
module guide.results;

import std.int;
import std.result;

pub fn halve(value: Int) -> Result<Int, String> {
  if int.rem_euclid(dividend: value, divisor: 2) == 0 {
    Ok(value: int.div_floor(dividend: value, divisor: 2))
  } else {
    Err(error: "odd")
  }
}

example and_then_stops_at_the_first_error for halve {
  result.and_then(
    outcome: halve(value: 8),
    apply: fn(value: Int) -> Result<Int, String> {
      halve(value: value)
    },
  )
    == Ok(value: 2)
    and result.and_then(
      outcome: halve(value: 3),
      apply: fn(value: Int) -> Result<Int, String> {
        halve(value: value)
      },
    )
      == Err(error: "odd")
    and result.and_then(
      outcome: halve(value: 6),
      apply: fn(value: Int) -> Result<Int, String> {
        halve(value: value)
      },
    )
      == Err(error: "odd")
}

example is_ok_and_is_err for halve {
  result.is_ok(outcome: halve(value: 4))
    and not result.is_err(outcome: halve(value: 4))
    and result.is_err(outcome: halve(value: 3))
    and not result.is_ok(outcome: halve(value: 3))
}

example map_changes_only_the_value for halve {
  result.map(
    outcome: halve(value: 4),
    apply: fn(value: Int) -> Int {
      value * 3
    },
  )
    == Ok(value: 6)
    and result.map(
      outcome: halve(value: 3),
      apply: fn(value: Int) -> Int {
        value * 3
      },
    )
      == Err(error: "odd")
}

example map_error_changes_only_the_error for halve {
  result.map_error(
    outcome: halve(value: 3),
    apply: fn(error: String) -> Int {
      error.length
    },
  )
    == Err(error: 3)
    and result.map_error(
      outcome: halve(value: 4),
      apply: fn(error: String) -> Int {
        error.length
      },
    )
      == Ok(value: 2)
}

example ok_drops_the_error for halve {
  result.ok(outcome: halve(value: 4)) == Some(value: 2)
    and result.ok(outcome: halve(value: 3)) == None
}

example unwrap_or_evaluates_the_default_even_for_an_ok for halve {
  defects(
    of: result.unwrap_or(
      outcome: halve(value: 4),
      default: int.div_floor(dividend: 1, divisor: 0),
    ),
  )
}

example unwrap_or_gives_the_default_for_an_error for halve {
  result.unwrap_or(outcome: halve(value: 4), default: 0) == 2
    and result.unwrap_or(outcome: halve(value: 3), default: 0) == 0
}

pub fn lookup(code: Result<Int, Int>) -> Result<Int, String> {
  result.and_then(
    outcome: result.map_error(
      outcome: code,
      apply: fn(error: Int) -> String {
        "code"
      },
    ),
    apply: fn(value: Int) -> Result<Int, String> {
      halve(value: value)
    },
  )
}

example map_error_aligns_the_error_types for lookup {
  lookup(code: Ok(value: 4)) == Ok(value: 2)
    and lookup(code: Ok(value: 3)) == Err(error: "odd")
    and lookup(code: Err(error: 7)) == Err(error: "code")
}

pub fn quarter(value: Int) -> Result<Int, String> {
  let half = halve(value: value)?;
  halve(value: half)
}

example question_mark_is_and_then for quarter {
  quarter(value: 8) == Ok(value: 2)
    and quarter(value: 6) == Err(error: "odd")
    and quarter(value: 3) == Err(error: "odd")
    and quarter(value: 8)
      == result.and_then(
        outcome: halve(value: 8),
        apply: fn(value: Int) -> Result<Int, String> {
          halve(value: value)
        },
      )
    and quarter(value: 6)
      == result.and_then(
        outcome: halve(value: 6),
        apply: fn(value: Int) -> Result<Int, String> {
          halve(value: value)
        },
      )
}
```

## Related

- [option](option.md): functions on `Option<T>`. `option.ok_or` converts an `Option` to a `Result`
- [07 Records and variants](../language/07-records-and-variants.md): taking apart with `match`
- [11 Effects](../language/11-effects.md): effect polymorphism
- [12 Errors and defects](../language/12-errors-and-defects.md): choosing between errors and defects, and `?`
