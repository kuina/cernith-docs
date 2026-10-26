# std.parallel

リストの各要素に関数を適用する関数のモジュールです。[list](list.md) の `map`、`filter`、`fold`、`any`、`all` と同じ結果を返しますが、渡す関数に効果を持てない代わりに、要素ごとの呼び出しを複数のワーカーで並べて実行できます。使うには `import std.parallel;` と書き、`parallel.map(items: ..., apply: ...)` のように呼びます。

## 関数一覧

| 関数 | 戻り値の型 | 意味 | `std.list` の対応する関数 |
| --- | --- | --- | --- |
| `parallel.all(items, test)` | `Bool` | すべての要素で `test` が `true` | `list.all` |
| `parallel.any(items, test)` | `Bool` | `test` が `true` の要素がある | `list.any` |
| `parallel.filter(items, keep)` | `List<T>` | `keep` が `true` の要素だけ（順序は保つ） | `list.filter` |
| `parallel.fold(items, initial, combine)` | `T` | `combine` で要素を 1 つにまとめる | `list.fold`（ただし `combine` の型が違う。下記） |
| `parallel.map(items, apply)` | `List<B>` | 各要素に `apply` を適用 | `list.map` |

`std.list` の他の関数（`find`、`count`、`flat_map` など）に、並列版はありません。

## 規則

### 結果は `std.list` の同名の関数と同じ

`parallel` の各関数の結果は、定義により、同じ引数で呼んだ `std.list` の同名の関数の結果です。結果の順序は、`items` の順序です。要素ごとの呼び出しがどの順に行われても、結果の並びは変わりません。

`std.list` の関数を、そのまま `parallel` の関数に置き換えられるとは限りません。次の 2 点で、受け取る関数の条件が違います。

### 渡す関数に効果がない

`parallel` の関数が受け取る関数（`test`、`keep`、`apply`、`combine`）の型には、`uses` がありません。`std.list` の同名の関数のように、効果多相（`effect E`）ではありません。効果のある関数（`uses io` など）を渡すと、エラー `E0404` です（[11 効果](../language/11-effects.md)）。

```cernith-error E0404
module guide.parallel_effects;

import std.parallel;

pub fn logged(items: List<Int>) -> List<Int> uses io {
  parallel.map(
    items: items,
    apply: fn(item: Int) -> Int uses io {
      item
    },
  )
}
```

効果がなければ、ある要素の結果が、ほかの要素の結果に左右されることはありません。そのため、呼び出しの順序は結果に影響せず、実行環境は要素ごとの呼び出しを複数のワーカーで並べて実行してかまいません。実行環境によっては、実際に並列に実行されます。そうでない環境では、逐次に実行されます。どちらでも結果は同じです。

`parallel` の関数自身も純粋で、呼び出し側に効果の宣言は要りません（`async` 効果も不要です。[18 並行処理](../language/18-concurrency.md)）。`parallel` の関数の署名には、`uses` がありません。

渡した関数が欠陥になる場合は、逐次に呼んだ場合と同じく、プログラムが欠陥で止まります。

### fold の `combine` は結合的で、`initial` は単位元であること

```text
pub fn fold<T>(
  items: List<T>,
  initial: T,
  combine: fn(total: T, item: T) -> T,
) -> T
```

`parallel.fold` は、`list.fold` とは型が違います。`list.fold` は、畳み込みの結果の型 `A` が要素の型 `T` と違ってかまいませんが、`parallel.fold` では、`initial`、`total`、`item`、結果がすべて同じ型 `T` です。

`combine` と `initial` には、次の条件があります。

- `combine` が結合的であること。任意の `a`、`b`、`c` について、`combine(total: combine(total: a, item: b), item: c)` と `combine(total: a, item: combine(total: b, item: c))` が等しい。
- `initial` が `combine` の単位元であること。任意の `x` について、`combine(total: initial, item: x)` と `combine(total: x, item: initial)` が、どちらも `x` に等しい。

この条件が成り立つと、要素をどのようなまとまりに分けて結合しても、結果が逐次の `list.fold` の結果と等しくなります。`parallel.fold` の結果が `list.fold` と等しいことが保証されるのは、この条件が成り立つときです。

条件を満たす例は、`+` と `0`、`*` と `1`、文字列の `++` と `""` です。満たさない例は、`total - item`（結合的でない）や、`+` に `initial: 10`（`10` は単位元でない）です。

条件が成り立つことは、ライブラリには検査できません。条件は渡す関数の性質で、呼び出し側が保証します。整数の足し算については、`property` で確かめられます（下の例の `fold_of_a_split_is_the_fold_of_the_halves`）。

## 関数の詳細

### parallel.map

```text
pub fn map<A, B>(items: List<A>, apply: fn(item: A) -> B) -> List<B>
```

各要素に `apply` を適用した結果を、同じ順序で並べたリストです。

- 事前条件：なし。
- 事後条件：`result.length == items.length`。

### parallel.filter

```text
pub fn filter<T>(items: List<T>, keep: fn(item: T) -> Bool) -> List<T>
```

`keep` が `true` を返す要素だけを、元の順序のまま残します。

- 事前条件：なし。
- 事後条件：`result.length <= items.length`。

### parallel.fold

```text
pub fn fold<T>(
  items: List<T>,
  initial: T,
  combine: fn(total: T, item: T) -> T,
) -> T
```

要素を `combine` で 1 つにまとめます。条件は上の節のとおりです。空のリストなら、`initial` が返ります。

- 事前条件：なし。ただし、`combine` と `initial` には上の条件があります。

### parallel.any と parallel.all

```text
pub fn any<T>(items: List<T>, test: fn(item: T) -> Bool) -> Bool
pub fn all<T>(items: List<T>, test: fn(item: T) -> Bool) -> Bool
```

`parallel.any` は `test` が `true` の要素が 1 つでもあれば `true`、`parallel.all` はすべての要素で `test` が `true` なら `true` です。空のリストでは、`parallel.any` は `false`、`parallel.all` は `true` です。結果は `list.any`、`list.all` と同じで、結果を決める要素より後ろの要素で `test` が欠陥になっても、欠陥にはなりません。

- 事前条件：なし。

## 例

```cernith
module guide.parallel_demo;

import std.int;
import std.list;
import std.parallel;

pub fn all_small(items: List<Int>) -> Bool {
  parallel.all(
    items: items,
    test: fn(item: Int) -> Bool {
      item < 100
    },
  )
}

example all_checks_every_item for all_small {
  all_small(items: [2, 4, 6])
    and not all_small(items: [2, 400])
    and parallel.all(
      items: [20, 0],
      test: fn(item: Int) -> Bool {
        int.div_floor(dividend: 10, divisor: item) > 1
      },
    )
      == false
}

example all_of_an_empty_list_is_true for all_small {
  all_small(items: [])
}

pub fn any_large(items: List<Int>) -> Bool {
  parallel.any(
    items: items,
    test: fn(item: Int) -> Bool {
      item >= 100
    },
  )
}

example any_looks_for_one_item for any_large {
  any_large(items: [0, 500])
    and not any_large(items: [0, 5])
    and parallel.any(
      items: [5, 0],
      test: fn(item: Int) -> Bool {
        int.div_floor(dividend: 10, divisor: item) > 1
      },
    )
}

example any_of_an_empty_list_is_false for any_large {
  not any_large(items: [])
}

pub fn big_ones(items: List<Int>) -> List<Int> {
  parallel.filter(
    items: items,
    keep: fn(item: Int) -> Bool {
      item >= 100
    },
  )
}

example filter_can_drop_everything for big_ones {
  big_ones(items: [1, 2]) == [] and big_ones(items: []) == []
}

example filter_keeps_the_order for big_ones {
  big_ones(items: [300, 1, 400, 1, 500]) == [300, 400, 500]
}

property filter_agrees_with_the_sequential_filter for big_ones(
  items: List<Int>,
) {
  big_ones(items: items)
    == list.filter(
      items: items,
      keep: fn(item: Int) -> Bool {
        item >= 100
      },
    )
}

pub fn sum_of_squares(items: List<Int>) -> Int {
  parallel.fold(
    items: parallel.map(
      items: items,
      apply: fn(item: Int) -> Int {
        item * item
      },
    ),
    initial: 0,
    combine: fn(total: Int, item: Int) -> Int {
      total + item
    },
  )
}

example fold_adds_in_any_grouping for sum_of_squares {
  sum_of_squares(items: [1, 2, 3]) == 14
    and parallel.fold(
      items: [1, 2, 3, 4],
      initial: 0,
      combine: fn(total: Int, item: Int) -> Int {
        total + item
      },
    )
      == 10
}

example fold_of_an_empty_list_is_the_initial_value for sum_of_squares {
  sum_of_squares(items: []) == 0
}

example fold_with_other_associative_functions for sum_of_squares {
  parallel.fold(
    items: [2, 3, 4],
    initial: 1,
    combine: fn(total: Int, item: Int) -> Int {
      total * item
    },
  )
    == 24
    and parallel.fold(
      items: ["a", "b", "c"],
      initial: "",
      combine: fn(total: String, item: String) -> String {
        total ++ item
      },
    )
      == "abc"
}

example map_agrees_with_the_sequential_map for sum_of_squares {
  parallel.map(
    items: [3, 1, 2],
    apply: fn(item: Int) -> Int {
      item + 1
    },
  )
    == list.map(
      items: [3, 1, 2],
      apply: fn(item: Int) -> Int {
        item + 1
      },
    )
    and parallel.map(
      items: [],
      apply: fn(item: Int) -> Int {
        item
      },
    )
      == []
}

example map_defect_ends_the_program_as_sequentially for sum_of_squares {
  defects(
    of: parallel.map(
      items: [1, 0],
      apply: fn(item: Int) -> Int {
        int.div_floor(dividend: 10, divisor: item)
      },
    ),
  )
}

example map_keeps_the_order for sum_of_squares {
  parallel.map(
    items: [1, 2, 3],
    apply: fn(item: Int) -> Int {
      item * item
    },
  )
    == [1, 4, 9]
}

property fold_of_a_split_is_the_fold_of_the_halves for sum_of_squares(
  left: List<Int>,
  right: List<Int>,
) {
  parallel.fold(
    items: left ++ right,
    initial: 0,
    combine: fn(total: Int, item: Int) -> Int {
      total + item
    },
  )
    == parallel.fold(
      items: left,
      initial: 0,
      combine: fn(total: Int, item: Int) -> Int {
        total + item
      },
    )
      + parallel.fold(
        items: right,
        initial: 0,
        combine: fn(total: Int, item: Int) -> Int {
          total + item
        },
      )
}
```

## 関連

- [list](list.md)：逐次版の `map`、`filter`、`fold`、`any`、`all`（効果多相）
- [11 効果](../language/11-effects.md)：純粋な関数と効果
- [18 並行処理](../language/18-concurrency.md)：`async` とタスク
