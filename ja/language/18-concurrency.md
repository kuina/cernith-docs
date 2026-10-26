# 並行処理

Cernith の値は変更できず、共有された可変状態を作る構文もないので、値が競合することはありません。並行処理は 3 つの層でできていて、どれもソースに現れます。

1. **`parallel`**：純粋な関数をリストの各要素に適用する。結果は、逐次に適用した結果と同じに定義されています。
2. **構造化されたタスク**：`scope` の中でタスクを開始する。スコープは、すべてのタスクが終わってから終わります。中断しうる関数は効果 `async` を宣言し、中断する点は `await` と書きます。
3. **チャネル**：同じスコープのタスクの間の、型付きで容量のある、不変の値を運ぶ通路。

ロック、アトミック変数、切り離されたタスク、スレッドのハンドル、グローバルな可変状態はありません。時間と順序は世界から来ます。スケジューリングは入力です。

## parallel

`std.parallel` は、`map`、`filter`、`fold`、`any`、`all` を持ちます（[stdlib/parallel.md](../stdlib/parallel.md)）。結果は、同じ名前の `std.list` の関数と同じです。関数の引数に**効果を持てません**（効果のある関数を渡すと `E0404`）。計算は各要素で独立なので、処理系は複数のワーカーで、任意の順序で実行できます。欠陥が起きたら、逐次のときと同じく止まります。`async` を宣言しなくても呼べます。

## スコープとタスク

```text
await async.scope(
  body: fn(tasks: async.Scope) -> Int uses async {
    let t = async.spawn(tasks: tasks, run: fn() -> Int uses async { ... });
    await t
  },
)
```

- `async.scope(body)` は、`Scope` を渡して `body` を実行し、**そのスコープで開始されたすべてのタスクが終わったあと**に `body` の値を返します。`async` の関数なので、`await` して呼び、囲む関数は `async` を宣言します。
- `async.spawn(tasks: s, run: f)` は、`f` を `s` のタスクとして開始し、すぐに `Task<T>` を返します。タスクは、現在のタスクが待つときに実行されます。`await` を付けずに書ける、`async` の関数です。`f` はパラメータがなく、効果は囲む関数の部分集合（`async` を宣言してもよい）です。値は取り込めますが、リソース（`E1101`）、`var`（`E0322`）、`Task`（`E1202`）は取り込めません。
- **すべてのタスクは、スコープの終わりに達するすべての経路で、ちょうど 1 回 `await` されなければなりません。** `await` されない経路があると `E1201`、2 回以上 `await` する（ループの中も）と `E1204` です。`?` や `return` で抜ける経路は、`await` が要りません（スコープが待ちます）。
- `Task` は、コピー・格納・受け渡し・取り込みができず、`await` することしかできません（`E1202`）。`Task`、`Scope`、チャネルの端は、スコープの値にできません（`E1202`）。`Scope` とチャネルの端は、同じスコープのタスクが使えます。
- タスクは、同じスコープのタスクを開始でき、自分のスコープも開けます。スコープは、そのすべてのタスクが終わると終わります。
- 失敗：タスクの値は、その関数が返す値です。起こりうる失敗は `T` の中の `Result` で、`await` した側が処理します。タスクの中の欠陥は、プログラムを終わらせます（終了ステータス 70）。
- タスクはターンで実行されます。次の中断点（待つ `await`、`pause`、`sleep`、待つチャネル操作）まで走ります。プログラムは、タスクの順序に依存してはなりません。

## async 効果と await

- 中断しうる関数は `uses async` を宣言します（`async fn` という構文はありません）。`async` を持たない関数は、持つ関数を呼べません（`E0401`）。`example` と `property` は純粋なので、`async.block_on` か `test.with_world` を使います。
- **`async` の関数の呼び出しは、`await 呼び出し(...)` と書かなければなりません**（`E1203`）。呼び出しが中断点です。`Task` 以外に、値としての future はありません。`await` は `Task` にも書けます。
- `await e?` は `(await e)?` の意味です。
- `await` は、`async` を宣言する関数かラムダの中だけに書けます。中断できないものの前の `await` は `E1206` です。
- 停止性の規則は、`async` の関数でも同じです（[13 停止性](13-termination.md)）。
- `using` の本体の中には `await` を書けません（`E1207`）。読み出しを中で、待つのを外で行います。
- `async` は、効果パラメータを通して渡せません（`E1205`）。`async` の関数を `list.map` に渡さず、ループで書きます。

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

`async.block_on(run: ...)` は、タスクを持つプログラムを実行して値を返す、普通の関数です（`async` を宣言しないので、`example` や `main` から呼べます）。

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

## 時間とキャンセル

- `async.sleep(ms: n)` は、スケジューラの時計が現在の値に `n` を足した値になるまで、タスクを中断します。**時計は、すべてのタスクが待っているときだけ**、最も早い起床時刻まで進みます。実際の時間はかからず、時間を使うテストも決定的です。`sleep(ms: 0)` は `pause` です。
- `async.pause()` は、ほかの実行可能なタスクを先に走らせます。
- `async.with_timeout(tasks: s, ms: n, run: f)` は、`f` を `s` のタスクとして走らせ、時計で `n` ミリ秒以内（ちょうど `n` ミリ秒で終わる場合を含む）に終われば `Ok(value)`、終わらなければキャンセルして `Err(Timeout.TimedOut)` を返します。`n` は正です。
- `async.cancel(task: t)` は、タスクをすべての待ち行列から外し、二度と実行しません。そのタスクが開いたスコープのタスクもキャンセルされます。キャンセルされたタスクを `await` すると欠陥です。
- `async.all(tasks: ts)` は、タスクの値を、リストの順序で返します。
- `async.race(tasks: s, candidates: ts)` は、最初に終わったタスクの値を `Some` で返し、残りをキャンセルします。候補が空なら `None` です。同時に準備できたときは、スケジューラの方針が決めます。

タスクのリストは、`var` に `spawn` の結果を追加して作ります（`started = started ++ [async.spawn(...)]`）。`spawn` の結果を先に別の名前に束縛してリストに入れると、`E1202` です。`async.cancel(task: ...)` も同じです。`let` で束縛した名前を渡すと `E1202`（そのタスクを `await` していないので `E1201` も）になるので、このようなリストの要素を渡します（`await async.cancel(task: started[0])`）。

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

## チャネル

`std.async` が、タスクとチャネルの両方を提供します。

| 関数 | 型 | 説明 |
| --- | --- | --- |
| `bounded::<T>(tasks, capacity)` | `Channel<T>`（`sender: Sender<T>` と `receiver: Receiver<T>` を持つレコード）`uses async` | `capacity >= 1`。スコープの中で作り、スコープの終わりに閉じる |
| `send(to, value)` | `Result<Unit, SendError>` `uses async` | いっぱいなら待つ。閉じていれば `Err(error: SendError.Closed)` |
| `receive(from)` | `Option<T>` `uses async` | 空なら待つ。閉じていて空なら `None` |
| `close(sender)` | `Unit` `uses async` | 以後の `send` は失敗し、受信側は残りを読み出せる |

`T` は第一級の値の型です（リソースでも、スコープに束縛された値でもない）。チャネルの端は、同じスコープのタスクが使えます。送信側が 1 つ、受信側が 1 つで、タスクが純粋なら、やりとりは決定的です。すべてのタスクが待っていて、起こせるタスクがなければデッドロックで、スケジューラが欠陥で止めます。

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

待つループも、ほかのループと同じ停止性の規則に従います（[13 停止性](13-termination.md)）。ここでは、受信する回数が `count` と決まっているので、`for _ in 0..count` のように有限の範囲で書きます。

## 値の分類

| 分類 | 例 | `spawn` のラムダが取り込める | スコープの外に出せる |
| --- | --- | --- | --- |
| 第一級 | 数、レコード、ケイパビリティ | できる | できる |
| 第二級（リソース） | ファイル、ウィンドウ | できない（`E1101`） | できない（`E1101`） |
| スコープに束縛 | `Scope`、`Sender`、`Receiver` | スコープの中でできる | できない（`E1202`） |
| スコープに束縛、取り込み不可 | `Task` | できない | できない（`E1202`） |

## スケジューリングは入力

スケジューラは `std.async` の Cernith のコード（`block_on`、`block_on_with`）で、すべての実行環境で同じものが動きます。`Policy.Fifo` は、実行可能になった順にタスクを走らせます。`Policy.Seeded(seed: n)` は、`n` で決まる乱数で、次のタスクを実行可能なものから選びます。同じ種なら、どこでも同じ順序になります。

プロパティで種を入力にすると、順序に依存する誤りが見つかったとき、失敗した種が反例として報告され、`block_on_with(policy: Policy.Seeded(seed: s), ...)` で再現できます。

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

## 診断コード

| コード | 名前 | 意味 |
| --- | --- | --- |
| `E1201` | task-not-awaited | スコープの終わりに達する経路で、タスクを `await` していない |
| `E1202` | task-escapes-scope | タスク・スコープ・チャネルの端が、スコープの値になった、またはタスクを、コピー・格納・受け渡し・取り込みした |
| `E1203` | missing-await | `async` の関数の呼び出しに `await` がない |
| `E1204` | task-awaited-twice | タスクを 2 回以上（ループの中も）`await` した |
| `E1205` | async-through-effect-parameter | 効果 `async` を効果パラメータに通した |
| `E1206` | await-of-nothing | 中断できないものの前に `await` を書いた |
| `E1207` | await-in-using | `using` の本体が中断点を含む |
| `W1203` | wait-cannot-complete | チャネルの受信側を使い、送信側を一度も使わない |

ほかに、`E0401`（`await` や `async` の呼び出しを `async` なしで行った）、`E0404`（`parallel` に効果のある関数を渡した）、`E0322`（タスクが `var` を取り込んだ）、`E1101`（[17 リソースとケイパビリティ](17-resources-and-capabilities.md)）が関係します。

## 関連

- 効果：[11 効果](11-effects.md)
- `std.async` の関数：[stdlib/async.md](../stdlib/async.md)
- `std.parallel`：[stdlib/parallel.md](../stdlib/parallel.md)
