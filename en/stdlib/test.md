# std.test

This module lets `example` and `property` tests exercise code with effects (`io`, `clock`, `random`). Import it with `import std.test;` and call `test.with_world(world, run)`. `with_world` executes `run` in a **simulated world** and returns its result along with the lines written during execution.

Calling effectful code directly from an `example` or `property` produces `E0403`. Use `with_world` to test input/output, time, or randomness with fixed inputs and reproducible results. For general testing syntax, see [16 Testing](../language/16-testing.md).

## Types and functions

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

| Name | Meaning |
| --- | --- |
| `test.World(input, now_ms, seed)` | The contents of the simulated world. `input` is the lines of input, `now_ms` is the clock value, and `seed` is the random seed |
| `test.with_world(world, run)` | Runs `run` inside `world` and returns an `Outcome<T>` |
| `test.Outcome<T>` | The result of the run. `value` is the return value of `run`, and `output` is the lines that were written |

`with_world` itself is pure. Its return type has no `uses`, so it can be called from inside an `example`. The world is rebuilt from `world` every time, so the result is fixed (the same `world` and the same `run` always give the same `Outcome`).

## The simulated world

Inside the world, the functions of `std.io`, `std.clock`, and `std.random` behave as follows instead of performing real input and output or using the real clock and randomness.

| Call | Behavior inside the world |
| --- | --- |
| `io.read_all_lines()` | Returns `Ok(value: world.input)`. No matter how many times it is called, it returns the whole `input` each time |
| `io.write_line(text)` | Appends `text` to the end of the output and returns `Ok(value: ())`. The lines appear in `Outcome.output` in the order they were written |
| `clock.now_ms()` | Always `world.now_ms`. Time does not advance inside the world |
| `random.int_between(low, high)` | The next value in the sequence determined by `world.seed` |

Each element of `input` is returned as one line, exactly as the string was given. Each element of `output` is the very `text` that was passed to `io.write_line`.

### How random numbers correspond to the seed

`random.int_between(low, high)` is built on SplitMix64 with `world.seed` as the seed. The seed and the value of each call are determined as follows. All calculations are modulo $2^{64}$.

1. When the world is created, the state is set to `seed mod 2^64` (a negative `seed` can also be used; `-1` is the same as $2^{64}-1$).
2. For each call, do the following.
   - Add `0x9E3779B97F4A7C15` to the state.
   - `z = state`
   - `z = (z ^ (z >> 30)) * 0xBF58476D1CE4E5B9`
   - `z = (z ^ (z >> 27)) * 0x94D049BB133111EB`
   - `draw = z ^ (z >> 31)`
3. The result is `low + draw mod (high - low + 1)`.

As an example, the values obtained by calling `random.int_between(low: 1, high: 6)` repeatedly are as follows.

| `seed` | First call | Second call | Third call | Fourth call |
| --- | --- | --- | --- | --- |
| `0` | 2 | 1 | 2 | 5 |
| `1` | 6 | 2 | 1 | 6 |
| `-1` (the same as `18446744073709551615`) | 3 | 4 | 2 | 1 |

The sequence of values is determined only by `seed` and by the order and arguments of the calls to `int_between`. Calls to `io` and `clock` do not affect it. The world is rebuilt every time `with_world` is called, so with the same `seed`, the same sequence starts from the beginning in every call.

The precondition of `random.int_between` (`low <= high`) is the same inside the world. Violating it is a defect.

### Effects allowed in run

The type of `run` is `fn() -> T uses clock, io, random`. The effects that `run` can have are some of these three (or no effects). You can pass a function that uses only `io`, only `random`, and so on. Passing a function that has an effect other than these three is error `E0404`.

The only effects that `with_world` handles are these three. A world that includes files, environment variables, and HTTP is built with [sim](sim.md).

## Example

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

Calling a function that has effects from an `example` without `with_world` is error `E0403`.

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

If `run` has an effect other than `clock`, `io`, and `random`, the result is error `E0404`.

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

## Related

- [16 Testing](../language/16-testing.md): `example`, `property`, and `defects(of: ...)`
- [11 Effects](../language/11-effects.md): effects such as `io`, `clock`, and `random`
- [io-clock-random](io-clock-random.md): the functions whose behavior is determined inside the world
- [sim](sim.md): a simulated world that includes files and HTTP
