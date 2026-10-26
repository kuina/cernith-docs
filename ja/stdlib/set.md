# std.set

重複のない値の集まり `Set<T>` のモジュールです。要素の型 `T` は順序のある型（`Ord`）でなければなりません。このモジュールの関数を使うには、`import std.set;` と書き、`set.add(...)` のように呼びます。

`Set` は値です。`set.add` や `set.union` などは元の集合を変更せず、新しい集合を返します。同じ要素を持つ 2 つの `Set` は、追加した順序や重複の有無によらず `==` で等しくなります。

## 型

```text
pub record Set<T: Ord> {
  members: map.Map<T, Unit>,
}
```

`Set<T>` は、`T` をキーとし、値を持たない（`Unit` の）[`Map`](map.md) として作られています。そのため、`T` に使える型は `Map` のキーと同じです。`Int`、`Float64`、`String`、`Bool` と、それらを土台にした `distinct` 型・制約付き型だけで、リストやレコードを要素にするとエラー `E0316` です（[03 型](../language/03-types.md)）。`Set` は、`set.empty` や `set.from_list` などの関数で作ります。`members` を直接さわる必要はありません。

## 関数一覧

| 関数 | 戻り値の型 | 意味 |
| --- | --- | --- |
| `set.empty()` | `Set<T>` | 要素のない集合 |
| `set.from_list(items)` | `Set<T>` | リストの要素を集めた集合。重複は 1 つになる |
| `set.add(set, item)` | `Set<T>` | `item` を加えた集合。既にあればそのまま |
| `set.remove(set, item)` | `Set<T>` | `item` を除いた集合。無ければそのまま |
| `set.has(set, item)` | `Bool` | `item` があれば `true` |
| `set.size(set)` | `Int` | 要素の数 |
| `set.to_list(set)` | `List<T>` | すべての要素。**昇順** |
| `set.first(set)` | `Option<T>` | 最小の要素。空なら `None` |
| `set.last(set)` | `Option<T>` | 最大の要素。空なら `None` |
| `set.union(first, second)` | `Set<T>` | どちらかにある要素の集合（和集合） |
| `set.intersection(first, second)` | `Set<T>` | 両方にある要素の集合（共通部分） |
| `set.difference(first, second)` | `Set<T>` | `first` にあって `second` に無い要素の集合（差集合） |

どの関数にも事前条件はありません。要素が無い場合や空の集合は、`None` や `false` で表します。

`add`、`remove`、`has` は、要素数 n に対して、おおよそ log n 回の手順で済みます。

## 関数の詳細

### set.empty

```text
pub fn empty<T: Ord>() -> Set<T> {
  ensures size(set: result) == 0;
  ...
}
```

要素のない集合です。型引数が文脈から決まらないときは、`set.empty::<Int>()` のように書きます。

### set.from_list

```text
pub fn from_list<T: Ord>(items: List<T>) -> Set<T> {
  ensures size(set: result) <= items.length;
  ...
}
```

`items` の要素を集めた集合です。同じ値が複数あっても 1 つにまとまるので、要素数は `items.length` 以下です。`items` が空なら空の集合です。

### set.add

```text
pub fn add<T: Ord>(set: Set<T>, item: T) -> Set<T> {
  ensures size(set: result) >= size(set: set)
    and size(set: result) <= size(set: set) + 1;
  ...
}
```

`item` を加えます。`item` が既にあれば、集合は変わりません。無かったときだけ、要素数が 1 増えます。

### set.remove

```text
pub fn remove<T: Ord>(set: Set<T>, item: T) -> Set<T> {
  ensures size(set: result) <= size(set: set)
    and size(set: result) + 1 >= size(set: set);
  ...
}
```

`item` を除きます。`item` が無ければ、`set` と等しい集合を返します（欠陥にはなりません）。

### set.has と set.size

```text
pub fn has<T: Ord>(set: Set<T>, item: T) -> Bool
pub fn size<T: Ord>(set: Set<T>) -> Int {
  ensures result >= 0;
  ...
}
```

`has` は `item` が要素にあるかどうかを返します。`size` は要素の数です。

### set.to_list

```text
pub fn to_list<T: Ord>(set: Set<T>) -> List<T> {
  ensures result.length == size(set: set);
  ...
}
```

すべての要素を、昇順に並べたリストにします。順序は `Ord` の順序です（`Int` なら数の大小、`String` ならコードポイントの辞書式順序）。各要素は 1 回だけ現れます。集合の要素を順に処理したいときは、これでリストにして `for` で回します。

### set.first と set.last

```text
pub fn first<T: Ord>(set: Set<T>) -> Option<T>
pub fn last<T: Ord>(set: Set<T>) -> Option<T>
```

`first` は最小の要素、`last` は最大の要素を、`Some(value: ...)` で返します。集合が空なら、どちらも `None` です。

### set.union、set.intersection、set.difference

```text
pub fn union<T: Ord>(first: Set<T>, second: Set<T>) -> Set<T> {
  ensures size(set: result) >= size(set: first);
  ...
}

pub fn intersection<T: Ord>(first: Set<T>, second: Set<T>) -> Set<T> {
  ensures size(set: result) <= size(set: first);
  ...
}

pub fn difference<T: Ord>(first: Set<T>, second: Set<T>) -> Set<T> {
  ensures size(set: result) <= size(set: first);
  ...
}
```

2 つの集合を組み合わせます。`union` は、どちらかにある要素すべてです。`intersection` は、両方にある要素だけです。`difference` は、`first` の要素のうち `second` に無いものです。`union` と `intersection` では、引数を入れ替えても結果は（`==` の意味で）同じです。`difference` は、入れ替えると結果が変わります。結果はどれも新しい集合で、引数の集合は変わりません。

## 例

```cernith
module guide.sets;

import std.set;

pub fn count_distinct(items: List<Int>) -> Int {
  set.size(set: set.from_list(items: items))
}

example from_list_drops_repeats for count_distinct {
  count_distinct(items: [3, 1, 3, 2, 1]) == 3 and count_distinct(items: []) == 0
}

pub fn small_primes() -> set.Set<Int> {
  set.from_list(items: [7, 2, 3, 5, 3])
}

example add_keeps_the_items_sorted_and_unique for small_primes {
  set.to_list(set: set.add(set: small_primes(), item: 4)) == [2, 3, 4, 5, 7]
    and set.add(set: small_primes(), item: 3) == small_primes()
}

example difference_keeps_what_only_the_first_has for small_primes {
  set.to_list(
    set: set.difference(
      first: small_primes(),
      second: set.from_list(items: [3, 4, 5]),
    ),
  )
    == [2, 7]
    and set.size(
      set: set.difference(first: set.empty::<Int>(), second: small_primes()),
    )
      == 0
    and set.to_list(
      set: set.difference(
        first: set.from_list(items: [3, 4, 5]),
        second: small_primes(),
      ),
    )
      == [4]
}

example empty_has_no_items for small_primes {
  set.size(set: set.empty::<Int>()) == 0
    and not set.has(set: set.empty::<Int>(), item: 2)
    and set.to_list(set: set.empty::<Int>()) == []
}

example first_is_the_smallest_member for small_primes {
  set.first(set: small_primes()) == Some(value: 2)
    and set.first(set: set.empty::<Int>()) == None
}

example has_finds_members_only for small_primes {
  set.has(set: small_primes(), item: 5)
    and not set.has(set: small_primes(), item: 4)
}

example intersection_keeps_what_both_have for small_primes {
  set.to_list(
    set: set.intersection(
      first: small_primes(),
      second: set.from_list(items: [3, 4, 5, 6]),
    ),
  )
    == [3, 5]
    and set.size(
      set: set.intersection(first: small_primes(), second: set.empty::<Int>()),
    )
      == 0
    and set.intersection(
      first: small_primes(),
      second: set.from_list(items: [3, 4, 5, 6]),
    )
      == set.intersection(
        first: set.from_list(items: [3, 4, 5, 6]),
        second: small_primes(),
      )
}

example last_is_the_largest_member for small_primes {
  set.last(set: small_primes()) == Some(value: 7)
    and set.last(set: set.empty::<Int>()) == None
}

example remove_present_and_absent for small_primes {
  set.to_list(set: set.remove(set: small_primes(), item: 3)) == [2, 5, 7]
    and set.remove(set: small_primes(), item: 4) == small_primes()
}

example sets_with_the_same_members_are_equal for small_primes {
  small_primes() == set.from_list(items: [2, 3, 5, 7])
    and small_primes() != set.from_list(items: [2, 3, 5])
}

example size_counts_the_members for small_primes {
  set.size(set: small_primes()) == 4
}

example to_list_is_in_order for small_primes {
  set.to_list(set: small_primes()) == [2, 3, 5, 7]
    and set.to_list(set: set.from_list(items: ["b", "c", "a"]))
      == ["a", "b", "c"]
}

example union_has_what_either_has for small_primes {
  set.to_list(
    set: set.union(
      first: small_primes(),
      second: set.from_list(items: [1, 2, 4]),
    ),
  )
    == [1, 2, 3, 4, 5, 7]
    and set.union(first: small_primes(), second: set.empty::<Int>())
      == small_primes()
    and set.union(first: small_primes(), second: set.from_list(items: [1, 2]))
      == set.union(first: set.from_list(items: [1, 2]), second: small_primes())
}
```

要素にできない型を指定した例です。

```cernith-error E0316
module guide.set_items;

import std.set;

pub fn of_lists() -> set.Set<List<Int>> {
  set.empty()
}
```

## 関連

- [map](map.md)：`Set` の土台になっている `Map<K, V>`
- [list](list.md)：`from_list`、`to_list` が扱う `List`
- [option](option.md)：`first`、`last` が返す `Option`
- [03 型](../language/03-types.md)：`Eq` と `Ord`
