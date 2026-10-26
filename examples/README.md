# Cernith samples

Small, verified programs that show how to write Cernith. They are meant to be read next to the [language reference](../en/README.md); each one names the chapter it belongs to. The comments are in English only.

Every sample passes `cernith fmt --check`, `cernith check --release`, `cernith check --verify` (no diagnostics), `cernith test`, and, if it has `main`, `cernith run`. The files in `mistakes/` are the opposite: each one is rejected with the code on its first line.

Verified with Cernith cernith 2.0.0 (compiler repository commit `aae4ddcf`).

```text
cernith test examples/types/refined_percent.cnth
cernith check --verify examples/contracts/binary_search.cnth
cernith run examples/programs/todo_list_cli.cnth --args add:milk done:1
cernith test examples/modules/ledger
```

A file starts with comment lines: `Sample` (title), `Shows` (the features), `Chapter` (where to read about them), and, for programs with `main`, `Run`. Functions appear in the canonical order (by name), not in teaching order, so read the comments on each function rather than the file from top to bottom.

## Basics

The first programs: main, labels, control flow, text, integer division.

| File | What it shows | Chapter |
|---|---|---|
| [`control_flow.cnth`](basics/control_flow.cnth) | if as an expression, match on integers, for over a range, while with a termination measure | [`language/05-statements.md`](../en/language/05-statements.md) |
| [`hello.cnth`](basics/hello.cnth) | module and import, a pure function with an example, main with the io effect, matching on a Result | [`language/01-first-program.md`](../en/language/01-first-program.md) |
| [`integer_division.cnth`](basics/integer_division.cnth) | why `/` is rejected on Int, the three rounding rules and their remainders | [`language/03-types.md`](../en/language/03-types.md) |
| [`labeled_arguments.cnth`](basics/labeled_arguments.cnth) | labels at every call, let and var, the final expression as the value, explicit Int to Float64 conversion | [`language/06-functions.md`](../en/language/06-functions.md) |
| [`text_basics.cnth`](basics/text_basics.cnth) | std.text functions with their `of:` label, split and join, trim, padding, length in characters | [`stdlib/text.md`](../en/stdlib/text.md) |

## Data

Records, variants, Option, lists, and maps.

| File | What it shows | Chapter |
|---|---|---|
| [`lists.cnth`](data/lists.cnth) | list literals, indexing and length, std.list functions (map, filter, fold, sort, zip), for loops with an invariant | [`stdlib/list.md`](../en/stdlib/list.md) |
| [`option_lookup.cnth`](data/option_lookup.cnth) | Option<T>, looking up a key in a Map, unwrap_or, option.map, handling None with match | [`stdlib/option.md`](../en/stdlib/option.md) |
| [`records.cnth`](data/records.cnth) | record declaration, construction with labels, field access, copy-with-change, a record inside a record | [`language/07-records-and-variants.md`](../en/language/07-records-and-variants.md) |
| [`variants_shapes.cnth`](data/variants_shapes.cnth) | variant cases with and without fields, match that must cover every case, nested patterns | [`language/07-records-and-variants.md`](../en/language/07-records-and-variants.md) |
| [`word_count_map_set.cnth`](data/word_count_map_set.cnth) | map.insert and map.get in a loop, option.unwrap_or for a default, map.entries in key order, set.from_list and set.to_list | [`stdlib/map.md`](../en/stdlib/map.md) |

## Types

Refined types, distinct types, record invariants, and generics.

| File | What it shows | Chapter |
|---|---|---|
| [`distinct_ids.cnth`](types/distinct_ids.cnth) | distinct types that wrap an Int, wrapping with Type(value: ...) and unwrapping with .value, no mixing by accident | [`language/08-refined-and-distinct-types.md`](../en/language/08-refined-and-distinct-types.md) |
| [`generics.cnth`](types/generics.cnth) | type parameters on functions and records, bounds (Eq, Ord), type arguments inferred from the call | [`language/06-functions.md`](../en/language/06-functions.md) |
| [`record_invariant.cnth`](types/record_invariant.cnth) | a where condition on a record, constructors that the checker proves, Option for input that may break the invariant | [`language/07-records-and-variants.md`](../en/language/07-records-and-variants.md) |
| [`refined_percent.cnth`](types/refined_percent.cnth) | a type with a where condition, constructing it with a check, Percent.check returning an Option, passing it where an Int is expected | [`language/08-refined-and-distinct-types.md`](../en/language/08-refined-and-distinct-types.md) |

## Contracts and termination

requires, ensures, loop invariants, specification functions, recursion.

| File | What it shows | Chapter |
|---|---|---|
| [`binary_search.cnth`](contracts/binary_search.cnth) | spec fn and forall in a precondition, a loop invariant that keeps the answer in range, an ensures that describes both outcomes | [`language/09-contracts.md`](../en/language/09-contracts.md) |
| [`loop_invariants.cnth`](contracts/loop_invariants.cnth) | invariant (what holds at the top of every pass), decreases (why the loop ends), contracts that depend on the loop | [`language/09-contracts.md`](../en/language/09-contracts.md) |
| [`recursion_termination.cnth`](contracts/recursion_termination.cnth) | decreases on a recursive function, structural recursion on a variant, why Cernith asks for a reason to stop | [`language/13-termination.md`](../en/language/13-termination.md) |
| [`requires_ensures.cnth`](contracts/requires_ensures.cnth) | requires (the caller's duty), ensures (the function's promise), result, how the checker uses both | [`language/09-contracts.md`](../en/language/09-contracts.md) |

## Errors

Result and `?`, your own error types, errors versus defects.

| File | What it shows | Chapter |
|---|---|---|
| [`custom_error_variant.cnth`](errors/custom_error_variant.cnth) | one error variant per operation, converting an inner error into an outer one with result.map_error | [`language/12-errors-and-defects.md`](../en/language/12-errors-and-defects.md) |
| [`errors_vs_defects.cnth`](errors/errors_vs_defects.cnth) | a Result for failures the caller should handle, requires for mistakes of the caller, defects(of: ...) in tests | [`language/12-errors-and-defects.md`](../en/language/12-errors-and-defects.md) |
| [`result_and_question.cnth`](errors/result_and_question.cnth) | a function that can fail returns Result, `?` passes an error on, errors are values you can test | [`language/12-errors-and-defects.md`](../en/language/12-errors-and-defects.md) |

## Effects and capabilities

uses io / clock / random, simulated worlds, files through capabilities.

| File | What it shows | Chapter |
|---|---|---|
| [`clock_random_sim.cnth`](effects/clock_random_sim.cnth) | the clock and random effects, a simulated world with a fixed clock and seed, repeatable results | [`language/11-effects.md`](../en/language/11-effects.md) |
| [`file_report_sim.cnth`](effects/file_report_sim.cnth) | a manifest of capabilities, fs.read_text and fs.write_text, sim.with_sim with prepared files, checking what was written | [`language/17-resources-and-capabilities.md`](../en/language/17-resources-and-capabilities.md) |
| [`pure_and_io.cnth`](effects/pure_and_io.cnth) | uses io in a function type, keeping logic in pure functions, testing output with test.with_world | [`language/11-effects.md`](../en/language/11-effects.md) |

## Testing

example and property.

| File | What it shows | Chapter |
|---|---|---|
| [`examples_and_properties.cnth`](testing/examples_and_properties.cnth) | example (one concrete case), property (a law for generated inputs), typical / empty / boundary / error cases | [`language/16-testing.md`](../en/language/16-testing.md) |
| [`property_laws.cnth`](testing/property_laws.cnth) | algebraic laws (inverse, idempotence, preserved length, additivity) written as properties of small functions | [`language/16-testing.md`](../en/language/16-testing.md) |

## Modules and packages

Several modules, abstract types, a dependency between packages.

| File | What it shows | Chapter |
|---|---|---|
| [`ledger/`](modules/ledger/) | import with an alias, using only the public functions of another module, main as the entry point | [`language/14-modules-and-packages.md`](../en/language/14-modules-and-packages.md) |
| [`with_dependency/app/`](modules/with_dependency/app/) | [dependencies] with a path, importing a module of the dependency, main returning an exit status | [`language/14-modules-and-packages.md`](../en/language/14-modules-and-packages.md) |
| [`with_dependency/geometry/`](modules/with_dependency/geometry/) | what another package may use: a transparent record, an open refined type, an abstract distinct type | [`language/14-modules-and-packages.md`](../en/language/14-modules-and-packages.md) |

## Small programs

Short complete programs that combine the features above.

| File | What it shows | Chapter |
|---|---|---|
| [`rpn_calculator.cnth`](programs/rpn_calculator.cnth) | errors as values with ?, a for loop that updates a var, match on text, a thin main around pure logic | [`language/12-errors-and-defects.md`](../en/language/12-errors-and-defects.md) |
| [`todo_list_cli.cnth`](programs/todo_list_cli.cnth) | main with arguments, parsing commands into a variant, a pure state update, an exit status | [`language/01-first-program.md`](../en/language/01-first-program.md) |
| [`tokenizer.cnth`](programs/tokenizer.cnth) | while loops with decreases over a list of characters, a variant for the result, errors with positions | [`language/13-termination.md`](../en/language/13-termination.md) |
| [`traffic_light_state.cnth`](programs/traffic_light_state.cnth) | states and events as variants, a total transition function checked by exhaustive match, a fold over events | [`language/07-records-and-variants.md`](../en/language/07-records-and-variants.md) |

## Concurrency

Tasks, channels, and std.parallel.

| File | What it shows | Chapter |
|---|---|---|
| [`channels_pipeline.cnth`](concurrency/channels_pipeline.cnth) | bounded channels, send and receive, close, tasks of one scope connected like a pipeline | [`language/18-concurrency.md`](../en/language/18-concurrency.md) |
| [`parallel_map.cnth`](concurrency/parallel_map.cnth) | std.parallel (map, filter, fold, any, all), why its function arguments have no effects, a property that ties it to std.list | [`language/18-concurrency.md`](../en/language/18-concurrency.md) |
| [`tasks_and_scopes.cnth`](concurrency/tasks_and_scopes.cnth) | the async effect, await, async.scope and async.spawn, sleep on a virtual clock, race, with_timeout | [`language/18-concurrency.md`](../en/language/18-concurrency.md) |

## Mistakes the compiler rejects

Each file is expected to be rejected with the code on its first line; read the Mistake and Fix comments.

| File | What it shows | Chapter |
|---|---|---|
| [`broken_invariant.cnth`](mistakes/broken_invariant.cnth) | `E0503`: Building a record that breaks its own where invariant. Reported by check --verify. | [`language/20-diagnostics.md`](../en/language/20-diagnostics.md) |
| [`implicit_conversion.cnth`](mistakes/implicit_conversion.cnth) | `E0302`: Int and Float64 mixed in one expression. | [`language/20-diagnostics.md`](../en/language/20-diagnostics.md) |
| [`index_out_of_range.cnth`](mistakes/index_out_of_range.cnth) | `E0506`: An index the checker cannot show to be in range. Reported by check --verify. | [`language/20-diagnostics.md`](../en/language/20-diagnostics.md) |
| [`let_with_type.cnth`](mistakes/let_with_type.cnth) | `E0323`: A type annotation on `let`. | [`language/20-diagnostics.md`](../en/language/20-diagnostics.md) |
| [`missing_decreases.cnth`](mistakes/missing_decreases.cnth) | `E0601`: A while loop with no reason to stop. | [`language/20-diagnostics.md`](../en/language/20-diagnostics.md) |
| [`mixed_identities.cnth`](mistakes/mixed_identities.cnth) | `E0303`: Two distinct types are not interchangeable even though both wrap an Int. | [`language/20-diagnostics.md`](../en/language/20-diagnostics.md) |
| [`non_exhaustive_match.cnth`](mistakes/non_exhaustive_match.cnth) | `E0305`: A match that does not cover every case of the variant. | [`language/20-diagnostics.md`](../en/language/20-diagnostics.md) |
| [`positional_arguments.cnth`](mistakes/positional_arguments.cnth) | `E0304`: A call without labels. | [`language/20-diagnostics.md`](../en/language/20-diagnostics.md) |
| [`shadowing.cnth`](mistakes/shadowing.cnth) | `E0203`: A name bound again while the first binding is still in scope. | [`language/20-diagnostics.md`](../en/language/20-diagnostics.md) |
| [`undeclared_effect.cnth`](mistakes/undeclared_effect.cnth) | `E0401`: Printing without declaring the io effect. | [`language/20-diagnostics.md`](../en/language/20-diagnostics.md) |
| [`unproved_precondition.cnth`](mistakes/unproved_precondition.cnth) | `E0501`: A call that can break the callee's requires clause. Reported by check --verify. | [`language/20-diagnostics.md`](../en/language/20-diagnostics.md) |
| [`unproved_restriction.cnth`](mistakes/unproved_restriction.cnth) | `E0503`: Constructing a refined type from a value that can be out of range. Reported by check --verify. | [`language/20-diagnostics.md`](../en/language/20-diagnostics.md) |
| [`unqualified_division.cnth`](mistakes/unqualified_division.cnth) | `E0307`: Plain `/` on Int: the rounding rule is not named. | [`language/20-diagnostics.md`](../en/language/20-diagnostics.md) |
| [`unused_binding.cnth`](mistakes/unused_binding.cnth) | `E0204`: A name that is bound but never read. | [`language/20-diagnostics.md`](../en/language/20-diagnostics.md) |
| [`unused_result.cnth`](mistakes/unused_result.cnth) | `E0306`: A call whose result is thrown away silently. | [`language/20-diagnostics.md`](../en/language/20-diagnostics.md) |
