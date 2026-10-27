# Resources and capabilities

This is the mechanism by which a program handles external resources such as files and sockets safely. It consists of the following four principles.

1. The program does not manage memory. There are no pointers, no allocation and deallocation, and no destructors.
2. A **resource** (a file, a socket, a window) exists only inside the scope that releases it, and cannot leave that scope.
3. **Authority is a value.** An operation that acts on the outside world requires a **capability** received from the program’s `manifest`.
4. No code runs implicitly at a time the program has not written.

## Second-class values

Resource values (and `Scope`, tasks, and channel ends; see [18 Concurrency](18-concurrency.md)) are **second-class**. A second-class name can be used only as an **argument of a function call**, passed to a parameter of the same type (it cannot be an argument of a record or variant constructor or of an operator). A lambda cannot capture a second-class name (`E1101`); pass the name to the lambda as an argument instead.

It cannot be returned, put into a record, variant, list, `Option`, or `Result`, assigned to a `var`, or bound with `let` outside the scope (`E1101`, resource-escapes-scope). If a function parameter has a second-class type, the body of the function is checked by the same rules, so a called function cannot carry the resource out either.

## Resource types

```text
pub resource ReadFile release close_read;

fn close_read(file: ReadFile) -> Result<Unit, FsError> uses io;
```

`resource name release function;` declares an opaque type over an integer handle, similar to a `distinct` type. Only the module that defines the type creates or opens its values. The function given to `release` must be in the same module, take one parameter of that resource type, and return `Result<Unit, E>` (`E1105`).

A function that obtains a resource returns `Result<resource type, E>`. A resource type is neither `Eq` nor `Ord`, and cannot be a type argument or a field (`E1106`).

## using

```text
using name = acquire expression { body }
```

The acquire expression is a `Result<R, E>` (where `R` is a resource type), the body is a `Result<T, E>` with the same `E`, and the type of the whole `using` is `Result<T, E>`. A `using` is an expression and can be nested. `name` is a second-class immutable binding usable only inside the body.

Evaluation proceeds in this order.

1. Evaluate the acquire expression. If it is `Err(e)`, that is the value of the `using`. Nothing was acquired, so nothing is released.
2. If it is `Ok(r)`, bind `name` to `r` and evaluate the body.
3. However the body ends (reaching the end, an early exit with `?`, or cancellation), the release function runs **exactly once**. A `return` cannot be written inside the body (`E1107`). The value of the body is the value of the `using`.
4. If the body is `Ok` and the release returns `Err(e2)`, the value of the `using` is `Err(e2)`. If the body is `Err(e1)` and the release also fails, the value of the `using` is `Err(e1)`, and the `e2` from the release is recorded as a **suppressed** error and does not become a value.
5. A defect ends the program. The release function is **not run** (because the state cannot be trusted).

The following example opens a file, counts its lines, and closes the file without writing a call to close it.

```cernith
module guide.lines;

import std.fs;

pub fn line_count(
  files: fs.ReadOnlyFs,
  path: String,
) -> Result<Int, fs.FsError> uses io {
  using file = fs.open_read(access: files, path: path) {
    let lines = fs.read_lines(file: file)?;
    Ok(value: lines.length)
  }
}
```

Binding a call that acquires a resource with `let` outside a `using`, or unwrapping it with `?` or `match`, is `E1103` (resource-outside-using). If the error type of the body of a `using` differs from the error type of the acquire and release, it is `E1102`; align them with `result.map_error`.

```cernith-error E1103
module guide.lines;

import std.fs;

pub fn bad(files: fs.ReadOnlyFs) -> Result<Int, fs.FsError> uses io {
  let file = fs.open_read(access: files, path: "a")?;
  let text = fs.read_all(file: file)?;
  Ok(value: text.length)
}
```

```cernith-error E1101
module guide.lines;

import std.fs;

pub fn bad(files: fs.ReadOnlyFs) -> Result<Int, fs.FsError> uses io {
  using file = fs.open_read(access: files, path: "a") {
    let all = [file];
    Ok(value: all.length)
  }
}
```

```cernith-error E1102
module guide.lines;

import std.fs;

pub variant Other {
  Oops,
}

pub fn bad(files: fs.ReadOnlyFs) -> Result<Int, fs.FsError> uses io {
  using file = fs.open_read(access: files, path: "a") {
    discard fs.read_all(file: file);
    Err(error: Other.Oops)
  }
}
```

```cernith-error E1107
module guide.lines;

import std.fs;

pub fn bad(files: fs.ReadOnlyFs) -> Result<Int, fs.FsError> uses io {
  using file = fs.open_read(access: files, path: "a") {
    discard fs.read_all(file: file);
    return Ok(value: 1);
  }
}
```

## Capabilities

A **capability** is a value that represents the authority for an operation on the outside world.

```text
pub capability Fs {
  root: String,
};

pub capability Console;
```

- `capability name { field: type, ... };` (or `capability name;` if it carries no content of authority) declares an opaque, first-class type (one that can be passed around freely). The fields are the **content of the authority** (the grant), and only the module that defines the capability can read them. A capability is not `Eq`.
- Only the `manifest` and the module that defines a capability can create it. Creating one anywhere else is `E1301`.
- A **derived** capability that narrows a capability is created from the original capability by a library function (`fs.read_only(access: files)`, `fs.narrow(access: files, folder: "reports")`). There is no function that grants more authority than the original.
- An operation on a capability takes the capability as a labeled parameter **and declares `uses io`**. Holding a capability alone does not carry an effect. If a function that calls the operation does not declare `uses io`, it is `E0401` ([11 Effects](11-effects.md)). A function that has no capability parameter cannot touch the resource.
- `clock` and `random` remain effects that do not use capabilities.

### Capabilities of the standard library

| Capability | Module | Grant written in the `manifest` | Authority |
| --- | --- | --- | --- |
| `Console` | [std.console](../stdlib/console.md) | none | Standard input and output |
| `Fs`, `ReadOnlyFs` | [std.fs](../stdlib/fs.md) | `root`: a folder | Files below the root (read and write, or read only) |
| `Env` | [std.env](../stdlib/env.md) | none | Reading environment variables |
| `Http` | [std.http](../stdlib/http.md) | `hosts`: the host names that may be reached (`"*"` is for tests) | HTTP requests to those hosts |
| `Process` | [std.process](../stdlib/process.md) | `programs` (name, absolute path, and argument rule of each), `work_root`, `environment`, `max_output_bytes`, `max_seconds` | Starting the listed programs and reading what they print |

`Process` is the largest authority in the list, because the program it starts is not checked by Cernith. Its grant is an allow-list: a call names a program, never a path; it can pass only the arguments the rule of that program allows; the child sees only the environment variables the grant lists; and every call has a time limit and an output limit. `cernith audit` prints the whole grant ([std.process](../stdlib/process.md)).

### manifest

```text
pub manifest Caps {
  files: fs.Fs(root: "."),
  console: console.Console,
}

pub fn main(args: List<String>, caps: Caps) -> Int uses io { ... }
```

`manifest name { field: capability type(grant), ... }` is the declaration of all the authority a program needs, and is a record whose fields are capability values. The content of the authority is written in the source. A program has at most one `manifest`, and `main` receives it as its second parameter; otherwise it is `E1303`. If a `manifest` names a capability that the world of the backend being built does not provide, the program cannot be built for that backend, and the error is `E1305` (for example, a browser page has files only in its private area `@opfs`, not in the working directory, and does not provide `Process`). The functions of `std.io`, `std.clock`, and `std.random` do not take a capability and can be called by declaring `uses io` (`clock`, `random`). This is the same whether `main` has the form `main(args)` (which does not use a `manifest`) or the form that uses a `manifest`.

```cernith
module guide.report;

import std.console;
import std.fs;
import std.int;
import std.sim;

pub manifest Caps {
  files: fs.Fs(root: "."),
  console: console.Console,
}

pub fn line_count(
  files: fs.ReadOnlyFs,
  path: String,
) -> Result<Int, fs.FsError> uses io {
  using file = fs.open_read(access: files, path: path) {
    let lines = fs.read_lines(file: file)?;
    Ok(value: lines.length)
  }
}

example counts_the_lines_of_a_simulated_file for line_count {
  sim.with_sim(
    world: sim.quiet_world(),
    setup: [sim.file(path: "notes.txt", content: "a\nb\nc\n")],
    run: fn() -> Result<Int, fs.FsError> uses io {
      line_count(files: fs.simulated_read_only(), path: "notes.txt")
    },
  ).value
    == Ok(value: 3)
}

pub fn main(args: List<String>, caps: Caps) -> Int uses io {
  match line_count(files: fs.read_only(access: caps.files), path: "notes.txt") {
    Ok(value: count) => {
      discard console.write_line(
        access: caps.console,
        text: int.to_text(value: count),
      );
      0
    },
    Err(error: _) => 1,
  }
}
```

The `audit` command of the implementation lists the `manifest` together with all foreign functions as the trust boundary.

### Authority in tests

In tests, capabilities can be created by functions such as `fs.simulated()`, `fs.simulated_read_only()`, `console.simulated()`, and `process.simulated_with(...)`. These work against the simulated world, and can be called only inside the body of an `example` and a `property`, and inside the `run:` lambda passed to `std.sim.with_sim` and `std.test.with_world`. Calling them anywhere else (for example, inside an ordinary helper function) is `E1302`. Pass capabilities to helper functions as arguments. To use a `manifest` record in a test, assemble it field by field, as in `Caps(files: fs.simulated(), console: console.simulated())` ([std.sim](../stdlib/sim.md)).

```cernith-error E1302
module guide.report;

import std.fs;

pub fn make() -> fs.ReadOnlyFs {
  fs.simulated_read_only()
}
```

### Obligations of the library

A standard library function that touches an outside resource has a specification, an `example`, and a counterpart in the simulated world. A function that obtains a resource returns `Result<resource, E>`, where `E` is a closed variant. Files, the console, environment variables, the network, and other programs (`std.fs`, `std.console`, `std.env`, `std.http`, `std.process`, and `std.path` for paths) are built on this mechanism. `Fs` and `ReadOnlyFs` have a root directory, and every path is normalized so that it cannot leave the root. A path that leaves the root is `Denied`. The functions that the standard library uses internally (`std.sys.request`, `std.async.erase`, `std.async.reveal`, and others) can touch resources without a capability or change the type of a value, so a program that calls them gets `E1306`. Use the functions of `std.fs`, `std.env`, `std.http`, `std.console`, `std.process`, and the other modules that take a capability.

## Diagnostic codes

| Code | Name | Meaning |
| --- | --- | --- |
| `E1101` | resource-escapes-scope | A second-class value was returned, stored, captured, or used other than as a call argument |
| `E1102` | release-error-not-covered | The error type of the body of a `using` differs from the error type of the acquire and release |
| `E1103` | resource-outside-using | The acquisition of a resource was bound outside a `using` |
| `E1105` | bad-resource-declaration | The `release` function is missing or has the wrong signature |
| `E1106` | resource-as-type-argument | A resource type was used as a type argument or a field |
| `E1107` | return-inside-using | A `return` inside the body of a `using` |
| `E1301` | capability-outside-manifest | A capability was created outside the `manifest` and the defining module |
| `E1302` | simulated-outside-test | A capability for testing was created outside a test |
| `E1303` | bad-manifest | There are two `manifest` items, a field is not a capability, or `main` does not receive it |
| `E1305` | capability-unavailable-on-target | The `manifest` names a capability that the world of the backend being built does not provide |
| `E1306` | internal-function | A program called a function that is for internal use by the standard library |

## Related

- Effects: [11 Effects](11-effects.md)
- Concurrency (tasks and resources): [18 Concurrency](18-concurrency.md)
- Standard library: [std.fs](../stdlib/fs.md), [std.console](../stdlib/console.md), [std.http](../stdlib/http.md), [std.process](../stdlib/process.md), [std.sim](../stdlib/sim.md)
