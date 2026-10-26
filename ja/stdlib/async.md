# std.async

タスク、スコープ、チャネル、時間の待ち、そしてそれらを実行するスケジューラを提供するモジュールです。`import std.async;` と書き、`async.scope(...)`、`async.spawn(...)` のように呼びます。概念と規則（`uses async`、`await`、スコープの終わりまでにすべてのタスクを `await` する規則など）は [18 並行処理](../language/18-concurrency.md) にあります。このページは関数と型の一覧です。

`uses async` の関数の呼び出しは、`await async.sleep(ms: 10)` のように `await` を付けて書きます。例外は `spawn`、`bounded`、`close` で、これらには `await` が要りません。`spawn` は `Task` をすぐに返し、`await` を付けずに書きます。`bounded` と `close` は、`await` を付けても付けなくてもかまいません。

## 型

| 型 | 意味 |
| --- | --- |
| `async.Scope` | タスクを開始する場所。`scope` の本体が受け取る |
| `async.Task<T>` | 開始したタスク。`await` して値 `T` を得る |
| `async.Channel<T>` | `sender: Sender<T>` と `receiver: Receiver<T>` を持つレコード |
| `async.Sender<T>`、`async.Receiver<T>` | チャネルの送信側と受信側 |
| `async.Policy` | スケジューラの方針。`Policy.Fifo`、`Policy.Seeded(seed: Int)` |
| `async.SendError` | 送信の失敗。`SendError.Closed` |
| `async.Timeout` | 時間切れ。`Timeout.TimedOut` |

`Scope`、`Task`、`Sender`、`Receiver` は、スコープの外に出せません（[18 並行処理](../language/18-concurrency.md)）。

## 関数一覧

| 関数 | 戻り値の型 | 意味 |
| --- | --- | --- |
| `async.block_on(run)` | `T` | `run` と、それが開始するタスクを、先に実行可能になったものから順に実行して、`run` の値を返す。`async` を宣言しない普通の関数 |
| `async.block_on_with(policy, run)` | `T` | 方針を指定して実行する |
| `async.scope(body)` | `T` `uses async` | `body` に `Scope` を渡して実行し、そのスコープのすべてのタスクが終わったあとに `body` の値を返す |
| `async.spawn(tasks, run)` | `Task<T>` `uses async` | `run` を `tasks` のタスクとして開始する。`await` を付けずに書く |
| `async.all(tasks)` | `List<T>` `uses async` | タスクのリストの値を、リストの順序で返す |
| `async.race(tasks, candidates)` | `Option<T>` `uses async` | 最初に終わったタスクの値を `Some` で返し、残りをキャンセルする。候補が空なら `None` |
| `async.cancel(task)` | `Unit` `uses async` | タスクをキャンセルする。二度と実行されない |
| `async.sleep(ms)` | `Unit` `uses async` | スケジューラの時計で `ms` ミリ秒待つ。`ms >= 0` |
| `async.pause()` | `Unit` `uses async` | ほかの実行可能なタスクを先に走らせる |
| `async.with_timeout(tasks, ms, run)` | `Result<T, Timeout>` `uses async` | `run` を `ms` ミリ秒以内の期限で走らせる。期限を過ぎるとキャンセルして `Err(error: Timeout.TimedOut)`。`ms > 0` |
| `async.bounded::<T>(tasks, capacity)` | `Channel<T>` `uses async` | 容量 `capacity` のチャネルを作る。`capacity >= 1` |
| `async.send(to, value)` | `Result<Unit, SendError>` `uses async` | 送る。いっぱいなら待つ。閉じていれば `Err(error: SendError.Closed)` |
| `async.receive(from)` | `Option<T>` `uses async` | 受け取る。空なら待つ。閉じていて空なら `None` |
| `async.close(sender)` | `Unit` `uses async` | チャネルを閉じる。以後の `send` は失敗し、受信側は残りを読んで、そのあと `None` を受け取る |

このモジュールには、`await` の仕組みが内部で使う関数と型（`bind`、`suspend`、`join`、`erase`、`reveal`、`Any`、`Request`、`Step`）もあります。利用者が直接使うものではありません。

## 事前条件

次の呼び出しは、事前条件に違反すると欠陥になります。

| 関数 | 事前条件 |
| --- | --- |
| `async.sleep(ms)` | `ms >= 0` |
| `async.with_timeout(tasks, ms, run)` | `ms > 0` |
| `async.bounded::<T>(tasks, capacity)` | `capacity >= 1` |

キャンセルされたタスクを `await` すると欠陥です。すべてのタスクが待っていて、誰も起こせない状態（デッドロック）になっても、スケジューラが欠陥で止めます。

## 時計と順序

- スケジューラの時計は、**すべてのタスクが待っているときだけ**、最も早い起床時刻まで進みます。`sleep` は実際の時間を使いません。
- `Policy.Fifo`（`block_on` の既定）は、実行可能になった順にタスクを走らせます。`Policy.Seeded(seed: n)` は、`n` で決まる擬似乱数で、次のタスクを選びます。同じ種なら、どこでも同じ順序になります。

## 例

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

## 関連

- [18 並行処理](../language/18-concurrency.md)：規則と例
- [parallel](parallel.md)：効果のない関数をリストの各要素に適用する
