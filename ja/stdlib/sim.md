# std.sim

ファイル・環境変数・ネットワークの応答を持つ**シミュレート世界**を作り、その中で関数を実行するモジュールです。`import std.sim;` と書き、`sim.with_sim(...)` や `sim.file(...)` のように呼びます。世界の外のファイルやネットワークには、一切触れません。

[fs](fs.md)、[env](env.md)、[http](http.md)、[console](console.md) の関数は、ケイパビリティを通して世界に問い合わせます。テストでは、`fs.simulated()` などのテスト用ケイパビリティを使い、問い合わせの相手をシミュレート世界にします。`std.test.with_world`（[test](test.md)）の世界が持つのは、標準入力の行・時計・乱数だけです。ファイル・環境変数・ネットワークも持つ世界が必要なテストでは、`sim.with_sim` を使います。

流れは 3 段階です。

1. `sim.file`、`sim.env_var`、`sim.route` などで、世界の初期状態（`setup`）を書く。
2. `sim.with_sim` で、テスト対象の関数を世界の中で実行する。
3. 返ってきた `SimOutcome` から、戻り値・出力・実行後のファイルなどを調べる。

## 型

```text
pub record SimOutcome<T> {
  value: T,
  output: List<String>,
  snapshot: List<String>,
}
```

- `value` は、実行した関数の戻り値です。
- `output` は、実行中に `console.write_line` や `io.write_line` が書いた行です。書いた順に並びます。
- `snapshot` は、実行が終わったときの世界の状態を表す文字列のリストです。中身の形式に頼らず、後述の `file_content` などの関数で調べます。

## 世界を実行する

### sim.with_sim

```text
pub fn with_sim<T>(
  world: test.World,
  setup: List<String>,
  run: fn() -> T uses clock, io, random,
) -> SimOutcome<T>
```

`setup` で作った世界の中で、`run` を実行し、結果を `SimOutcome` にして返します。`with_sim` 自体は純粋なので、`example` の中で使えます。

- `world` は、`std.test` の `World`（`input`、`now_ms`、`seed`）です。標準入力の行、時計の値、乱数の種が、`std.test.with_world` と同じ意味で使われます（[io-clock-random](io-clock-random.md)）。ほかに指定がなければ、`sim.quiet_world()` を使います。
- `setup` は、`sim.file`、`sim.env_var`、`sim.route` などが返す文字列のリストです。ほかの文字列を渡しても、世界には何の影響もありません（エラーにもなりません）。
- `run` は、引数のない関数です。`fs.simulated()`、`env.simulated()`、`http.simulated()`、`console.simulated()` などのテスト用ケイパビリティを作れるのは、この関数の中（または `example` や `property` の中）だけです。補助関数など、ほかの場所で呼ぶとエラー `E1302` です。

世界は呼び出しごとに新しく作られます。実行が終わるまでの変更（書いたファイルなど）は、その世界の中にだけ残り、`SimOutcome` から調べられます。

### sim.quiet_world

```text
pub fn quiet_world() -> test.World
```

標準入力が空で、時計が 0、乱数の種が 0 の世界を返します。`test.World(input: [], now_ms: 0, seed: 0)` と同じです。

## setup の項目

`setup` に並べる項目は、次の関数で作ります。同じパス、同じ名前、同じ URL の項目を複数書いたときは、**あとに書いたものが有効**です。

| 関数 | 世界に作るもの |
| --- | --- |
| `sim.file(path, content)` | テキストのファイル |
| `sim.file_bytes(path, content)` | バイト列のファイル |
| `sim.env_var(name, value)` | 環境変数 |
| `sim.route(url, status, body)` | `http.get` が返す応答（テキスト） |
| `sim.route_bytes(url, status, body)` | `http.get_bytes` が返す応答（バイト列） |
| `sim.close_fails(path)` | 閉じるのに失敗するファイル |

### sim.file と sim.file_bytes

```text
pub fn file(path: String, content: String) -> String
pub fn file_bytes(path: String, content: Bytes) -> String
```

`path` にファイルを置きます。`path` は、`/` で区切った相対パスで書きます（`reports/2024/a.txt`）。途中のフォルダは自動的にできます。`fs` の関数は、パスを整えて（`path.normalize`）から世界に問い合わせるので、`a/../b.txt` のように整える前の形や、`/` で始まる絶対パスで置いたファイルには、届きません。

`file` と `file_bytes` の違いは、内容を文字列とバイト列のどちらで渡すかだけです。テキストのファイルを `fs.read_file_bytes` で読むと、その UTF-8 のバイト列が得られ、UTF-8 でないバイト列のファイルを `fs.read_text` で読むと `Failed(detail: "malformed-text")` になります。

### sim.env_var

```text
pub fn env_var(name: String, value: String) -> String
```

環境変数 `name` を、値 `value` で設定します。`value` は空文字列でもかまいません（`env.get` は `Some(value: "")` を返します）。

### sim.route と sim.route_bytes

```text
pub fn route(url: String, status: Int, body: String) -> String
pub fn route_bytes(url: String, status: Int, body: Bytes) -> String
```

`http.get` と `http.get_bytes` が、`url`（完全に一致する文字列）に対して、ステータスコード `status` と本文 `body` で応答するようにします。`route` で作った応答を `get_bytes` で、`route_bytes` で作った応答を `get` で受け取ることもできます。ルートがない URL へのリクエストは、`HttpError.Network(detail: "network")` になります（[http](http.md)）。

### sim.close_fails

```text
pub fn close_fails(path: String) -> String
```

`path` のファイルを閉じる処理が失敗するようにします。`using` の本体が成功していれば、`using` 全体が `Err(error: FsError.CloseFailed(path: ...))` になります（[fs](fs.md)）。ファイルを閉じる失敗は、現実にはめったに起きないので、プログラムがそれを正しく扱うかどうかを確かめるために使います。

## 実行後の観察

実行後の世界は、次の関数で調べます。どれも純粋です。

| 関数 | 戻り値の型 | 調べるもの |
| --- | --- | --- |
| `sim.file_content(outcome, path)` | `Option<String>` | ファイルの内容（テキスト） |
| `sim.file_content_bytes(outcome, path)` | `Option<Bytes>` | ファイルの内容（バイト列） |
| `sim.file_paths(outcome)` | `List<String>` | すべてのファイルのパス |
| `sim.open_handles(outcome)` | `Int` | 閉じられなかったファイルの数 |
| `sim.stdout_chunks(outcome)` | `List<Bytes>` | `console.write_bytes` で書いたバイト列 |
| `sim.stderr_lines(outcome)` | `List<String>` | `console.write_error_line` で書いた行 |

### sim.file_content と sim.file_content_bytes

```text
pub fn file_content<T>(outcome: SimOutcome<T>, path: String) -> Option<String>
pub fn file_content_bytes<T>(
  outcome: SimOutcome<T>,
  path: String,
) -> Option<Bytes>
```

実行後のファイル `path` の内容を返します。そのパスにファイルがなければ `None` です。`path` は、`fs` に渡したのと同じく、ルートからの相対パスで、整えた形（`a/b.txt`）で書きます。

`file_content` は、内容がテキストのファイルに対してだけ `Some` を返します。UTF-8 のテキストでないバイト列のファイルに対しては `None` です。`file_content_bytes` は、どのファイルにも `Some` を返し、テキストのファイルでは、その UTF-8 のバイト列です。空のファイルは、`Some("")` と空のバイト列です。

### sim.file_paths

```text
pub fn file_paths<T>(outcome: SimOutcome<T>) -> List<String>
```

実行後の世界にあるすべてのファイルのパスを、コードポイント順に返します。`setup` で置いたファイルと、実行中に作られたファイルの両方を含みます。フォルダは含みません。ファイルがなければ `[]` です。「このプログラムは、このファイルしか作らなかった」ことを確かめるのに使います。

### sim.open_handles

```text
pub fn open_handles<T>(outcome: SimOutcome<T>) -> Int
```

実行が終わった時点で、閉じられていない開いたファイルの数を返します。`using` で開いたファイルは必ず閉じられるので、通常は 0 です。

### sim.stdout_chunks と sim.stderr_lines

```text
pub fn stdout_chunks<T>(outcome: SimOutcome<T>) -> List<Bytes>
pub fn stderr_lines<T>(outcome: SimOutcome<T>) -> List<String>
```

`stdout_chunks` は、`console.write_bytes` の呼び出しごとに 1 つ、書いたバイト列を呼んだ順に返します。`console.write_line` で書いた行は含まれず、`outcome.output` に入ります。`stderr_lines` は、`console.write_error_line` で書いた行を、呼んだ順に返します（[console](console.md)）。

> 補足：`std.sys` は、`fs`、`env`、`http`、`console` の関数が、世界（実際の環境かシミュレート世界）にリクエストを送るためのモジュールです。プログラムが直接使うものではありません。

## 例

次のモジュールは、環境変数 `REPORT_URL` の URL からレポートを取得して `out` フォルダに保存し、画像を取得して保存と標準出力への書き出しを行うプログラムです。このページで説明している `sim` のすべての関数を使って、環境変数、ネットワーク、ファイル、標準出力・標準エラー出力の関わる動作をテストしています。

`simulate` は、`with_sim` に `Caps`（`manifest` と同じ形のレコード）を渡す形に包んだ補助関数です。

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

## 関連

- [test](test.md)：`with_world`（標準入力・時計・乱数だけの世界）
- [fs](fs.md)、[env](env.md)、[http](http.md)、[console](console.md)：シミュレート世界に問い合わせる関数
- [io-clock-random](io-clock-random.md)：`World` の `input`、`now_ms`、`seed`
- [16 テスト](../language/16-testing.md)
- [17 リソースとケイパビリティ](../language/17-resources-and-capabilities.md)
