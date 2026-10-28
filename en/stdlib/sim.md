# std.sim

This module runs functions in a **simulated world** containing files, environment variables, and network responses. Import it with `import std.sim;` and call functions such as `sim.with_sim(...)` and `sim.file(...)`. Operations stay inside the simulated world and never access external files or networks.

The functions in [fs](fs.md), [env](env.md), [http](http.md), [console](console.md), and [process](process.md) interact with the world through capabilities. In tests, capabilities such as `fs.simulated()` direct these operations to the simulated world. Use `sim.with_sim` when a test needs files, environment variables, or network responses in addition to the input lines, clock, and randomness provided by `std.test.with_world` ([test](test.md)).

The flow has three steps.

1. Write the initial state of the world (`setup`) with `sim.file`, `sim.env_var`, `sim.route`, and so on.
2. Run the function under test inside the world with `sim.with_sim`.
3. Examine the return value, the output, the files after the run, and so on in the returned `SimOutcome`.

## Types

```text
pub record SimOutcome<T> {
  value: T,
  output: List<String>,
  snapshot: List<String>,
}
```

- `value` is the return value of the function that was run.
- `output` holds the lines written by `console.write_line` and `io.write_line` during the run, in the order they were written.
- `snapshot` is a list of strings representing the state of the world when the run ended. Do not rely on its internal format; examine it with functions such as `file_content`, described below.

The requests that a run sent with `http.send` are returned by `sim.requests_made` as a list of the following record. `headers` holds the header lines as they were sent (`Name: value`).

```text
pub record SentRequest {
  method: String,
  url: String,
  headers: List<String>,
  body: Bytes,
}
```

## Running a world

### sim.with_sim

```text
pub fn with_sim<T>(
  world: test.World,
  setup: List<String>,
  run: fn() -> T uses clock, io, random,
) -> SimOutcome<T>
```

Runs `run` inside the world built from `setup` and returns the result as a `SimOutcome`. `with_sim` itself is pure, so it can be used inside an `example`.

- `world` is the `World` of `std.test` (`input`, `now_ms`, `seed`). The lines of standard input, the clock value, and the random seed have the same meaning as in `std.test.with_world` ([io-clock-random](io-clock-random.md)). When nothing else is needed, use `sim.quiet_world()`.
- `setup` is a list of strings returned by `sim.file`, `sim.env_var`, `sim.route`, and so on. Passing any other string has no effect on the world (and is not an error).
- `run` is a function with no arguments. Test capabilities such as `fs.simulated()`, `env.simulated()`, `http.simulated()`, and `console.simulated()` can be created only inside this function (or directly inside an `example` or a `property`). Calling them anywhere else, for example in a separate helper function, is error `E1302`.

A new world is created for each call. Changes made until the run ends (such as files written) remain only inside that world and can be examined through the `SimOutcome`.

### sim.quiet_world

```text
pub fn quiet_world() -> test.World
```

Returns a world with empty standard input, a clock at 0, and a random seed of 0. It is the same as `test.World(input: [], now_ms: 0, seed: 0)`.

## Setup items

Build the items listed in `setup` with the following functions. When several items have the same path, the same name, or the same URL, **the one written later takes effect**.

| Function | What it creates in the world |
| --- | --- |
| `sim.file(path, content)` | A text file |
| `sim.file_bytes(path, content)` | A file of bytes |
| `sim.env_var(name, value)` | An environment variable |
| `sim.route(url, status, body)` | A response (text) returned by `http.get` |
| `sim.route_bytes(url, status, body)` | A response (bytes) returned by `http.get_bytes` |
| `sim.route_for(method, url, status, headers, body)` | A response to `http.send` for one method |
| `sim.close_fails(path)` | A file whose closing fails |
| `sim.program(path, arguments, exit_code, stdout, stderr)` | A program that ends with an exit code when it is started with exactly these arguments |
| `sim.program_any(path, exit_code, stdout, stderr)` | The same for any arguments |
| `sim.program_signaled(path, arguments, signal)` | A program that is killed by a signal |
| `sim.program_timed_out(path, arguments, stdout)` | A program that does not end in time |
| `sim.terminal(width)` | Standard output and standard error are a terminal of that width |

### sim.file and sim.file_bytes

```text
pub fn file(path: String, content: String) -> String
pub fn file_bytes(path: String, content: Bytes) -> String
```

Places a file at `path`. Write `path` as a relative path separated by `/` (`reports/2024/a.txt`). Intermediate folders are created automatically. The functions of `fs` normalize the path (`path.normalize`) before querying the world, so they cannot reach a file placed under a form that is not yet normalized, such as `a/../b.txt`, or under an absolute path that starts with `/`.

The only difference between `file` and `file_bytes` is whether the content is passed as a string or as bytes. Reading a text file with `fs.read_file_bytes` gives its UTF-8 bytes, and reading a file of bytes that are not UTF-8 with `fs.read_text` gives `Failed(detail: "malformed-text")`.

### sim.env_var

```text
pub fn env_var(name: String, value: String) -> String
```

Sets the environment variable `name` to the value `value`. `value` may be an empty string (`env.get` then returns `Some(value: "")`).

### sim.route and sim.route_bytes

```text
pub fn route(url: String, status: Int, body: String) -> String
pub fn route_bytes(url: String, status: Int, body: Bytes) -> String
```

Makes `http.get` and `http.get_bytes` respond to `url` (an exactly matching string) with the status code `status` and the body `body`. A response created with `route` can also be received with `get_bytes`, and a response created with `route_bytes` can also be received with `get`. A request to a URL that has no route results in `HttpError.Network(detail: "network")` ([http](http.md)).

### sim.route_for

```text
pub fn route_for(
  method: String,
  url: String,
  status: Int,
  headers: List<String>,
  body: Bytes,
) -> String
```

Makes `http.send` respond to the request `method url` with the status code `status`, the header lines `headers` (each of the form `Name: value`), and the body `body`. `method` is the word of the method (`"POST"`; see `http.method_name`), and the URL must match exactly. A `GET` for which there is no `route_for` falls back to the plain routes of `sim.route` and `sim.route_bytes`; the other methods do not. A request that no route answers results in `HttpError.Network(detail: "network")`. A response of any status is a `Response` for `send` ([http](http.md)).

### sim.close_fails

```text
pub fn close_fails(path: String) -> String
```

Makes the closing of the file at `path` fail. If the body of the `using` succeeded, the whole `using` becomes `Err(error: FsError.CloseFailed(path: ...))` ([fs](fs.md)). A failure to close a file rarely happens in reality, so use this to check whether a program handles it correctly.

### sim.program, sim.program_any, sim.program_signaled, and sim.program_timed_out

```text
pub fn program(
  path: String,
  arguments: List<String>,
  exit_code: Int,
  stdout: String,
  stderr: String,
) -> String

pub fn program_any(
  path: String,
  exit_code: Int,
  stdout: String,
  stderr: String,
) -> String

pub fn program_signaled(
  path: String,
  arguments: List<String>,
  signal: String,
) -> String

pub fn program_timed_out(
  path: String,
  arguments: List<String>,
  stdout: String,
) -> String
```

The simulated world runs no program. It answers `process.run` ([process](process.md)) from these scripts.

- `path` is the path of the program **as the grant gives it** (the `path` of the `ProgramGrant`), not the name that a call uses. The grant of `process.simulated()` has the program `tool` at `/sim/tool`.
- `program` answers the call whose arguments are exactly `arguments`: the program ends with `Exit.Exited(code: exit_code)` and prints `stdout` and `stderr`. `program_any` answers a call with any arguments; a script for exact arguments wins over it.
- `program_signaled` ends the program with `Exit.Signaled(name: signal)`, with no output. `program_timed_out` makes the call return `Exit.TimedOut` with the output `stdout` that was printed so far, without waiting.
- A program with no script is `ProcessError.NotFound`. When the same program or arguments are written more than once, the later one takes effect.

### sim.terminal

```text
pub fn terminal(width: Int) -> String
```

Makes the standard streams a terminal of `width` columns: `console.is_terminal` says `true` for both streams and `console.width` gives `Some(value: width)` ([console](console.md)). Without this item the streams are pipes: `is_terminal` is `false` and `width` is `None`.

## Observing after the run

Examine the world after the run with the following functions. All of them are pure.

| Function | Return type | What it examines |
| --- | --- | --- |
| `sim.file_content(outcome, path)` | `Option<String>` | The content of a file (text) |
| `sim.file_content_bytes(outcome, path)` | `Option<Bytes>` | The content of a file (bytes) |
| `sim.file_paths(outcome)` | `List<String>` | The paths of all files |
| `sim.open_handles(outcome)` | `Int` | The number of files that were not closed |
| `sim.stdout_chunks(outcome)` | `List<Bytes>` | The bytes written by `console.write_bytes` |
| `sim.stderr_lines(outcome)` | `List<String>` | The lines written by `console.write_error_line` |
| `sim.requests_made(outcome)` | `List<SentRequest>` | The requests sent with `http.send` |
| `sim.processes_started(outcome)` | `List<List<String>>` | The commands that were started |

### sim.file_content and sim.file_content_bytes

```text
pub fn file_content<T>(outcome: SimOutcome<T>, path: String) -> Option<String>
pub fn file_content_bytes<T>(
  outcome: SimOutcome<T>,
  path: String,
) -> Option<Bytes>
```

Return the content of the file at `path` after the run. If there is no file at that path, the result is `None`. As with the paths passed to `fs`, write `path` as a path relative to the root, in normalized form (`a/b.txt`).

`file_content` returns `Some` only for a file whose content is text. For a file whose bytes are not UTF-8 text, the result is `None`. `file_content_bytes` returns `Some` for every file; for a text file, it is the UTF-8 bytes of the text. An empty file gives `Some("")` and empty bytes, respectively.

### sim.file_paths

```text
pub fn file_paths<T>(outcome: SimOutcome<T>) -> List<String>
```

Returns the paths of all files in the world after the run, in code point order. It includes both the files placed by `setup` and the files created during the run. It does not include folders. If there are no files, the result is `[]`. Use it to check that “this program created only this file.”

### sim.open_handles

```text
pub fn open_handles<T>(outcome: SimOutcome<T>) -> Int
```

Returns the number of open files that were not closed at the moment the run ended. A file opened with `using` is always closed, so the number is normally 0.

### sim.stdout_chunks and sim.stderr_lines

```text
pub fn stdout_chunks<T>(outcome: SimOutcome<T>) -> List<Bytes>
pub fn stderr_lines<T>(outcome: SimOutcome<T>) -> List<String>
```

`stdout_chunks` returns the bytes written, one entry per call of `console.write_bytes`, in the order of the calls. Lines written with `console.write_line` are not included; they go into `outcome.output`. `stderr_lines` returns the lines written by `console.write_error_line`, in the order of the calls ([console](console.md)).

### sim.requests_made and sim.processes_started

```text
pub fn requests_made<T>(outcome: SimOutcome<T>) -> List<SentRequest>
pub fn processes_started<T>(outcome: SimOutcome<T>) -> List<List<String>>
```

`requests_made` returns the requests that the run sent with `http.send`, in order. A request that `send` refused before sending (`BadRequest`, `Denied`) is not in the list, and neither is a call of `get` or `get_bytes`.

`processes_started` returns the commands that the run started, in order. Each command is a list: the path of the program, then its arguments. A command that has no script (and so is `NotFound`) is recorded too, since it reached the world. A call that `process.run` refused by the grant (`Denied`, `ArgumentsDenied`, `BadRequest`) never reaches the world and is not recorded.

> Note: `std.sys` is the module through which the functions of `fs`, `env`, `http`, and `console` send requests to the world (the real environment or the simulated world). Programs do not use it directly.

## Example

The following module is a program that fetches a report from the URL in the environment variable `REPORT_URL` and saves it in the `out` folder, then fetches an image, saves it, and writes it to standard output. It uses every function described on this page to test behavior that involves environment variables, the network, files, and standard output and standard error.

`simulate` is a helper function that wraps `with_sim` so that it receives `Caps` (a record of the same shape as the `manifest`).

```cernith
module guide.sim_demo;

import std.bytes;
import std.clock;
import std.console;
import std.env;
import std.fs;
import std.http;
import std.int;
import std.option;
import std.result;
import std.sim;
import std.test;

pub manifest Caps {
  console: console.Console,
  env: env.Env,
  files: fs.Fs(root: "out"),
  net: http.Http(hosts: ["reports.example.com"]),
}

fn console_error(error: console.ConsoleError) -> String {
  console.describe(error: error)
}

fn fetch_outcome(setup: List<String>) -> Result<Unit, String> {
  simulate(
    world: sim.quiet_world(),
    setup: setup,
    body: fn(caps: Caps) -> Result<Unit, String> uses clock, io {
      fetch_report(caps: caps)
    },
  ).value
}

pub fn fetch_picture(caps: Caps) -> Result<Unit, String> uses io {
  let picture = result.map_error(
    outcome: http.get_bytes(
      access: caps.net,
      url: "https://reports.example.com/picture",
    ),
    apply: net_error,
  )?;
  result.map_error(
    outcome: fs.write_file_bytes(
      access: caps.files,
      path: "picture.bin",
      bytes: picture,
    ),
    apply: file_error,
  )?;
  result.map_error(
    outcome: console.write_bytes(access: caps.console, bytes: picture),
    apply: console_error,
  )
}

example fetch_picture_saves_and_prints_the_bytes for fetch_picture {
  match simulate(
    world: sim.quiet_world(),
    setup: [
      sim.route_bytes(
        url: "https://reports.example.com/picture",
        status: 200,
        body: picture(),
      ),
      sim.file_bytes(path: "picture.bin", content: bytes.empty()),
    ],
    body: fn(caps: Caps) -> Result<Unit, String> uses clock, io {
      fetch_picture(caps: caps)
    },
  ) {
    outcome => outcome.value == Ok(value: ())
      and sim.file_content_bytes(outcome: outcome, path: "picture.bin")
        == Some(value: picture())
      and sim.stdout_chunks(outcome: outcome) == [picture()]
      and outcome.output == [],
  }
}

pub fn fetch_report(caps: Caps) -> Result<Unit, String> uses clock, io {
  let url = option.ok_or(
    option: env.get(access: caps.env, name: "REPORT_URL"),
    error: "REPORT_URL is not set",
  )?;
  let body = result.map_error(
    outcome: http.get(access: caps.net, url: url),
    apply: net_error,
  )?;
  result.map_error(
    outcome: fs.write_text(
      access: caps.files,
      path: "report.txt",
      text: "fetched " ++ int.to_text(value: clock.now_ms()) ++ "\n" ++ body,
    ),
    apply: file_error,
  )?;
  result.map_error(
    outcome: console.write_line(access: caps.console, text: "saved report.txt"),
    apply: console_error,
  )
}

example fetch_report_saves_the_body for fetch_report {
  match simulate(
    world: test.World(input: [], now_ms: 1234, seed: 0),
    setup: report_setup(status: 200)
      ++ [sim.file(path: "report.txt", content: "old")],
    body: fn(caps: Caps) -> Result<Unit, String> uses clock, io {
      fetch_report(caps: caps)
    },
  ) {
    outcome => outcome.value == Ok(value: ())
      and sim.file_content(outcome: outcome, path: "report.txt")
        == Some(value: "fetched 1234\nbody text")
      and sim.file_paths(outcome: outcome) == ["report.txt"]
      and outcome.output == ["saved report.txt"]
      and sim.stderr_lines(outcome: outcome) == []
      and sim.open_handles(outcome: outcome) == 0,
  }
}

example fetch_report_starts_from_the_quiet_world for fetch_report {
  match simulate(
    world: sim.quiet_world(),
    setup: report_setup(status: 200),
    body: fn(caps: Caps) -> Result<Unit, String> uses clock, io {
      fetch_report(caps: caps)
    },
  ) {
    outcome => sim.file_content(outcome: outcome, path: "report.txt")
      == Some(value: "fetched 0\nbody text"),
  }
}

example fetch_report_stops_at_the_first_failure for fetch_report {
  fetch_outcome(setup: []) == Err(error: "REPORT_URL is not set")
    and fetch_outcome(
      setup: [
        sim.env_var(
          name: "REPORT_URL",
          value: "https://reports.example.com/r1",
        ),
      ],
    )
      == Err(error: "network: network")
    and fetch_outcome(setup: report_setup(status: 404))
      == Err(error: "status 404")
    and fetch_outcome(
      setup: report_setup(status: 200) ++ [sim.close_fails(path: "report.txt")],
    )
      == Err(error: "report.txt: closing failed")
}

fn file_error(error: fs.FsError) -> String {
  fs.describe(error: error)
}

pub fn main(args: List<String>, caps: Caps) -> Int uses clock, io {
  match fetch_report(caps: caps) {
    Ok(value: _) => 0,
    Err(error: message) => {
      discard console.write_error_line(access: caps.console, text: message);
      1
    },
  }
}

example main_returns_one_and_complains_on_the_error_stream for main {
  match simulate(
    world: sim.quiet_world(),
    setup: [],
    body: fn(caps: Caps) -> Int uses clock, io {
      main(args: [], caps: caps)
    },
  ) {
    outcome => outcome.value == 1
      and sim.stderr_lines(outcome: outcome) == ["REPORT_URL is not set"]
      and sim.file_paths(outcome: outcome) == [],
  }
}

example main_returns_zero_when_the_report_is_saved for main {
  match simulate(
    world: sim.quiet_world(),
    setup: report_setup(status: 200),
    body: fn(caps: Caps) -> Int uses clock, io {
      main(args: [], caps: caps)
    },
  ) {
    outcome => outcome.value == 0
      and sim.stderr_lines(outcome: outcome) == []
      and sim.file_paths(outcome: outcome) == ["report.txt"],
  }
}

fn net_error(error: http.HttpError) -> String {
  http.describe(error: error)
}

fn picture() -> Bytes {
  bytes.new(count: 3, fill: 200)
}

fn report_setup(status: Int) -> List<String> {
  [
    sim.env_var(name: "REPORT_URL", value: "https://reports.example.com/r1"),
    sim.route(
      url: "https://reports.example.com/r1",
      status: status,
      body: "body text",
    ),
  ]
}

fn simulate<T>(
  world: test.World,
  setup: List<String>,
  body: fn(caps: Caps) -> T uses clock, io,
) -> sim.SimOutcome<T> {
  sim.with_sim(
    world: world,
    setup: setup,
    run: fn() -> T uses clock, io, random {
      body(
        caps: Caps(
          console: console.simulated(),
          env: env.simulated(),
          files: fs.simulated(),
          net: http.simulated(),
        ),
      )
    },
  )
}

pub fn stored_picture_size(caps: Caps) -> Result<Int, String> uses io {
  let stored = result.map_error(
    outcome: fs.read_file_bytes(
      access: fs.read_only(access: caps.files),
      path: "picture.bin",
    ),
    apply: file_error,
  )?;
  Ok(value: bytes.length(bytes: stored))
}

example stored_picture_size_reads_a_file_set_up_as_bytes for stored_picture_size {
  simulate(
    world: sim.quiet_world(),
    setup: [sim.file_bytes(path: "picture.bin", content: picture())],
    body: fn(caps: Caps) -> Result<Int, String> uses clock, io {
      stored_picture_size(caps: caps)
    },
  ).value
    == Ok(value: 3)
    and simulate(
      world: sim.quiet_world(),
      setup: [],
      body: fn(caps: Caps) -> Result<Int, String> uses clock, io {
        stored_picture_size(caps: caps)
      },
    ).value
      == Err(error: "picture.bin: not found")
}
```

### Scripts, terminals, and recorded requests

The following module scripts a command's response, simulates a terminal, and inspects requests sent with `send`.

```cernith
module guide.sim_scripts;

import std.bytes;
import std.console;
import std.http;
import std.process;
import std.sim;

pub fn announce(
  access: console.Console,
) -> Result<Unit, console.ConsoleError> uses io {
  if console.is_terminal(access: access, stream: console.Stream.Output) {
    console.write_line(access: access, text: "terminal")
  } else {
    console.write_line(access: access, text: "pipe")
  }
}

example the_world_is_a_pipe_unless_the_setup_says_otherwise for announce {
  sim.with_sim(
    world: sim.quiet_world(),
    setup: [sim.terminal(width: 100)],
    run: fn() -> Result<Unit, console.ConsoleError> uses clock, io, random {
      announce(access: console.simulated())
    },
  ).output
    == ["terminal"]
    and sim.with_sim(
      world: sim.quiet_world(),
      setup: [],
      run: fn() -> Result<Unit, console.ConsoleError> uses clock, io, random {
        announce(access: console.simulated())
      },
    ).output
      == ["pipe"]
}

pub fn ping(access: http.Http) -> Result<Int, http.HttpError> uses io {
  let request = http.new_request(
    method: http.Method.Put,
    url: "https://api.example.com/ping",
  );
  let response = http.send(access: access, request: request)?;
  Ok(value: response.status)
}

example send_is_answered_by_the_route_of_its_method for ping {
  match sim.with_sim(
    world: sim.quiet_world(),
    setup: [
      sim.route_for(
        method: "PUT",
        url: "https://api.example.com/ping",
        status: 204,
        headers: [],
        body: bytes.empty(),
      ),
    ],
    run: fn() -> Result<Int, http.HttpError> uses clock, io, random {
      ping(access: http.simulated())
    },
  ) {
    outcome => outcome.value == Ok(value: 204)
      and sim.requests_made(outcome: outcome)
        == [
          sim.SentRequest(
            method: "PUT",
            url: "https://api.example.com/ping",
            headers: [],
            body: bytes.empty(),
          ),
        ],
  }
}

pub fn version_of_tool(
  tools: process.Process,
) -> Result<Int, process.ProcessError> uses io {
  let output = process.run(
    access: tools,
    program: "tool",
    arguments: ["--version"],
    options: process.options(seconds: 5),
  )?;
  match output.exit {
    process.Exit.Exited(code: code) => Ok(value: code),
    _ => Ok(value: -1),
  }
}

example a_program_that_ends_otherwise for version_of_tool {
  sim.with_sim(
    world: sim.quiet_world(),
    setup: [
      sim.program_signaled(
        path: "/sim/tool",
        arguments: ["--version"],
        signal: "SIGSEGV",
      ),
    ],
    run: fn() -> Result<Int, process.ProcessError> uses clock, io, random {
      version_of_tool(tools: process.simulated())
    },
  ).value
    == Ok(value: -1)
    and sim.with_sim(
      world: sim.quiet_world(),
      setup: [
        sim.program_timed_out(
          path: "/sim/tool",
          arguments: ["--version"],
          stdout: "",
        ),
      ],
      run: fn() -> Result<Int, process.ProcessError> uses clock, io, random {
        version_of_tool(tools: process.simulated())
      },
    ).value
      == Ok(value: -1)
}

example a_script_answers_the_exact_command for version_of_tool {
  match sim.with_sim(
    world: sim.quiet_world(),
    setup: [
      sim.program_any(path: "/sim/tool", exit_code: 2, stdout: "", stderr: ""),
      sim.program(
        path: "/sim/tool",
        arguments: ["--version"],
        exit_code: 0,
        stdout: "tool 1.0\n",
        stderr: "",
      ),
    ],
    run: fn() -> Result<Int, process.ProcessError> uses clock, io, random {
      version_of_tool(tools: process.simulated())
    },
  ) {
    outcome => outcome.value == Ok(value: 0)
      and sim.processes_started(outcome: outcome)
        == [["/sim/tool", "--version"]],
  }
}
```

## Related

- [test](test.md): `with_world` (a world with only standard input, the clock, and randomness)
- [fs](fs.md), [env](env.md), [http](http.md), [console](console.md), [process](process.md): the functions that query the simulated world
- [io-clock-random](io-clock-random.md): `input`, `now_ms`, and `seed` of `World`
- [16 Testing](../language/16-testing.md)
- [17 Resources and capabilities](../language/17-resources-and-capabilities.md)
