# std.io、std.clock、std.random

標準入出力・時刻・乱数を扱う 3 つのモジュールです。`import std.io;`、`import std.clock;`、`import std.random;` で読み込み、`io.write_line(...)`、`clock.now_ms()`、`random.int_between(...)` のように呼び出します。

この 3 つが扱うのは、**ambient な効果**（`io`、`clock`、`random`）です。呼び出すには、その関数が効果を `uses` で宣言していれば足り、ケイパビリティを受け取る必要はありません（[11 効果](../language/11-effects.md)）。ファイル・環境変数・ネットワークのように、プログラムが宣言して受け取った権限を通して行う操作は、別のモジュールが担当します（[fs](fs.md)、[env](env.md)、[http](http.md)、[console](console.md)。仕組みは [17 リソースとケイパビリティ](../language/17-resources-and-capabilities.md)）。

効果は、宣言した名前ごとに独立です。`clock` を使う関数は `uses clock` と書く必要があり、`uses io` では足りません（足りないとエラー `E0401`）。複数使うときは `uses clock, io, random` のように並べます。

## 関数一覧

| 関数 | 戻り値の型 | 効果 | 意味 |
| --- | --- | --- | --- |
| `io.read_all_lines()` | `Result<List<String>, IoError>` | `io` | 標準入力の全行を読む |
| `io.write_line(text)` | `Result<Unit, IoError>` | `io` | 標準出力に 1 行書く |
| `clock.now_ms()` | `Int` | `clock` | 現在の時刻（ミリ秒） |
| `clock.monotonic_ms()` | `Int` | `clock`、`io` | 決して戻らない時計（ミリ秒） |
| `clock.sleep_ms(milliseconds)` | `Unit` | `clock`、`io` | 実際にそのミリ秒だけ待つ |
| `random.int_between(low, high)` | `Int` | `random` | `low` 以上 `high` 以下の整数を 1 つ選ぶ |
| `random.bytes(count)` | `Result<Bytes, RandomError>` | `io`、`random` | システムの安全な乱数源から `count` バイトを得る |
| `random.describe(error)` | `String` | なし | `RandomError` を文字列にする |

`std.io` にも `std.random` にも、エラーの型があります。

```text
pub variant IoError {
  Failed(reason: String),
}

pub variant RandomError {
  Unsupported,
  Failed(detail: String),
}
```

## 関数の詳細

### io.read_all_lines

```text
pub fn read_all_lines() -> Result<List<String>, IoError> uses io
```

標準入力を最後まで読み、行ごとのリストにして返します。引数はありません。入力が空なら `Ok(value: [])` です。読めなかったときは `Err(error: IoError.Failed(reason: ...))` で、`reason` に理由の文字列が入ります。入力が UTF-8 のテキストでないときは、`Err(error: IoError.Failed(reason: "malformed-text"))` です。

テストでは、`std.test` の `World` の `input` が返る行になります（後述）。

### io.write_line

```text
pub fn write_line(text: String) -> Result<Unit, IoError> uses io
```

`text` を標準出力に 1 行として書きます。書けなかったときは `Err(error: IoError.Failed(reason: ...))` です。

テストでは、書いた行が `Outcome` の `output` に、書いた順で入ります。

### clock.now_ms

```text
pub fn now_ms() -> Int uses clock
```

現在時刻を、1970-01-01 00:00:00 UTC からの経過ミリ秒を表す整数として返します。呼び出すたびに時刻を取得するため、引数が同じ（この場合は引数なし）でも結果は変わります。

テストでは、`World` の `now_ms` の値が返ります。この時計は自分では進まないので、`with_world` の中で何度呼んでも同じ値です。進めるのは `clock.sleep_ms` だけです。

### clock.monotonic_ms

```text
pub fn monotonic_ms() -> Int uses clock, io
```

単調増加する時計の値をミリ秒で返します。起点は未規定なので、2 回の取得値の差を使って経過時間を測ります。一方、`now_ms` は実際の日時を表し、システムの時刻調整で戻ることがあります。シミュレート世界（`with_world`、`sim.with_sim`）では、どちらもその世界の時計を使います。単調時計のないターゲットでは 0 を返します。

### clock.sleep_ms

```text
pub fn sleep_ms(milliseconds: Int) -> Unit uses clock, io {
  requires milliseconds >= 0;
  ...
}
```

`milliseconds` だけ待ちます。プログラムは実際にその間止まります。（`std.async.sleep` はスケジューラの仮想的な待ちで、プログラムを止めません。[18 並行処理](../language/18-concurrency.md)）

**事前条件：** `milliseconds >= 0`。満たさない呼び出しは欠陥です。

シミュレート世界では、実際に待つことはありません。世界の時計が `milliseconds` だけ進み、`monotonic_ms` と `now_ms` にその分が反映されます。

### random.int_between

```text
pub fn int_between(low: Int, high: Int) -> Int uses random {
  requires low <= high;
  ensures result >= low and result <= high;
  ...
}
```

`low` 以上 `high` 以下の整数を 1 つ返します。両端を含みます。`low == high` なら、結果は `low` です。

**事前条件：** `low <= high`。満たさない呼び出しは欠陥です。どの整数を返すかは契約では決まらず、事後条件は範囲だけを保証します。

> 補足：`low` と `high` を実行時に受け取る関数が `random.int_between` をそのまま呼ぶと、`cernith check --verify` は、`low > high` の入力がありうるとして事前条件違反を報告します（診断 `E0501`）。呼び出し側の関数に `requires low <= high;` を付けて、条件を引き継ぎます。次の例の `roll` がその形です。

### random.bytes

```text
pub fn bytes(
  count: Int,
) -> Result<Bytes, RandomError> uses io, random {
  requires count >= 0 and count <= 1048576;
  ...
}
```

プログラムが種を与える生成器ではなく、オペレーティングシステムの安全な乱数源から `count` バイトを返します。トークンや鍵に使います。秘密にする必要のない値には `int_between` を使います。`count` が 0 なら、空のバイト列です。

**事前条件：** `count >= 0 and count <= 1048576`。満たさない呼び出しは欠陥です。

安全な乱数源のないターゲットでは `Err(error: RandomError.Unsupported)` を返します。その場合、安全でないバイト列を代用して処理を続けてはいけません。そのほかの失敗は `Err(error: RandomError.Failed(detail: ...))` です。シミュレート世界では、世界のシードに基づく乱数源を使うためテストを再現できますが、そのバイト列を秘密情報として使うことはできません。

### random.describe

```text
pub fn describe(error: RandomError) -> String
```

| エラー | 文字列 |
| --- | --- |
| `Unsupported` | `"no secure random source here"` |
| `Failed(detail: "x")` | `"random: x"` |

## テストでの扱い

`example` と `property` は純粋なコードしか呼べないので、`io`・`clock`・`random` を使う関数を直接呼ぶとエラー `E0403` です。これらの効果を持つ関数は、`std.test` の `with_world` に渡して、シミュレートされた世界の中で実行します。

```text
test.with_world(
  world: test.World(input: ["a", "b"], now_ms: 1500, seed: 0),
  run: fn() -> T uses clock, io, random { ... },
)
```

`World` の 3 つのフィールドが、この 3 つのモジュールに対応します。

| `World` のフィールド | 使われ方 |
| --- | --- |
| `input` | `io.read_all_lines()` が返す行 |
| `now_ms` | `clock.now_ms()` が返す値 |
| `seed` | `random.int_between` と `random.bytes` が使う乱数列の種。同じ種なら同じ数列になる |

`with_world` は、`run` の値と書き出された行を持つ `Outcome` を返します。`with_world` 自体は純粋なので、`example` の中で使えます。`with_world` の詳細は [test](test.md) にあります。

ファイル・環境変数・ネットワークも使うテストは、`std.sim.with_sim` を使います（[sim](sim.md)）。

## 例

```cernith
module guide.ambient;

import std.clock;
import std.io;
import std.random;
import std.test;

pub fn echo_lines() -> Result<Int, io.IoError> uses io {
  let lines = io.read_all_lines()?;
  for line in lines {
    io.write_line(text: "> " ++ line)?;
  }
  Ok(value: lines.length)
}

example echo_lines_writes_what_it_reads for echo_lines {
  test.with_world(
    world: test.World(input: ["a", "b"], now_ms: 0, seed: 0),
    run: fn() -> Result<Int, io.IoError> uses clock, io, random {
      echo_lines()
    },
  )
    == test.Outcome(value: Ok(value: 2), output: ["> a", "> b"])
    and test.with_world(
      world: test.World(input: [], now_ms: 0, seed: 0),
      run: fn() -> Result<Int, io.IoError> uses clock, io, random {
        echo_lines()
      },
    )
      == test.Outcome(value: Ok(value: 0), output: [])
}

pub fn elapsed(since: Int) -> Int uses clock {
  clock.now_ms() - since
}

example elapsed_reads_the_clock for elapsed {
  test.with_world(
    world: test.World(input: [], now_ms: 1500, seed: 0),
    run: fn() -> Int uses clock, io, random {
      elapsed(since: 1000)
    },
  ).value
    == 500
}

pub fn roll(sides: Int) -> Int uses random {
  requires sides >= 1;
  ensures result >= 1 and result <= sides;

  random.int_between(low: 1, high: sides)
}

example roll_of_zero_sides_is_a_defect for roll {
  defects(
    of: test.with_world(
      world: test.World(input: [], now_ms: 0, seed: 42),
      run: fn() -> Int uses clock, io, random {
        roll(sides: 0)
      },
    ),
  )
}

example roll_stays_in_range for roll {
  test.with_world(
    world: test.World(input: [], now_ms: 0, seed: 42),
    run: fn() -> Bool uses clock, io, random {
      let first = roll(sides: 6);
      let second = roll(sides: 1);
      first >= 1 and first <= 6 and second == 1
    },
  ).value
}
```

次の例は、`random.int_between` を直接呼ぶ例です。範囲が 1 点のとき結果が決まること、同じ種なら同じ値になること、`low > high` が欠陥であることを確かめています。

```cernith
module guide.between;

import std.random;
import std.test;

pub fn pick(low: Int, high: Int) -> Int uses random {
  requires low <= high;
  ensures result >= low and result <= high;

  random.int_between(low: low, high: high)
}

example pick_of_one_point_is_that_point for pick {
  test.with_world(
    world: test.World(input: [], now_ms: 0, seed: 1),
    run: fn() -> Int uses clock, io, random {
      pick(low: 5, high: 5)
    },
  ).value
    == 5
}

example pick_repeats_for_the_same_seed for pick {
  test.with_world(
    world: test.World(input: [], now_ms: 0, seed: 3),
    run: fn() -> Int uses clock, io, random {
      pick(low: 0, high: 1000000)
    },
  )
    == test.with_world(
      world: test.World(input: [], now_ms: 0, seed: 3),
      run: fn() -> Int uses clock, io, random {
        pick(low: 0, high: 1000000)
      },
    )
}

example pick_with_reversed_bounds_is_a_defect for pick {
  defects(
    of: test.with_world(
      world: test.World(input: [], now_ms: 0, seed: 1),
      run: fn() -> Int uses clock, io, random {
        pick(low: 2, high: 1)
      },
    ),
  )
}
```

`io.read_all_lines`、`io.write_line`、`clock.now_ms` は、上の `guide.ambient` の例で確かめています。

次の例では、待機時間を計測し、ランダムなバイト列を取得します。シミュレート世界での待機に実時間はかかりません。

```cernith
module guide.clock_and_bytes;

import std.bytes;
import std.clock;
import std.random;
import std.test;

pub fn failure_text(error: random.RandomError) -> String {
  random.describe(error: error)
}

example failure_text_names_the_failure for failure_text {
  failure_text(error: random.RandomError.Unsupported)
    == "no secure random source here"
    and failure_text(error: random.RandomError.Failed(detail: "x"))
      == "random: x"
}

pub fn timed_wait(milliseconds: Int) -> Int uses clock, io {
  requires milliseconds >= 0;

  let start = clock.monotonic_ms();
  clock.sleep_ms(milliseconds: milliseconds);
  clock.monotonic_ms() - start
}

example a_negative_wait_is_a_defect for timed_wait {
  defects(
    of: test.with_world(
      world: test.World(input: [], now_ms: 0, seed: 0),
      run: fn() -> Int uses clock, io, random {
        timed_wait(milliseconds: -1)
      },
    ),
  )
}

example a_wait_moves_the_clock_of_the_world for timed_wait {
  test.with_world(
    world: test.World(input: [], now_ms: 1000, seed: 0),
    run: fn() -> List<Int> uses clock, io, random {
      let waited = timed_wait(milliseconds: 250);
      [waited, clock.now_ms(), clock.monotonic_ms()]
    },
  ).value
    == [250, 1250, 1250]
}

pub fn token(count: Int) -> String uses io, random {
  requires count >= 0 and count <= 1048576;

  match random.bytes(count: count) {
    Ok(value: data) => bytes.to_hex(bytes: data),
    Err(error: error) => random.describe(error: error),
  }
}

example the_same_seed_gives_the_same_bytes for token {
  token_in(seed: 7, count: 8) == token_in(seed: 7, count: 8)
    and token_in(seed: 7, count: 8).length == 16
    and token_in(seed: 7, count: 0) == ""
}

fn token_in(seed: Int, count: Int) -> String {
  requires count >= 0 and count <= 1048576;

  test.with_world(
    world: test.World(input: [], now_ms: 0, seed: seed),
    run: fn() -> String uses clock, io, random {
      token(count: count)
    },
  ).value
}
```

## 関連

- [console](console.md)：ケイパビリティ `Console` を通した標準入出力（`io` の代わりに使う形）
- [test](test.md)：`with_world`
- [sim](sim.md)：ファイル・環境変数・ネットワークを含むシミュレート世界
- [11 効果](../language/11-effects.md)
- [17 リソースとケイパビリティ](../language/17-resources-and-capabilities.md)
