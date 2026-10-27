# std.process

ケイパビリティ `Process` を通して、ほかのプログラムを実行し、そのプログラムが出力したものを読むモジュールです。`import std.process;` と書き、`process.run(access: ..., program: ..., arguments: ..., options: ...)` のように呼びます。

プログラムを実行することは、プログラムが求められる権限のうち最大のものです。実行されるプログラムは、Cernith が検査したものではないからです。そのため、**`manifest` に書くグラント（grant。権限の内容）が境界**になります。グラントは、起動してよいプログラム（名前、絶対パス、それぞれが受け取ってよい引数）、作業してよいフォルダ、子プロセスが見てよい環境変数、時間と出力の上限を列挙します。呼び出しは、グラントが許すものを狭めることしかできません。`cernith audit` はグラントを出力します（[17 リソースとケイパビリティ](../language/17-resources-and-capabilities.md)）。

`run` は、プログラムが終わるまで待ちます。一度に起動するのは 1 つのプログラムで、呼び出しより長く生きるプログラムはありません。

## 型

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

`Process` はケイパビリティです。作れるのは `manifest` と `std.process` 自身だけで（ほかの場所ではエラー `E1301`）、フィールドは読めず（エラー `E0215`）、`==` で比較することもできません。

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

## グラント

| フィールド | 意味 |
| --- | --- |
| `programs` | 起動してよいプログラム。`ProgramGrant` は、呼び出しが使う `name`、プログラムの絶対パス `path`、受け取ってよい `arguments` を持つ |
| `work_root` | 子プロセスの作業フォルダがその下になければならないフォルダ（`Fs` と同じ規則で、外に出るパスは拒否される） |
| `environment` | 子プロセスが見てよい環境変数の**名前**。それ以外は捨てられる |
| `max_output_bytes` | 子プロセスが標準出力と標準エラー出力のそれぞれに出力してよい最大のバイト数。1 から 67108864（64 MiB） |
| `max_seconds` | 1 回の呼び出しにかかってよい最長の秒数。1 から 3600 |

`ArgumentRule` は、呼び出しが渡してよい引数を決めます。

| 規則 | 呼び出しが渡してよいもの |
| --- | --- |
| `Any` | どんな引数でも |
| `Prefix(words: [...])` | これらの語で始まる引数（そのあとは何でもよい） |
| `OneOf(choices: [...])` | リストのどれかと等しい引数 |

呼び出しは、プログラムを `name` で指定します。パスは決して渡しません。グラントは `manifest` に書きます。下の最初の例がそれを示しています。

## マシンを守る規則

`run` は、ホストに尋ねる**前**に、呼び出しをグラントと照らして検査します。ホストも、できることはもう一度検査します（たとえば、パスの形を、そのシステムの規則で）。

- **シェルも `PATH` の検索もありません。** 実行されるのは、グラントの `path` にあるプログラムです。引数は、1 つずつそのまま渡されるので、シェルに読み直されることはありません。
- **新しい環境。** 子プロセスが見るのは、`environment` に名前のある変数だけです。呼び出しは、許可された名前の値を `Options.environment` で与えられます。値のない名前には、ホスト自身の値があればそれが入ります。グラントに載っていない変数は、呼び出しから指定できません。
- **プログラムが読み込むものを変える変数は、グラントに名前があっても設定できません。** `LD_PRELOAD`、`LD_LIBRARY_PATH`、`DYLD_INSERT_LIBRARIES`、`DYLD_LIBRARY_PATH`、`NODE_OPTIONS`、`PYTHONPATH`、`PYTHONSTARTUP`、`BASH_ENV`、`ENV`、`PROMPT_COMMAND`、`GIT_SSH_COMMAND` です（文字の大小は無視します）。このような名前には、グラントを通して、ホスト自身の値が渡されます。
- **バッチファイルは実行しません。** `.bat`、`.cmd`、`.com`、`.lnk` で終わるパスは、決して起動されません。Windows が引数をコマンドインタープリタでもう一度読むからです。
- **絶対パス。** プログラムのパスは、ルート（POSIX では `/`）かドライブ文字（`C:\` か `C:/`）で始まります。Windows のホストはドライブ文字も要求するので、そこでは `/usr/bin/git` は `BadRequest("the path of the program is not absolute")` として拒否されます。
- **3 本のパイプだけで、ほかのハンドルはありません。** 子プロセスが受け取るのは、標準入力、標準出力、標準エラー出力だけで、プログラムのほかのものは渡りません。
- **上限。** 呼び出しの時間は、`Options.seconds` と `max_seconds` の小さいほうです。終了時にまだ動いている子プロセスは、**起動したものすべてとともに**殺され、呼び出しは、それまでに出力されたものとともに `Exit.TimedOut` を返します。1 つのストリームに `max_output_bytes` を超えて出力した子プロセスは殺され、呼び出しは `OutputLimit` を返します。どのグラントも、上限を 1 時間、各出力ストリーム 64 MiB、入力 64 MiB より上げることはできません。

これらの規則は、間違いが起こせる被害を狭めるもので、サンドボックスではありません。許可されたプログラムは、そのプログラムができることを何でもできます。アプリケーションが必要とするプログラムと引数だけを許可してください。

## 関数一覧

| 関数 | 戻り値の型 | 意味 |
| --- | --- | --- |
| `process.run(access, program, arguments, options)` | `Result<Output, ProcessError>` | 許可されたプログラムを実行し、待って、どう終わったかと出力を返す |
| `process.options(seconds)` | `Options` | フォルダ `"."`、変数なし、入力なし、時間の上限つきの呼び出し |
| `process.allows(rule, arguments)` | `Bool` | 規則が、これらの引数を渡すことを許すか |
| `process.describe(error)` | `String` | `ProcessError` を文字列にする |
| `process.describe_exit(exit)` | `String` | `Exit` を文字列にする |
| `process.simulated()` | `Process` | テスト用のケイパビリティ。プログラムは `tool` が 1 つ |
| `process.simulated_with(programs, work_root, environment, max_output_bytes, max_seconds)` | `Process` | テストが選んだグラントを持つ、テスト用のケイパビリティ |

`run` は `uses io` が必要です。ほかは純粋です。

## 関数の詳細

### process.run

```text
pub fn run(
  access: Process,
  program: String,
  arguments: List<String>,
  options: Options,
) -> Result<Output, ProcessError> uses io
```

グラントで `program` という名前のプログラムを、`arguments` を付けて実行し、終わるまで待ちます。子プロセスの標準入力は `options.input`、作業フォルダは `work_root` の下の `options.folder` です。プログラムが終わったときの結果は、**終了コードにかかわらず** `Output` を持つ `Ok` です。1 で終了したプログラムは `Exit.Exited(code: 1)` の `Ok` で、それが何を意味するかは、プログラムが決めます。`stdout` と `stderr` は `Bytes` です。テキストとして読むには `bytes.to_text` を使います。

呼び出しは次の順に検査され、最初の失敗が結果になります。

1. 名前が `programs` にない：`Err(error: ProcessError.Denied(program: ...))`。
2. 引数がそのプログラムの規則を満たさない：`ArgumentsDenied(program: ...)`。
3. グラントか呼び出しに、ほかの問題がある：`BadRequest(detail: ...)`。`detail` は次のとおりです。

   | `detail` | 原因 |
   | --- | --- |
   | `the path of the program is not absolute` | グラントの `path` |
   | `a batch file or shortcut is never started` | `path` が `.bat`、`.cmd`、`.com`、`.lnk` で終わる |
   | `the grant needs max_seconds from 1 to 3600` | グラント |
   | `the grant needs max_output_bytes from 1 to 67108864` | グラント |
   | `seconds is at least 1` | `options.seconds` が 1 未満 |
   | `the input is larger than 64 MiB` | `options.input` |
   | `an argument has a zero character` | 引数に `"\u{0}"` が含まれる |
   | `the grant lists the bad variable name NAME` | `environment` の名前に、英字、数字、`_` 以外の文字がある |
   | `bad variable NAME` | `options.environment` の名前が変数名でない、または値にゼロ文字がある |
   | `the variable NAME is not granted` | `options.environment` の名前がグラントにない |
   | `the variable NAME cannot be set` | プログラムが読み込むものを変える変数 |
   | `the folder is outside the work root` | `options.folder` が `work_root` の外に出る |

4. ホストがプログラムを起動します。その失敗は、`NotFound(program: ...)`（パスにファイルがない）、`Denied(program: ...)`（ホストが拒否した）、`SpawnFailed(detail: ...)`（プログラムを起動できなかった）、`BadRequest(detail: ...)`、`OutputLimit`（ストリームが `max_output_bytes` を超えた。子プロセスは殺された）、`Unsupported`（この対象はプログラムを実行できない）です。

プログラムがどう終わったかは `output.exit` にあります。

| `Exit` | 意味 |
| --- | --- |
| `Exited(code: ...)` | プログラムがその終了コードで終わった |
| `Signaled(name: ...)` | シグナルがプログラムを殺した（`name` は `SIGKILL` のようなシグナルの名前） |
| `TimedOut` | 時間の上限でもプログラムがまだ動いていたので、起動したものすべてとともに殺された。`stdout` と `stderr` には、それまでに出力されたものが入る |

### process.options

```text
pub fn options(seconds: Int) -> Options
```

`Options(folder: ".", environment: [], seconds: seconds, input: bytes.empty())` を返します。作業フォルダは `work_root` そのもの、変数は与えず、標準入力はなく、呼び出しは `seconds` 秒までかかってよい、という呼び出しです。フォルダ、変数、入力を決めるには、`Options` レコードを書き並べます。フォルダは `work_root` からの相対パスで、`std.fs` と同じように正規化されるので、外に出ることはできません。

### process.allows

```text
pub fn allows(rule: ArgumentRule, arguments: List<String>) -> Bool
```

規則が `arguments` を渡すことを許すかどうかを返します。`Any` は常に許し、`Prefix` は `arguments` がその語で始まるときに許し（語より短いリストは許しません）、`OneOf` は `arguments` がリストのどれかと等しいときに許します（選択肢が空のリストでは何も許しません）。`run` は、手順 2 の検査にこれを使います。

### process.describe と process.describe_exit

```text
pub fn describe(error: ProcessError) -> String
pub fn describe_exit(exit: Exit) -> String
```

| エラー | 文字列 |
| --- | --- |
| `Denied(program: "git")` | `"git: not granted"` |
| `ArgumentsDenied(program: "git")` | `"git: these arguments are not granted"` |
| `NotFound(program: "git")` | `"git: not found"` |
| `BadRequest(detail: "x")` | `"bad request: x"` |
| `SpawnFailed(detail: "x")` | `"could not start: x"` |
| `OutputLimit` | `"output limit reached"` |
| `Unsupported` | `"unsupported here"` |

| Exit | 文字列 |
| --- | --- |
| `Exited(code: 3)` | `"exited 3"` |
| `Signaled(name: "SIGKILL")` | `"killed by SIGKILL"` |
| `TimedOut` | `"timed out"` |

### process.simulated と process.simulated_with

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

テスト用の `Process` を返します。`simulated()` には、`tool` という名前のプログラムが、パス `/sim/tool`（実行はされません）に 1 つだけあり、引数は何でもよく、作業フォルダは `.` の下、環境変数はなく、上限は 65536 バイトと 60 秒です。`simulated_with` はグラントをテストから受け取るので、テストは、プログラムが拒否にどう反応するかを確かめられます。

どちらも、`example` と `property` の中、および `std.test.with_world` や `std.sim.with_sim` に渡す関数の中でだけ呼べます。ほかの場所で呼ぶとエラー `E1302` です。

シミュレート世界では、**プログラムは実行されません**。`process.run` には、`std.sim.with_sim` の `setup` に書いたスクリプト（`sim.program`、`sim.program_any`、`sim.program_signaled`、`sim.program_timed_out`）で答え、世界は起動されたコマンドをすべて記録します（`sim.processes_started`。[sim](sim.md)）。スクリプトのないプログラムは `NotFound` です。上で述べた `run` の検査はシミュレート世界でも働くので、テストは、何も実行せずに、`Denied`、`ArgumentsDenied`、`BadRequest`、`OutputLimit` を確かめられます。スクリプトの `path` は、グラントの `path` です。

## 動く場所

プログラムを実行できるのは、インタープリタ、Python、Node 上の JavaScript と Wasm（POSIX のみ）、Wasmtime 上の Wasm、Windows のネイティブコードです。ほかの対象では `Unsupported` になります。ページ、WebView シェル、Linux ウィンドウは `Process` を提供しません。`manifest` がそれを求めるプログラムは、それらの対象向けにはビルドできません（エラー `E1305`。`cernith build --target web`、`webview`、`linux-window` で確認しました）。

## 例

### 許可されたプログラムを実行する

次のモジュールは、プログラム `git` を 1 つだけ、しかも 2 通りの引数のリストだけを求めています。`run_git` はそれを実行し、結果を `String` にします。テストは、同じグラントの `process.simulated_with` を使い、プログラムにスクリプトを与えます。実際のプログラムは実行されません。

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

`manifest` は、プログラムのグラントとして `cernith audit` が出力します。

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

### 拒否と上限

次のモジュールは、引数の規則と、何かを起動する前に `run` が行う拒否、そして時間切れと出力の上限がどう見えるかを確かめます。

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

## 関連

- [sim](sim.md)：`program`、`program_any`、`program_signaled`、`program_timed_out`、`processes_started`
- [console](console.md)、[fs](fs.md)、[http](http.md)：ほかのケイパビリティ
- [bytes](bytes.md)：`bytes.to_text` など `Bytes` の関数
- [17 リソースとケイパビリティ](../language/17-resources-and-capabilities.md)
