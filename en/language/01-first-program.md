# First program

A Cernith source file contains one module and has the extension `.cnth`. A file consists, in order, of a `module` line, `import` lines, and definitions (types, constants, functions, and tests).

## The smallest runnable program

```cernith
module guide.hello;

import std.io;

pub fn main(args: List<String>) -> Int uses io {
  match io.write_line(text: "hello") {
    Ok(value: _) => 0,
    Err(error: _) => 1,
  }
}
```

- `module guide.hello;` declares that this file is the module `guide.hello`.
- `import std.io;` declares that the standard library module `std.io` is used. From then on, names from it are called with the prefix `io.`, as in `io.write_line(...)` ([14 Modules and packages](14-modules-and-packages.md)).
- `pub fn main(args: List<String>) -> Int uses io { ... }` is the entry point of the program. It receives the command-line arguments as a `List<String>` and returns an `Int`. This `Int` is the exit status of the program. `uses io` declares that the function performs input and output ([11 Effects](11-effects.md)).
- `io.write_line(text: "hello")` writes one line to standard output. **An argument always carries a label (`text:`).**
- `io.write_line` returns success or failure as a `Result`. Each case of a `match` has the form `pattern => expression,`, and the last line also ends with a comma. The `_` in `Ok(value: _)` means “the value is not used.”

## A program with a contract and tests

```cernith
module guide.first;

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

example clamps_high for clamp {
  clamp(value: 12, low: 0, high: 9) == 9
}

example keeps_inside for clamp {
  clamp(value: 5, low: 0, high: 9) == 5
}

property result_in_range for clamp(value: Int, low: Int, high: Int) {
  requires low <= high;

  let r = clamp(value: value, low: low, high: high);
  r >= low and r <= high
}
```

The body of a function is structured in the following order.

1. **Clauses** (`requires`, `ensures`, `decreases`). Each ends with `;`. `requires` is a condition that the caller must satisfy, and `ensures` is a condition that the function guarantees; `result` refers to the return value ([09 Contracts](09-contracts.md)).
2. One blank line.
3. Statements (`let`, `var`, assignment, `for`, and so on).
4. The final expression. **The final expression, which has no trailing `;`, is the value of the function.** `return` can also be used, but it cannot be the last statement of the body ([05 Statements and bindings](05-statements.md)).

An `example` and a `property` follow the function. `example name for function { Bool expression }` states a fact about concrete inputs, and `property name for function(arguments) { ... }` states a law that holds for all inputs. The implementation generates inputs for a property automatically and tries them; if it finds an input for which the property does not hold, it reports a minimal counterexample ([16 Testing](16-testing.md)).

## Using the implementation

The command of the implementation is `cernith`. Only the minimum needed in this documentation is shown here.

| Command | What it does and its exit code |
| --- | --- |
| `cernith check file` | Checks syntax, names, types, effects, exhaustiveness, termination, and more. Exits with 0 if there are no errors |
| `cernith check --verify file` | In addition to the above, tries to prove contracts and constraints with the solver |
| `cernith test file` | Runs the `example`s and `property`s. Exits with 0 if all of them pass |
| `cernith run file` | Runs `main` |
| `cernith fmt file` | Rewrites the file into canonical form ([19 Canonical form](19-canonical-form.md)) |
| `cernith api [module] [--root folder]` | Prints the public interface of the modules in the folder (the current folder by default) without the bodies: for each public definition, the comment lines above it, its head, its `requires` and `ensures`, and the first example that names it. `--format json` gives JSON. Exits with 2 if nothing is printed |
| `cernith eval file 'expression'` | Evaluates an expression in the scope of the module of the file and prints its type and value as JSON. Exits with 0 for a value, 1 if the expression does not check, 70 for a defect, and 71 if the step budget is used up |

By default, diagnostics are output as JSON. With `--format compact`, each diagnostic takes one line.

```text
t.cnth:4:3 E0307 unqualified-integer-division: Integer division must name its rounding rule (int.div_floor, int.div_trunc, int.div_euclid). [fix (needs-review): ...]
```

The format is `file:line:column code name: message`, and the meanings of the codes are collected in [20 Diagnostic codes](20-diagnostics.md).

The output of `cernith test` has the following form.

```text
passed   example  guide.first.clamps_high
passed   example  guide.first.keeps_inside
passed   property guide.first.result_in_range
3 tests: 3 passed
```

`cernith eval` shows what a call gives without writing an `example`. It uses the module of the first program above, saved as `first.cnth`.

```text
$ cernith eval first.cnth 'clamp(value: 12, low: 0, high: 9)'
{
  "ok": true,
  "schema": "cernith-eval/0.1",
  "steps": 24,
  "type": "Int",
  "value": "9"
}
```

The expression must be pure. All the names of the module, including private ones, can be used. With `--session`, `eval` reads one line at a time from standard input (an expression, `let name = expression`, or `import module`) and prints one JSON line for each.

Other commands, one line each:

| Command | What it does |
| --- | --- |
| `cernith profile file` | Runs `main` on the reference interpreter and counts the calls, evaluation steps, and allocations of each function. The counts are the same on every run. `--tests` profiles the tests instead, and `--format table` prints a table |
| `cernith debug file` | Records one run of `main` and then answers questions read from standard input, one JSON line each (`summary`, `stack`, `locals`, `find-call`, `first-where`, `step`, and so on) |
| `cernith lsp` | A language server for editors (LSP 3.17) on standard input and output: diagnostics, quick fixes, hover, definition, references, symbols, formatting, and rename |
| `cernith graph folder` | Prints, as JSON, the files and dependencies of a package and the fingerprint of its interface, for build tools |
| `cernith outline [module]`, `cernith show module::name` | List the definitions of a package with their addresses and hashes, and print one definition (JSON). The kinds include `interface`, `impl`, and `manifest` |

## What to read next

The basics of types and expressions are in [03 Types](03-types.md) and [04 Expressions](04-expressions.md). The rules specific to Cernith (contracts, effects, and termination) begin with [09 Contracts](09-contracts.md).
