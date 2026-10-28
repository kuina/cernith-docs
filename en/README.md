# Cernith language reference

Cernith is a programming language by Kuina-chan with static checks for types, effects, and contracts (preconditions and postconditions). Before a program runs, the checker validates names, types, effects, exhaustiveness, and termination, and attempts to prove its contracts. Source files use the `.cnth` extension.

This repository describes the language specification of Cernith 2.0 and the core modules of the standard library `std`.

```cernith
module guide.overview;

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
```

## Features of the language

- **Every argument has a label.** A call such as `clamp(value: 12, low: 0, high: 9)` shows the purpose of each argument. Omitting a label is an error.
- **Conversions and omissions must be explicit.** `Int` and `Float64` are not converted automatically. There is no `null`, `match` must cover every case, and a function's result cannot be silently discarded.
- **Failures are values.** There are no exceptions. Failures that can happen appear in the type as `Result` or `Option` and are passed to the caller with `?`. A mistake in the program itself (a contract violation, an out-of-range index, and so on) is called a “defect,” and the program stops there.
- **Contracts and refined types express constraints.** `requires` / `ensures` and refined types such as `type Percent = Int where self >= 0 and self <= 100;` describe conditions on functions and values. The checker attempts to prove these conditions with a solver; unproved conditions are checked at run time.
- **Side effects are declared in the function type.** Effects are declared as in `uses io, clock`, and a function with no declaration is pure.
- **Termination is checked.** A `while` loop needs a decreasing measure (`decreases`), and recursion needs structural recursion or a measure. A function that might not stop is declared with `uses diverge`.
- **Testing is built in.** Write an `example` for a concrete case or a `property` for a law intended to hold for all inputs, immediately after the function it tests. The test runner executes them.
- **Formatting is consistent.** A single canonical form defines whitespace and declaration order.

## Contents

### Language specification ([language/](language/))

| Page | Content |
| --- | --- |
| [01 First program](language/01-first-program.md) | The shape of a module, and how to check, test, and run it |
| [02 Lexical structure](language/02-lexical-structure.md) | Identifiers, literals, comments, keywords, the prelude |
| [03 Types](language/03-types.md) | Built-in types and the basic type rules |
| [04 Expressions](language/04-expressions.md) | Operators, calls, `if` / `match`, `?` |
| [05 Statements and bindings](language/05-statements.md) | `let` / `var`, naming rules, loops |
| [06 Functions](language/06-functions.md) | Functions, function values, generics, type inference |
| [07 Records and variants](language/07-records-and-variants.md) | User-defined types, patterns, exhaustiveness |
| [08 Refined types and distinct types](language/08-refined-and-distinct-types.md) | Restricting value ranges and distinguishing values with different meanings |
| [09 Contracts](language/09-contracts.md) | `requires`, `ensures`, `where`, specification expressions |
| [10 Proof obligations and assurance levels](language/10-verification.md) | What the checker proves, and what it leaves to run time |
| [11 Effects](language/11-effects.md) | `uses`, effect parameters |
| [12 Errors and defects](language/12-errors-and-defects.md) | `Result`, `?`, defects, exit status |
| [13 Termination](language/13-termination.md) | Why loops and recursion stop |
| [14 Modules and packages](language/14-modules-and-packages.md) | `import`, visibility, packages |
| [15 Foreign functions](language/15-foreign-functions.md) | Functions implemented in other languages |
| [16 Testing](language/16-testing.md) | `example`, `property`, the simulated world |
| [17 Resources and capabilities](language/17-resources-and-capabilities.md) | `using`, `capability`, `manifest` |
| [18 Concurrency](language/18-concurrency.md) | `parallel`, `scope`, `await`, channels |
| [19 Canonical form](language/19-canonical-form.md) | The rules of the format |
| [20 Diagnostic codes](language/20-diagnostics.md) | A list of the codes the checker reports |
| [21 Grammar](language/21-grammar.md) | The definition of the syntax and operator precedence |
| [22 Interfaces](language/22-interfaces.md) | `interface`, `impl`, bounds, laws, `derive` |
| [23 Units of measure](language/23-units-of-measure.md) | `unit`, quantity types such as `Float64<m / s>`, and unit mismatch errors |

### Standard library ([stdlib/](stdlib/))

See [stdlib/README.md](stdlib/README.md) for conventions and the module index.

### Samples ([../examples/](../examples/README.md))

The [examples/](../examples/README.md) directory contains compiler-checked programs illustrating how to write Cernith. Comments are in English.

## Notation

- Code examples have been checked and their tests run. Examples intended to fail include the expected diagnostic code, such as `E0304`.
- An example beginning with `module guide.…;` is a complete module that can be saved and checked as a single file.
- A function’s parameters are written in the form `name: type`, and a call passes them in the form `name: value`. In explanations, a function is sometimes written with only its labels, as in `module.name(label, label)`.
- In a diagnostic code, the leading letter `E` means an error and `W` means a warning. The list is in [20 Diagnostic codes](language/20-diagnostics.md).
