# std.console

This module provides access to standard input, standard output, and standard error through the `Console` capability. Import it with `import std.console;` and call functions such as `console.write_line(access: ..., text: ...)`. Each function takes `access: console.Console` as its first argument.

`std.io` ([io-clock-random](io-clock-random.md)) can be called by anyone who declares `uses io`. In contrast, the functions of `std.console` can be called only from a function that has received a `Console` value. A `Console` is passed as the second argument of `main` when it is written in the program’s `manifest` declaration. It cannot be created anywhere else (error `E1301`). This makes it possible to tell whether a function uses standard input and output just by looking at the function’s arguments ([17 Resources and capabilities](../language/17-resources-and-capabilities.md)).

## Types

```text
pub capability Console;

pub variant ConsoleError {
  Failed(reason: String),
}

pub variant Stream {
  Output,
  Error,
}
```

- `Console` is a capability with no fields. It has no grant contents; only holding it is meaningful. It cannot be compared with `==`.
- `ConsoleError` is the error for when reading or writing fails. `reason` holds a string giving the reason.
- `Stream` names the two output streams a program can ask about: `Output` (standard output) and `Error` (standard error).

## Program shape

A program that uses `Console` requests `Console` in its `manifest` and receives it as the second argument of `main`.

```text
pub manifest Caps {
  console: console.Console,
}

pub fn main(args: List<String>, caps: Caps) -> Int uses io { ... }
```

A function that uses `Console` takes an `access: console.Console` argument and declares `uses io`. Merely having the argument does not count as declaring the effect, so forgetting `uses io` is error `E0401`.

```cernith-error E0401
module guide.console_forgot_io;

import std.console;

pub fn greet(
  access: console.Console,
) -> Result<Unit, console.ConsoleError> {
  console.write_line(access: access, text: "hello")
}
```

Trying to create a `Console` yourself is error `E1301`.

```cernith-error E1301
module guide.console_made_by_hand;

import std.console;

pub fn greet() -> Result<Unit, console.ConsoleError> uses io {
  console.write_line(access: console.Console(), text: "hello")
}
```

## Function list

| Function | Return type | Meaning |
| --- | --- | --- |
| `console.read_all_lines(access)` | `Result<List<String>, ConsoleError>` | All lines of standard input |
| `console.read_all_bytes(access)` | `Result<Bytes, ConsoleError>` | All bytes of standard input |
| `console.fold_lines(access, initial, step)` | `Result<A, ConsoleError>` | Folds the lines of standard input in order |
| `console.write_line(access, text)` | `Result<Unit, ConsoleError>` | Writes one line to standard output |
| `console.write_bytes(access, bytes)` | `Result<Unit, ConsoleError>` | Writes a byte sequence to standard output as it is |
| `console.write_error_line(access, text)` | `Result<Unit, ConsoleError>` | Writes one line to standard error |
| `console.is_terminal(access, stream)` | `Bool` | Whether the stream is a terminal |
| `console.width(access, stream)` | `Option<Int>` | The width of the terminal in columns |
| `console.describe(error)` | `String` | Turns a `ConsoleError` into a string |
| `console.simulated()` | `Console` | A capability for tests |

Except for `describe` and `simulated`, `uses io` is required. `fold_lines` is effect-polymorphic, and the caller also declares the effect of `step` ([11 Effects](../language/11-effects.md)).

## Function details

### console.read_all_lines

```text
pub fn read_all_lines(
  access: Console,
) -> Result<List<String>, ConsoleError> uses io
```

Returns standard input as a list of lines. The line terminators at the ends of lines (`\n`, `\r\n`) are not included. A last line without a terminator still counts as a line. If the input is empty, the result is `Ok(value: [])`. If reading fails, the result is `Err(error: ConsoleError.Failed(reason: ...))`. If standard input is not valid UTF-8 text, the result is `Err(error: ConsoleError.Failed(reason: "malformed-text"))`; `console.read_all_bytes` still reads the raw bytes.

Calling it a second or later time in the same program returns the same lines.

### console.read_all_bytes

```text
pub fn read_all_bytes(
  access: Console,
) -> Result<Bytes, ConsoleError> uses io
```

Returns the whole of standard input as a byte sequence. Use it to read data that is not text, such as images.

- If it is called **before** a function that reads lines (`read_all_lines`, `fold_lines`), it returns the bytes of the input as they are. In this case standard input has been read to the end, so a later `read_all_lines` returns an empty list.
- If it is called **after** lines have been read, it returns the lines that were read, each with `\n` appended at the end, joined together (even if the original line terminator was `\r\n`, it becomes `\n`).
- The test world (`input` of `World`) does not have the original bytes, so it is always in this form.

### console.fold_lines

```text
pub fn fold_lines<A, effect E>(
  access: Console,
  initial: A,
  step: fn(total: A, line: String) -> A uses E,
) -> Result<A, ConsoleError> uses io, E
```

Reads standard input line by line and applies `step` in order from the first line, combining the lines into a single value. The labels of the arguments of `step` are `total` (the value so far) and `line` (that line). The first `total` is `initial`, and if there are no lines the result is `Ok(value: initial)`. The result is the same as reading the lines with `read_all_lines` and then running a `for` loop yourself.

### console.write_line

```text
pub fn write_line(
  access: Console,
  text: String,
) -> Result<Unit, ConsoleError> uses io
```

Writes `text` to standard output as one line. Write the result of a program with this function. In a test, the lines written go into `output` of the `Outcome` in the order they were written.

### console.write_bytes

```text
pub fn write_bytes(
  access: Console,
  bytes: Bytes,
) -> Result<Unit, ConsoleError> uses io
```

Writes `bytes` to standard output as they are. No line terminator is added. When it is called alternately with `write_line`, the output appears in the order written. In a test, the bytes do not go into `output`; they can be read as the byte sequences of each call with `sim.stdout_chunks` ([sim](sim.md)).

### console.write_error_line

```text
pub fn write_error_line(
  access: Console,
  text: String,
) -> Result<Unit, ConsoleError> uses io
```

Writes `text` to standard error as one line. Write the warnings and error reports that a program emits here, separately from its result. In a test, the lines do not go into `output`; they can be read with `sim.stderr_lines` ([sim](sim.md)).

### console.is_terminal

```text
pub fn is_terminal(access: Console, stream: Stream) -> Bool uses io
```

Returns `true` if the stream is connected to a terminal, or `false` if it is a pipe or file. Targets without a terminal, such as a page or a module without a host, return `false`. Use this to decide whether to display colors or progress indicators. Simulated standard streams are pipes by default; adding `sim.terminal(width: ...)` to the setup of `sim.with_sim` makes both streams terminals ([sim](sim.md)).

### console.width

```text
pub fn width(access: Console, stream: Stream) -> Option<Int> uses io
```

Returns the terminal width in columns as `Some(value: columns)` if the stream is a terminal and the system can report its size; otherwise, returns `None`. In a simulated world, the width is the value passed to `sim.terminal`.

### console.describe

```text
pub fn describe(error: ConsoleError) -> String
```

Turns the error into a string consisting of `"console: "` followed by the reason. For `ConsoleError.Failed(reason: "closed")`, the result is `"console: closed"`. It is a pure function.

### console.simulated

```text
pub fn simulated() -> Console
```

Returns a `Console` for tests. It can be called only inside `example` and `property`, and inside functions passed to `std.test.with_world` or `std.sim.with_sim`. Calling it anywhere else is error `E1302`. A program receives the real `Console` from its `manifest`.

```cernith-error E1302
module guide.console_simulated_outside_a_test;

import std.console;

pub fn greet() -> Result<Unit, console.ConsoleError> uses io {
  console.write_line(access: console.simulated(), text: "hello")
}
```

## Example

The following module contains a program (`report_total` and `main`) that adds up the lines of standard input as integers and writes the result to standard output, together with small functions that use the functions of `Console`. A line that cannot be read as a number is treated as 0.

In tests, a `Console` is created with `console.simulated()`, and `main` is given the record `Caps(console: console.simulated())`, which has the same shape as the `manifest`. The input lines are given by `input` of `test.World`.

```cernith
module guide.console_tools;

import std.bytes as bytes_lib;
import std.console;
import std.int;
import std.option;
import std.sim;
import std.test;

pub manifest Caps {
  console: console.Console,
}

pub fn byte_count(
  access: console.Console,
) -> Result<Int, console.ConsoleError> uses io {
  let bytes = console.read_all_bytes(access: access)?;
  Ok(value: bytes_lib.length(bytes: bytes))
}

example byte_count_counts_the_lines_with_their_line_feeds for byte_count {
  test.with_world(
    world: input_of(lines: ["ab", "c"]),
    run: fn() -> Result<Int, console.ConsoleError> uses clock, io, random {
      byte_count(access: console.simulated())
    },
  ).value
    == Ok(value: 5)
    and test.with_world(
      world: input_of(lines: []),
      run: fn() -> Result<Int, console.ConsoleError> uses clock, io, random {
        byte_count(access: console.simulated())
      },
    ).value
      == Ok(value: 0)
}

pub fn copy_lines(
  access: console.Console,
) -> Result<Int, console.ConsoleError> uses io {
  let lines = console.read_all_lines(access: access)?;
  for line in lines {
    console.write_line(access: access, text: line)?;
  }
  Ok(value: lines.length)
}

example copy_lines_writes_every_line for copy_lines {
  test.with_world(
    world: input_of(lines: ["x", "", "z"]),
    run: fn() -> Result<Int, console.ConsoleError> uses clock, io, random {
      copy_lines(access: console.simulated())
    },
  )
    == test.Outcome(value: Ok(value: 3), output: ["x", "", "z"])
    and test.with_world(
      world: input_of(lines: []),
      run: fn() -> Result<Int, console.ConsoleError> uses clock, io, random {
        copy_lines(access: console.simulated())
      },
    )
      == test.Outcome(value: Ok(value: 0), output: [])
}

pub fn emit(
  access: console.Console,
  bytes: Bytes,
) -> Result<Unit, console.ConsoleError> uses io {
  console.write_bytes(access: access, bytes: bytes)
}

example emit_writes_the_bytes_as_they_are for emit {
  match run_emit() {
    outcome => outcome.value == Ok(value: ())
      and sim.stdout_chunks(outcome: outcome)
        == [bytes_lib.from_text(text: "AB")]
      and outcome.output == [],
  }
}

pub fn failure_text(reason: String) -> String {
  console.describe(error: console.ConsoleError.Failed(reason: reason))
}

example failure_text_names_the_failure for failure_text {
  failure_text(reason: "closed") == "console: closed"
}

fn input_of(lines: List<String>) -> test.World {
  test.World(input: lines, now_ms: 0, seed: 0)
}

pub fn main(args: List<String>, caps: Caps) -> Int uses io {
  match report_total(access: caps.console) {
    Ok(value: _) => 0,
    Err(error: error) => {
      discard warn(access: caps.console, text: console.describe(error: error));
      1
    },
  }
}

example main_of_a_simulated_world for main {
  test.with_world(
    world: input_of(lines: ["5", "6"]),
    run: fn() -> Int uses clock, io, random {
      main(args: [], caps: Caps(console: console.simulated()))
    },
  )
    == test.Outcome(value: 0, output: ["total 11"])
}

pub fn report_total(
  access: console.Console,
) -> Result<Unit, console.ConsoleError> uses io {
  let total = console.fold_lines(
    access: access,
    initial: 0,
    step: fn(total: Int, line: String) -> Int {
      total + option.unwrap_or(option: int.parse(text: line), default: 0)
    },
  )?;
  console.write_line(
    access: access,
    text: "total " ++ int.to_text(value: total),
  )
}

example report_total_sums_the_lines for report_total {
  test.with_world(
    world: input_of(lines: ["1", "2", "x", "4"]),
    run: fn() -> Result<Unit, console.ConsoleError> uses clock, io, random {
      report_total(access: console.simulated())
    },
  )
    == test.Outcome(value: Ok(value: ()), output: ["total 7"])
    and test.with_world(
      world: input_of(lines: []),
      run: fn() -> Result<Unit, console.ConsoleError> uses clock, io, random {
        report_total(access: console.simulated())
      },
    )
      == test.Outcome(value: Ok(value: ()), output: ["total 0"])
}

fn run_emit() -> sim.SimOutcome<Result<Unit, console.ConsoleError>> {
  sim.with_sim(
    world: input_of(lines: []),
    setup: [],
    run: fn() -> Result<Unit, console.ConsoleError> uses clock, io, random {
      let first = emit(
        access: console.simulated(),
        bytes: bytes_lib.from_text(text: "AB"),
      );
      first
    },
  )
}

fn run_warn() -> sim.SimOutcome<Result<Unit, console.ConsoleError>> {
  sim.with_sim(
    world: input_of(lines: []),
    setup: [],
    run: fn() -> Result<Unit, console.ConsoleError> uses clock, io, random {
      warn(access: console.simulated(), text: "careful")
    },
  )
}

pub fn warn(
  access: console.Console,
  text: String,
) -> Result<Unit, console.ConsoleError> uses io {
  console.write_error_line(access: access, text: text)
}

example warn_goes_to_the_error_stream for warn {
  match run_warn() {
    outcome => outcome.value == Ok(value: ())
      and sim.stderr_lines(outcome: outcome) == ["careful"]
      and outcome.output == [],
  }
}
```

The exit status of `main` is 0 if `report_total` succeeds. If it fails with a `ConsoleError`, it writes the description to standard error and the exit status is 1.

### Asking about the terminal

The following module chooses a width and decides on colors. The simulated world is a pipe unless the setup says it is a terminal.

```cernith
module guide.console_terminal;

import std.console;
import std.sim;

pub fn columns(access: console.Console) -> Int uses io {
  match console.width(access: access, stream: console.Stream.Output) {
    Some(value: width) => width,
    None => 80,
  }
}

example columns_is_the_width_of_a_terminal_or_80 for columns {
  look(
    setup: [sim.terminal(width: 120)],
    body: fn(access: console.Console) -> Int uses io {
      columns(access: access)
    },
  )
    == 120
    and look(
      setup: [],
      body: fn(access: console.Console) -> Int uses io {
        columns(access: access)
      },
    )
      == 80
}

fn look<T>(
  setup: List<String>,
  body: fn(access: console.Console) -> T uses io,
) -> T {
  sim.with_sim(
    world: sim.quiet_world(),
    setup: setup,
    run: fn() -> T uses clock, io, random {
      body(access: console.simulated())
    },
  ).value
}

pub fn wants_color(access: console.Console) -> Bool uses io {
  console.is_terminal(access: access, stream: console.Stream.Output)
    and not console.is_terminal(access: access, stream: console.Stream.Error)
}

example color_only_for_a_terminal for wants_color {
  look(
    setup: [sim.terminal(width: 80)],
    body: fn(access: console.Console) -> Bool uses io {
      console.is_terminal(access: access, stream: console.Stream.Error)
    },
  )
    == true
    and look(
      setup: [],
      body: fn(access: console.Console) -> Bool uses io {
        wants_color(access: access)
      },
    )
      == false
}
```

## Related

- [io-clock-random](io-clock-random.md): `std.io`, which does not use capabilities
- [sim](sim.md): observing standard error and the output of `write_bytes`
- [test](test.md): `with_world`
- [fs](fs.md), [env](env.md), [http](http.md): other capabilities
- [17 Resources and capabilities](../language/17-resources-and-capabilities.md)
