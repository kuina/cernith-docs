# Modules and packages

## Modules

A **module** is one source file. Its first line of code is `module a.b.c;`. The name of a module must match the path of the file from the package’s source root (`src/`): replace `/` with `.` and drop the extension. The file `src/a/b/c.cnth` is `module a.b.c;`. A mismatch is `E0218`. Each part of the name is snake_case, and the first part `std` is reserved for the standard library (creating a module of your own under `std` is `E0219`).

The contents of a module come in this order: `import` lines, declarations (types, constants, functions), and the `example` and `property` items attached to functions. Nothing else may be written.

## import

```text
import std.list;                  // used as list.take(items: xs, count: 2)
import geometry.shapes as shapes; // used as shapes.Rect, shapes.area(...)
```

- An `import` gives the full name of the module. The **alias** is the last part, or the name after `as`. Names from other modules are always written with the alias as a prefix (`list.take(...)`, `shapes.Rect`, `shapes.Length(value: 2)`).
- There is no wildcard `import`, no `import` that brings names in without an alias, and no re-export.
- An `import` takes one line each, and in canonical form the lines are sorted by path.
- Using the same alias twice is `E0207`. An alias cannot be the name of a parameter or a local variable (`E0202`).
- An unused `import` is `E0209`. A module used only inside a type (`i: shapes.Length`) counts as used.
- An `import` of a module that does not exist is `E0206`.
- **Import cycles are prohibited** (`E0210`). As a result, functions in different modules cannot be recursive with each other, and the termination check ([13 Termination](13-termination.md)) can be done module by module.

```cernith-error E0207
module guide.imports;

import std.int;
import std.int;

pub fn magnitude(value: Int) -> Int {
  int.abs(value: value)
}
```

## Visibility

A declaration belongs only to its own module unless it is marked `pub`. There are only two levels of visibility, and there are no `pub(...)` variants.

| Declaration | With `pub` | Without `pub` |
| --- | --- | --- |
| Function, `spec fn` | Callable from other modules. The signature and contract become its specification | Inside the module only |
| Constant | Readable | Inside the module only |
| Record | Fields are readable, and `R(...)` and `R.with(...)` can be used (the type is transparent) | Inside the module only |
| Variant | All cases can be used and matched with `match` | Inside the module only |
| Refined type | Open. `T(value: n)` and `T.check(value: n)` can be used, and the condition acts as a guard. A value can be used as its base type | Inside the module only |
| `distinct` type | **Abstract**. The type name can be used, but neither `T(value: ...)` nor `.value` can be used outside (`E0215`) | Inside the module only |
| `example`, `property` | Not public | |

Using something that is not `pub` from another module is `E0208` (not-exported).

### Abstract types

A `distinct` type is the language’s “sealed type.” Only the module that defines it creates and reads its values. Other modules can touch the values only through the functions that module makes public. To hide a record, make public a `distinct` type over a private record.

```text
type AccountData = record ...;           // private
pub type Account = distinct AccountData; // abstract from outside
pub fn open_account(owner: String) -> Account { ... }
```

The base type of a `pub` `distinct` type may be private.

### No private names in public specifications

The parameter and return types of a `pub` function, the field types of a `pub` record or variant, and the type of a `pub` constant must be public types. The contract of a `pub` function may call only `pub` `spec fn` items and may read only `pub` constants. A violation is `E0216` (private-in-public), because a caller must be able to read everything that binds it.

```cernith-error E0216
module guide.visibility;

record Inner {
  value: Int,
}

pub fn make() -> Inner {
  Inner(value: 1)
}
```

### Where tests go

An `example` and a `property` are attached to **a function in the same module** (in `example e for f`, `f` is a function declared in that module). A test for a function in another module is written as an ordinary function in the calling module.

## Constants

A constant is pure and terminates. It is evaluated once, before the program starts, in dependency order. A cycle in which a constant depends on itself (through other constants or functions) is `E0211`. The order of independent constants is fixed by module path and then by name, and does not depend on the order of the file system.

## Packages

A **package** is a directory that has `cernith.toml` at its root and keeps its sources under `src/`. A program is made of one package, the standard library `std`, and dependency packages specified by path. There is no package registry and there are no versions.

```toml
name = "inventory"
entry = "app.main"      # the module that has main (omit for a library)

[dependencies]
geometry = { path = "../geometry" }
```

- `name` is required and is a snake_case name (`std` is not allowed).
- `entry` is the module that has the `main` that `cernith run` executes.
- An unknown key is `E0224`.
- `targets` is the list of backends that the package supports (`python`, `javascript`, `wasm`, `native`). If it is omitted, the package supports all of them. Wasm builds for hosts and for the browser all count as `wasm`. `cernith build` refuses a backend that is not in the list (`E0229`), and `cernith difftest` runs only the declared backends. A package can support only the backends that all of its dependencies support (`E0228`).
- `minimum-level` and `[budget]` are also checked as keys, but they are not yet enforced.

The directory of a package can be passed to a command as it is (`cernith check app/`, `cernith test app/`, `cernith run app/`).

### Dependencies

Each key in `[dependencies]` must equal the `name` in the `cernith.toml` of the dependency (`E0225`). **Every module name in a dependency begins with that name.** The file `../geometry/src/geometry/shapes.cnth` is `module geometry.shapes;`. The using side writes `import geometry.shapes as shapes;`, just as for an ordinary module. The visibility rules are as above.

`cernith.lock` holds the SHA-256 hashes of the manifest and the `.cnth` files of every dependency, including transitive ones. `cernith lock` writes it. It is `E0226` if the file is missing, if the contents of a dependency differ from the hash, or if a dependency is not listed. Check what changed, and then run `cernith lock`.

The following is an example of an `app` package that uses `geometry`.

```text
geometry/
  cernith.toml          name = "geometry"
  src/geometry/shapes.cnth
app/
  cernith.toml          name = "app" / entry = "app.main" / [dependencies] geometry = { path = "../geometry" }
  src/app/main.cnth
```

```text
module geometry.shapes;

pub type Length = Int where self >= 0;

pub record Rect {
  width: Length,
  height: Length,
}

pub type Side = distinct Int;

pub fn area(rect: Rect) -> Int {
  rect.width * rect.height
}

pub fn make_side(value: Int) -> Side {
  requires value >= 0;

  Side(value: value)
}

pub fn side_length(side: Side) -> Int {
  side.value
}
```

```text
module app.main;

import geometry.shapes as shapes;

pub fn main(args: List<String>) -> Int {
  let rect = shapes.Rect(
    width: shapes.Length(value: 2),
    height: shapes.Length(value: 5),
  );
  shapes.area(rect: rect) + shapes.side_length(side: shapes.make_side(value: 4))
}
```

`shapes.Rect(...)` and `shapes.Length(value: 2)` can be created from outside because `Rect` is a transparent `pub` record and `Length` is an open refined type. `shapes.Side(value: 4)` is `E0215`, because `Side` is an abstract type. Calling a function of `geometry` that is not `pub` (for example, a private `fn hidden()` added to `shapes.cnth`) is `E0208`.

## Interfaces and fingerprints

The **interface** of a module is its public declarations without their bodies. For a function it is the signature and contract (for a `pub spec fn`, the body as well); for a record, variant, or refined type it is the entire declaration; for a `distinct` type it is the existence only; and for a constant it is the name and type. The **fingerprint** of a module is a hash of its interface and the fingerprints of the modules it imports.

- Changing a function body, a private declaration, an `example`, or the inside of a `distinct` type does not change the fingerprint.
- Changing a public signature, a contract, a record or variant, or the condition of a refined type changes the fingerprint of that module and of every module that imports it (transitively).

The implementation can reuse the check results for a module whose source and whose imports’ fingerprints have not changed. The results always agree with a full check.

## Diagnostic codes

| Code | Name | Meaning |
| --- | --- | --- |
| `E0206` | unknown-module | `import` of a module that does not exist |
| `E0207` | duplicate-definition | A name defined twice, or a duplicate `import` |
| `E0208` | not-exported | Used a private name from another module |
| `E0209` | unused-import | An unused `import` |
| `E0210` | import-cycle | A cycle of `import` |
| `E0211` | constant-cycle | A constant depends on itself |
| `E0215` | abstract-type | Constructed a `distinct` type or used `.value` outside the defining module |
| `E0216` | private-in-public | A public signature, field, or contract contains a private name |
| `E0218` | module-path-mismatch | The module name does not match the file path |
| `E0219` | reserved-module-name | A module name under `std` |
| `E0224` | invalid-manifest | `cernith.toml` is unusable |
| `E0225` | dependency-not-found | A dependency is missing, has the wrong name, or appears in two places |
| `E0226` | lock-out-of-date | `cernith.lock` is missing or does not match the dependencies |
| `E0227` | dependency-cycle | Packages depend on each other in a cycle |
| `E0228` | dependency-lacks-target | A dependency does not support a backend that the package supports |
| `E0229` | target-not-declared | A build was attempted for a backend that the package does not declare in `targets` |

## Related

- `distinct` types and refined types: [08 Refined types and distinct types](08-refined-and-distinct-types.md)
- Functions implemented in other languages: [15 Foreign functions](15-foreign-functions.md)
