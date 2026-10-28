# std.set

This module provides `Set<T>`, a collection of unique values. Elements must satisfy `Ord`. Import it with `import std.set;` and call functions such as `set.add(...)`.

A `Set` is a value. `set.add`, `set.union`, and the other functions do not modify the original set; they return a new one. Two `Set`s with the same elements are equal under `==`, regardless of the order in which the elements were added and of whether duplicates were added.

## Types

```text
pub record Set<T: Ord> {
  members: map.Map<T, Unit>,
}
```

`Set<T>` is built as a [`Map`](map.md) whose keys are `T` and which holds no values (the values are `Unit`). Therefore, the types usable as `T` are the same as the keys of a `Map`: only `Int`, `Float64`, `String`, `Bool`, and `distinct` types and refined types built on them. Using a list or a record as the element type is error `E0316` ([03 Types](../language/03-types.md)). A `Set` is created with functions such as `set.empty` and `set.from_list`. There is no need to touch `members` directly.

## Function list

| Function | Return type | Meaning |
| --- | --- | --- |
| `set.empty()` | `Set<T>` | A set with no elements |
| `set.from_list(items)` | `Set<T>` | A set of the elements of the list. Duplicates become one |
| `set.add(set, item)` | `Set<T>` | A set with `item` added. If it already exists, the set is unchanged |
| `set.remove(set, item)` | `Set<T>` | A set with `item` removed. If it does not exist, the set is unchanged |
| `set.has(set, item)` | `Bool` | `true` if `item` exists |
| `set.size(set)` | `Int` | The number of elements |
| `set.to_list(set)` | `List<T>` | All elements, in **ascending order** |
| `set.first(set)` | `Option<T>` | The smallest element, or `None` if the set is empty |
| `set.last(set)` | `Option<T>` | The largest element, or `None` if the set is empty |
| `set.union(first, second)` | `Set<T>` | The set of elements in either set (the union) |
| `set.intersection(first, second)` | `Set<T>` | The set of elements in both sets (the intersection) |
| `set.difference(first, second)` | `Set<T>` | The set of elements in `first` that are not in `second` (the difference) |

No function has a precondition. A missing element or an empty set is represented by `None` or `false`.

`add`, `remove`, and `has` finish in roughly log n steps for n elements.

## Function details

### set.empty

```text
pub fn empty<T: Ord>() -> Set<T> {
  ensures size(set: result) == 0;
  ...
}
```

A set with no elements. When the type argument cannot be determined from the context, write it as in `set.empty::<Int>()`.

### set.from_list

```text
pub fn from_list<T: Ord>(items: List<T>) -> Set<T> {
  ensures size(set: result) <= items.length;
  ...
}
```

Builds a set from `items`, merging duplicate values. The result has at most `items.length` elements and is empty when `items` is empty.

### set.add

```text
pub fn add<T: Ord>(set: Set<T>, item: T) -> Set<T> {
  ensures size(set: result) >= size(set: set)
    and size(set: result) <= size(set: set) + 1;
  ...
}
```

Adds `item`. If `item` already exists, the set does not change. Only when it did not exist does the number of elements increase by one.

### set.remove

```text
pub fn remove<T: Ord>(set: Set<T>, item: T) -> Set<T> {
  ensures size(set: result) <= size(set: set)
    and size(set: result) + 1 >= size(set: set);
  ...
}
```

Removes `item`. If `item` does not exist, it returns a set equal to `set` (this is not a defect).

### set.has and set.size

```text
pub fn has<T: Ord>(set: Set<T>, item: T) -> Bool
pub fn size<T: Ord>(set: Set<T>) -> Int {
  ensures result >= 0;
  ...
}
```

`has` returns whether `item` is an element. `size` is the number of elements.

### set.to_list

```text
pub fn to_list<T: Ord>(set: Set<T>) -> List<T> {
  ensures result.length == size(set: set);
  ...
}
```

Makes a list of all elements in ascending order. The order is the `Ord` order (for `Int`, the numeric order, and for `String`, the lexicographic order of code points). Each element appears exactly once. To process the elements of a set one by one, turn it into a list with this function and loop over it with `for`.

### set.first and set.last

```text
pub fn first<T: Ord>(set: Set<T>) -> Option<T>
pub fn last<T: Ord>(set: Set<T>) -> Option<T>
```

`first` returns the smallest element, and `last` returns the largest element, as `Some(value: ...)`. If the set is empty, both return `None`.

### set.union, set.intersection, and set.difference

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

These combine two sets. `union` is all the elements that are in either set. `intersection` is only the elements that are in both. `difference` is those elements of `first` that are not in `second`. For `union` and `intersection`, swapping the arguments gives the same result (in the sense of `==`). For `difference`, swapping them changes the result. Each result is a new set, and the argument sets do not change.

## Example

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

An example that specifies a type that cannot be an element.

```cernith-error E0316
module guide.set_items;

import std.set;

pub fn of_lists() -> set.Set<List<Int>> {
  set.empty()
}
```

## Related

- [map](map.md): the `Map<K, V>` that `Set` is built on
- [list](list.md): the `List` that `from_list` and `to_list` handle
- [option](option.md): the `Option` that `first` and `last` return
- [03 Types](../language/03-types.md): `Eq` and `Ord`
