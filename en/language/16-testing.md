# Testing

Cernith has built-in support for two kinds of tests: `example` and `property`. Write tests immediately after the function they test, in the same module, and run them with `cernith test`.

```cernith
module guide.testing;

pub fn clamp(value: Int, low: Int, high: Int) -> Int {
  requires low <= high;
  ensures result >= low and result <= high;

  if value < low {
    low
  } else if value > high {
    high
  } else {
    value
  }
}

example clamp_below for clamp {
  clamp(value: -5, low: 0, high: 10) == 0
}

property clamp_is_idempotent for clamp(value: Int, low: Int, high: Int) {
  requires low <= high;

  let once = clamp(value: value, low: low, high: high);
  clamp(value: once, low: low, high: high) == once
}
```

- An **example** is a single `Bool` expression. It is evaluated exactly once. It states a fact about concrete inputs.
- A **property** has parameters, an optional `requires` clause, and a block that ends in a `Bool`. It states a law that holds for all inputs of the parameter types. The implementation generates inputs and tries them, and if any input makes it false, it reports a minimal counterexample.
- The function named after `for` must be **a function declared in the same module** (`E0205`). Test names are unique within a module. Tests are not public.
- Tests are evaluated in the `test` context. They can use pure code, `diverge` code, `spec fn`, and everything usable in specifications ([09 Contracts](09-contracts.md)). Code with effects is tried inside a simulated world (described below).

## Running an example

`cernith test` evaluates the expression while checking every contract. The result is one of the following.

| Status | Meaning |
| --- | --- |
| `passed` | The expression is `true` |
| `failed` | The expression is `false` |
| `error` | Evaluation stopped on a defect, or the budget (described below) was used up. A report is attached. **An example that stopped on a defect is not a success** |

A test that breaks a precondition by accident is not a failure of the function but an `error` of the test. A test that is meant to confirm that something becomes a defect says so explicitly with `defects(of: ...)`.

## defects(of: ...)

A defect cannot be caught inside code ([12 Errors and defects](12-errors-and-defects.md)). A test that wants to state “this call violates the precondition” is written as follows.

```cernith
module guide.testing_defects;

pub fn halve(value: Int) -> Int {
  requires value >= 0;
  ensures result >= 0;

  value
}

example halve_of_a_negative_is_a_defect for halve {
  defects(of: halve(value: -1)) and not defects(of: halve(value: 6))
}
```

`defects(of: expression)` is a `Bool`: `true` if evaluating the expression stops on a defect, and `false` if it returns. It can be written only inside an `example` and a `property`, and using it anywhere else is `E0222`. The classification and blame of the defect cannot be examined. No proof obligations are created for calls inside `defects` (because becoming a defect is the purpose). Resource exhaustion is not caught.

## Running a property

For each run of a property (one case), parameter values are generated, `requires` is evaluated, and the block is evaluated.

| Case result | Meaning |
| --- | --- |
| `pass` | The block is `true` |
| `fail` | The block is `false`, or it stopped on a defect (a defect report is attached) |
| `discard` | `requires` is false. The case is discarded |

The run stops when **100 cases pass** (changed with `--cases N`) or at the first failure. If the discarded cases exceed 10 times that number (more than 1,000 with the default 100), the result is `gave-up`. This means that a meaningful number of inputs could not be tried, and it is not a success.

```cernith-snippet
module guide.testing_gave_up;

pub fn identity(value: Int) -> Int {
  value
}

property rarely_applies for identity(value: Int) {
  requires value == 123456789;

  identity(value: value) == value
}
```

The property above results in `gave-up` (the output of `cernith test` is shown later). Write properties in the form “holds for all inputs,” without narrowing the inputs too much with `requires`.

### Input generation

The test runner generates inputs from their types. You do not need to write generators.

| Type | Values |
| --- | --- |
| `Int` | Uniform values in `[-size, size]`. With 15% probability, a boundary value (`0`, `±1`, `±2`, `10`, `100`, `255`, `256`, `2^31-1`, `-2^31`, `2^53`, `2^63`, `-2^63`) |
| `Float64` | Finite values only. A multiple of 0.25 or a uniform value in `[-size, size]`. With 15% probability, a boundary value (`0.0`, `±1.0`, `0.5`, `1e-9`, `1e9`, `±1e100`) |
| `Bool`, `Unit` | Both values, `()` |
| `String` | Length up to `size / 4`. Drawn from a small character set that includes non-ASCII characters, whitespace, commas, and `=` |
| `List<T>` | Length up to `size / 4`. Elements are generated at half the size |
| `Option<T>`, `Result<T, E>` | Each case. `None` one time in four |
| Record | Each field. As with a refined type, an invariant (`where`) selects only values that satisfy it. If no satisfying value is found in 400 tries, the property is `error` |
| Variant | A random case with generated fields. Beyond depth 4, a case without fields is chosen if there is one (so recursive values stay finite) |
| `distinct` type | Wraps a base value |
| Refined type | Selects values that satisfy the condition from base values. Tries several scales (near 0, far from 0). If none is found in 400 tries, the property is `error` |
| Function type | Cannot be generated. The property is `error` |

`size` starts at 3 and grows as the case number increases (`2 * case + 3`, at most 200). The first cases use small values.

### Reproducibility

A run is deterministic. The random seed is determined by the global seed (`--seed`, default 0), the module path, and a hash of the property name. The same program, seed, and number of cases give the same inputs and the same results on any machine. The seed is included in the report.

### Shrinking

A failing input is **shrunk**. In order, each parameter is replaced with simpler candidates, and the first candidate that still fails is adopted. This repeats until no replacement is possible or 2000 candidates have been tried (stopping on a defect also counts as a failure).

| Type | Simpler candidates (in this order) |
| --- | --- |
| `Int` | `0`, `-v` if negative, `v / 2` (toward 0), `v - 1` (toward 0) |
| `Float64` | `0.0`, truncation, `v / 2` |
| `Bool` | `false` |
| `String` | `""`, the first half, the string with one character removed at a time |
| `List<T>` | `[]`, each half, the list with each element removed, the list with each element shrunk |
| `Option<T>` | `None`, the contents shrunk |
| Record | Each field shrunk |
| Variant | An earlier case (with its fields at their simplest values); the fields shrunk |
| `distinct`, refined type | The base shrunk (for a refined type, kept inside the condition) |

The resulting counterexample is locally minimal: no single step that is tried gives something simpler that still fails. For example, an input that makes `x < 100` false shrinks to `x: 100`, and an input that makes `xs.length < 3` false shrinks to `xs: [0, 0, 0]`.

```text
passed   property guide.props.double_is_even
failed   property guide.props.double_doubles  counterexample: value: 101
passed   example  guide.props.small
3 tests: 2 passed, 1 failed
```

This `double_doubles` asserts `double(value) == value * 2` for a function in which `double(value)` returns `value` unchanged when `value > 100`. The first counterexample found was `9223372036854775808` (2^63), but the result of shrinking is the boundary value `101`.

## Regression tests from counterexamples

When a property fails, the test runner generates a **regression test** as an `example` in Cernith source code.

```text
example regression_double_doubles_722e5041 for double {
  double_doubles(value: 101)
}
```

- A property can be **called** like a function inside an `example`, a `property`, and specifications (it cannot be called inside executable code: `E0331`). If the `requires` clause is false it returns `true` (the clause works as an implication), and otherwise it returns the value of the block. A generated example writes the shrunk values as Cernith expressions and calls the property.
- The name is `regression_<property name>_<8 hexadecimal digits>`. The digits are a hash of the argument string, so the same counterexample gives the same name. Names that begin with `regression_` are reserved, and it is the error `E0223` unless the example calls a property.
- It fails before the bug is fixed and succeeds after the fix, and keeps guarding that case.

```cernith-error E0223
module guide.testing_names;

pub fn identity(value: Int) -> Int {
  value
}

example regression_something_1234abcd for identity {
  identity(value: 1) == 1
}
```

## Testing code with effects

An `example` or `property` must be pure ([11 Effects](11-effects.md)). To test code that uses input/output, the clock, or randomness, run it in a **simulated world** using `with_world` from `std.test`.

```text
test.with_world(
  world: test.World(input: ["1", "2", "3"], now_ms: 0, seed: 0),
  run: fn() -> Int uses io {
    main(args: [])
  },
).output == ["total 6"]
```

`with_world` runs a function in a world with fixed input lines (what `io.read_all_lines` returns), a fixed `clock.now_ms` value, and a fixed `random` seed, and returns an `Outcome` that holds the value of the function and the lines written. The world is deterministic, and `with_world` itself is pure. Details are in [std.test](../stdlib/test.md). A world that includes files, environment variables, the network, and capabilities is [std.sim](../stdlib/sim.md). The execution of asynchronous code can be fixed by giving the schedule as input ([18 Concurrency](18-concurrency.md)).

## Budget

Every example, and each case of a property, runs under an execution budget (the number of evaluator steps, 1,000,000 by default; `--budget N`). When it is used up, the result is `error` (`resource-exhausted`). This is a mechanism for trying `diverge` code, and it is not a time limit.

## Contracts and tests

While a test runs, every contract clause is checked, including the clauses in the functions that are called. A violation stops the test on a defect (for a precondition, the caller is blamed; for a postcondition, the callee). Clauses that cannot be executed (quantifiers that range over a whole type) are skipped rather than counted as failures. A clause that was executed and held in all tests but has not been proved has the status `tested-only` ([10 Proof obligations and assurance levels](10-verification.md)).

## Running some tests, and what the tests miss

`cernith test --only TEXT` runs only the tests whose name, target function, or `module.name` contains `TEXT` (case-sensitive). To run the tests of one function, give its name. If no test matches, the command says so on standard error and exits with `2`, so that a misspelt name is not mistaken for “all passed”.

`--coverage` and `--mutate` show what the tests leave out. The module below has a function that no test calls and an `if` whose last branch no example takes.

```cernith
module guide.coverage;

pub fn double(value: Int) -> Int {
  value * 2
}

example double_of_three for double {
  double(value: 3) == 6
}

pub fn sign(value: Int) -> Int {
  if value > 0 {
    1
  } else if value < 0 {
    -1
  } else {
    0
  }
}

example sign_of_negative for sign {
  sign(value: -5) == -1
}

example sign_of_positive for sign {
  sign(value: 5) == 1
}

pub fn unused(value: Int) -> Int {
  value + 1
}
```

`--coverage` reports a summary and locations no test reached: uncalled functions, untaken `if` branches and `match` arms, and unevaluated contract clauses. Saving the module as `coverage.cnth` produces the following output:

```text
$ cernith test --coverage coverage.cnth
passed   example  guide.coverage.double_of_three
passed   example  guide.coverage.sign_of_negative
passed   example  guide.coverage.sign_of_positive
3 tests: 3 passed
functions    2/3 (66.7%)
if branches  3/4 (75.0%)
match arms   0/0
clauses      0/0
  coverage.cnth:14:10 if-branch sign: the `else` branch of `if value < 0` never ran
  coverage.cnth:29:1 function unused: no test calls it
```

`--mutate` runs the tests, and if they pass, applies small changes (mutants) to the code under test one at a time, running again only the tests that reach the changed place. A mutant that makes some test fail is *killed*. One that no test notices *survived*, which points to a missing example (here, `sign(value: 0)`).

```text
$ cernith test --mutate coverage.cnth
passed   example  guide.coverage.double_of_three
passed   example  guide.coverage.sign_of_negative
passed   example  guide.coverage.sign_of_positive
3 tests: 3 passed
mutants 9 of 12 run: 5 killed, 4 survived, 3 unreached, score 55.6%
  SURVIVED coverage.cnth:12:6 sign: `>` becomes `>=` in `value > 0`
  SURVIVED coverage.cnth:12:14 sign: the constant 0 becomes 1
  SURVIVED coverage.cnth:14:13 sign: `<` becomes `<=` in `value < 0`
  SURVIVED coverage.cnth:14:21 sign: the constant 0 becomes 1
```

- The changes are: exchange of comparisons (`==` and `!=`; `<` and `<=`; `>` and `>=`), exchange of arithmetic (`+` and `-`, `*` to `+`), an integer constant plus one, a swap of the two branches of an `if`, a swap of two neighbouring `match` arms, and removing one `requires`, `ensures`, or loop `invariant`.
- A mutant is *unreached* when no test runs the changed place. The score is `killed / (killed + survived)`.
- A survivor is evidence of a missing example, not proof: a change that does not alter the meaning (such as `x + 0`) survives forever. Mutants are chosen evenly and the run is repeatable. `--mutants N` limits the number run (default 200), `--mutation-cases N` sets the cases per property (default 20), and `--min-score PERCENT` makes the exit status `1` below that score.
- With `--json`, the document has the key `coverage` (`cernith-coverage/0.1`) or `mutation` (`cernith-mutation/0.1`).

## Laws of interfaces

A `law` of an interface is run as a test for every `impl` of that interface that has no type parameters ([22 Interfaces](22-interfaces.md)). The test runner instantiates the law at the type of the `impl` and runs it on generated values, like a `property`. It is named `law_<Interface>_<law>_for_<Type>`, so `--only` can select it. A failing case names the law and the `impl` and shows the counterexample. In the following module, the law `not_negative` holds for `Square` (whose fields are restricted by `where`) and does not hold for `Strip` (whose fields are not).

```cernith-snippet
module guide.testing_laws;

pub interface Area {
  fn area(shape: Self) -> Int;

  law not_negative(shape: Self) {
    area(shape: shape) >= 0
  }
}

pub record Square {
  side: Int,
} where side >= 0;

pub record Strip {
  length: Int,
  width: Int,
}

impl Area for Square {
  fn area(shape: Square) -> Int {
    shape.side * shape.side
  }
}

impl Area for Strip {
  fn area(shape: Strip) -> Int {
    shape.length * shape.width
  }
}
```

The tests are the following (saved as `area.cnth`).

```text
$ cernith test --only law_ area.cnth
passed   property guide.testing_laws.law_Area_not_negative_for_Square
failed   property guide.testing_laws.law_Area_not_negative_for_Strip  counterexample: shape: Strip(length: -1, width: 1)
2 tests: 1 passed, 1 failed
```

The verifier is not asked to prove a law; a law that is false of an `impl` is a defect of the `impl`.

## Results and exit status

`cernith test` prints one line per test. With `--json`, the output is `cernith-test-results` JSON.

```text
failed   property guide.props.double_doubles  counterexample: value: 101
gave-up  property guide.testing_gave_up.rarely_applies  only 0 of 100 generated inputs satisfied the requires clauses; ...
error    example  guide.c.never_ends  resource-exhausted:
```

| JSON field | Meaning |
| --- | --- |
| `module`, `kind`, `name`, `target` | Which test it is |
| `status` | `passed`, `failed`, `error`, `gave-up` |
| `cases`, `discarded`, `seed` | For a property |
| `counterexample`, `original`, `shrink_steps` | For a failing property: the minimal counterexample, the first counterexample found, and the number of candidates tried |
| `defect` | The defect report, when the test stopped on a defect |
| `regression` | The generated regression test example |

| Option | Meaning |
| --- | --- |
| `--seed N` | Global seed (default 0) |
| `--cases N` | Number of passing cases for a property (default 100) |
| `--budget N` | Number of steps for each example and each property case |
| `--only TEXT` | Run only the tests whose name, target function, or `module.name` contains `TEXT` |
| `--coverage` | Also report the functions, branches, arms, and contract clauses that no test reached |
| `--mutate` | Also apply small changes to the code under test and list the ones that no test notices |
| `--json` | Output as JSON |

The exit status is `0` if everything passes, `1` if there is a failure or `gave-up`, and `2` if the program does not pass the type check, there is an `error` test, or `--only` matches no test. Tests are run only for a program that passes the check without errors.

## Diagnostic codes

| Code | Name | Meaning |
| --- | --- | --- |
| `E0205` | unknown-example-target | The function named after `for` is not in that module |
| `E0222` | test-construct-in-code | Used `defects(of: ...)` outside an `example` and a `property` |
| `E0223` | reserved-test-name | A name that begins with `regression_` does not call a property |
| `E0331` | spec-function-in-code | Called a property (or a `spec fn`) from executable code |

## Related

- Contracts: [09 Contracts](09-contracts.md)
- Defects: [12 Errors and defects](12-errors-and-defects.md)
- `std.test`: [stdlib/test.md](../stdlib/test.md)
