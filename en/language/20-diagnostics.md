# Diagnostic codes

A diagnostic reported by the checker has a **code** (five characters, such as `E0304`), a **name** (a name in one-to-one correspondence with the code, such as `missing-argument-label`), a message, and a location. If the first letter of the code is `E`, the diagnostic is an error; if it is `W`, a warning. When there are errors, `cernith check` exits with a non-zero exit code.

This page lists the codes that relate to the rules of the language. It shows the meaning of each code and the chapter that covers it. The codes of the intermediate representation (`E09xx`), the edit API (`E10xx`), and host functions (`E1304`) are not listed.

```text
file:line:col E0304 missing-argument-label: Arguments must be named: expected a, b. [fix (safe): Add the parameter names as labels.]
```

- A diagnostic may have a **fix** attached. Fixes are either `safe` (can be applied without changing the meaning) or `needs-review` (the intent must be confirmed).
- Diagnostics that originate from proof obligations (`E05xx`) carry a status (`refuted`, `unknown`), the party blamed (such as `blame caller`), and the counterexample inputs ([10 Proof obligations and assurance levels](10-verification.md)).
- `cernith explain code` shows an explanation of the code, how to fix it, and an example.
- Diagnostics that arise in a chain from the same cause are suppressed, and only one is reported per cause.

In the code numbers, the hundreds digit roughly indicates the area: `E01xx` formatting, `E02xx` names and modules (including the test codes `E0222` and `E0223`), `E03xx` types, `E04xx` effects, `E05xx` contracts and proof obligations (including the termination code `E0509`), `E06xx` termination, `E07xx` unfinished code, `E11xx` resources, `E12xx` concurrency, and `E13xx` capabilities. Warnings (`W`) are numbered the same way: `W0102` (formatting), `W0504` and `W0801` (contracts and proof obligations), `W0701` (unfinished code), and `W1203` (concurrency).

## Lexical structure, syntax, and formatting

| Code | Name | Meaning | Chapter |
| --- | --- | --- | --- |
| `E0001` | syntax-error | The text cannot be read as the grammar (including lexical errors) | [02](02-lexical-structure.md) |
| `E0101` | non-canonical-form | The text is valid but not in canonical form (release build) | [19](19-canonical-form.md) |
| `W0102` | non-canonical-text | The text is valid but not in canonical form (`check`). The canonical form is attached | [19](19-canonical-form.md) |

## Name resolution

| Code | Name | Meaning | Chapter |
| --- | --- | --- | --- |
| `E0201` | unresolved-name | A name, type, member, or case is not defined | [04](04-expressions.md) |
| `E0202` | name-shadows-module | A parameter or variable has the same name as the alias of an imported module | [05](05-statements.md) |
| `E0203` | shadowing | A name was bound twice in one function | [05](05-statements.md) |
| `E0204` | unused-binding | A bound name is never read | [05](05-statements.md) |
| `E0205` | unknown-example-target | An `example` or `property` refers to a function that is not in the module | [16](16-testing.md) |

## Modules and packages

| Code | Name | Meaning | Chapter |
| --- | --- | --- | --- |
| `E0206` | unknown-module | A module that does not exist was imported | [14](14-modules-and-packages.md) |
| `E0207` | duplicate-definition | A name was defined or imported twice | [14](14-modules-and-packages.md) |
| `E0208` | not-exported | A private function, constant, or type of another module was used | [14](14-modules-and-packages.md) |
| `E0209` | unused-import | An `import` that is not used | [14](14-modules-and-packages.md) |
| `E0210` | import-cycle | Modules import each other (a cycle) | [14](14-modules-and-packages.md) |
| `E0211` | constant-cycle | A constant depends on itself | [14](14-modules-and-packages.md) |
| `E0215` | abstract-type | A `distinct` type was constructed or unwrapped outside the module that defines it | [08](08-refined-and-distinct-types.md) |
| `E0216` | private-in-public | A public signature, field, or contract contains a private type or name | [14](14-modules-and-packages.md) |
| `E0218` | module-path-mismatch | The module name does not match the file path | [14](14-modules-and-packages.md) |
| `E0219` | reserved-module-name | A module name under `std` was used | [14](14-modules-and-packages.md) |
| `E0224` | invalid-manifest | The package’s `cernith.toml` cannot be used | [14](14-modules-and-packages.md) |
| `E0225` | dependency-not-found | A dependency cannot be found, has the wrong name, or appears in two places | [14](14-modules-and-packages.md) |
| `E0226` | lock-out-of-date | `cernith.lock` is missing or does not match the dependencies | [14](14-modules-and-packages.md) |
| `E0227` | dependency-cycle | Packages depend on each other in a cycle | [14](14-modules-and-packages.md) |
| `E0228` | dependency-lacks-target | A dependency does not support a backend that the package supports | [14](14-modules-and-packages.md) |
| `E0229` | target-not-declared | A build was attempted for a backend that the package does not declare in `targets` | [14](14-modules-and-packages.md) |

## Foreign functions

| Code | Name | Meaning | Chapter |
| --- | --- | --- | --- |
| `E0212` | bad-foreign-declaration | A `foreign` line names an unknown backend, is duplicated, or names an invalid symbol | [15](15-foreign-functions.md) |
| `E0213` | foreign-type-not-supported | A type that cannot cross the foreign boundary | [15](15-foreign-functions.md) |
| `E0220` | missing-foreign-target | A foreign function has no implementation for the backend being built | [15](15-foreign-functions.md) |

## Testing

| Code | Name | Meaning | Chapter |
| --- | --- | --- | --- |
| `E0222` | test-construct-in-code | `defects(of: ...)` was used outside an `example` or `property` | [16](16-testing.md) |
| `E0223` | reserved-test-name | A name beginning with `regression_` does not call a property | [16](16-testing.md) |

## Types

| Code | Name | Meaning | Chapter |
| --- | --- | --- | --- |
| `E0301` | type-mismatch | A value of a type other than the expected type (`E0303` when a `distinct` type is involved) | [03](03-types.md) |
| `E0302` | implicit-conversion | `Int` and `Float64` were mixed | [03](03-types.md) |
| `E0303` | incompatible-identities | A `distinct` type was used in place of another type (or the reverse) | [08](08-refined-and-distinct-types.md) |
| `E0304` | missing-argument-label | An argument of a call has no label | [04](04-expressions.md) |
| `E0305` | non-exhaustive-match | A `match` does not cover all cases | [07](07-records-and-variants.md) |
| `E0306` | unused-result | A value that is not `Unit` was silently discarded | [05](05-statements.md) |
| `E0307` | unqualified-integer-division | `/` was used on two `Int` values | [03](03-types.md) |
| `E0309` | error-type-mismatch | With `?`, the error types do not match, or it was used in a function that does not return `Result` or `Option` | [12](12-errors-and-defects.md) |
| `E0310` | argument-labels-mismatch | Labels are missing, unknown, or in a different order from the declaration (calls, constructors, `with`, patterns) | [04](04-expressions.md) |
| `E0311` | var-needs-type | A `var` has no type annotation | [05](05-statements.md) |
| `E0312` | return-in-tail-position | `return` is the last statement of a function body | [05](05-statements.md) |
| `E0313` | misplaced-spec-keyword | `result` or `self` was used outside the clauses where it is allowed | [09](09-contracts.md) |
| `E0314` | type-alias | A type alias was declared | [03](03-types.md) |
| `E0315` | wrong-type-arguments | The number of type arguments is wrong | [06](06-functions.md) |
| `E0316` | bound-not-satisfied | A type that must satisfy `Eq` or `Ord` does not | [06](06-functions.md) |
| `E0317` | no-such-field | A field that does not exist (including `.value` on a refined type) | [07](07-records-and-variants.md) |
| `E0318` | unknown-bound | A bound other than `Eq` and `Ord` | [06](06-functions.md) |
| `E0319` | not-a-value | A type or module was used as a value | [04](04-expressions.md) |
| `E0320` | cannot-infer | Neither the arguments nor the context determine the type | [06](06-functions.md) |
| `E0321` | assign-to-non-var | An assignment to a `let` or a parameter | [05](05-statements.md) |
| `E0322` | closure-captures-var | A function expression used a `var` of the surrounding function | [05](05-statements.md) |
| `E0323` | let-has-type | A type annotation was written on a `let` | [05](05-statements.md) |
| `E0331` | spec-function-in-code | A `spec fn` or property was called from executable code | [09](09-contracts.md) |
| `E0332` | spec-construct-in-code | A quantifier or lambda inside a specification, or a quantifier inside code | [09](09-contracts.md) |

## Effects

| Code | Name | Meaning | Chapter |
| --- | --- | --- | --- |
| `E0401` | undeclared-effect | An effect that was not declared was used | [11](11-effects.md) |
| `E0402` | effect-in-spec | An effect was used inside a specification, or a `spec fn` declares an effect | [11](11-effects.md) |
| `E0403` | effect-in-test | An `example` or `property` called code with effects | [11](11-effects.md) |
| `E0404` | effect-mismatch | A function value has more effects than are allowed | [11](11-effects.md) |
| `E0405` | unknown-effect | The name of an effect that does not exist | [11](11-effects.md) |

## Contracts and proof obligations

| Code | Name | Meaning | Chapter |
| --- | --- | --- | --- |
| `E0501` | precondition-violated | A call may violate the `requires` of the callee | [10](10-verification.md) |
| `E0502` | postcondition-violated | The body may violate the `ensures` | [10](10-verification.md) |
| `E0503` | restriction-violated | A construction may violate the `where` condition | [10](10-verification.md) |
| `E0504` | assertion-violated | An `assert` may fail | [10](10-verification.md) |
| `E0505` | invariant-violated | A loop invariant may be broken on entry or after one iteration | [10](10-verification.md) |
| `E0506` | index-out-of-range | An index may be out of range | [10](10-verification.md) |
| `E0507` | division-by-zero | The divisor of a `Float64` division may be 0 | [10](10-verification.md) |
| `E0508` | float-overflow | A `Float64` operation may produce a non-finite value (an error for `pub` definitions by default, otherwise only a warning; see [10](10-verification.md)) | [10](10-verification.md) |
| `W0504` | obligation-unproved | The solver could not decide a proof obligation | [10](10-verification.md) |
| `W0801` | vacuous-contract | A contract holds trivially (vacuous) | [10](10-verification.md) |

## Termination

| Code | Name | Meaning | Chapter |
| --- | --- | --- | --- |
| `E0509` | measure-not-decreasing | The `decreases` measure does not get smaller in a recursive call | [13](13-termination.md) |
| `E0601` | termination-unproved | A loop or a group of recursive functions has no basis for termination | [13](13-termination.md) |
| `E0602` | non-positive-recursive-type | A recursive type has a function that takes a value of its own recursive group | [13](13-termination.md) |

## Unfinished code

| Code | Name | Meaning | Chapter |
| --- | --- | --- | --- |
| `E0702` | hole-in-release | A `hole` remains in a release build | [02](02-lexical-structure.md) |
| `W0701` | hole-present | A function has a `hole` | [02](02-lexical-structure.md) |

## Resources and capabilities

| Code | Name | Meaning | Chapter |
| --- | --- | --- | --- |
| `E1101` | resource-escapes-scope | A resource (a second-class value) was returned, stored, copied, captured by a closure, or used anywhere other than as a call argument | [17](17-resources-and-capabilities.md) |
| `E1102` | release-error-not-covered | The error type of the body of `using` differs from the error type of acquiring and releasing | [17](17-resources-and-capabilities.md) |
| `E1103` | resource-outside-using | The acquisition of a resource was bound outside `using` | [17](17-resources-and-capabilities.md) |
| `E1105` | bad-resource-declaration | A resource has no `release` function, or its signature is wrong | [17](17-resources-and-capabilities.md) |
| `E1106` | resource-as-type-argument | A resource type was made a type argument or a field | [17](17-resources-and-capabilities.md) |
| `E1107` | return-inside-using | `return` was used inside the body of `using` | [17](17-resources-and-capabilities.md) |
| `E1301` | capability-outside-manifest | A capability was created outside the `manifest` and the defining module | [17](17-resources-and-capabilities.md) |
| `E1302` | simulated-outside-test | A capability for testing was created outside a test | [17](17-resources-and-capabilities.md) |
| `E1303` | bad-manifest | There are two `manifest` declarations, a field is not a capability, or `main` does not receive it | [17](17-resources-and-capabilities.md) |
| `E1305` | capability-unavailable-on-target | The `manifest` names a capability that the world of the backend being built does not provide | [17](17-resources-and-capabilities.md) |
| `E1306` | internal-function | A program called a function that is for internal use by the standard library | [17](17-resources-and-capabilities.md) |

## Concurrency

| Code | Name | Meaning | Chapter |
| --- | --- | --- | --- |
| `E1201` | task-not-awaited | A task is not awaited on every path | [18](18-concurrency.md) |
| `E1202` | task-escapes-scope | A task, scope, or channel endpoint became the value of a scope, or a task was copied, stored, passed, or captured | [18](18-concurrency.md) |
| `E1203` | missing-await | A call to an `async` function has no `await` | [18](18-concurrency.md) |
| `E1204` | task-awaited-twice | A task was awaited two or more times | [18](18-concurrency.md) |
| `E1205` | async-through-effect-parameter | The effect `async` was passed through an effect parameter | [18](18-concurrency.md) |
| `E1206` | await-of-nothing | `await` was written in front of something that cannot suspend | [18](18-concurrency.md) |
| `E1207` | await-in-using | The body of `using` contains a suspension point | [18](18-concurrency.md) |
| `W1203` | wait-cannot-complete | Receives from a channel but never sends | [18](18-concurrency.md) |

## Related

- The details of the rules are on the page named in the “Chapter” column for each code.
