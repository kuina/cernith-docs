# std.io、std.clock、std.random

標準入出力・時計・乱数を扱う 3 つの小さなモジュールです。`import std.io;`、`import std.clock;`、`import std.random;` と書き、`io.write_line(...)`、`clock.now_ms()`、`random.int_between(...)` のように呼びます。

この 3 つが扱うのは、**ambient な効果**（`io`、`clock`、`random`）です。呼び出すには、その関数が効果を `uses` で宣言していれば足り、ケイパビリティを受け取る必要はありません（[11 効果](../language/11-effects.md)）。ファイル・環境変数・ネットワークのように、プログラムが宣言して受け取った権限を通して行う操作は、別のモジュールが担当します（[fs](fs.md)、[env](env.md)、[http](http.md)、[console](console.md)。仕組みは [17 リソースとケイパビリティ](../language/17-resources-and-capabilities.md)）。

効果は、宣言した名前ごとに独立です。`clock` を使う関数は `uses clock` と書く必要があり、`uses io` では足りません（足りないとエラー `E0401`）。複数使うときは `uses clock, io, random` のように並べます。

## 関数一覧

| 関数 | 戻り値の型 | 効果 | 意味 |
| --- | --- | --- | --- |
| `io.read_all_lines()` | `Result<List<String>, IoError>` | `io` | 標準入力の全行を読む |
| `io.write_line(text)` | `Result<Unit, IoError>` | `io` | 標準出力に 1 行書く |
| `clock.now_ms()` | `Int` | `clock` | 現在の時刻（ミリ秒） |
| `random.int_between(low, high)` | `Int` | `random` | `low` 以上 `high` 以下の整数を 1 つ選ぶ |

`std.io` にはエラーの型もあります。

```text
pub variant IoError {
  Failed(reason: String),
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

現在の時刻をミリ秒の整数で返します。実際に実行したときは、1970-01-01 00:00:00 UTC からの経過ミリ秒です。呼ぶたびに時刻を読み直すので、同じ引数（なし）でも結果は変わります。

テストでは、`World` の `now_ms` の値が返ります。この時計は進まないので、`with_world` の中で何度呼んでも同じ値です。

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
| `seed` | `random.int_between` が使う乱数列の種。同じ種なら同じ数列になる |

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

## 関連

- [console](console.md)：ケイパビリティ `Console` を通した標準入出力（`io` の代わりに使う形）
- [test](test.md)：`with_world`
- [sim](sim.md)：ファイル・環境変数・ネットワークを含むシミュレート世界
- [11 効果](../language/11-effects.md)
- [17 リソースとケイパビリティ](../language/17-resources-and-capabilities.md)
