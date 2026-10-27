# std.process

A module that runs another program and reads what it printed, through the capability `Process`. Write `import std.process;` and call `process.run(access: ..., program: ..., arguments: ..., options: ...)`.

Running a program is the largest authority a program can ask for: the other program is not checked by Cernith. For this reason, **the grant in the `manifest` is the boundary**. It lists the programs that may be started (by a name, with their absolute paths and the arguments each may receive), the folders they may work in, the environment variables they may see, and the limits on time and output. A call can only narrow what the grant allows. `cernith audit` prints the grant ([17 Resources and capabilities](../language/17-resources-and-capabilities.md)).

`run` waits for the program to end. It starts one program at a time, and no program outlives the call.

## Types

```text
pub capability Process {
  programs: List<ProgramGrant>,
  work_root: String,
  environment: List<String>,
  max_output_bytes: Int,
  max_seconds: Int,
}

pub record ProgramGrant {
  name: String,
  path: String,
  arguments: ArgumentRule,
}

pub variant ArgumentRule {
  Any,
  Prefix(words: List<String>),
  OneOf(choices: List<List<String>>),
}

pub record EnvValue {
  name: String,
  value: String,
}

pub record Options {
  folder: String,
  environment: List<EnvValue>,
  seconds: Int,
  input: Bytes,
}

pub record Output {
  exit: Exit,
  stdout: Bytes,
  stderr: Bytes,
}

pub variant Exit {
  Exited(code: Int),
  Signaled(name: String),
  TimedOut,
}

pub variant ProcessError {
  Denied(program: String),
  ArgumentsDenied(program: String),
  NotFound(program: String),
  BadRequest(detail: String),
  SpawnFailed(detail: String),
  OutputLimit,
  Unsupported,
}
```

`Process` is a capability. It can be created only in the `manifest` and in `std.process` itself (error `E1301` elsewhere), its fields cannot be read (error `E0215`), and it cannot be compared with `==`.

```cernith-error E1301
module guide.process_made_by_hand;

import std.process;

pub fn open_access() -> process.Process {
  process.Process(
    programs: [],
    work_root: ".",
    environment: [],
    max_output_bytes: 10,
    max_seconds: 2,
  )
}
```

## The grant

| Field | Meaning |
| --- | --- |
| `programs` | The programs that may be started. Each `ProgramGrant` has a `name` that a call uses, the absolute `path` of the program, and the `arguments` it may receive |
| `work_root` | The folder below which the working folder of a child must be (the rule of `Fs`: a path that leaves it is refused) |
| `environment` | The **names** of the environment variables that the child may see. All others are dropped |
| `max_output_bytes` | The most bytes the child may print on each of standard output and standard error. From 1 to 67108864 (64 MiB) |
| `max_seconds` | The longest time one call may take, in seconds. From 1 to 3600 |

An `ArgumentRule` says which arguments a call may pass:

| Rule | A call may pass |
| --- | --- |
| `Any` | any arguments |
| `Prefix(words: [...])` | arguments that start with these words (and then anything) |
| `OneOf(choices: [...])` | arguments that are equal to one of the lists |

The call names the program by `name`; it never gives a path. The grant is written in the `manifest`, and the first example below shows it.

## Rules that protect the machine

`run` checks the call against the grant **before** it asks the host, and the host checks again what it can, for example the form of the path by the rules of its system.

- **No shell and no search of `PATH`.** The program that runs is the one at the `path` of the grant. The arguments are passed as they are, one by one, so they are not read a second time by any shell.
- **A fresh environment.** The child sees only the variables named in `environment`. A call may give a value to a granted name in `Options.environment`; a name without a value takes the host's own value if there is one. A call cannot name a variable that the grant does not list.
- **Variables that change what a program loads cannot be set**, even if the grant lists the name: `LD_PRELOAD`, `LD_LIBRARY_PATH`, `DYLD_INSERT_LIBRARIES`, `DYLD_LIBRARY_PATH`, `NODE_OPTIONS`, `PYTHONPATH`, `PYTHONSTARTUP`, `BASH_ENV`, `ENV`, `PROMPT_COMMAND`, and `GIT_SSH_COMMAND` (the case of the letters is ignored). The grant passes the host's own value of such a name.
- **No batch files.** A path that ends in `.bat`, `.cmd`, `.com`, or `.lnk` is never started, because Windows would read the arguments again through the command interpreter.
- **An absolute path.** The path of a program starts at a root (`/` on POSIX) or at a drive letter (`C:\` or `C:/`). On Windows the host also requires a drive letter, so `/usr/bin/git` is refused there as `BadRequest("the path of the program is not absolute")`.
- **Three pipes and no other handle.** The child gets standard input, standard output, and standard error, and nothing else of the program.
- **Limits.** The time of a call is the smaller of `Options.seconds` and `max_seconds`. A child that is still running at the end is killed **with everything it started**, and the call returns `Exit.TimedOut` with what was printed so far. A child that prints more than `max_output_bytes` on one stream is killed and the call returns `OutputLimit`. No grant can raise the limits above one hour, 64 MiB for each output stream, and 64 MiB of input.

These rules narrow the damage a mistake can do, and they are not a sandbox: a granted program can do whatever that program does. Grant only the programs, and only the arguments, that the application needs.

## Function list

| Function | Return type | Meaning |
| --- | --- | --- |
| `process.run(access, program, arguments, options)` | `Result<Output, ProcessError>` | Runs the granted program, waits, and returns how it ended and what it printed |
| `process.options(seconds)` | `Options` | A call with the folder `"."`, no variables, no input, and the time limit |
| `process.allows(rule, arguments)` | `Bool` | Whether the rule lets a call pass these arguments |
| `process.describe(error)` | `String` | Turns a `ProcessError` into a string |
| `process.describe_exit(exit)` | `String` | Turns an `Exit` into a string |
| `process.simulated()` | `Process` | A capability for tests: one program `tool` |
| `process.simulated_with(programs, work_root, environment, max_output_bytes, max_seconds)` | `Process` | A capability for tests with a grant that the test chooses |

`run` requires `uses io`. The others are pure.

## Function details

### process.run

```text
pub fn run(
  access: Process,
  program: String,
  arguments: List<String>,
  options: Options,
) -> Result<Output, ProcessError> uses io
```

Runs the program that the grant names `program`, with `arguments`, and waits for it. The standard input of the child is `options.input`, and its working folder is `options.folder` below `work_root`. The result is `Ok` with an `Output` when the program ended, **whatever its exit code was**: a program that exits with 1 is an `Ok` with `Exit.Exited(code: 1)`, and the program decides what that means. `stdout` and `stderr` are `Bytes`; use `bytes.to_text` to read them as text.

The call is checked in this order, and the first failure is the result.

1. The name is not in `programs`: `Err(error: ProcessError.Denied(program: ...))`.
2. The arguments do not satisfy the rule of the program: `ArgumentsDenied(program: ...)`.
3. Something else is wrong with the grant or the call: `BadRequest(detail: ...)`. The details are the following.

   | `detail` | Cause |
   | --- | --- |
   | `the path of the program is not absolute` | The `path` of the grant |
   | `a batch file or shortcut is never started` | The `path` ends in `.bat`, `.cmd`, `.com`, or `.lnk` |
   | `the grant needs max_seconds from 1 to 3600` | The grant |
   | `the grant needs max_output_bytes from 1 to 67108864` | The grant |
   | `seconds is at least 1` | `options.seconds` is less than 1 |
   | `the input is larger than 64 MiB` | `options.input` |
   | `an argument has a zero character` | An argument contains `"\u{0}"` |
   | `the grant lists the bad variable name NAME` | A name in `environment` that has a character other than a letter, a digit, or `_` |
   | `bad variable NAME` | A name in `options.environment` that is not a variable name, or a value with a zero character |
   | `the variable NAME is not granted` | A name in `options.environment` that the grant does not list |
   | `the variable NAME cannot be set` | A variable that changes what a program loads |
   | `the folder is outside the work root` | `options.folder` leaves `work_root` |

4. The host starts the program. Its failures are `NotFound(program: ...)` (there is no file at the path), `Denied(program: ...)` (the host refused), `SpawnFailed(detail: ...)` (the program could not be started), `BadRequest(detail: ...)`, `OutputLimit` (a stream passed `max_output_bytes`; the child was killed), and `Unsupported` (this target cannot run programs).

How the program ended is in `output.exit`:

| `Exit` | Meaning |
| --- | --- |
| `Exited(code: ...)` | The program ended with that exit code |
| `Signaled(name: ...)` | A signal killed the program (`name` is the name of the signal, such as `SIGKILL`) |
| `TimedOut` | The program was still running at the time limit and was killed with everything it started. `stdout` and `stderr` hold what was printed until then |

### process.options

```text
pub fn options(seconds: Int) -> Options
```

Returns `Options(folder: ".", environment: [], seconds: seconds, input: bytes.empty())`: the working folder is `work_root` itself, no variable is given, there is no standard input, and the call may take `seconds`. Write the `Options` record out to set the folder, the variables, or the input. The folder is a path relative to `work_root`, normalized as in `std.fs`, so it cannot leave it.

### process.allows

```text
pub fn allows(rule: ArgumentRule, arguments: List<String>) -> Bool
```

Returns whether the rule lets a call pass `arguments`: always for `Any`, when `arguments` starts with the words for `Prefix` (a list shorter than the words does not), and when `arguments` equals one of the lists for `OneOf` (nothing for an empty list of choices). `run` uses it for the check of step 2.

### process.describe and process.describe_exit

```text
pub fn describe(error: ProcessError) -> String
pub fn describe_exit(exit: Exit) -> String
```

| Error | String |
| --- | --- |
| `Denied(program: "git")` | `"git: not granted"` |
| `ArgumentsDenied(program: "git")` | `"git: these arguments are not granted"` |
| `NotFound(program: "git")` | `"git: not found"` |
| `BadRequest(detail: "x")` | `"bad request: x"` |
| `SpawnFailed(detail: "x")` | `"could not start: x"` |
| `OutputLimit` | `"output limit reached"` |
| `Unsupported` | `"unsupported here"` |

| Exit | String |
| --- | --- |
| `Exited(code: 3)` | `"exited 3"` |
| `Signaled(name: "SIGKILL")` | `"killed by SIGKILL"` |
| `TimedOut` | `"timed out"` |

### process.simulated and process.simulated_with

```text
pub fn simulated() -> Process
pub fn simulated_with(
  programs: List<ProgramGrant>,
  work_root: String,
  environment: List<String>,
  max_output_bytes: Int,
  max_seconds: Int,
) -> Process
```

Return a `Process` for testing. `simulated()` has one program, named `tool`, at the path `/sim/tool` (it never runs), with any arguments, the working folders below `.`, no variables, and the limits 65536 bytes and 60 seconds. `simulated_with` takes the grant from the test, so a test can check how the program reacts to a refusal.

Both can be called only inside an `example` or a `property`, and inside a function passed to `std.test.with_world` or `std.sim.with_sim`. Calling them anywhere else is error `E1302`.

In the simulated world **no program runs**. `process.run` is answered from the scripts written in the `setup` of `std.sim.with_sim` (`sim.program`, `sim.program_any`, `sim.program_signaled`, and `sim.program_timed_out`), and the world records every command that was started (`sim.processes_started`; see [sim](sim.md)). A program that has no script is `NotFound`. The checks of `run` described above work in the simulated world as well, so a test sees `Denied`, `ArgumentsDenied`, `BadRequest`, and `OutputLimit` without running anything. The `path` of a script is the `path` of the grant.

## Where it works

The interpreter, Python, JavaScript and Wasm on Node (on POSIX only), Wasm under Wasmtime, and native code on Windows can run programs. On other targets the result is `Unsupported`. A page, the WebView shell, and the Linux window do not provide `Process` at all, and a program whose `manifest` asks for it cannot be built for them (error `E1305`; confirmed with `cernith build --target web`, `webview`, and `linux-window`).

## Examples

### Running a granted program

The following module asks for one program, `git`, and only for two argument lists. `run_git` runs it and turns the result into a `String`. The tests use `process.simulated_with` with the same grant and give the program a script; no real program runs.

```cernith
module guide.process_tools;

import std.bytes;
import std.console;
import std.option;
import std.process;
import std.sim;
import std.text;

pub manifest Caps {
  console: console.Console,
  tools: process.Process(
    programs: [
      process.ProgramGrant(
        name: "git",
        path: "/usr/bin/git",
        arguments: process.ArgumentRule.OneOf(
          choices: [["status", "--short"], ["log", "--oneline"]],
        ),
      ),
    ],
    work_root: ".",
    environment: ["LANG"],
    max_output_bytes: 65536,
    max_seconds: 30,
  ),
}

fn grants() -> List<process.ProgramGrant> {
  [
    process.ProgramGrant(
      name: "git",
      path: "/usr/bin/git",
      arguments: process.ArgumentRule.OneOf(
        choices: [["status", "--short"], ["log", "--oneline"]],
      ),
    ),
  ]
}

pub fn main(args: List<String>, caps: Caps) -> Int uses io {
  match run_git(tools: caps.tools, arguments: ["status", "--short"]) {
    Ok(value: changes) => {
      discard console.write_line(access: caps.console, text: changes);
      0
    },
    Err(error: message) => {
      discard console.write_error_line(access: caps.console, text: message);
      1
    },
  }
}

pub fn run_git(
  tools: process.Process,
  arguments: List<String>,
) -> Result<String, String> uses io {
  match process.run(
    access: tools,
    program: "git",
    arguments: arguments,
    options: process.options(seconds: 10),
  ) {
    Ok(value: output) => match output.exit {
      process.Exit.Exited(code: 0) => Ok(value: text_of(data: output.stdout)),
      other => Err(error: process.describe_exit(exit: other)),
    },
    Err(error: error) => Err(error: process.describe(error: error)),
  }
}

example run_git_needs_a_script_in_the_simulated_world for run_git {
  simulate(
    setup: [],
    body: fn(tools: process.Process) -> Result<String, String> uses io {
      run_git(tools: tools, arguments: ["log", "--oneline"])
    },
  ).value
    == Err(error: "git: not found")
}

example run_git_refuses_arguments_outside_the_grant for run_git {
  match simulate(
    setup: [
      sim.program_any(
        path: "/usr/bin/git",
        exit_code: 0,
        stdout: "",
        stderr: "",
      ),
    ],
    body: fn(tools: process.Process) -> Result<String, String> uses io {
      run_git(tools: tools, arguments: ["push", "--force"])
    },
  ) {
    outcome => outcome.value
      == Err(error: "git: these arguments are not granted")
      and sim.processes_started(outcome: outcome) == [],
  }
}

example run_git_reports_a_time_out for run_git {
  simulate(
    setup: [
      sim.program_timed_out(
        path: "/usr/bin/git",
        arguments: ["status", "--short"],
        stdout: "partial",
      ),
    ],
    body: fn(tools: process.Process) -> Result<String, String> uses io {
      run_git(tools: tools, arguments: ["status", "--short"])
    },
  ).value
    == Err(error: "timed out")
}

example run_git_reports_how_the_program_ended for run_git {
  simulate(
    setup: [
      sim.program(
        path: "/usr/bin/git",
        arguments: ["log", "--oneline"],
        exit_code: 128,
        stdout: "",
        stderr: "fatal",
      ),
      sim.program_signaled(
        path: "/usr/bin/git",
        arguments: ["status", "--short"],
        signal: "SIGKILL",
      ),
    ],
    body: fn(tools: process.Process) -> List<Result<String, String>> uses io {
      [
        run_git(tools: tools, arguments: ["log", "--oneline"]),
        run_git(tools: tools, arguments: ["status", "--short"]),
      ]
    },
  ).value
    == [Err(error: "exited 128"), Err(error: "killed by SIGKILL")]
}

example run_git_returns_the_text_the_program_printed for run_git {
  match simulate(
    setup: [
      sim.program(
        path: "/usr/bin/git",
        arguments: ["status", "--short"],
        exit_code: 0,
        stdout: " M a.txt\n",
        stderr: "",
      ),
    ],
    body: fn(tools: process.Process) -> Result<String, String> uses io {
      run_git(tools: tools, arguments: ["status", "--short"])
    },
  ) {
    outcome => outcome.value == Ok(value: "M a.txt")
      and sim.processes_started(outcome: outcome)
        == [["/usr/bin/git", "status", "--short"]],
  }
}

fn simulate<T>(
  setup: List<String>,
  body: fn(tools: process.Process) -> T uses io,
) -> sim.SimOutcome<T> {
  sim.with_sim(
    world: sim.quiet_world(),
    setup: setup,
    run: fn() -> T uses clock, io, random {
      body(
        tools: process.simulated_with(
          programs: grants(),
          work_root: ".",
          environment: ["LANG"],
          max_output_bytes: 65536,
          max_seconds: 30,
        ),
      )
    },
  )
}

fn text_of(data: Bytes) -> String {
  text.trim(
    of: option.unwrap_or(option: bytes.to_text(bytes: data), default: ""),
  )
}
```

The `manifest` is shown by `cernith audit`, as the grant of the program:

```text
$ cernith audit process_tools.cnth
...
  "field": "tools",
  "grant": {
    "environment": [
      "LANG"
    ],
    "max_output_bytes": 65536,
    "max_seconds": 30,
    "programs": [
      {
        "arguments": {
          "case": "OneOf",
          "choices": [
            [
              "status",
              "--short"
            ],
            [
              "log",
              "--oneline"
            ]
          ]
        },
        "name": "git",
        "path": "/usr/bin/git"
      }
    ],
    "work_root": "."
  },
  "type": "std.process::Process"
...
```

### Refusals and limits

The next module checks the argument rules and the refusals that `run` makes before it starts anything, and what a time out and an output limit look like.

```cernith
module guide.process_limits;

import std.bytes;
import std.int;
import std.process;
import std.sim;

pub fn allowed(arguments: List<String>) -> List<Bool> {
  [
    process.allows(rule: process.ArgumentRule.Any, arguments: arguments),
    process.allows(
      rule: process.ArgumentRule.Prefix(words: ["status"]),
      arguments: arguments,
    ),
    process.allows(
      rule: process.ArgumentRule.OneOf(
        choices: [["status"], ["log", "-n", "5"]],
      ),
      arguments: arguments,
    ),
  ]
}

example allows_follows_the_rule for allowed {
  allowed(arguments: ["status"]) == [true, true, true]
    and allowed(arguments: ["status", "-s"]) == [true, true, false]
    and allowed(arguments: ["-c", "status"]) == [true, false, false]
    and allowed(arguments: []) == [true, false, false]
}

pub fn attempt(
  program: String,
  options: process.Options,
) -> Result<process.Output, process.ProcessError> {
  sim.with_sim(
    world: sim.quiet_world(),
    setup: [
      sim.program_any(
        path: "/opt/tool",
        exit_code: 0,
        stdout: "hello",
        stderr: "",
      ),
    ],
    run: fn() -> Result<process.Output, process.ProcessError> uses clock, io, random {
      process.run(
        access: process.simulated_with(
          programs: [
            process.ProgramGrant(
              name: "tool",
              path: "/opt/tool",
              arguments: process.ArgumentRule.Any,
            ),
          ],
          work_root: "work",
          environment: ["LANG", "LD_PRELOAD"],
          max_output_bytes: 4,
          max_seconds: 30,
        ),
        program: program,
        arguments: [],
        options: options,
      )
    },
  ).value
}

example a_call_is_checked_before_anything_runs for attempt {
  attempt(program: "rm", options: process.options(seconds: 5))
    == Err(error: process.ProcessError.Denied(program: "rm"))
    and attempt(program: "tool", options: process.options(seconds: 0))
      == Err(
        error: process.ProcessError.BadRequest(detail: "seconds is at least 1"),
      )
    and attempt(
      program: "tool",
      options: process.Options(
        folder: "..",
        environment: [],
        seconds: 5,
        input: bytes.empty(),
      ),
    )
      == Err(
        error: process.ProcessError.BadRequest(
          detail: "the folder is outside the work root",
        ),
      )
    and attempt(
      program: "tool",
      options: process.Options(
        folder: ".",
        environment: [process.EnvValue(name: "HOME", value: "/")],
        seconds: 5,
        input: bytes.empty(),
      ),
    )
      == Err(
        error: process.ProcessError.BadRequest(
          detail: "the variable HOME is not granted",
        ),
      )
    and attempt(
      program: "tool",
      options: process.Options(
        folder: ".",
        environment: [process.EnvValue(name: "LD_PRELOAD", value: "x.so")],
        seconds: 5,
        input: bytes.empty(),
      ),
    )
      == Err(
        error: process.ProcessError.BadRequest(
          detail: "the variable LD_PRELOAD cannot be set",
        ),
      )
}

example output_over_the_limit_is_an_error for attempt {
  attempt(program: "tool", options: process.options(seconds: 5))
    == Err(error: process.ProcessError.OutputLimit)
}

pub fn how_it_ended(setup: List<String>) -> String {
  sim.with_sim(
    world: sim.quiet_world(),
    setup: setup,
    run: fn() -> String uses clock, io, random {
      match process.run(
        access: process.simulated(),
        program: "tool",
        arguments: ["build"],
        options: process.options(seconds: 5),
      ) {
        Ok(value: output) => process.describe_exit(exit: output.exit)
          ++ " after "
          ++ int.to_text(value: bytes.length(bytes: output.stdout))
          ++ " bytes",
        Err(error: error) => process.describe(error: error),
      }
    },
  ).value
}

example a_time_out_keeps_the_output_so_far for how_it_ended {
  how_it_ended(
    setup: [
      sim.program_timed_out(
        path: "/sim/tool",
        arguments: ["build"],
        stdout: "step 1\n",
      ),
    ],
  )
    == "timed out after 7 bytes"
    and how_it_ended(
      setup: [
        sim.program(
          path: "/sim/tool",
          arguments: ["build"],
          exit_code: 3,
          stdout: "",
          stderr: "no",
        ),
      ],
    )
      == "exited 3 after 0 bytes"
    and how_it_ended(setup: []) == "tool: not found"
}
```

## Related

- [sim](sim.md): `program`, `program_any`, `program_signaled`, `program_timed_out`, and `processes_started`
- [console](console.md), [fs](fs.md), [http](http.md): other capabilities
- [bytes](bytes.md): `bytes.to_text` and the other functions of `Bytes`
- [17 Resources and capabilities](../language/17-resources-and-capabilities.md)
