# Errors and defects

There are three kinds of failure, and they are handled differently.

| Kind | What it is | Appears in the type? | Can the program handle it? |
| --- | --- | --- | --- |
| **Error** | A possible outcome that the caller must decide how to handle (a parse failure, a missing key, an input/output failure) | Yes. `Result<T, E>` or `Option<T>` as the return type | Yes. `match`, `?` |
| **Defect** | A broken claim (a precondition, a postcondition, a restriction, an `assert`, an invariant, an index range, division by zero, a non-finite `Float64`, a `hole`) | No. However, the checker reports whether it could prove that the defect cannot occur | No. The program stops |
| **Divergence** | A computation that does not end | Yes. The effect `diverge` | No (the lack of a result is the point) |

Separately, there is **resource exhaustion**, which means exceeding the limit on memory, stack depth, or execution steps. It is reported with its own status, it cannot be caught, and it is neither a Cernith error nor a defect.

There are no exceptions and no `catch`. A function that can fail returns a `Result`, and the caller either handles it or passes it on with `?`.

## Errors as values

An error type is an ordinary variant. Define one per module or per operation. There is no type common to all errors, and there is no inheritance. Use `Option` to express “no value” when there is no particular reason to give.

```text
variant ParseError {
  EmptyInput,
  NotANumber(piece: String, position: Int),
}
```

`e?` extracts the value if `e` is a success, and if it is an error, **returns from the enclosing function** with that error.

- When used on a `Result<T, E>` value, the enclosing function must return `Result<_, E2>`, and `E` must be the same as `E2` (or a subtype of it). If the error types differ, the result is `E0309`. Conversion to a different error is written explicitly, with `match` or `result.map_error`.
- When used on an `Option<T>` value, the enclosing function must also return `Option<_>`.

Because error types are not converted automatically, every path along which an error travels appears in the code.

A `Result` or `Option` value cannot be discarded without being used (`E0306`). To discard one intentionally, write `discard` ([05 Statements and bindings](05-statements.md)).

```cernith
module guide.ports;

import std.int;
import std.result;

pub variant ConfigError {
  BadPort(error: PortError),
}

pub type Port = Int where self >= 1 and self <= 65535;

pub variant PortError {
  NotANumber,
  OutOfRange(value: Int),
}

pub fn config_port(text: String) -> Result<Port, ConfigError> {
  result.map_error(
    outcome: parse_port(text: text),
    apply: fn(error: PortError) -> ConfigError {
      ConfigError.BadPort(error: error)
    },
  )
}

example config_wraps_the_error for config_port {
  config_port(text: "x")
    == Err(error: ConfigError.BadPort(error: PortError.NotANumber))
}

pub fn head_plus_one(items: List<Int>) -> Option<Int> {
  let first = list_head(items: items)?;
  Some(value: first + 1)
}

example head_plus_one_propagates_none for head_plus_one {
  head_plus_one(items: []) == None
    and head_plus_one(items: [4]) == Some(value: 5)
}

pub fn list_head(items: List<Int>) -> Option<Int> {
  if items.length == 0 {
    None
  } else {
    Some(value: items[0])
  }
}

pub fn next_port(text: String) -> Result<Int, PortError> {
  let port = parse_port(text: text)?;
  Ok(value: port + 1)
}

example next_of_80 for next_port {
  next_port(text: "80") == Ok(value: 81)
}

example next_of_a_word for next_port {
  next_port(text: "x") == Err(error: PortError.NotANumber)
}

example next_of_zero for next_port {
  next_port(text: "0") == Err(error: PortError.OutOfRange(value: 0))
}

pub fn parse_port(text: String) -> Result<Port, PortError> {
  match int.parse(text: text) {
    None => Err(error: PortError.NotANumber),
    Some(value: number) => match Port.check(value: number) {
      Some(value: port) => Ok(value: port),
      None => Err(error: PortError.OutOfRange(value: number)),
    },
  }
}
```

```cernith-error E0309
module guide.errors;

pub variant First {
  One,
}

pub variant Second {
  Two,
}

pub fn from_first() -> Result<Int, First> {
  Err(error: First.One)
}

pub fn convert() -> Result<Int, Second> {
  let n = from_first()?;
  Ok(value: n)
}
```

### Error or precondition

An outcome that can occur even when the caller is written correctly (invalid user input, a missing file) is returned as an error, in a `Result`. Only **situations that a correct caller cannot cause** become preconditions. A call that violates a precondition is a defect, and the program stops. For example, if you want to return an error code for invalid input, check the input inside the function and return `Err(error: ...)`. Do not forbid the input with `requires`.

## Defects

When a defect occurs, the program stops (inside `cernith test`, that test fails). A defect cannot be caught, turned into a value, or ignored.

The causes of defects are as follows.

| Class (`class`) | Cause |
| --- | --- |
| `precondition` | A call that violates `requires` |
| `postcondition` | A return value that violates `ensures` |
| `restriction` | Construction that violates the condition of a refined type or of a record with an invariant |
| `assertion` | A failed `assert` |
| `invariant` | A violated loop invariant |
| `measure` | The `decreases` measure does not decrease |
| `index` | An out-of-range index |
| `float-div` | Division of a `Float64` by zero |
| `float-range` | A `Float64` operation whose result is not finite |
| `hole` | Reaching a `hole(reason: ...)` |
| `foreign-failure` | A failure of a foreign function ([15 Foreign functions](15-foreign-functions.md)) |

Which defects can occur is a question that is decided statically. The proof obligations in [10 Proof obligations and assurance levels](10-verification.md) are exactly the defects a program can have. The checker reports those that it has not yet been able to prove cannot occur.

### Defect reports

A defect is reported on standard error as a single JSON object, and the program ends with exit status **70**.

| Field | Meaning |
| --- | --- |
| `schema` | The name of the format (`cernith-defect/0.1`) |
| `class` | The class above |
| `blame` | The responsibility: `caller` for a precondition, `callee` for a postcondition, and `site` otherwise (the function that contains the place where the defect occurred) |
| `clause` | A string of the violated clause, or the operation |
| `location` | The module and line |
| `function` | The function that was running |
| `values` | The values of the variables the clause refers to |
| `message` | A description |

This is the case when the following program is run.

```text
module guide.defect;

pub fn halve(value: Int) -> Int {
  requires value >= 0;

  value
}

pub fn main(args: List<String>) -> Int {
  halve(value: 0 - args.length - 1)
}
```

```text
{
 "schema": "cernith-defect/0.1",
 "class": "precondition",
 "blame": "caller",
 "clause": "value >= 0",
 "location": {
  "line": 10,
  "module": "guide.defect"
 },
 "function": "halve",
 "values": {
  "value": "-1"
 },
 "message": "the call to halve violates its precondition"
}
```

The exit status that a program chooses (the return value of `main`) can be any value other than 70 and 71. These two are reserved by the implementation.

### Resource exhaustion

When the limit on memory, call depth, or execution steps is exceeded, the program stops with exit status **71** and a report whose `class` is `resource-exhausted` is produced. The `resource` in the report is one of `memory`, `call-depth`, or `steps`. Even with a termination guarantee, a function that terminates can still exceed a limit.

```text
{"schema": "cernith-defect/0.1", "class": "resource-exhausted", "resource": "steps"}
```

### Observing defects in tests

A defect cannot be caught in code, but inside an `example` or `property` you can state that “this call is a defect.”

```cernith
module guide.non_negative;

pub fn keep(value: Int) -> Int {
  requires value >= 0;

  value
}

example keep_rejects_a_negative_number for keep {
  defects(of: keep(value: -1)) and not defects(of: keep(value: 4))
}
```

`defects(of: expression)` is `true` if the expression stops with a defect, and `false` if it returns normally. It can be written only inside tests; elsewhere it is `E0222`. The class and the responsibility of the defect cannot be extracted. Resource exhaustion is not caught. Details are in [16 Testing](16-testing.md).

## Divergence

A function that may not stop declares `uses diverge`. The checker guarantees that a function that does not declare it always terminates ([13 Termination](13-termination.md)).

## Related

- Contracts: [09 Contracts](09-contracts.md)
- Effects: [11 Effects](11-effects.md)
- Functions on `Option` and `Result`: [stdlib/option.md](../stdlib/option.md), [stdlib/result.md](../stdlib/result.md)
