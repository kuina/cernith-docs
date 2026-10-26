# std.async

A module that provides tasks, scopes, channels, waiting for time, and the scheduler that runs them. Write `import std.async;` and call functions such as `async.scope(...)` and `async.spawn(...)`. The concepts and rules (`uses async`, `await`, the rule that every task must be awaited before the end of its scope, and so on) are in [18 Concurrency](../language/18-concurrency.md). This page is a list of functions and types.

A call to a `uses async` function is written with `await`, as in `await async.sleep(ms: 10)`. The exceptions are `spawn`, `bounded`, and `close`, for which `await` is not required. `spawn` returns a `Task` immediately and is written without `await`; `bounded` and `close` may be written with or without it.

## Types

| Type | Meaning |
| --- | --- |
| `async.Scope` | The place where tasks are started. Received by the body of `scope` |
| `async.Task<T>` | A task that was started. Await it to get a value `T` |
| `async.Channel<T>` | A record with `sender: Sender<T>` and `receiver: Receiver<T>` |
| `async.Sender<T>`, `async.Receiver<T>` | The sending side and the receiving side of a channel |
| `async.Policy` | The policy of the scheduler: `Policy.Fifo` or `Policy.Seeded(seed: Int)` |
| `async.SendError` | A failure to send: `SendError.Closed` |
| `async.Timeout` | A timeout: `Timeout.TimedOut` |

`Scope`, `Task`, `Sender`, and `Receiver` cannot be taken out of the scope ([18 Concurrency](../language/18-concurrency.md)).

## Function list

| Function | Return type | Meaning |
| --- | --- | --- |
| `async.block_on(run)` | `T` | Runs `run` and the tasks it starts, in order from whichever became runnable first, and returns the value of `run`. An ordinary function that does not declare `async` |
| `async.block_on_with(policy, run)` | `T` | Runs with a specified policy |
| `async.scope(body)` | `T` `uses async` | Runs `body` with a `Scope` passed to it, and returns the value of `body` after all tasks of that scope have finished |
| `async.spawn(tasks, run)` | `Task<T>` `uses async` | Starts `run` as a task of `tasks`. Written without `await` |
| `async.all(tasks)` | `List<T>` `uses async` | Returns the values of the list of tasks, in the order of the list |
| `async.race(tasks, candidates)` | `Option<T>` `uses async` | Returns the value of the task that finished first as `Some` and cancels the rest. If the candidates are empty, the result is `None` |
| `async.cancel(task)` | `Unit` `uses async` | Cancels a task. It is never run again |
| `async.sleep(ms)` | `Unit` `uses async` | Waits `ms` milliseconds on the scheduler’s clock. `ms >= 0` |
| `async.pause()` | `Unit` `uses async` | Lets the other runnable tasks run first |
| `async.with_timeout(tasks, ms, run)` | `Result<T, Timeout>` `uses async` | Runs `run` with a deadline of `ms` milliseconds. When the deadline passes, it cancels `run` and returns `Err(error: Timeout.TimedOut)`. `ms > 0` |
| `async.bounded::<T>(tasks, capacity)` | `Channel<T>` `uses async` | Creates a channel with the capacity `capacity`. `capacity >= 1` |
| `async.send(to, value)` | `Result<Unit, SendError>` `uses async` | Sends. If the channel is full, it waits. If the channel is closed, the result is `Err(error: SendError.Closed)` |
| `async.receive(from)` | `Option<T>` `uses async` | Receives. If the channel is empty, it waits. If the channel is closed and empty, the result is `None` |
| `async.close(sender)` | `Unit` `uses async` | Closes the channel. Later `send` calls fail, and the receiving side reads what remains and then receives `None` |

This module also has functions and types that the `await` mechanism uses internally (`bind`, `suspend`, `join`, `erase`, `reveal`, `Any`, `Request`, and `Step`). They are not meant to be used directly.

## Preconditions

The following calls become defects when they violate their preconditions.

| Function | Precondition |
| --- | --- |
| `async.sleep(ms)` | `ms >= 0` |
| `async.with_timeout(tasks, ms, run)` | `ms > 0` |
| `async.bounded::<T>(tasks, capacity)` | `capacity >= 1` |

Awaiting a canceled task is a defect. Even when every task is waiting and nobody can wake any of them (a deadlock), the scheduler stops the program with a defect.

## Clock and order

- The scheduler’s clock advances to the earliest wake-up time **only while every task is waiting**. `sleep` does not use real time.
- `Policy.Fifo` (the default of `block_on`) runs tasks in the order in which they became runnable. `Policy.Seeded(seed: n)` chooses the next task with pseudo-random numbers determined by `n`. With the same seed, the order is the same everywhere.

## Example

```cernith
module guide.async_reference;

import std.async;

pub fn exchange() -> List<Int> {
  async.block_on(
    run: fn() -> List<Int> uses async {
      await async.scope(
        body: fn(tasks: async.Scope) -> List<Int> uses async {
          let channel = await async.bounded::<Int>(tasks: tasks, capacity: 2);
          discard await async.send(to: channel.sender, value: 7);
          await async.close(sender: channel.sender);
          var seen: List<Int> = [];
          for _ in 0..2 {
            match await async.receive(from: channel.receiver) {
              Some(value: value) => {
                seen = seen ++ [value];
              },
              None => {
                seen = seen ++ [0 - 1];
              },
            }
          }
          match await async.send(to: channel.sender, value: 8) {
            Ok(value: _) => {
              seen = seen ++ [1];
            },
            Err(error: async.SendError.Closed) => {
              seen = seen ++ [0];
            },
          }
          seen
        },
      )
    },
  )
}

example exchange_reads_what_was_sent_before_the_close for exchange {
  exchange() == [7, -1, 0]
}

pub fn nap(ms: Int) -> Unit {
  async.block_on(
    run: fn() -> Unit uses async {
      await async.sleep(ms: ms)
    },
  )
}

example sleep_rejects_a_negative_time for nap {
  defects(of: nap(ms: -1)) and not defects(of: nap(ms: 0))
}

pub fn small_channel(capacity: Int) -> Int {
  async.block_on(
    run: fn() -> Int uses async {
      await async.scope(
        body: fn(tasks: async.Scope) -> Int uses async {
          let channel = await async.bounded::<Int>(
            tasks: tasks,
            capacity: capacity,
          );
          await async.close(sender: channel.sender);
          capacity
        },
      )
    },
  )
}

example bounded_needs_a_positive_capacity for small_channel {
  defects(of: small_channel(capacity: 0)) and small_channel(capacity: 1) == 1
}
```

## Related

- [18 Concurrency](../language/18-concurrency.md): rules and examples
- [parallel](parallel.md): applies a function without effects to each element of a list
