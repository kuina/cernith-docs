# Standard library std

The standard library is a collection of modules whose names begin with `std.`, written in Cernith source. To use one, `import` it, as in `import std.list;`, which makes **the last part its alias**, and call its functions as in `list.take(items: xs, count: 2)`. Writing `std.list.take(...)` is not allowed ([14 Modules and packages](../language/14-modules-and-packages.md)).

The prelude types and names (`Int`, `Float64`, `Bool`, `String`, `Unit`, `List`, `Option`, `Result`, `Bytes`, `Floats`, `Ints`, `Some`, `None`, `Ok`, `Err`, `hole`) are available without an `import`. Other types, such as `Map` and `Set`, are used from their modules with the module name attached (`map.Map<String, Int>`).

## Module list

### Data structures and values

| Module | Contents |
| --- | --- |
| [std.list](list.md) | Functions on `List<T>`: find, slice, transform, fold, sort |
| [std.text](text.md) | Functions on `String`: split, search, replace, format |
| [std.int](int.md) | Functions on `Int`: division with a choice of rounding, powers, bit operations, conversion to and from strings |
| [std.float](float.md) | Functions on `Float64`: conversion, maximum and minimum, `sqrt` |
| [std.math](math.md) | Mathematical functions and constants: trigonometric functions, exponentials and logarithms, rounding |
| [std.word32 and std.word64](word32-word64.md) | 32-bit and 64-bit unsigned words (they wrap around on overflow) |
| [std.map](map.md) | `Map<K, V>`: a dictionary with ordered keys |
| [std.set](set.md) | `Set<T>`: a set with ordered elements |
| [std.option](option.md) | Functions on `Option<T>` |
| [std.result](result.md) | Functions on `Result<T, E>` |
| [std.bytes](bytes.md) | `Bytes`: byte sequences, reading and writing numbers, hexadecimal and Base64 |
| [std.floats and std.ints](floats-ints.md) | `Floats`, `Ints`: arrays of numbers |
| [std.seq](seq.md) | `Seq<T>`: lazy sequences that end, with a bound on their length |
| [std.show](show.md) | The interface `Show`: a value as the Cernith text that builds it |
| [std.codec](codec.md) | The interfaces `Encode` and `Decode`: a value as the shape of data that formats read and write |

### Concurrency and parallelism

| Module | Contents |
| --- | --- |
| [std.parallel](parallel.md) | Apply a pure function to each element of a list in parallel |
| [std.async](async.md) | Tasks, scopes, channels, and waiting for time |

### Interaction with the world

| Module | Contents |
| --- | --- |
| [std.io, std.clock, and std.random](io-clock-random.md) | Standard input and output, time (including a monotonic clock and a real wait), and random numbers (including secure random bytes); effects that do not use capabilities |
| [std.console](console.md) | Standard input and output, and whether the output is a terminal (using a capability) |
| [std.env](env.md) | Environment variables |
| [std.fs](fs.md) | Files and folders: reading, writing, appending, replacing, renaming, and looking at a path |
| [std.path](path.md) | String operations on paths |
| [std.http](http.md) | HTTP requests with any method, headers, and a body |
| [std.process](process.md) | Running a program that the grant allows, and reading what it printed |

### Testing

| Module | Contents |
| --- | --- |
| [std.test](test.md) | Try out code with effects in a simulated world |
| [std.sim](sim.md) | A simulated world of files, environment variables, the network, programs, and the terminal |

## Conventions

The functions of the standard library follow these conventions.

- **Labels follow roles.** The subject of an operation comes first and has a name such as `items`, `of`, `option`, `outcome`, or `map`. Parameters that take a function are `apply`, `keep`, `test`, `combine`, and `less_than`, and the parameters of those functions are named `item`, `total`, `first`, `second`, and `value`.
- **A function that touches the world takes a capability, or is an ambient effect.** `std.fs`, `std.console`, `std.env`, `std.http`, and `std.process` take a capability received from the `manifest` (`access:`) and declare `uses io`. `std.io`, `std.clock`, and `std.random` need only the effect (`io`, `clock`, `random`; a function such as `clock.sleep_ms` or `random.bytes` declares `io` as well) ([17 Resources and capabilities](../language/17-resources-and-capabilities.md)).
- **Most functions that take a function as an argument are effect-polymorphic** (`<effect E>`). If you pass a pure function, the call is also pure ([11 Effects](../language/11-effects.md)). The exceptions are `list.sort_by` (the comparison must be pure), `std.parallel` (no effects are allowed in the function), and `test.with_world` and `sim.with_sim` (the function may have only the effects `clock`, `io`, and `random`).
- **Total operations return an `Option`, and partial operations have a precondition.** `first`, `last`, `max`, `min`, `get`, `index_of`, and `parse` return an `Option` because there may be no result. Functions that take a range, such as `take`, `drop`, `slice`, and `range`, make a valid range a precondition, and a call that violates it is a defect ([12 Errors and defects](../language/12-errors-and-defects.md)).
- **Positions and lengths are counted in Unicode scalar values**, not in bytes or UTF-16 units.
- **Sorting is stable.** `sort_by` takes a strict comparison that means “is less than.”
- **Case conversion applies to ASCII letters only.** `to_lower` and `to_upper` have neither locale nor Unicode case mapping.
- A function whose result is a `Float64` must produce a finite result. An operation that would produce infinity or NaN is a defect.
- **The signature of a function that has a contract is shown with `{ ... }`.** Only the `requires` and `ensures` clauses are written, and the body is omitted as `...`.

The code examples on each page have been run and checked.
