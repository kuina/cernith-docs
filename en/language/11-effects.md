# Effects

An **effect** is an observable interaction with the outside world or with time. Functions declare their possible effects in their signatures; a function with no declared effects is **pure**. This lets you see how a function interacts with its environment without reading its body.

| Effect | Meaning |
| --- | --- |
| `io` | Input and output with the outside world (console, files, network, environment variables) |
| `clock` | Reading the current time |
| `random` | Nondeterministic choice (random numbers) |
| `diverge` | May not terminate ([13 Termination](13-termination.md)) |
| `async` | May be suspended midway ([18 Concurrency](18-concurrency.md)) |

Failure is not an effect. A failure that can occur is a value, `Result` or `Option` ([12 Errors and defects](12-errors-and-defects.md)), and a contract violation is a defect. Updating a local `var` is not an effect either, because it cannot be observed from outside the function.

## Declaration and checking

Effects are listed after the return type with `uses`.

```text
pub fn stamp() -> Int uses clock, io { ... }
```

- The order of the list has no meaning. In the canonical form, effect names are sorted alphabetically (effect parameters come after them) and duplicates are removed ([19 Canonical form](19-canonical-form.md)).
- **At every call, the effects of the called function must be a subset of the effects declared by the calling function (or lambda).** Otherwise the result is `E0401` (undeclared-effect), and the missing effects are shown.
- Declaring more effects than the body needs is allowed.
- The name of an effect that does not exist is `E0405`.
- The initializer expression of a constant is pure. Calling a function with effects is `E0401`.

```cernith-error E0401
module guide.effects;

import std.io;

pub fn greet() -> Int {
  discard io.write_line(text: "hi");
  0
}
```

```cernith
module guide.effects;

import std.clock;
import std.io;

pub fn stamp() -> Int uses clock, io {
  discard io.write_line(text: "stamp");
  clock.now_ms()
}
```

`stamp` cannot be called from an `example` (`E0403`). Code with effects is tested inside a simulated world ([16 Testing](16-testing.md)).

## Effects of function values

A function type has effects. `fn(value: Int) -> Unit uses io` is the type of a function that performs input and output. A function value with fewer effects can be placed where more effects are allowed (`fn(...) -> T` is a subtype of `fn(...) -> T uses io`). The opposite is `E0404` (effect-mismatch).

A function expression (lambda) declares its own effects, independently of the enclosing function. Only those declared effects are available inside the lambda. Calling it from a function that does not declare all of its effects produces `E0401`.

```cernith-error E0404
module guide.effects;

import std.io;

pub fn run(apply: fn(value: Int) -> Int) -> Int {
  apply(value: 1)
}

pub fn noisy() -> Int {
  run(
    apply: fn(value: Int) -> Int uses io {
      discard io.write_line(text: "x");
      value
    },
  )
}
```

## Effect parameters

A function that takes a function as an argument can be made **polymorphic** over the effects of that argument. Declare `effect E` in the parameter list, and use it as `uses E` in the function type and as `uses E` on the function itself.

```cernith
module guide.polymorphism;

import std.io;

pub fn map_list<A, B, effect E>(
  items: List<A>,
  apply: fn(value: A) -> B uses E,
) -> List<B> uses E {
  var mapped: List<B> = [];
  for item in items {
    mapped = mapped ++ [apply(value: item)];
  }
  mapped
}

pub fn noisy(numbers: List<Int>) -> List<Int> uses io {
  map_list(
    items: numbers,
    apply: fn(value: Int) -> Int uses io {
      discard io.write_line(text: "x");
      value
    },
  )
}

pub fn pure_double(numbers: List<Int>) -> List<Int> {
  map_list(
    items: numbers,
    apply: fn(value: Int) -> Int {
      value * 2
    },
  )
}

example pure_double_doubles for pure_double {
  pure_double(numbers: [1, 2]) == [2, 4]
}
```

At a call, `E` is the union of the effects of the function arguments whose types contain `E`. If a pure lambda is passed, the call is pure; if a lambda with `uses io` is passed, the call is also `uses io`, and the calling side must declare `io`. Inside the function, `E` is treated like a declared effect. `apply(...)` can be called because the function declares `uses E`.

`list.map` and `list.filter` in the standard library are written in this form ([stdlib/list.md](../stdlib/list.md)).

The effect `async` cannot be passed through an effect parameter (`E1205`). An asynchronous function cannot be passed to `list.map`; write a loop instead ([18 Concurrency](18-concurrency.md)).

## Inside specifications and tests

- The clauses of a contract, `where`, the clauses of a loop, and the body of a `spec fn` are pure. Calling a function with effects is `E0402`, and so is a `spec fn` that declares `uses`.
- `example` and `property` can call pure code and `diverge` functions (which run under an execution budget). Calling code with `io`, `clock`, or `random` is `E0403`. To test such code, use a simulated world ([16 Testing](16-testing.md)).

## Capabilities and io

Operations on the outside world, such as file and network access, require a capability argument **and** a `uses io` declaration. Possessing the capability alone is not enough: the calling function must also declare `uses io`, or the checker reports `E0401` ([17 Resources and capabilities](17-resources-and-capabilities.md)).

## Diagnostic codes

| Code | Name | Meaning |
| --- | --- | --- |
| `E0401` | undeclared-effect | A call (or the body of a lambda) uses an effect that is not declared |
| `E0402` | effect-in-spec | A call with effects inside a specification, or a `spec fn` that declares effects |
| `E0403` | effect-in-test | A call with effects inside an `example` or `property` |
| `E0404` | effect-mismatch | A function value that has more effects than are allowed |
| `E0405` | unknown-effect | The name of an effect that does not exist |
| `E1203` | missing-await | A call to an `async` function without `await` |
| `E1205` | async-through-effect-parameter | The effect `async` was passed through an effect parameter |

## Related

- Function declarations: [06 Functions](06-functions.md)
- Concurrency: [18 Concurrency](18-concurrency.md)
