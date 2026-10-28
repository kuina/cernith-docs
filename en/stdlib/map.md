# std.map

This module provides `Map<K, V>`, a collection of key-value pairs. Keys must satisfy `Ord`. Import it with `import std.map;` and call functions such as `map.insert(...)`.

A `Map` is a value. `map.insert` and `map.remove` do not modify the original map; they return a new one.

## Types

```text
pub record Entry<K, V> {
  key: K,
  value: V,
}

pub record Map<K: Ord, V> {
  root: Tree<K, V>,
}

pub variant Tree<K, V> {
  Leaf,
  Node(
    left: Tree<K, V>,
    key: K,
    value: V,
    priority: Int,
    size: Int,
    right: Tree<K, V>,
  ),
}
```

- `Entry<K, V>` is a pair of a key and a value. `map.entries`, `map.first`, and `map.last` return it.
- `Map<K, V>` is the map itself. Because `K` has an `Ord` bound, the only types usable as keys are `Int`, `Float64`, `String`, `Bool`, and `distinct` types and refined types built on them ([03 Types](../language/03-types.md)). Using a list or a record as the key is error `E0316`.
- `Tree<K, V>` is the binary search tree used internally by `Map`. It is public because it is the type of `Map`'s `root` field, but users do not need to work with it directly.

A `Map` is created and read with the functions of this module (`map.empty`, `map.insert`, and so on). Assembling a `Tree` value yourself and putting it in `Map(root: ...)` can be written as far as types go. However, the functions are written on the assumption that the tree satisfies rules such as “the keys are in ascending order from left to right,” so the results of the functions on a tree that violates the rules are not defined.

Two `Map`s with the same entries (pairs of a key and a value) are equal under `==` regardless of the order in which the entries were added, because the shape of the tree is determined by the keys alone. To use `==`, the value type `V` must also support equality comparison (`Eq`).

> Note: The function `map.priority(key)` is also public, but it is a function the map uses internally to determine the shape of the tree. Users do not need to use it.

## Function list

| Function | Return type | Meaning |
| --- | --- | --- |
| `map.empty()` | `Map<K, V>` | A map with no entries |
| `map.insert(map, key, value)` | `Map<K, V>` | A map in which `key` is associated with `value`. If `key` already exists, its value is replaced |
| `map.remove(map, key)` | `Map<K, V>` | A map with `key` removed. If `key` does not exist, the map is unchanged |
| `map.get(map, key)` | `Option<V>` | The value associated with `key`, or `None` if there is none |
| `map.has(map, key)` | `Bool` | `true` if `key` exists |
| `map.size(map)` | `Int` | The number of entries |
| `map.entries(map)` | `List<Entry<K, V>>` | All entries, in **ascending order of keys** |
| `map.first(map)` | `Option<Entry<K, V>>` | The entry with the smallest key, or `None` if the map is empty |
| `map.last(map)` | `Option<Entry<K, V>>` | The entry with the largest key, or `None` if the map is empty |

No function has a precondition. A missing key or an empty map is represented by `None` of an `Option` or by `false`.

`get`, `has`, `insert`, `remove`, and `size` do not examine other entries. For n entries, they finish in roughly log n steps. The same holds for `first` and `last`, which do not build a list of the entries. Only `entries` builds a list of all entries.

## Function details

### map.empty

```text
pub fn empty<K: Ord, V>() -> Map<K, V> {
  ensures size(map: result) == 0;
  ...
}
```

A map with no entries. When the type arguments cannot be determined from the context, write it as in `map.empty::<String, Int>()` ([06 Functions](../language/06-functions.md)).

### map.insert

```text
pub fn insert<K: Ord, V>(map: Map<K, V>, key: K, value: V) -> Map<K, V> {
  ensures size(map: result) >= size(map: map)
    and size(map: result) <= size(map: map) + 1;
  ...
}
```

Associates `key` with `value`. When `key` already exists, its value is replaced with `value`, so the number of entries does not change. Only when `key` did not exist does the number of entries increase by one. The original `map` does not change.

### map.remove

```text
pub fn remove<K: Ord, V>(map: Map<K, V>, key: K) -> Map<K, V> {
  ensures size(map: result) <= size(map: map)
    and size(map: result) + 1 >= size(map: map);
  ...
}
```

Removes the entry for `key`. If `key` does not exist, it returns a map equal to `map` (this is not a defect). The number of entries in the result is the same as in `map` or one fewer.

### map.get and map.has

```text
pub fn get<K: Ord, V>(map: Map<K, V>, key: K) -> Option<V>
pub fn has<K: Ord, V>(map: Map<K, V>, key: K) -> Bool
```

`get` returns the value associated with `key` as `Some(value: ...)`, or `None` if there is none. When you do not need the value and only want to know whether the key exists, use `has`.

### map.size

```text
pub fn size<K: Ord, V>(map: Map<K, V>) -> Int {
  ensures result >= 0;
  ...
}
```

Returns the number of entries without constructing a list, unlike calling `entries` and measuring its length.

### map.entries

```text
pub fn entries<K: Ord, V>(map: Map<K, V>) -> List<Entry<K, V>>
```

Makes a list of all entries in ascending order of keys. The order is the `Ord` order: for `Int`, the numeric order, and for `String`, the lexicographic order of code points ([03 Types](../language/03-types.md)). The order in which entries were added is irrelevant. Each key appears exactly once.

### map.first and map.last

```text
pub fn first<K: Ord, V>(map: Map<K, V>) -> Option<Entry<K, V>>
pub fn last<K: Ord, V>(map: Map<K, V>) -> Option<Entry<K, V>>
```

`first` returns the entry with the smallest key, and `last` returns the entry with the largest key, as `Some(value: Entry(key: ..., value: ...))`. If the map is empty, both return `None`. They return the same entries as the first and last of `entries`, but without building the list.

## Example

```cernith
module guide.maps;

import std.map;

pub fn by_number() -> map.Map<Int, String> {
  map.insert(
    map: map.insert(
      map: map.insert(map: map.empty(), key: 10, value: "ten"),
      key: 9,
      value: "nine",
    ),
    key: 2,
    value: "two",
  )
}

example entries_are_in_key_order for by_number {
  map.entries(map: by_number())
    == [
      map.Entry(key: 2, value: "two"),
      map.Entry(key: 9, value: "nine"),
      map.Entry(key: 10, value: "ten"),
    ]
}

pub fn prices() -> map.Map<String, Int> {
  map.insert(
    map: map.insert(
      map: map.insert(map: map.empty(), key: "pear", value: 120),
      key: "apple",
      value: 80,
    ),
    key: "melon",
    value: 400,
  )
}

example empty_has_no_entries for prices {
  map.size(map: map.empty::<String, Int>()) == 0
    and map.entries(map: map.empty::<String, Int>()) == []
    and map.first(map: map.empty::<String, Int>()) == None
    and map.last(map: map.empty::<String, Int>()) == None
}

example first_and_last_are_the_ends_of_the_keys for prices {
  map.first(map: prices()) == Some(value: map.Entry(key: "apple", value: 80))
    and map.last(map: prices())
      == Some(value: map.Entry(key: "pear", value: 120))
}

example get_present_and_absent for prices {
  map.get(map: prices(), key: "pear") == Some(value: 120)
    and map.get(map: prices(), key: "fig") == None
}

example has_present_and_absent for prices {
  map.has(map: prices(), key: "melon")
    and not map.has(map: prices(), key: "fig")
}

example insert_adds_a_new_key for prices {
  map.size(map: map.insert(map: prices(), key: "fig", value: 300)) == 4
    and map.size(map: prices()) == 3
}

example insert_replaces_an_existing_key for prices {
  map.size(map: map.insert(map: prices(), key: "pear", value: 150)) == 3
    and map.get(
      map: map.insert(map: prices(), key: "pear", value: 150),
      key: "pear",
    )
      == Some(value: 150)
}

example maps_with_the_same_entries_are_equal for prices {
  prices()
    == map.insert(
      map: map.insert(
        map: map.insert(map: map.empty(), key: "melon", value: 400),
        key: "apple",
        value: 80,
      ),
      key: "pear",
      value: 120,
    )
    and prices() != map.insert(map: prices(), key: "pear", value: 150)
}

example remove_present_and_absent for prices {
  map.size(map: map.remove(map: prices(), key: "pear")) == 2
    and not map.has(map: map.remove(map: prices(), key: "pear"), key: "pear")
    and map.remove(map: prices(), key: "fig") == prices()
}

example size_counts_the_entries for prices {
  map.size(map: prices()) == 3
}
```

An example that specifies a type that cannot be used as a key.

```cernith-error E0316
module guide.map_keys;

import std.map;

pub fn by_list() -> map.Map<List<Int>, Int> {
  map.empty()
}
```

## Related

- [set](set.md): the set `Set<T>`, which has only keys. It is built on `std.map`
- [list](list.md): the `List` that `entries` returns
- [option](option.md): the `Option` that `get`, `first`, and `last` return
- [03 Types](../language/03-types.md): `Eq` and `Ord`
