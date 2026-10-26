# Foreign functions

The implementation of a function can be written in a language other than Cernith. Such a function is called a **foreign function**. It is used for processing that Cernith cannot express, such as calling an existing library.

## Declaration

```text
foreign python "math.isqrt";
foreign javascript "cernith_runtime.isqrt";
fn isqrt(value: Int) -> Int {
  requires value >= 0;
  ensures result >= 0;
}
```

- A `foreign` line gives the name of the implementing language (the backend) and a string with the symbol name in the form `module.symbol`. The backends are `python`, `javascript`, `wasm`, and `native`. Any other name, a symbol without a dot, or a duplicate of the same backend is `E0212`.
- A `foreign` line is placed immediately before the corresponding `fn`. Writing several declares an implementation for each backend.
- The declaration contains only contract clauses and no body statements.
- `foreign native` refers to a C function (in the form `foreign native "c.name"`).
- Arguments and return values may use only types that can cross the boundary exactly.

| Usable types |
| --- |
| `Int`, `Float64`, `Bool`, `String`, `Unit` |
| `List` of those |
| Refined types of those |

Records, variants, `Option`, `Result`, `distinct` types, and functions cannot cross (`E0213`). They are wrapped and used on the Cernith side. These limits are for `foreign python`, `foreign javascript`, and `foreign native`. A `foreign wasm` function is a host function and may also use records, variants, `Option`, and `Result` (functions and resources still cannot cross); if decoding its result can fail (for example, the result contains a `String`), the result type must be `Result<T, host.BoundaryError>` (`E1304`). The types that a `native` C function can pass are more limited still (`Int`, `Float64`, `Bool`, `String`, and `Unit` as a return value).

```cernith
module guide.root;

pub fn is_perfect_square(value: Int) -> Bool {
  requires value >= 0;

  let root = isqrt(value: value);
  root * root == value
}

example squares_and_non_squares for is_perfect_square {
  is_perfect_square(value: 9) and not is_perfect_square(value: 10)
}

foreign javascript "cernith_runtime.isqrt";
foreign python "math.isqrt";
fn isqrt(value: Int) -> Int {
  requires value >= 0;
  ensures result >= 0;
  ensures result * result <= value;
  ensures (result + 1) * (result + 1) > value;
}
```

```cernith-error E0213
module guide.root;

foreign python "math.isqrt";
fn isqrt(value: Option<Int>) -> Int {
  requires true;
}
```

## Contracts are trusted

**The body of a foreign function is not checked.** Its contract is believed as written. As with an ordinary function, the caller carries the proof obligation for `requires`, and `ensures` is used as something that holds. In other words, the contracts of foreign functions are the foundation of trust in a program. The `audit` command of the implementation lists all foreign functions and their contracts as the trust boundary.

- If the signature has no `uses`, the function claims to be pure.
- It also claims to terminate. A foreign function that might not stop declares `uses diverge`.
- If the foreign code fails (for example, a Python exception) or returns a value of a type different from the declaration, it is a `foreign-failure` defect. The foreign function is blamed (because its contract was not kept).

## Which implementation is used

The standard execution of `cernith run` and `cernith test` (the reference interpreter) uses the implementation on the `foreign python` line. Running a foreign function that has no such line is a `foreign-failure` defect.

When building an executable for a specific backend (`cernith build --target ...`), a foreign function with no `foreign` line for that backend is `E0220`.

## Diagnostic codes

| Code | Name | Meaning |
| --- | --- | --- |
| `E0212` | bad-foreign-declaration | An unknown backend, a duplicate backend, or a symbol that is not `module.symbol` |
| `E0213` | foreign-type-not-supported | A type that cannot cross the boundary |
| `E0220` | missing-foreign-target | No implementation for the backend being built |

## Related

- Effects: [11 Effects](11-effects.md)
- Defects: [12 Errors and defects](12-errors-and-defects.md)
- Modules: [14 Modules and packages](14-modules-and-packages.md)
