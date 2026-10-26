# std.test

効果（`io`、`clock`、`random`）を持つコードを `example` や `property` から試すためのモジュールです。`import std.test;` と書き、`test.with_world(world, run)` のように呼びます。`with_world` は、関数 `run` を**シミュレート世界**の中で実行し、結果とその間に書き出された行を返します。

`example` と `property` は純粋なコードしか呼べません（効果を持つ関数を直接呼ぶと、エラー `E0403` です）。入出力や時計や乱数を使う関数は、`with_world` の中で呼ぶことで、決まった入力と決まった結果のテストにできます。テスト全般の書き方は [16 テスト](../language/16-testing.md) にあります。

## 型と関数

```text
pub record World {
  input: List<String>,
  now_ms: Int,
  seed: Int,
}

pub record Outcome<T> {
  value: T,
  output: List<String>,
}

pub fn with_world<T>(
  world: World,
  run: fn() -> T uses clock, io, random,
) -> Outcome<T>
```

| 名前 | 意味 |
| --- | --- |
| `test.World(input, now_ms, seed)` | シミュレート世界の中身。`input` は入力の行、`now_ms` は時計の値、`seed` は乱数の種 |
| `test.with_world(world, run)` | `run` を `world` の中で実行して、`Outcome<T>` を返す |
| `test.Outcome<T>` | 実行の結果。`value` は `run` の戻り値、`output` は書き出された行 |

`with_world` 自体は純粋です。戻り値の型に `uses` が付いていないので、`example` の中から呼べます。世界は毎回 `world` から作り直されるので、結果は決まっています（同じ `world` と同じ `run` なら、必ず同じ `Outcome` です）。

## シミュレート世界

世界の中では、`std.io`、`std.clock`、`std.random` の関数が、実際の入出力や時計や乱数の代わりに、次のように振る舞います。

| 呼び出し | 世界の中での振る舞い |
| --- | --- |
| `io.read_all_lines()` | `Ok(value: world.input)` を返す。何度呼んでも、毎回同じ `input` 全体 |
| `io.write_line(text)` | `text` を出力の末尾に加えて、`Ok(value: ())` を返す。`Outcome.output` に、書いた順に並ぶ |
| `clock.now_ms()` | 常に `world.now_ms`。世界の中で時間は進まない |
| `random.int_between(low, high)` | `world.seed` から決まる列の、次の値 |

`input` の各要素は 1 行として、与えた文字列のまま返されます。`output` の各要素は、`io.write_line` に渡した `text` そのものです。

### 乱数と seed の対応

`random.int_between(low, high)` は、`world.seed` を種とする SplitMix64 で作られます。種と、1 回ごとの値は、次のように決まります。計算はすべて $2^{64}$ を法とします。

1. 世界を作るとき、状態を `seed mod 2^64` にします（負の `seed` も使えます。`-1` は $2^{64}-1$ と同じです）。
2. 呼び出しごとに、次を行います。
   - 状態に `0x9E3779B97F4A7C15` を足す。
   - `z = 状態`
   - `z = (z ^ (z >> 30)) * 0xBF58476D1CE4E5B9`
   - `z = (z ^ (z >> 27)) * 0x94D049BB133111EB`
   - `draw = z ^ (z >> 31)`
3. 結果は `low + draw mod (high - low + 1)` です。

例として、`random.int_between(low: 1, high: 6)` を続けて呼んだときの値は、次のとおりです。

| `seed` | 1 回目 | 2 回目 | 3 回目 | 4 回目 |
| --- | --- | --- | --- | --- |
| `0` | 2 | 1 | 2 | 5 |
| `1` | 6 | 2 | 1 | 6 |
| `-1`（`18446744073709551615` と同じ） | 3 | 4 | 2 | 1 |

値の並びは、`seed` と、`int_between` を呼んだ順序・引数だけで決まります。`io` や `clock` の呼び出しには影響されません。`with_world` を呼ぶたびに世界が作り直されるので、同じ `seed` なら、呼び出しごとに同じ並びが最初から始まります。

`random.int_between` の事前条件（`low <= high`）は、世界の中でも同じです。違反すれば欠陥です。

### run に書ける効果

`run` の型は `fn() -> T uses clock, io, random` です。`run` が持てる効果は、この 3 つのうちの一部（または効果なし）です。`io` だけ、`random` だけ、といった関数を渡せます。3 つ以外の効果を持つ関数を渡すと、エラー `E0404` です。

`with_world` が扱う効果は、この 3 つだけです。ファイル、環境変数、HTTP を含む世界は、[sim](sim.md) で作ります。

## 例

```cernith
module guide.worlds;

import std.clock;
import std.int;
import std.io;
import std.random;
import std.test;

pub fn count_lines() -> Int uses io {
  let lines = match io.read_all_lines() {
    Ok(value: all) => all,
    Err(error: _) => {
      return -1;
    },
  };
  discard io.write_line(text: "lines: " ++ int.to_text(value: lines.length));
  lines.length
}

example count_lines_reads_the_input_and_writes_a_line for count_lines {
  test.with_world(
    world: test.World(input: ["a", "b", "c"], now_ms: 0, seed: 0),
    run: fn() -> Int uses io {
      count_lines()
    },
  )
    == test.Outcome(value: 3, output: ["lines: 3"])
    and test.with_world(
      world: test.World(input: [], now_ms: 0, seed: 0),
      run: fn() -> Int uses io {
        count_lines()
      },
    ).value
      == 0
}

pub fn count_twice() -> Int uses io {
  let first = match io.read_all_lines() {
    Ok(value: all) => all,
    Err(error: _) => [],
  };
  let second = match io.read_all_lines() {
    Ok(value: all) => all,
    Err(error: _) => [],
  };
  first.length + second.length
}

example every_read_returns_the_whole_input for count_twice {
  test.with_world(
    world: test.World(input: ["x", "y"], now_ms: 0, seed: 0),
    run: fn() -> Int uses io {
      count_twice()
    },
  ).value
    == 4
}

pub fn elapsed() -> Int uses clock {
  let start = clock.now_ms();
  let end = clock.now_ms();
  end - start
}

example the_clock_shows_now_ms_and_does_not_advance for elapsed {
  test.with_world(
    world: test.World(input: [], now_ms: 1234, seed: 0),
    run: fn() -> Int uses clock {
      clock.now_ms()
    },
  ).value
    == 1234
    and test.with_world(
      world: test.World(input: [], now_ms: 1234, seed: 0),
      run: fn() -> Int uses clock {
        elapsed()
      },
    ).value
      == 0
}

pub fn greet(name: String) -> Bool uses io {
  match io.write_line(text: "hello " ++ name) {
    Ok(value: _) => true,
    Err(error: _) => false,
  }
}

example write_line_succeeds_and_is_recorded for greet {
  test.with_world(
    world: test.World(input: [], now_ms: 0, seed: 0),
    run: fn() -> Bool uses io {
      greet(name: "ada")
    },
  )
    == test.Outcome(value: true, output: ["hello ada"])
}

pub fn report() -> Unit uses clock, io, random {
  discard io.write_line(text: "time " ++ int.to_text(value: clock.now_ms()));
  discard io.write_line(text: "roll " ++ int.to_text(value: roll()));
}

example a_pure_run_is_allowed for report {
  test.with_world(
    world: test.World(input: [], now_ms: 0, seed: 0),
    run: fn() -> Int {
      3
    },
  )
    == test.Outcome(value: 3, output: [])
}

example all_three_effects_in_one_world for report {
  test.with_world(
    world: test.World(input: [], now_ms: 500, seed: 0),
    run: fn() -> Unit uses clock, io, random {
      report()
    },
  ).output
    == ["time 500", "roll 2"]
}

pub fn roll() -> Int uses random {
  random.int_between(low: 1, high: 6)
}

example an_empty_range_is_a_defect_in_the_world_too for roll {
  defects(
    of: test.with_world(
      world: test.World(input: [], now_ms: 0, seed: 0),
      run: fn() -> Int uses random {
        random.int_between(low: 5, high: 1)
      },
    ).value,
  )
}

example each_world_starts_the_sequence_again for roll {
  test.with_world(
    world: test.World(input: [], now_ms: 0, seed: 0),
    run: fn() -> Int uses random {
      roll()
    },
  ).value
    == 2
    and test.with_world(
      world: test.World(input: [], now_ms: 0, seed: 0),
      run: fn() -> Int uses random {
        roll()
      },
    ).value
      == 2
}

example the_range_gives_low_plus_the_remainder for roll {
  test.with_world(
    world: test.World(input: [], now_ms: 0, seed: 0),
    run: fn() -> Int uses random {
      random.int_between(low: 0, high: 99)
    },
  ).value
    == 35
    and test.with_world(
      world: test.World(input: [], now_ms: 0, seed: 0),
      run: fn() -> Int uses random {
        random.int_between(low: 7, high: 7)
      },
    ).value
      == 7
}

property a_roll_is_from_one_to_six_for_every_seed for roll(seed: Int) {
  test.with_world(
    world: test.World(input: [], now_ms: 0, seed: seed),
    run: fn() -> Int uses random {
      roll()
    },
  ).value
    >= 1
    and test.with_world(
      world: test.World(input: [], now_ms: 0, seed: seed),
      run: fn() -> Int uses random {
        roll()
      },
    ).value
      <= 6
}

pub fn roll_four() -> List<Int> uses random {
  [roll(), roll(), roll(), roll()]
}

example a_negative_seed_wraps_around_modulo_2_to_the_64 for roll_four {
  test.with_world(
    world: test.World(input: [], now_ms: 0, seed: -1),
    run: fn() -> List<Int> uses random {
      roll_four()
    },
  ).value
    == [3, 4, 2, 1]
    and test.with_world(
      world: test.World(input: [], now_ms: 0, seed: 18446744073709551615),
      run: fn() -> List<Int> uses random {
        roll_four()
      },
    ).value
      == [3, 4, 2, 1]
}

example the_seed_decides_the_rolls for roll_four {
  test.with_world(
    world: test.World(input: [], now_ms: 0, seed: 0),
    run: fn() -> List<Int> uses random {
      roll_four()
    },
  ).value
    == [2, 1, 2, 5]
    and test.with_world(
      world: test.World(input: [], now_ms: 0, seed: 1),
      run: fn() -> List<Int> uses random {
        roll_four()
      },
    ).value
      == [6, 2, 1, 6]
}
```

効果を持つ関数を、`with_world` なしで `example` から呼ぶと、エラー `E0403` です。

```cernith-error E0403
module guide.worlds_direct;

import std.io;

pub fn hello() -> Unit uses io {
  discard io.write_line(text: "hello");
}

example calls_an_effect_directly for hello {
  hello() == ()
}
```

`run` が、`clock`、`io`、`random` 以外の効果を持つと、エラー `E0404` です。

```cernith-error E0404
module guide.worlds_effects;

import std.io;
import std.test;

pub fn hello() -> Unit uses io {
  discard io.write_line(text: "hello");
}

example run_has_another_effect for hello {
  test.with_world(
    world: test.World(input: [], now_ms: 0, seed: 0),
    run: fn() -> Unit uses io, diverge {
      hello()
    },
  ).output
    == ["hello"]
}
```

## 関連

- [16 テスト](../language/16-testing.md)：`example`、`property`、`defects(of: ...)`
- [11 効果](../language/11-effects.md)：`io`、`clock`、`random` などの効果
- [io-clock-random](io-clock-random.md)：世界の中で振る舞いが決まる関数
- [sim](sim.md)：ファイルや HTTP を含むシミュレート世界
