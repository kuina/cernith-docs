# std.console

コマンドラインプログラムの標準入力・標準出力・標準エラー出力を、ケイパビリティ `Console` を通して扱うモジュールです。`import std.console;` と書き、`console.write_line(access: ..., text: ...)` のように呼びます。どの関数も、最初の引数に `access: console.Console` を取ります。

`std.io`（[io-clock-random](io-clock-random.md)）は、`uses io` を宣言すれば誰でも呼べます。これに対して `std.console` の関数は、`Console` の値を受け取った関数からしか呼べません。`Console` は、プログラムの `manifest` 宣言に書くことで、`main` の第 2 引数として渡されます。ほかの場所では作れません（エラー `E1301`）。こうすると、ある関数が標準入出力を使うかどうかが、その関数の引数を見るだけで分かります（[17 リソースとケイパビリティ](../language/17-resources-and-capabilities.md)）。

## 型

```text
pub capability Console;

pub variant ConsoleError {
  Failed(reason: String),
}
```

- `Console` は、フィールドを持たないケイパビリティです。許可の中身（grant）はなく、持っていることだけが意味を持ちます。`==` では比較できません。
- `ConsoleError` は、読み書きに失敗したときのエラーです。`reason` に理由の文字列が入ります。

## プログラムの形

`Console` を使うプログラムは、`manifest` で `Console` を求め、`main` の第 2 引数で受け取ります。

```text
pub manifest Caps {
  console: console.Console,
}

pub fn main(args: List<String>, caps: Caps) -> Int uses io { ... }
```

`Console` を使う関数は、`access: console.Console` の引数を取り、`uses io` を宣言します。引数を持っているだけでは効果の宣言にならないので、`uses io` を書き忘れるとエラー `E0401` です。

```cernith-error E0401
module guide.console_forgot_io;

import std.console;

pub fn greet(
  access: console.Console,
) -> Result<Unit, console.ConsoleError> {
  console.write_line(access: access, text: "hello")
}
```

`Console` を自分で作ろうとすると、エラー `E1301` です。

```cernith-error E1301
module guide.console_made_by_hand;

import std.console;

pub fn greet() -> Result<Unit, console.ConsoleError> uses io {
  console.write_line(access: console.Console(), text: "hello")
}
```

## 関数一覧

| 関数 | 戻り値の型 | 意味 |
| --- | --- | --- |
| `console.read_all_lines(access)` | `Result<List<String>, ConsoleError>` | 標準入力の全行 |
| `console.read_all_bytes(access)` | `Result<Bytes, ConsoleError>` | 標準入力の全バイト |
| `console.fold_lines(access, initial, step)` | `Result<A, ConsoleError>` | 標準入力の各行を順に畳み込む |
| `console.write_line(access, text)` | `Result<Unit, ConsoleError>` | 標準出力に 1 行書く |
| `console.write_bytes(access, bytes)` | `Result<Unit, ConsoleError>` | 標準出力にバイト列をそのまま書く |
| `console.write_error_line(access, text)` | `Result<Unit, ConsoleError>` | 標準エラー出力に 1 行書く |
| `console.describe(error)` | `String` | `ConsoleError` を文字列にする |
| `console.simulated()` | `Console` | テスト用のケイパビリティ |

`describe` と `simulated` 以外は、`uses io` が必要です。`fold_lines` は効果多相で、`step` の効果も呼び出し側が宣言します（[11 効果](../language/11-effects.md)）。

## 関数の詳細

### console.read_all_lines

```text
pub fn read_all_lines(
  access: Console,
) -> Result<List<String>, ConsoleError> uses io
```

標準入力を行ごとのリストにして返します。行末の改行（`\n`、`\r\n`）は含まれません。最後の行に改行がなくても、1 行として数えます。入力が空なら `Ok(value: [])` です。読めなかったときは `Err(error: ConsoleError.Failed(reason: ...))` です。標準入力が UTF-8 のテキストでないときは、`Err(error: ConsoleError.Failed(reason: "malformed-text"))` です。生のバイト列は `console.read_all_bytes` で読めます。

同じプログラムの中で 2 回目以降に呼んでも、同じ行が返ります。

### console.read_all_bytes

```text
pub fn read_all_bytes(
  access: Console,
) -> Result<Bytes, ConsoleError> uses io
```

標準入力の全体をバイト列で返します。画像などテキストでないデータを読むときに使います。

- 行を読む関数（`read_all_lines`、`fold_lines`）を**呼ぶ前**に呼ぶと、入力のバイト列そのものを返します。この場合、標準入力は読み切られているので、そのあとの `read_all_lines` は空のリストを返します。
- 行を読んだ**あと**に呼ぶと、読んだ各行の末尾に `\n` を付けてつないだものを返します（元の改行が `\r\n` でも `\n` になります）。
- テストの世界（`World` の `input`）には元のバイト列がないので、常にこの形です。

### console.fold_lines

```text
pub fn fold_lines<A, effect E>(
  access: Console,
  initial: A,
  step: fn(total: A, line: String) -> A uses E,
) -> Result<A, ConsoleError> uses io, E
```

標準入力を行ごとに読み、先頭の行から順に `step` を適用して、1 つの値にまとめます。`step` の引数のラベルは `total`（これまでの値）と `line`（その行）です。最初の `total` は `initial` で、行がなければ `Ok(value: initial)` です。`read_all_lines` で行を読んでから自分で `for` を回すのと同じ結果になります。

### console.write_line

```text
pub fn write_line(
  access: Console,
  text: String,
) -> Result<Unit, ConsoleError> uses io
```

`text` を標準出力に 1 行として書きます。プログラムの結果は、これで書きます。テストでは、書いた行が `Outcome` の `output` に、書いた順に入ります。

### console.write_bytes

```text
pub fn write_bytes(
  access: Console,
  bytes: Bytes,
) -> Result<Unit, ConsoleError> uses io
```

`bytes` を標準出力に、そのまま書きます。改行は足されません。`write_line` と交互に呼んだときは、書いた順に出力されます。テストでは、`output` には入らず、`sim.stdout_chunks` で呼び出しごとのバイト列として読めます（[sim](sim.md)）。

### console.write_error_line

```text
pub fn write_error_line(
  access: Console,
  text: String,
) -> Result<Unit, ConsoleError> uses io
```

`text` を標準エラー出力に 1 行として書きます。プログラムが出す警告やエラーの報告は、結果と分けて、こちらに書きます。テストでは、`output` には入らず、`sim.stderr_lines` で読めます（[sim](sim.md)）。

### console.describe

```text
pub fn describe(error: ConsoleError) -> String
```

エラーを `"console: "` に理由をつないだ文字列にします。`ConsoleError.Failed(reason: "closed")` なら `"console: closed"` です。純粋な関数です。

### console.simulated

```text
pub fn simulated() -> Console
```

テストのための `Console` を返します。`example` と `property` の中、および `std.test.with_world` や `std.sim.with_sim` に渡す関数の中でだけ呼べます。ほかの場所で呼ぶとエラー `E1302` です。プログラムは、本物の `Console` を `manifest` から受け取ります。

```cernith-error E1302
module guide.console_simulated_outside_a_test;

import std.console;

pub fn greet() -> Result<Unit, console.ConsoleError> uses io {
  console.write_line(access: console.simulated(), text: "hello")
}
```

## 例

次のモジュールは、標準入力の各行を整数として足し、結果を標準出力に書くプログラム（`report_total` と `main`）と、`Console` の各関数を使う小さな関数です。数として読めない行は、0 として扱います。

テストでは、`Console` は `console.simulated()` で作り、`main` には `manifest` と同じ形のレコード `Caps(console: console.simulated())` を渡します。入力の行は `test.World` の `input` で与えます。

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

`main` の終了ステータスは、`report_total` が成功すれば 0、`ConsoleError` で失敗すれば標準エラー出力にその説明を書いて 1 です。

## 関連

- [io-clock-random](io-clock-random.md)：ケイパビリティを使わない `std.io`
- [sim](sim.md)：標準エラー出力や `write_bytes` の出力を観察する
- [test](test.md)：`with_world`
- [fs](fs.md)、[env](env.md)、[http](http.md)：ほかのケイパビリティ
- [17 リソースとケイパビリティ](../language/17-resources-and-capabilities.md)
