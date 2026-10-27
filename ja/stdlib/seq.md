# std.seq

必ず終わる遅延列のモジュールです。`Seq<T>` は、消費する側が要求したときにだけ要素を計算します。長さの上限 `bound` を持つので、`Seq` に対するループは必ず終わります。使うには `import std.seq;` と書き、`seq.map(source: ..., apply: ...)` のように呼びます。引数はすべてラベル付きで、列を受け取る引数のラベルは、`first` と `second` を取る `chain` と `zip` を除いて `source` です（[06 関数](../language/06-functions.md)）。

既定はやはりリストです。`Seq` は、大きい系列、必要になったときに作る系列、最初に一致した 1 つだけが欲しい場合に使います。要素 1 つにつきクロージャの呼び出しが数回かかり、ターゲットはステップを融合しません。

## 型

```text
pub record Seq<T> {
  bound: Int,
  step: fn() -> Step<T>,
}

pub variant Step<T> {
  Done,
  Yield(item: T, rest: Seq<T>),
}

pub record Indexed<T> {
  index: Int,
  item: T,
}

pub record Unfolded<S, T> {
  item: T,
  state: S,
}

pub record Zipped<A, B> {
  first: A,
  second: B,
}
```

- `Seq<T>` は `bound` と `step` です。`step()` を呼ぶと、`Step.Done` か、最初の要素と残りの列を持つ `Step.Yield(item: ..., rest: ...)` が返ります。消費する側が取り出すまで、何も計算されません。
- `Indexed<T>`、`Unfolded<S, T>`、`Zipped<A, B>` は、`enumerate`、`unfold`、`zip` の結果のレコードです。モジュール名を付けて `seq.Unfolded(item: ..., state: ...)` のように書きます。
- `Seq` はクロージャを持つので、`==` では比較できません（`E0316`）。

## 関数一覧

| 関数 | 戻り値の型 | 意味 | 結果の `bound` |
| --- | --- | --- | --- |
| `seq.empty()` | `Seq<T>` | 要素なし | `0` |
| `seq.from_list(items)` | `Seq<T>` | リストの要素を順に | リストの長さ |
| `seq.range(from, to)` | `Seq<Int>` | `from`（含む）から `to`（含まない）までの整数 | `to - from`。負なら `0` |
| `seq.repeat(item, count)` | `Seq<T>` | `item` を `count` 個 | `count`。負なら `0` |
| `seq.iterate(seed, next, bound)` | `Seq<T>` | `seed`、その `next`、さらにその `next`、…… | `bound` |
| `seq.unfold(seed, bound, next)` | `Seq<T>` | 状態から要素を作る。`next` が `None` を返すと止まる | `bound` |
| `seq.map(source, apply)` | `Seq<B>` | 各要素に `apply` を適用する | `source` のもの |
| `seq.filter(source, keep)` | `Seq<T>` | `keep` が `true` の要素 | `source` のもの |
| `seq.enumerate(source)` | `Seq<Indexed<T>>` | 各要素に 0 からの位置を付ける | `source` のもの |
| `seq.take_while(source, test)` | `Seq<T>` | `test` を満たさない最初の要素より前の要素 | `source` のもの |
| `seq.zip(first, second)` | `Seq<Zipped<A, B>>` | 同じ位置の要素の組 | 2 つのうち小さいほう |
| `seq.chain(first, second)` | `Seq<T>` | 最初の列、続いて 2 番目の列 | 合計 |
| `seq.take(source, count)` | `Seq<T>` | 最初の `count` 個 | `count` と `source` のもののうち小さいほう |
| `seq.skip(source, count)` | `Seq<T>` | 最初の `count` 個を除いたもの | `source` のものから `count` を引いた値。`0` を下回らない |
| `seq.fold(source, initial, combine)` | `A` | 先頭から順に畳み込む | |
| `seq.to_list(source)` | `List<T>` | すべての要素をリストにする | |
| `seq.count(source)` | `Int` | 要素の数 | |
| `seq.first(source)` | `Option<T>` | 最初の要素 | |
| `seq.last(source)` | `Option<T>` | 最後の要素 | |
| `seq.find(source, test)` | `Option<T>` | `test` が `true` の最初の要素 | |
| `seq.any(source, test)` | `Bool` | `test` を満たす要素がある（空なら `false`） | |
| `seq.all(source, test)` | `Bool` | すべての要素が `test` を満たす（空なら `true`） | |

## 共通の規則

### bound

列の `bound` は、長さの上限です。消費する側は、最大でも `bound` ステップしか進まず、`step` がそれ以上を返しうる場合でも、`bound` を超えた要素は読まれません。そのため、`Seq` に対するループは `decreases` 節なしで終わります。

組み合わせる関数は、構成上正しい `bound` を保ちます。表のとおりです。`bound` は上限にすぎません。100 万個の要素に `seq.filter` を適用した結果の `bound` は、1 つしか通らなくても 100 万です。

自身には終わりのない系列（`1, 2, 4, 8, ...`）は、`seq.iterate` か `seq.repeat` で書き、`bound` は書く側が決めます。大きな `bound` でも、消費する側が要素を求めるまでは何もかかりません。`seq.take_while` と `seq.find` は早く止まり、残りは計算されません。

### 関数は純粋

組み合わせる関数や消費する関数が受け取る関数（`apply`、`keep`、`test`、`next`、`combine`）は、効果を持てません。効果のある関数は `E0404` です。ステップは効果を持たないので、読む人に見えない時点で効果が実行されることはありません。

### 事前条件はない

どの関数にも事前条件はなく、自分の引数のせいで欠陥になることもありません。`count` や `bound` が負の場合と、`to` が `from` より大きくない `range` は、空の列になります。

### for

`for item in source { ... }` は、`Seq` をたどります。最大で `source.bound` 回だけ繰り返し、`invariant` 節も `decreases` 節も取れません（`E1406`）。[05 文と束縛](../language/05-statements.md)を見てください。

## 関数の詳細

### 作る

```text
pub fn empty<T>() -> Seq<T>
pub fn from_list<T>(items: List<T>) -> Seq<T>
pub fn range(from: Int, to: Int) -> Seq<Int>
pub fn repeat<T>(item: T, count: Int) -> Seq<T>
```

`seq.range(from: 2, to: 5)` は列 `2, 3, 4` で、`seq.range(from: 5, to: 2)` は空です。`seq.repeat(item: "a", count: 3)` は `"a", "a", "a"` です。`seq.empty::<Int>()` は、`T` が他から決まらない場所では、`[]` と同じように型引数を取ります（[06 関数](../language/06-functions.md)）。

```text
pub fn iterate<T>(seed: T, next: fn(value: T) -> T, bound: Int) -> Seq<T>
pub fn unfold<S, T>(
  seed: S,
  bound: Int,
  next: fn(state: S) -> Option<Unfolded<S, T>>,
) -> Seq<T>
```

`seq.iterate(seed: 1, next: ..., bound: 5)` は、`next` が 2 倍にするなら `1, 2, 4, 8, 16` です。`seq.unfold` は、状態から要素を 1 つずつ作ります。`next` は、要素と次の状態を持つ `Some(value: seq.Unfolded(item: ..., state: ...))` か、止めるための `None` を返します。作られる要素は最大で `bound` 個です。

### 組み合わせる

```text
pub fn map<A, B>(source: Seq<A>, apply: fn(value: A) -> B) -> Seq<B>
pub fn filter<T>(source: Seq<T>, keep: fn(item: T) -> Bool) -> Seq<T>
pub fn enumerate<T>(source: Seq<T>) -> Seq<Indexed<T>>
pub fn take_while<T>(source: Seq<T>, test: fn(item: T) -> Bool) -> Seq<T>
```

どれも、`source` の `bound` を持つ遅延列を返します。`seq.enumerate` は `seq.Indexed(index: 0, item: ...)`、`seq.Indexed(index: 1, item: ...)`、…… を返します。`seq.take_while` は、`test` を満たさない最初の要素で止まり、あとの要素が満たしていても続けません。

```text
pub fn zip<A, B>(first: Seq<A>, second: Seq<B>) -> Seq<Zipped<A, B>>
pub fn chain<T>(first: Seq<T>, second: Seq<T>) -> Seq<T>
pub fn take<T>(source: Seq<T>, count: Int) -> Seq<T>
pub fn skip<T>(source: Seq<T>, count: Int) -> Seq<T>
```

`seq.zip` は短いほうの列で止まり、`seq.Zipped(first: ..., second: ...)` を返します。`seq.chain` は `first` のすべて、続いて `second` のすべてを返します。`seq.take(source: s, count: 3)` は最初の 3 個で、足りなければすべてです。`seq.skip(source: s, count: 3)` は最初の 3 個を除きます。足りなければ、結果は空です。

### 消費する

```text
pub fn fold<T, A>(
  source: Seq<T>,
  initial: A,
  combine: fn(total: A, item: T) -> A,
) -> A
pub fn to_list<T>(source: Seq<T>) -> List<T>
pub fn count<T>(source: Seq<T>) -> Int
pub fn first<T>(source: Seq<T>) -> Option<T>
pub fn last<T>(source: Seq<T>) -> Option<T>
```

`seq.fold` は、`list.fold` と同じように、先頭から順に畳み込みます。`seq.first` は最初のステップだけを求めます。`seq.last`、`seq.count`、`seq.to_list`、`seq.fold` は、`bound` までのすべての要素をたどります。

```text
pub fn find<T>(source: Seq<T>, test: fn(item: T) -> Bool) -> Option<T>
pub fn any<T>(source: Seq<T>, test: fn(item: T) -> Bool) -> Bool
pub fn all<T>(source: Seq<T>, test: fn(item: T) -> Bool) -> Bool
```

結果を決める要素で止まり、残りは計算されません。`seq.find(source: seq.range(from: 0, to: 1000000000), test: ...)` は、早い要素が一致すれば、10 億個をたどることはありません。

## 例

次のモジュールは、ほとんどの関数を使います。`first_square_above` と `powers_of_two_below` は、読む部分よりはるかに長い系列を扱います。

```cernith
module guide.sequences;

import std.seq;

pub fn checks(items: List<Int>) -> Bool {
  requires items.length > 0;

  let source = seq.from_list(items: items);
  seq.all(
    source: source,
    test: fn(item: Int) -> Bool {
      item > 0
    },
  )
    and seq.any(
      source: source,
      test: fn(item: Int) -> Bool {
        item > 5
      },
    )
    and seq.first(source: source) == Some(value: items[0])
    and seq.last(source: source) == Some(value: items[items.length - 1])
}

example checks_runs_the_consumers for checks {
  checks(items: [1, 7, 3])
}

pub fn countdown(start: Int) -> List<Int> {
  seq.to_list(
    source: seq.unfold(
      seed: start,
      bound: 100,
      next: fn(state: Int) -> Option<seq.Unfolded<Int, Int>> {
        if state <= 0 {
          None
        } else {
          Some(value: seq.Unfolded(item: state, state: state - 3))
        }
      },
    ),
  )
}

example countdown_stops_at_none for countdown {
  countdown(start: 10) == [10, 7, 4, 1] and countdown(start: 0) == []
}

pub fn first_square_above(limit: Int) -> Option<Int> {
  seq.find(
    source: seq.map(
      source: seq.range(from: 1, to: 1000000000),
      apply: fn(value: Int) -> Int {
        value * value
      },
    ),
    test: fn(item: Int) -> Bool {
      item > limit
    },
  )
}

example first_square_above_stops_early for first_square_above {
  first_square_above(limit: 50) == Some(value: 64)
}

pub fn large_ones(items: List<Int>, limit: Int) -> List<Int> {
  seq.to_list(
    source: seq.filter(
      source: seq.from_list(items: items),
      keep: fn(item: Int) -> Bool {
        item > limit
      },
    ),
  )
}

example large_ones_keeps_the_items_above for large_ones {
  large_ones(items: [3, 9, 1, 7], limit: 5) == [9, 7]
}

pub fn middle(count: Int) -> List<Int> {
  seq.to_list(
    source: seq.skip(
      source: seq.chain(
        first: seq.range(from: 0, to: count),
        second: seq.repeat(item: 9, count: 2),
      ),
      count: 1,
    ),
  )
}

example middle_chains_and_skips for middle {
  middle(count: 3) == [1, 2, 9, 9]
}

pub fn pair_sums(left: List<Int>, right: List<Int>) -> Int {
  var total: Int = 0;
  for pair in seq.zip(
    first: seq.from_list(items: left),
    second: seq.from_list(items: right),
  ) {
    total = total + pair.first * pair.second;
  }
  total
}

example pair_sums_stops_at_the_shorter for pair_sums {
  pair_sums(left: [1, 2, 3], right: [4, 5]) == 14
}

pub fn powers_of_two_below(limit: Int) -> List<Int> {
  seq.to_list(
    source: seq.take_while(
      source: seq.iterate(
        seed: 1,
        next: fn(value: Int) -> Int {
          value * 2
        },
        bound: 1000000000,
      ),
      test: fn(item: Int) -> Bool {
        item < limit
      },
    ),
  )
}

example powers_of_two_below_cuts_the_series for powers_of_two_below {
  powers_of_two_below(limit: 40) == [1, 2, 4, 8, 16, 32]
}

pub fn summary(items: List<Int>) -> Int {
  let source = seq.from_list(items: items);
  seq.count(source: source)
    + seq.fold(
      source: source,
      initial: 0,
      combine: fn(total: Int, item: Int) -> Int {
        total + item
      },
    )
}

example summary_counts_and_adds for summary {
  summary(items: [5, 6]) == 13
}

pub fn weighted_total(items: List<Int>) -> Int {
  var total: Int = 0;
  for entry in seq.enumerate(source: seq.from_list(items: items)) {
    total = total + entry.index * entry.item;
  }
  total
}

example weighted_total_uses_positions for weighted_total {
  weighted_total(items: [5, 6, 7]) == 20 and weighted_total(items: []) == 0
}
```

## 関連

- [list](list.md)：`List<T>` の関数。`seq.from_list` と `seq.to_list` で相互に変換する
- [05 文と束縛](../language/05-statements.md)：`Seq` に対する `for`
- [06 関数](../language/06-functions.md)：関数値と型引数
- [11 効果](../language/11-effects.md)：列の関数が純粋である理由
- [13 停止性](../language/13-termination.md)：`Seq` に対する `for` に `decreases` が要らない理由
