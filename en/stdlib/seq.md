# std.seq

This module provides finite lazy sequences. A `Seq<T>` computes elements only when requested and carries an upper bound on its length, `bound`, so iteration always terminates. Import it with `import std.seq;` and call functions such as `seq.map(source: ..., apply: ...)`. All arguments are labeled. Sequence arguments use `source`, except `chain` and `zip`, which use `first` and `second` ([06 Functions](../language/06-functions.md)).

Use a list for most sequences. Choose `Seq` for large or on-demand data, or when you only need the first match. Each element requires a few closure calls; targets do not fuse these steps.

## Types

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

- `Seq<T>` is a `bound` and a `step`. Calling `step()` gives `Step.Done`, or `Step.Yield(item: ..., rest: ...)` with the first item and the rest of the sequence. Nothing is computed until a consumer pulls.
- `Indexed<T>`, `Unfolded<S, T>`, and `Zipped<A, B>` are the result records of `enumerate`, `unfold`, and `zip`. Write them with the module name: `seq.Unfolded(item: ..., state: ...)`.
- A `Seq` holds closures, so it is not compared with `==` (`E0316`).

## Function list

| Function | Return type | Meaning | Bound of the result |
| --- | --- | --- | --- |
| `seq.empty()` | `Seq<T>` | No items | `0` |
| `seq.from_list(items)` | `Seq<T>` | The items of a list, in order | The length of the list |
| `seq.range(from, to)` | `Seq<Int>` | The integers from `from` (inclusive) to `to` (exclusive) | `to - from`, or `0` if that is negative |
| `seq.repeat(item, count)` | `Seq<T>` | `count` copies of `item` | `count`, or `0` if negative |
| `seq.iterate(seed, next, bound)` | `Seq<T>` | `seed`, then `next` of it, and so on | `bound` |
| `seq.unfold(seed, bound, next)` | `Seq<T>` | Items made from a state; `next` returns `None` to stop | `bound` |
| `seq.map(source, apply)` | `Seq<B>` | Applies `apply` to each item | That of `source` |
| `seq.filter(source, keep)` | `Seq<T>` | The items for which `keep` is `true` | That of `source` |
| `seq.enumerate(source)` | `Seq<Indexed<T>>` | Each item with its position from 0 | That of `source` |
| `seq.take_while(source, test)` | `Seq<T>` | The items before the first one that fails `test` | That of `source` |
| `seq.zip(first, second)` | `Seq<Zipped<A, B>>` | Pairs of the items at the same position | The smaller of the two |
| `seq.chain(first, second)` | `Seq<T>` | The first sequence, then the second | The sum |
| `seq.take(source, count)` | `Seq<T>` | The first `count` items | The smaller of `count` and that of `source` |
| `seq.skip(source, count)` | `Seq<T>` | Without the first `count` items | That of `source` less `count`, not below `0` |
| `seq.fold(source, initial, combine)` | `A` | Folds in order from the start | |
| `seq.to_list(source)` | `List<T>` | All the items as a list | |
| `seq.count(source)` | `Int` | The number of items | |
| `seq.first(source)` | `Option<T>` | The first item | |
| `seq.last(source)` | `Option<T>` | The last item | |
| `seq.find(source, test)` | `Option<T>` | The first item for which `test` is `true` | |
| `seq.any(source, test)` | `Bool` | Some item passes `test` (`false` for an empty sequence) | |
| `seq.all(source, test)` | `Bool` | Every item passes `test` (`true` for an empty sequence) | |

## Common rules

### Bounds

The `bound` of a sequence is an upper bound on its length. A consumer takes at most `bound` steps, and an item past the bound is never read, even if `step` would give more. This is why a loop over a `Seq` ends without a `decreases` clause.

A combinator keeps a bound that is true by construction, as the table shows. The bound is only an upper bound: a `seq.filter` over a million items has the bound one million even if one item passes.

A series with no end of its own (`1, 2, 4, 8, ...`) is written with `seq.iterate` or `seq.repeat`, and the writer states the bound. A large bound costs nothing until a consumer asks for the items. `seq.take_while` and `seq.find` stop early, and the rest is not computed.

### Functions are pure

Callbacks passed to combinators and consumers (`apply`, `keep`, `test`, `next`, `combine`) must be pure; effectful callbacks produce `E0404`. This prevents lazy evaluation from triggering effects at unexpected points.

### No preconditions

None of the functions has a precondition, and none of them causes a defect for its own arguments. A negative `count` or `bound`, and a `range` whose `to` is not above `from`, give an empty sequence.

### for

`for item in source { ... }` traverses a `Seq`. It takes at most `source.bound` turns and takes no `invariant` or `decreases` clause (`E1406`). See [05 Statements and bindings](../language/05-statements.md).

## Function details

### Producers

```text
pub fn empty<T>() -> Seq<T>
pub fn from_list<T>(items: List<T>) -> Seq<T>
pub fn range(from: Int, to: Int) -> Seq<Int>
pub fn repeat<T>(item: T, count: Int) -> Seq<T>
```

`seq.range(from: 2, to: 5)` is the sequence `2, 3, 4`, and `seq.range(from: 5, to: 2)` is empty. `seq.repeat(item: "a", count: 3)` is `"a", "a", "a"`. `seq.empty::<Int>()` takes a type argument where nothing else decides `T`, as `[]` does ([06 Functions](../language/06-functions.md)).

```text
pub fn iterate<T>(seed: T, next: fn(value: T) -> T, bound: Int) -> Seq<T>
pub fn unfold<S, T>(
  seed: S,
  bound: Int,
  next: fn(state: S) -> Option<Unfolded<S, T>>,
) -> Seq<T>
```

`seq.iterate(seed: 1, next: ..., bound: 5)` with `next` doubling is `1, 2, 4, 8, 16`. `seq.unfold` makes each item from a state: `next` returns `Some(value: seq.Unfolded(item: ..., state: ...))` with the item and the next state, or `None` to stop. At most `bound` items are made.

### Combinators

```text
pub fn map<A, B>(source: Seq<A>, apply: fn(value: A) -> B) -> Seq<B>
pub fn filter<T>(source: Seq<T>, keep: fn(item: T) -> Bool) -> Seq<T>
pub fn enumerate<T>(source: Seq<T>) -> Seq<Indexed<T>>
pub fn take_while<T>(source: Seq<T>, test: fn(item: T) -> Bool) -> Seq<T>
```

Each returns a lazy sequence with the bound of `source`. `seq.enumerate` gives `seq.Indexed(index: 0, item: ...)`, `seq.Indexed(index: 1, item: ...)`, and so on. `seq.take_while` stops at the first item that fails `test`, even if later items pass.

```text
pub fn zip<A, B>(first: Seq<A>, second: Seq<B>) -> Seq<Zipped<A, B>>
pub fn chain<T>(first: Seq<T>, second: Seq<T>) -> Seq<T>
pub fn take<T>(source: Seq<T>, count: Int) -> Seq<T>
pub fn skip<T>(source: Seq<T>, count: Int) -> Seq<T>
```

`seq.zip` stops at the shorter sequence and gives `seq.Zipped(first: ..., second: ...)`. `seq.chain` gives all of `first`, then all of `second`. `seq.take(source: s, count: 3)` is the first three items, or all of them if there are fewer. `seq.skip(source: s, count: 3)` drops the first three; if there are fewer, the result is empty.

### Consumers

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

`seq.fold` combines the items in order from the start, as `list.fold` does. `seq.first` only asks for the first step. `seq.last`, `seq.count`, `seq.to_list`, and `seq.fold` walk every item, up to the bound.

```text
pub fn find<T>(source: Seq<T>, test: fn(item: T) -> Bool) -> Option<T>
pub fn any<T>(source: Seq<T>, test: fn(item: T) -> Bool) -> Bool
pub fn all<T>(source: Seq<T>, test: fn(item: T) -> Bool) -> Bool
```

They stop at the item that decides the result, and the rest is not computed. `seq.find(source: seq.range(from: 0, to: 1000000000), test: ...)` does not walk a billion items if an early item passes.

## Example

The following module uses most of the functions. `first_square_above` and `powers_of_two_below` work on series that are far longer than the part they read.

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

## Related

- [list](list.md): the functions for `List<T>`; `seq.from_list` and `seq.to_list` convert between the two
- [05 Statements and bindings](../language/05-statements.md): `for` over a `Seq`
- [06 Functions](../language/06-functions.md): function values and type arguments
- [11 Effects](../language/11-effects.md): why the functions of a sequence are pure
- [13 Termination](../language/13-termination.md): why a `for` over a `Seq` needs no `decreases`
