# Concurrency

Cernith values cannot be changed, and there is no syntax for creating shared mutable state, so values cannot race. Concurrency is built in three layers, and all of them appear in the source.

1. **`parallel`**: applies a pure function to each element of a list. The result is defined to be the same as the result of applying it sequentially.
2. **Structured tasks**: tasks are started inside a `scope`. A scope ends only after all of its tasks have ended. A function that may suspend declares the effect `async`, and the point where it suspends is written `await`.
3. **Channels**: typed, bounded passages between tasks of the same scope that carry immutable values.

There are no locks, atomic variables, detached tasks, thread handles, or global mutable state. Time and ordering come from the world. Scheduling is an input.

## parallel

`std.parallel` provides `map`, `filter`, `fold`, `any`, and `all` ([stdlib/parallel.md](../stdlib/parallel.md)). The results are the same as those of the `std.list` functions of the same names. The function arguments **cannot have effects** (passing a function with effects gives `E0404`). Because the computation for each element is independent, the implementation may run it on multiple workers, in any order. If a defect occurs, execution stops, just as in the sequential case. These functions can be called without declaring `async`.

## Scopes and tasks

```text
await async.scope(
  body: fn(tasks: async.Scope) -> Int uses async {
    let t = async.spawn(tasks: tasks, run: fn() -> Int uses async { ... });
    await t
  },
)
```

- `async.scope(body)` runs `body`, passing it a `Scope`, and returns the value of `body` **after all tasks started in that scope have ended**. It is an `async` function, so it is called with `await`, and the enclosing function declares `async`.
- `async.spawn(tasks: s, run: f)` starts `f` as a task of `s` and returns a `Task<T>` immediately. The task runs when the current task waits. It is an `async` function that can be written without `await`. `f` has no parameters, and its effects are a subset of those of the enclosing function (it may declare `async`). It can capture values, but it cannot capture resources (`E1101`), `var` (`E0322`), or `Task` (`E1202`).
- **Every task must be awaited exactly once on every path that reaches the end of the scope.** If any path does not `await` it, the result is `E1201`; awaiting it two or more times (including inside a loop) gives `E1204`. A path that exits through `?` or `return` needs no `await` (the scope waits).
- A `Task` cannot be copied, stored, passed, or captured; it can only be awaited (`E1202`). `Task`, `Scope`, and channel endpoints cannot be made the value of a scope (`E1202`). `Scope` and channel endpoints can be used by the tasks of their own scope.
- A task can start other tasks in the same scope, and it can open its own scopes. A scope ends when all of its tasks have ended.
- Failure: the value of a task is the value that its function returns. A failure that can occur is a `Result` inside `T`, and the side that awaits handles it. A defect inside a task ends the program (exit status 70).
- Tasks run in turns. A task runs until the next suspension point (a waiting `await`, `pause`, `sleep`, or a waiting channel operation). A program must not depend on the order of tasks.

## The async effect and await

- A function that may suspend declares `uses async` (there is no `async fn` syntax). A function without `async` cannot call one that has it (`E0401`). Because `example` and `property` are pure, use `async.block_on` or `test.with_world`.
- **A call to an `async` function must be written as `await call(...)`** (`E1203`). The call is a suspension point. Apart from `Task`, there are no futures as values. `await` can also be written on a `Task`.
- `await e?` means `(await e)?`.
- `await` can be written only inside a function or lambda that declares `async`. An `await` in front of something that cannot suspend is `E1206`.
- The termination rules are the same for `async` functions ([13 Termination](13-termination.md)).
- `await` cannot be written inside the body of `using` (`E1207`). Do the read inside, and the waiting outside.
- `async` cannot be passed through an effect parameter (`E1205`). Do not pass an `async` function to `list.map`; write a loop instead.

```cernith
module guide.tasks;

import std.async;

pub record Pair {
  left: Int,
  right: Int,
}

pub fn run() -> Pair {
  async.block_on(
    run: fn() -> Pair uses async {
      await squares(a: 3, b: 4)
    },
  )
}

example both_squares for run {
  run() == Pair(left: 9, right: 16)
}

fn square_later(n: Int) -> Int uses async {
  await async.sleep(ms: n);
  n * n
}

pub fn squares(a: Int, b: Int) -> Pair uses async {
  await async.scope(
    body: fn(tasks: async.Scope) -> Pair uses async {
      let left = async.spawn(
        tasks: tasks,
        run: fn() -> Int uses async {
          await square_later(n: a)
        },
      );
      let right = async.spawn(
        tasks: tasks,
        run: fn() -> Int uses async {
          await square_later(n: b)
        },
      );
      Pair(left: await left, right: await right)
    },
  )
}
```

`async.block_on(run: ...)` is an ordinary function that runs a program containing tasks and returns its value (it does not declare `async`, so it can be called from an `example` or from `main`).

```cernith-error E1201
module guide.tasks;

import std.async;

pub fn forgotten() -> Int uses async {
  await async.scope(
    body: fn(tasks: async.Scope) -> Int uses async {
      discard async.spawn(
        tasks: tasks,
        run: fn() -> Int uses async {
          1
        },
      );
      2
    },
  )
}
```

```cernith-error E1203
module guide.tasks;

import std.async;

pub fn slow() -> Int uses async {
  await async.sleep(ms: 1);
  1
}

pub fn caller() -> Int uses async {
  slow()
}
```

```cernith-error E1204
module guide.tasks;

import std.async;

pub fn twice() -> Int uses async {
  await async.scope(
    body: fn(tasks: async.Scope) -> Int uses async {
      let t = async.spawn(
        tasks: tasks,
        run: fn() -> Int uses async {
          1
        },
      );
      let a = await t;
      let b = await t;
      a + b
    },
  )
}
```

## Time and cancellation

- `async.sleep(ms: n)` suspends the task until the scheduler’s clock equals its current value plus `n`. **The clock advances only when all tasks are waiting**, and only to the earliest wake-up time. No real time passes, so tests that use time are also deterministic. `sleep(ms: 0)` is `pause`.
- `async.pause()` lets other runnable tasks run first.
- `async.with_timeout(tasks: s, ms: n, run: f)` runs `f` as a task of `s`. If it finishes within `n` milliseconds on the clock (finishing exactly at `n` counts as within), the result is `Ok(value)`; if it does not, the task is canceled and the result is `Err(Timeout.TimedOut)`. `n` is positive.
- `async.cancel(task: t)` removes the task from all wait queues and never runs it again. The tasks of any scope that the task opened are also canceled. Awaiting a canceled task is a defect.
- `async.all(tasks: ts)` returns the values of the tasks in list order.
- `async.race(tasks: s, candidates: ts)` returns the value of the first task to finish as `Some` and cancels the rest. If the candidates are empty, the result is `None`. When several become ready at the same time, the scheduler’s policy decides.

A list of tasks is built by appending the result of `spawn` to a `var` (`started = started ++ [async.spawn(...)]`). Binding the result of `spawn` to a separate name first and then putting it in the list gives `E1202`. `async.cancel(task: ...)` takes a task in the same way: a name bound by `let` gives `E1202` (and `E1201`, because that task is not awaited), so pass an element of such a list, as in `await async.cancel(task: started[0])`.

```cernith
module guide.timing;

import std.async;

fn after(ms: Int, value: Int) -> Int uses async {
  await async.sleep(ms: ms);
  value
}

pub fn all_values(delays: List<Int>) -> List<Int> {
  async.block_on(
    run: fn() -> List<Int> uses async {
      await async.scope(
        body: fn(tasks: async.Scope) -> List<Int> uses async {
          var started: List<async.Task<Int>> = [];
          for delay in delays {
            started = started
              ++ [
                async.spawn(
                  tasks: tasks,
                  run: fn() -> Int uses async {
                    await after(ms: delay, value: delay)
                  },
                ),
              ];
          }
          await async.all(tasks: started)
        },
      )
    },
  )
}

example all_in_order for all_values {
  all_values(delays: [20, 10, 30]) == [20, 10, 30]
}

pub fn fastest(delays: List<Int>) -> Option<Int> {
  async.block_on(
    run: fn() -> Option<Int> uses async {
      await async.scope(
        body: fn(tasks: async.Scope) -> Option<Int> uses async {
          var runners: List<async.Task<Int>> = [];
          for delay in delays {
            runners = runners
              ++ [
                async.spawn(
                  tasks: tasks,
                  run: fn() -> Int uses async {
                    await after(ms: delay, value: delay)
                  },
                ),
              ];
          }
          await async.race(tasks: tasks, candidates: runners)
        },
      )
    },
  )
}

example fastest_wins for fastest {
  fastest(delays: [50, 5, 20]) == Some(value: 5) and fastest(delays: []) == None
}

pub fn timed_out() -> Bool {
  async.block_on(
    run: fn() -> Bool uses async {
      await async.scope(
        body: fn(tasks: async.Scope) -> Bool uses async {
          let result = await async.with_timeout(
            tasks: tasks,
            ms: 10,
            run: fn() -> Int uses async {
              await after(ms: 100, value: 1)
            },
          );
          result == Err(error: async.Timeout.TimedOut)
        },
      )
    },
  )
}

example times_out for timed_out {
  timed_out()
}
```

## Channels

`std.async` provides both tasks and channels.

| Function | Type | Description |
| --- | --- | --- |
| `bounded::<T>(tasks, capacity)` | `Channel<T>` (a record with `sender: Sender<T>` and `receiver: Receiver<T>`) `uses async` | `capacity >= 1`. Created inside a scope and closed at the end of the scope |
| `send(to, value)` | `Result<Unit, SendError>` `uses async` | Waits if full. If closed, `Err(error: SendError.Closed)` |
| `receive(from)` | `Option<T>` `uses async` | Waits if empty. `None` if closed and empty |
| `close(sender)` | `Unit` `uses async` | Later `send` calls fail, and the receiving side can read what remains |

`T` is a first-class value type (neither a resource nor a scope-bound value). Channel endpoints can be used by tasks of the same scope. With one sender and one receiver and pure tasks, the exchange is deterministic. If all tasks are waiting and no task can be woken, that is a deadlock, and the scheduler stops with a defect.

```cernith
module guide.channels;

import std.async;

pub fn total(count: Int) -> Int {
  requires count >= 0;

  async.block_on(
    run: fn() -> Int uses async {
      await async.scope(
        body: fn(tasks: async.Scope) -> Int uses async {
          let numbers = await async.bounded::<Int>(tasks: tasks, capacity: 2);
          let producer = async.spawn(
            tasks: tasks,
            run: fn() -> Unit uses async {
              for i in 0..count {
                discard await async.send(to: numbers.sender, value: i);
              }
              await async.close(sender: numbers.sender);
            },
          );
          var sum: Int = 0;
          for _ in 0..count {
            match await async.receive(from: numbers.receiver) {
              Some(value: value) => {
                sum = sum + value;
              },
              None => {},
            }
          }
          await producer;
          sum
        },
      )
    },
  )
}

example the_sum for total {
  total(count: 5) == 10
}
```

A waiting loop follows the same termination rules as any other loop ([13 Termination](13-termination.md)). Here the number of receives is fixed at `count`, so the loop is written over a finite range, as in `for _ in 0..count`.

## Classification of values

| Category | Examples | Can a `spawn` lambda capture it? | Can it leave the scope? |
| --- | --- | --- | --- |
| First-class | numbers, records, capabilities | Yes | Yes |
| Second-class (resource) | file, window | No (`E1101`) | No (`E1101`) |
| Scope-bound | `Scope`, `Sender`, `Receiver` | Yes, inside the scope | No (`E1202`) |
| Scope-bound, not capturable | `Task` | No | No (`E1202`) |

## Scheduling is an input

The scheduler is Cernith code in `std.async` (`block_on`, `block_on_with`), and the same code runs in every execution environment. `Policy.Fifo` runs tasks in the order in which they became runnable. `Policy.Seeded(seed: n)` chooses the next task from the runnable ones using a random number determined by `n`. The same seed gives the same order everywhere.

If a property takes the seed as input, then when an order-dependent mistake is found, the failing seed is reported as the counterexample and can be reproduced with `block_on_with(policy: Policy.Seeded(seed: s), ...)`.

```cernith
module guide.schedules;

import std.async;

pub fn sum_of_two(seed: Int) -> Int {
  async.block_on_with(
    policy: async.Policy.Seeded(seed: seed),
    run: fn() -> Int uses async {
      await async.scope(
        body: fn(tasks: async.Scope) -> Int uses async {
          let first = async.spawn(
            tasks: tasks,
            run: fn() -> Int uses async {
              await async.pause();
              1
            },
          );
          let second = async.spawn(
            tasks: tasks,
            run: fn() -> Int uses async {
              await async.pause();
              2
            },
          );
          await first + await second
        },
      )
    },
  )
}

property order_does_not_matter for sum_of_two(seed: Int) {
  sum_of_two(seed: seed) == 3
}
```

## Diagnostic codes

| Code | Name | Meaning |
| --- | --- | --- |
| `E1201` | task-not-awaited | A task is not awaited on a path that reaches the end of the scope |
| `E1202` | task-escapes-scope | A task, scope, or channel endpoint became the value of a scope, or a task was copied, stored, passed, or captured |
| `E1203` | missing-await | A call to an `async` function has no `await` |
| `E1204` | task-awaited-twice | A task was awaited two or more times (including inside a loop) |
| `E1205` | async-through-effect-parameter | The effect `async` was passed through an effect parameter |
| `E1206` | await-of-nothing | `await` was written in front of something that cannot suspend |
| `E1207` | await-in-using | The body of `using` contains a suspension point |
| `W1203` | wait-cannot-complete | The receiving end of a channel is used, and the sending end is never used |

The following are also relevant: `E0401` (an `await` or a call to an `async` function was made without `async`), `E0404` (a function with effects was passed to `parallel`), `E0322` (a task captured a `var`), and `E1101` ([17 Resources and capabilities](17-resources-and-capabilities.md)).

## Related

- Effects: [11 Effects](11-effects.md)
- Functions of `std.async`: [stdlib/async.md](../stdlib/async.md)
- `std.parallel`: [stdlib/parallel.md](../stdlib/parallel.md)
