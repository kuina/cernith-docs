# std.io, std.clock, and std.random

These three modules provide standard input/output, time, and random numbers. Import them with `import std.io;`, `import std.clock;`, and `import std.random;`, then call functions such as `io.write_line(...)`, `clock.now_ms()`, and `random.int_between(...)`.

These three handle **ambient effects** (`io`, `clock`, `random`). To call them, it is enough for the function to declare the effect with `uses`; no capability needs to be received ([11 Effects](../language/11-effects.md)). Operations that go through an authority the program declares and receives, such as files, environment variables, and the network, are handled by other modules ([fs](fs.md), [env](env.md), [http](http.md), [console](console.md); for the mechanism, see [17 Resources and capabilities](../language/17-resources-and-capabilities.md)).

Effects are independent for each declared name. A function that uses `clock` must write `uses clock`; `uses io` is not enough (otherwise the error is `E0401`). To use several, list them as in `uses clock, io, random`.

## Function list

| Function | Return type | Effect | Meaning |
| --- | --- | --- | --- |
| `io.read_all_lines()` | `Result<List<String>, IoError>` | `io` | Reads all lines of standard input |
| `io.write_line(text)` | `Result<Unit, IoError>` | `io` | Writes one line to standard output |
| `clock.now_ms()` | `Int` | `clock` | The current time (in milliseconds) |
| `clock.monotonic_ms()` | `Int` | `clock`, `io` | A clock that never goes back (in milliseconds) |
| `clock.sleep_ms(milliseconds)` | `Unit` | `clock`, `io` | Really waits for that many milliseconds |
| `random.int_between(low, high)` | `Int` | `random` | Picks one integer from `low` to `high` inclusive |
| `random.bytes(count)` | `Result<Bytes, RandomError>` | `io`, `random` | `count` bytes from the secure random source of the system |
| `random.describe(error)` | `String` | none | Turns a `RandomError` into a string |

`std.io` also has an error type, and so does `std.random`.

```text
pub variant IoError {
  Failed(reason: String),
}

pub variant RandomError {
  Unsupported,
  Failed(detail: String),
}
```

## Function details

### io.read_all_lines

```text
pub fn read_all_lines() -> Result<List<String>, IoError> uses io
```

Reads standard input to the end and returns it as a list of lines. It takes no arguments. If the input is empty, the result is `Ok(value: [])`. If the input cannot be read, the result is `Err(error: IoError.Failed(reason: ...))`, where `reason` holds a string giving the reason. If the input is not valid UTF-8 text, the result is `Err(error: IoError.Failed(reason: "malformed-text"))`.

In tests, the `input` of the `World` of `std.test` becomes the lines that are returned (see below).

### io.write_line

```text
pub fn write_line(text: String) -> Result<Unit, IoError> uses io
```

Writes `text` to standard output as one line. If it cannot be written, the result is `Err(error: IoError.Failed(reason: ...))`.

In tests, the lines written go into the `output` of the `Outcome`, in the order written.

### clock.now_ms

```text
pub fn now_ms() -> Int uses clock
```

Returns the current time as an integer number of milliseconds. In an actual run, this is the number of milliseconds elapsed since 1970-01-01 00:00:00 UTC. It reads the time anew on every call, so the result changes even with the same arguments (here, none).

In tests, the value of the `now_ms` of the `World` is returned. This clock does not advance by itself, so it gives the same value no matter how many times it is called inside `with_world`; only `clock.sleep_ms` moves it forward.

### clock.monotonic_ms

```text
pub fn monotonic_ms() -> Int uses clock, io
```

Returns a monotonic clock reading in milliseconds. Its origin is unspecified; use the difference between two readings to measure elapsed time. Unlike this clock, `now_ms` reports wall-clock time and may be adjusted backward by the system. In a simulated world (`with_world`, `sim.with_sim`), both functions use the world's clock. Targets without a monotonic clock return 0.

### clock.sleep_ms

```text
pub fn sleep_ms(milliseconds: Int) -> Unit uses clock, io {
  requires milliseconds >= 0;
  ...
}
```

Waits for `milliseconds`: the program really stops for that long. (`std.async.sleep` is the scheduler's virtual wait and never stops the program, [18 Concurrency](../language/18-concurrency.md).)

**Precondition:** `milliseconds >= 0`. A call that does not satisfy it is a defect.

In a simulated world no time passes. The clock of the world moves forward by `milliseconds`, so `monotonic_ms` and `now_ms` show the wait at once.

### random.int_between

```text
pub fn int_between(low: Int, high: Int) -> Int uses random {
  requires low <= high;
  ensures result >= low and result <= high;
  ...
}
```

Returns one integer from `low` to `high`, both ends included. If `low == high`, the result is `low`.

**Precondition:** `low <= high`. A call that does not satisfy it is a defect. The contract does not determine which integer is returned, and the postcondition guarantees only the range.

> Note: If a function that receives `low` and `high` at run time calls `random.int_between` directly, `cernith check --verify` reports a precondition violation, because inputs with `low > high` are possible (diagnostic `E0501`). Attach `requires low <= high;` to the calling function to pass the condition along. `roll` in the following example has that form.

### random.bytes

```text
pub fn bytes(
  count: Int,
) -> Result<Bytes, RandomError> uses io, random {
  requires count >= 0 and count <= 1048576;
  ...
}
```

Returns `count` bytes from the secure source of the operating system, not from a generator that the program seeds. Use it for tokens and keys; use `int_between` for values that need not be secret. If `count` is 0, the result is empty bytes.

**Precondition:** `count >= 0 and count <= 1048576`. A call that does not satisfy it is a defect.

A target that has no secure source answers `Err(error: RandomError.Unsupported)`, and a program must not go on with bytes that are not random. Another failure is `Err(error: RandomError.Failed(detail: ...))`. In a simulated world the bytes come from the seeded source of the world, so a test repeats; they are not secret there.

### random.describe

```text
pub fn describe(error: RandomError) -> String
```

| Error | String |
| --- | --- |
| `Unsupported` | `"no secure random source here"` |
| `Failed(detail: "x")` | `"random: x"` |

## Handling in tests

An `example` and a `property` can call only pure code, so calling a function that uses `io`, `clock`, or `random` directly is error `E0403`. A function with these effects is passed to `with_world` of `std.test` and run inside a simulated world.

```text
test.with_world(
  world: test.World(input: ["a", "b"], now_ms: 1500, seed: 0),
  run: fn() -> T uses clock, io, random { ... },
)
```

The three fields of `World` correspond to these three modules.

| Field of `World` | How it is used |
| --- | --- |
| `input` | The lines that `io.read_all_lines()` returns |
| `now_ms` | The value that `clock.now_ms()` returns |
| `seed` | The seed of the random sequence that `random.int_between` and `random.bytes` use. The same seed gives the same sequence |

`with_world` returns an `Outcome` that holds the value of `run` and the lines that were written out. `with_world` itself is pure, so it can be used inside an `example`. The details of `with_world` are in [test](test.md).

A test that also uses files, environment variables, or the network uses `std.sim.with_sim` ([sim](sim.md)).

## Examples

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

The next example calls `random.int_between` directly. It checks that the result is determined when the range is a single point, that the same seed gives the same value, and that `low > high` is a defect.

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

`io.read_all_lines`, `io.write_line`, and `clock.now_ms` are checked in the `guide.ambient` example above.

The next example waits, measures elapsed time, and requests random bytes. The simulated wait takes no real time.

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

## Related

- [console](console.md): standard input and output through the capability `Console` (the form to use instead of `io`)
- [test](test.md): `with_world`
- [sim](sim.md): a simulated world that includes files, environment variables, and the network
- [11 Effects](../language/11-effects.md)
- [17 Resources and capabilities](../language/17-resources-and-capabilities.md)
