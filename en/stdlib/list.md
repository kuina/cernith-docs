# std.list

The module of functions for working with `List<T>`. A `List<T>` is an immutable sequence, and every function in this module returns a new list or a value without changing the list passed in. To use it, write `import std.list;` and call functions as in `list.map(items: ..., apply: ...)`. All arguments are labeled, and the label of the argument that receives a list is `items`, except for `flatten`, which takes `lists`, and `zip`, which takes `first` and `second` ([06 Functions](../language/06-functions.md)).

The length and indexing of a list (`items.length`, `items[i]`) are built-in features of the language, not functions of the module ([03 Types](../language/03-types.md)). This page covers the other operations. The module also defines a record type, `Pair<A, B>` (used by [list.zip](#listzip-and-listpair)).

## Function list

| Function | Return type | Meaning | Precondition |
| --- | --- | --- | --- |
| `list.all(items, test)` | `Bool` | `test` is `true` for every element (`true` for an empty list) | None |
| `list.any(items, test)` | `Bool` | Some element has `test` equal to `true` (`false` for an empty list) | None |
| `list.contains(items, item)` | `Bool` | Some element is equal to `item` | None |
| `list.count(items, test)` | `Int` | The number of elements for which `test` is `true` | None |
| `list.drop(items, count)` | `List<T>` | The rest after removing the first `count` elements | `count >= 0 and count <= items.length` |
| `list.drop_while(items, test)` | `List<T>` | The rest after removing the leading elements for which `test` is `true` | None |
| `list.filter(items, keep)` | `List<T>` | Only the elements for which `keep` is `true` (order is preserved) | None |
| `list.find(items, test)` | `Option<T>` | The first element for which `test` is `true` | None |
| `list.first(items)` | `Option<T>` | The first element | None |
| `list.flat_map(items, apply)` | `List<B>` | Applies `apply`, which returns a list, to each element and concatenates the results | None |
| `list.flatten(lists)` | `List<T>` | Concatenates a list of lists | None |
| `list.fold(items, initial, combine)` | `A` | Folds in order from the start | None |
| `list.index_of(items, item)` | `Option<Int>` | The position of the first element equal to `item` | None |
| `list.last(items)` | `Option<T>` | The last element | None |
| `list.map(items, apply)` | `List<B>` | Applies `apply` to each element | None |
| `list.max(items)` | `Option<T>` | The largest element | None |
| `list.min(items)` | `Option<T>` | The smallest element | None |
| `list.range(from, to)` | `List<Int>` | The integers from `from` (inclusive) to `to` (exclusive) | `from <= to` |
| `list.repeat(item, count)` | `List<T>` | A list of `count` copies of `item` | `count >= 0` |
| `list.reverse(items)` | `List<T>` | The reverse order | None |
| `list.slice(items, from, to)` | `List<T>` | The part from position `from` (inclusive) to `to` (exclusive) | `0 <= from and from <= to and to <= items.length` |
| `list.sort(items)` | `List<T>` | Sorts in ascending order | None |
| `list.sort_by(items, less_than)` | `List<T>` | Sorts in the order given by `less_than` (stable) | None |
| `list.sum(items)` | `Int` | The sum (`0` for an empty list) | None |
| `list.take(items, count)` | `List<T>` | The first `count` elements | `count >= 0 and count <= items.length` |
| `list.take_while(items, test)` | `List<T>` | The leading elements for which `test` is `true` | None |
| `list.unique(items)` | `List<T>` | Removes duplicates (the first occurrence is kept) | None |
| `list.windows(items, size)` | `List<List<T>>` | The sequence of contiguous parts of length `size` | `size >= 1` |
| `list.zip(first, second)` | `List<Pair<A, B>>` | Pairs up the elements at the same position | None |

## Common rules

### Function arguments and effects

A function that has an argument receiving a function, such as `test`, `keep`, `apply`, or `combine`, is effect-polymorphic. If the function passed in is pure, the call is pure, and if the function passed in is `uses io`, the caller must also declare `uses io` ([11 Effects](../language/11-effects.md)). Signatures express this with `effect E` and `uses E`.

The exception is `less_than` of `list.sort_by`, which cannot have effects. Passing a function with effects causes error `E0404`.

Functions are called in order from the start of the list to the end. `list.all`, `list.any`, `list.find`, `list.take_while`, and `list.drop_while` stop at the element that decides the result and do not call the function on the remaining elements.

### Equality and ordering bounds

`list.contains`, `list.index_of`, and `list.unique` require the bound `Eq` on the element type (because they compare with `==`). `list.max`, `list.min`, and `list.sort` require the bound `Ord` (because they compare with `<`). Which types satisfy these bounds is described in [03 Types](../language/03-types.md).

### A call that violates a precondition is a defect

`list.take`, `list.drop`, `list.slice`, `list.range`, `list.repeat`, and `list.windows` have preconditions (`requires`). A call that violates a precondition is a **defect**, and the program stops there. A defect cannot be caught inside the program ([12 Errors and defects](../language/12-errors-and-defects.md)).

In tests, `defects(of: ...)` confirms that an expression is a defect ([16 Testing](../language/16-testing.md)). The examples on this page use it for every function that has a precondition.

A call for which the checker finds inputs that violate the precondition is reported by `cernith check --verify` as error `E0501` (precondition-violated).

```cernith-error E0501
module guide.list_precondition;

import std.list;

pub fn preview(items: List<Int>) -> List<Int> {
  list.take(items: items, count: 3)
}
```

If `items` is shorter than three elements, the precondition of `list.take` is violated. The diagnostic shows `items=[]` as an input that causes this (the counterexample input). If the caller requires the same condition with its own `requires`, the code is accepted (`preview` in `guide.list_slicing`). How to write preconditions is described in [09 Contracts](../language/09-contracts.md), and how proofs are handled in [10 Proof obligations and assurance levels](../language/10-verification.md).

> Note: `list.take(items, count)` does not silently clamp a `count` that is out of range, because its postcondition guarantees `result.length == count`. A specification that is out of range does not return a result; it is a defect.

## Function details

### Building

#### list.range

```text
pub fn range(from: Int, to: Int) -> List<Int>
```

The list of integers from `from` (inclusive) to `to` (exclusive), in ascending order. `to` is not included. `list.range(from: 2, to: 5)` is `[2, 3, 4]`, and `list.range(from: -1, to: 1)` is `[-1, 0]`. If `from` and `to` are equal, `list.range(from: 3, to: 3)` is `[]`.

- Precondition: `from <= to`. A call in which `from` is greater than `to` (`list.range(from: 3, to: 2)`) is a defect.
- Postcondition: `result.length == to - from`.

#### list.repeat

```text
pub fn repeat<T>(item: T, count: Int) -> List<T>
```

A list of `count` copies of `item`. `list.repeat(item: "a", count: 3)` is `["a", "a", "a"]`, and if `count` is `0`, the result is `[]`.

- Precondition: `count >= 0`. A negative `count` (`list.repeat(item: "a", count: -1)`) is a defect.
- Postcondition: `result.length == count`.

### Taking and slicing

#### list.first and list.last

```text
pub fn first<T>(items: List<T>) -> Option<T>
pub fn last<T>(items: List<T>) -> Option<T>
```

Return the first element and the last element. `list.first(items: [7, 8])` is `Some(value: 7)`, and `list.last(items: [7, 8])` is `Some(value: 8)`. An empty list has no elements, so both return `None`.

- Precondition: none.
- Postcondition: the result is `None` if and only if `items.length == 0`.

The empty list literal `[]` alone does not determine `T`, so write a type argument, as in `list.first::<Int>(items: [])` ([06 Functions](../language/06-functions.md)).

#### list.take and list.drop

```text
pub fn take<T>(items: List<T>, count: Int) -> List<T>
pub fn drop<T>(items: List<T>, count: Int) -> List<T>
```

`list.take` returns the first `count` elements, and `list.drop` returns the rest after removing them. Joining `take` and `drop` with the same `count` using `++` gives back the original list.

- `list.take(items: [1, 2, 3], count: 2)` is `[1, 2]`; if `count` is `0`, the result is `[]`; if `count` equals the length, the result is the whole list.
- `list.drop(items: [1, 2, 3], count: 1)` is `[2, 3]`; if `count` equals the length, the result is `[]`; if `count` is `0`, the result is the whole list.

Precondition: `count >= 0 and count <= items.length`. A `count` that exceeds the length, or a negative `count`, is a defect. For example, `list.take(items: [1, 2], count: 3)` and `list.drop(items: [1, 2], count: 3)` are defects.

Postcondition: `result.length == count` for `take`, and `result.length == items.length - count` for `drop`.

#### list.slice

```text
pub fn slice<T>(items: List<T>, from: Int, to: Int) -> List<T>
```

Returns the part from position `from` (inclusive) to `to` (exclusive). Positions are counted from 0. `list.slice(items: [1, 2, 3, 4], from: 1, to: 3)` is `[2, 3]`, and if `from` and `to` are equal, the result is `[]`, as in `list.slice(items: [1, 2], from: 2, to: 2)`.

- Precondition: `0 <= from and from <= to and to <= items.length`. Each of `list.slice(items: [1, 2, 3], from: 2, to: 1)` (`from > to`), `list.slice(items: [1, 2, 3], from: 1, to: 4)` (`to` exceeds the length), and `list.slice(items: [1, 2, 3], from: -1, to: 2)` (`from` is negative) is a defect.
- Postcondition: `result.length == to - from`.

#### list.take_while and list.drop_while

```text
pub fn take_while<T, effect E>(
  items: List<T>,
  test: fn(item: T) -> Bool uses E,
) -> List<T> uses E

pub fn drop_while<T, effect E>(
  items: List<T>,
  test: fn(item: T) -> Bool uses E,
) -> List<T> uses E
```

Advance from the start only while `test` returns `true`. `list.take_while` returns that part, and `list.drop_while` returns the rest. Once `test` returns `false`, the scan stops there, even if later elements satisfy `test`. With `item < 3` as `test`, for `[1, 2, 5, 1]`, `list.take_while` returns `[1, 2]` and `list.drop_while` returns `[5, 1]`. If every element satisfies `test`, `list.take_while` returns the whole list and `list.drop_while` returns `[]`.

- Precondition: none.
- Postcondition: `result.length <= items.length`.

#### list.windows

```text
pub fn windows<T>(items: List<T>, size: Int) -> List<List<T>>
```

Lists the contiguous parts of length `size`, shifting one position at a time from the start. `list.windows(items: [1, 2, 3, 4], size: 2)` is `[[1, 2], [2, 3], [3, 4]]`, and with `size` equal to `1`, `list.windows(items: [1, 2], size: 1)` is `[[1], [2]]`. If `size` equals the length of the list, the result is a sequence containing only the whole list. If `size` is longer than the list, `list.windows(items: [1, 2], size: 3)` returns `[]`.

- Precondition: `size >= 1`. A call in which `size` is `0` or negative is a defect.
- Postcondition: none. The length of the result is `items.length - size + 1` (or `0` when that is negative).

### Searching and inspecting

#### list.all and list.any

```text
pub fn all<T, effect E>(
  items: List<T>,
  test: fn(item: T) -> Bool uses E,
) -> Bool uses E

pub fn any<T, effect E>(
  items: List<T>,
  test: fn(item: T) -> Bool uses E,
) -> Bool uses E
```

`list.all` is `true` when `test` is `true` for every element. `list.any` is `true` when at least one element has `test` equal to `true`. For an empty list, whatever `test` is, `list.all` is `true` and `list.any` is `false`.

`list.all` stops at the first element for which `test` is `false`, and `list.any` stops at the first element for which `test` is `true`; neither calls `test` on the remaining elements. Even a `test` that would cause a defect on a later element (for example, a division by `0`) does not cause a defect if the result is decided before that element.

- Precondition: none.

#### list.contains and list.index_of

```text
pub fn contains<T: Eq>(items: List<T>, item: T) -> Bool
pub fn index_of<T: Eq>(items: List<T>, item: T) -> Option<Int>
```

Return whether an element equal to `item` exists (`list.contains`) and, if so, the position of the first one (`list.index_of`). Positions are counted from 0. `list.index_of(items: [4, 5, 4], item: 4)` is `Some(value: 0)`, and if no such element is found, the result is `None`.

- Precondition: none.
- Postcondition (`index_of`): the `position` in `Some(value: position)` is at least `0` and less than `items.length`.

#### list.find

```text
pub fn find<T, effect E>(
  items: List<T>,
  test: fn(item: T) -> Bool uses E,
) -> Option<T> uses E
```

Returns the first element for which `test` returns `true`. With `item > 3` as `test`, `Some(value: 5)` is returned for `[1, 5, 7]` and `None` for `[1]`.

- Precondition: none.

#### list.count

```text
pub fn count<T, effect E>(
  items: List<T>,
  test: fn(item: T) -> Bool uses E,
) -> Int uses E
```

The number of elements for which `test` returns `true`. `test` is called on every element. With `item > 3` as `test`, the result for `[1, 5, 7, 2]` is `2`.

- Precondition: none.
- Postcondition: `result >= 0 and result <= items.length`.

#### list.max and list.min

```text
pub fn max<T: Ord>(items: List<T>) -> Option<T>
pub fn min<T: Ord>(items: List<T>) -> Option<T>
```

Return the largest and the smallest element. `list.max(items: [3, 9, 4])` is `Some(value: 9)`, and `list.min(items: [3, 9, 4])` is `Some(value: 3)`. A `String` is compared in lexicographic order of its code points. An empty list has neither a largest nor a smallest element, so `None` is returned.

- Precondition: none.
- Postcondition: the result is `None` if and only if `items.length == 0`.

### Transforming and folding

#### list.map

```text
pub fn map<A, B, effect E>(
  items: List<A>,
  apply: fn(item: A) -> B uses E,
) -> List<B> uses E
```

The list of the results of applying `apply` to each element, in the same order. The label of the argument of `apply` is `item`, and `B` may be a type different from `A`.

- Precondition: none.
- Postcondition: `result.length == items.length`.

#### list.filter

```text
pub fn filter<T, effect E>(
  items: List<T>,
  keep: fn(item: T) -> Bool uses E,
) -> List<T> uses E
```

Keeps only the elements for which `keep` returns `true`, in the original order.

- Precondition: none.
- Postcondition: `result.length <= items.length`.

#### list.flat_map and list.flatten

```text
pub fn flat_map<A, B, effect E>(
  items: List<A>,
  apply: fn(item: A) -> List<B> uses E,
) -> List<B> uses E

pub fn flatten<T>(lists: List<List<T>>) -> List<T>
```

`list.flatten` joins a list of lists, preserving the order. `list.flatten(lists: [[1, 2], [], [3]])` is `[1, 2, 3]`. `list.flat_map` applies `apply` (which returns a list) to each element and joins the results in the same way as `list.flatten`. Where `list.map` would give a `List<List<B>>`, this gives an unnested `List<B>`. An element for which `apply` returns `[]` disappears from the result.

- Precondition: none.

#### list.fold

```text
pub fn fold<T, A, effect E>(
  items: List<T>,
  initial: A,
  combine: fn(total: A, item: T) -> A uses E,
) -> A uses E
```

Starting from `initial`, applies `combine(total: ..., item: ...)` in order from the first element and returns the last `total`. Because the fold goes from left to right, when `combine` is `total - item`, `list.fold(items: [1, 2, 3], initial: 10, combine: ...)` is `((10 - 1) - 2) - 3`, which is `4`. For an empty list, `initial` is returned as it is. The type `A` of `total` may be different from the element type `T`.

- Precondition: none.

`list.fold` folds sequentially. The function that folds an associative `combine` in parallel is `parallel.fold` ([parallel](parallel.md)).

#### list.sum

```text
pub fn sum(items: List<Int>) -> Int
```

The sum of a list of `Int`. For an empty list, it is `0`. `Int` does not overflow, so the sum is always an exact value.

- Precondition: none.

#### list.unique

```text
pub fn unique<T: Eq>(items: List<T>) -> List<T>
```

Removes duplicates. Of equal values, the one that appears first is kept, and the order is unchanged. `list.unique(items: [3, 1, 3, 2, 1])` is `[3, 1, 2]`.

- Precondition: none.
- Postcondition: `result.length <= items.length`.

### Ordering and combining

#### list.reverse

```text
pub fn reverse<T>(items: List<T>) -> List<T>
```

The list in reverse order.

- Precondition: none.
- Postcondition: `result.length == items.length`.

#### list.sort and list.sort_by

```text
pub fn sort<T: Ord>(items: List<T>) -> List<T>

pub fn sort_by<T>(
  items: List<T>,
  less_than: fn(first: T, second: T) -> Bool,
) -> List<T>
```

`list.sort` sorts in ascending order. `list.sort_by` sorts so that `a` is placed before `b` when `less_than(first: a, second: b)` is `true`. Both are stable. Elements that cannot be ordered by comparison (those for which `less_than` is `false` in both directions) keep their original order. `list.sort` is the same as `list.sort_by` with `<` as `less_than`.

- Precondition: none.
- Postcondition: `result.length == items.length`. In addition, `list.sort_by` guarantees that, for every pair of adjacent elements `result[i]` and `result[i + 1]` of the result, `less_than(first: result[i + 1], second: result[i])` is `false` (that is, there is no pair in reverse order).

Give `less_than` a strict order, such as `<`, which is `false` between equal elements. Passing a function that is `true` between equal elements, such as `<=`, or a function that is always `true`, can leave the result unable to satisfy the postcondition, which is a defect. For example, sorting `[3, 1, 3, 2]` with `<=` as `less_than` is a defect (it is not a defect for `[3, 1, 2]`, which has no duplicates).

Passing a function with effects as `less_than` is error `E0404`.

```cernith-error E0404
module guide.list_effects;

import std.list;

pub fn logged_sort(items: List<Int>) -> List<Int> uses io {
  list.sort_by(
    items: items,
    less_than: fn(first: Int, second: Int) -> Bool uses io {
      first < second
    },
  )
}
```

#### list.zip and list.Pair

```text
pub record Pair<A, B> {
  first: A,
  second: B,
}

pub fn zip<A, B>(first: List<A>, second: List<B>) -> List<Pair<A, B>>
```

Pairs up the elements at the same position of two lists as `Pair` values and lists them. When the lengths differ, the result follows the shorter list and the surplus of the longer one is discarded. `list.zip(first: [1, 2, 3], second: ["a", "b"])` is `[list.Pair(first: 1, second: "a"), list.Pair(first: 2, second: "b")]`. Cernith has no tuples, so the two values of a pair are read through the fields `first` and `second` of `Pair` ([03 Types](../language/03-types.md)).

- Precondition: none.
- Postcondition: `result.length <= first.length and result.length <= second.length`.

## Examples

### Building

```cernith
module guide.list_build;

import std.list;

pub fn blank_row(width: Int) -> List<String> {
  requires width >= 0;

  list.repeat(item: " ", count: width)
}

example repeat_copies_the_item for blank_row {
  list.repeat(item: "a", count: 3) == ["a", "a", "a"]
    and blank_row(width: 2) == [" ", " "]
}

example repeat_negative_count_is_a_defect for blank_row {
  defects(of: list.repeat(item: "a", count: -1))
}

example repeat_zero_is_empty for blank_row {
  list.repeat(item: 1, count: 0) == [] and blank_row(width: 0) == []
}

pub fn squares(count: Int) -> List<Int> {
  requires count >= 0;

  list.map(
    items: list.range(from: 0, to: count),
    apply: fn(item: Int) -> Int {
      item * item
    },
  )
}

example range_backwards_is_a_defect for squares {
  defects(of: list.range(from: 3, to: 2))
}

example range_is_half_open for squares {
  list.range(from: 2, to: 5) == [2, 3, 4]
    and list.range(from: -1, to: 1) == [-1, 0]
    and squares(count: 4) == [0, 1, 4, 9]
}

example range_with_equal_ends_is_empty for squares {
  list.range(from: 3, to: 3) == [] and squares(count: 0) == []
}
```

### Taking and slicing

```cernith
module guide.list_slicing;

import std.int;
import std.list;

pub fn latest(items: List<Int>, count: Int) -> List<Int> {
  requires count >= 0 and count <= items.length;

  list.drop(items: items, count: items.length - count)
}

example drop_beyond_the_length_is_a_defect for latest {
  defects(of: list.drop(items: [1, 2], count: 3))
    and defects(of: list.drop(items: [1, 2], count: -1))
}

example drop_removes_the_prefix for latest {
  list.drop(items: [1, 2, 3], count: 1) == [2, 3]
    and list.drop(items: [1, 2, 3], count: 3) == []
    and list.drop(items: [1, 2, 3], count: 0) == [1, 2, 3]
    and latest(items: [1, 2, 3, 4], count: 2) == [3, 4]
}

pub fn leading_small(items: List<Int>) -> List<Int> {
  list.take_while(
    items: items,
    test: fn(item: Int) -> Bool {
      item < 3
    },
  )
}

example take_while_and_drop_while_stop_at_the_first_miss for leading_small {
  list.take_while(
    items: [20, 0],
    test: fn(item: Int) -> Bool {
      int.div_floor(dividend: 10, divisor: item) > 1
    },
  )
    == []
    and list.drop_while(
      items: [20, 0],
      test: fn(item: Int) -> Bool {
        int.div_floor(dividend: 10, divisor: item) > 1
      },
    )
      == [20, 0]
}

example take_while_stops_at_the_first_miss for leading_small {
  leading_small(items: [1, 2, 5, 1]) == [1, 2]
    and leading_small(items: [1, 2]) == [1, 2]
    and leading_small(items: [5, 1]) == []
    and leading_small(items: []) == []
}

pub fn middle(items: List<Int>) -> List<Int> {
  requires items.length >= 2;

  list.slice(items: items, from: 1, to: items.length - 1)
}

example slice_out_of_range_is_a_defect for middle {
  defects(of: list.slice(items: [1, 2, 3], from: 2, to: 1))
    and defects(of: list.slice(items: [1, 2, 3], from: 1, to: 4))
    and defects(of: list.slice(items: [1, 2, 3], from: -1, to: 2))
}

example slice_takes_a_range for middle {
  list.slice(items: [1, 2, 3, 4], from: 1, to: 3) == [2, 3]
    and list.slice(items: [1, 2], from: 2, to: 2) == []
    and middle(items: [1, 2, 3, 4]) == [2, 3]
}

pub fn moving_sums(items: List<Int>, size: Int) -> List<Int> {
  requires size >= 1;

  list.map(
    items: list.windows(items: items, size: size),
    apply: fn(item: List<Int>) -> Int {
      list.sum(items: item)
    },
  )
}

example windows_longer_than_the_list_is_empty for moving_sums {
  list.windows(items: [1, 2], size: 3) == []
    and list.windows::<Int>(items: [], size: 1) == []
}

example windows_of_size_zero_is_a_defect for moving_sums {
  defects(of: list.windows(items: [1, 2], size: 0))
    and defects(of: list.windows(items: [1, 2], size: -1))
}

example windows_slide_over_the_list for moving_sums {
  list.windows(items: [1, 2, 3, 4], size: 2) == [[1, 2], [2, 3], [3, 4]]
    and list.windows(items: [1, 2], size: 1) == [[1], [2]]
    and list.windows(items: [1, 2], size: 2) == [[1, 2]]
    and moving_sums(items: [1, 2, 3, 4], size: 2) == [3, 5, 7]
}

pub fn preview(items: List<Int>, count: Int) -> List<Int> {
  requires count >= 0 and count <= items.length;

  list.take(items: items, count: count)
}

example take_beyond_the_length_is_a_defect for preview {
  defects(of: list.take(items: [1, 2], count: 3))
    and defects(of: list.take(items: [1, 2], count: -1))
}

example take_keeps_the_prefix for preview {
  list.take(items: [1, 2, 3], count: 2) == [1, 2]
    and list.take(items: [1, 2, 3], count: 0) == []
    and list.take(items: [1], count: 1) == [1]
    and preview(items: [1, 2, 3], count: 2) == [1, 2]
    and list.take(items: [1, 2, 3], count: 2)
      ++ list.drop(items: [1, 2, 3], count: 2)
      == [1, 2, 3]
}

pub fn spread(items: List<Int>) -> Int {
  match list.first(items: items) {
    None => 0,
    Some(value: low) => match list.last(items: items) {
      None => 0,
      Some(value: high) => high - low,
    },
  }
}

example first_and_last_of_an_empty_list for spread {
  list.first::<Int>(items: []) == None
    and list.last::<Int>(items: []) == None
    and spread(items: []) == 0
}

example first_and_last_of_lists for spread {
  list.first(items: [7, 8]) == Some(value: 7)
    and list.last(items: [7, 8]) == Some(value: 8)
    and list.first(items: [7]) == Some(value: 7)
    and list.last(items: [7]) == Some(value: 7)
    and spread(items: [1, 5, 9]) == 8
}

pub fn without_leading_zeros(items: List<Int>) -> List<Int> {
  list.drop_while(
    items: items,
    test: fn(item: Int) -> Bool {
      item == 0
    },
  )
}

example drop_while_skips_the_prefix for without_leading_zeros {
  without_leading_zeros(items: [0, 0, 4, 0]) == [4, 0]
    and without_leading_zeros(items: [0, 0]) == []
    and without_leading_zeros(items: []) == []
    and list.drop_while(
      items: [1, 2, 5, 1],
      test: fn(item: Int) -> Bool {
        item < 3
      },
    )
      == [5, 1]
}
```

### Searching and inspecting

```cernith
module guide.list_search;

import std.int;
import std.list;

pub fn all_in_range(scores: List<Int>) -> Bool {
  list.all(
    items: scores,
    test: fn(item: Int) -> Bool {
      item >= 0 and item <= 100
    },
  )
}

example all_checks_every_item for all_in_range {
  all_in_range(scores: [2, 4])
    and not all_in_range(scores: [2, 400])
    and all_in_range(scores: [])
}

example all_stops_at_the_first_false for all_in_range {
  list.all(
    items: [20, 0],
    test: fn(item: Int) -> Bool {
      int.div_floor(dividend: 10, divisor: item) > 1
    },
  )
    == false
}

pub fn any_failing(scores: List<Int>) -> Bool {
  list.any(
    items: scores,
    test: fn(item: Int) -> Bool {
      item < 60
    },
  )
}

example any_looks_for_one_item for any_failing {
  any_failing(scores: [80, 50]) and not any_failing(scores: [80, 90])
}

example any_of_an_empty_list_is_false for any_failing {
  not any_failing(scores: [])
}

example any_stops_at_the_first_true for any_failing {
  list.any(
    items: [5, 0],
    test: fn(item: Int) -> Bool {
      int.div_floor(dividend: 10, divisor: item) > 1
    },
  )
}

pub fn best(scores: List<Int>) -> Option<Int> {
  list.max(items: scores)
}

example max_of_an_empty_list_is_none for best {
  best(scores: []) == None
}

example max_of_lists for best {
  best(scores: [3, 9, 4]) == Some(value: 9)
    and list.max(items: ["b", "a"]) == Some(value: "b")
    and best(scores: [1]) == Some(value: 1)
}

pub fn first_failing(scores: List<Int>) -> Option<Int> {
  list.find(
    items: scores,
    test: fn(item: Int) -> Bool {
      item < 60
    },
  )
}

example find_returns_the_first_match for first_failing {
  first_failing(scores: [80, 50, 40]) == Some(value: 50)
    and first_failing(scores: [80]) == None
    and first_failing(scores: []) == None
}

example find_stops_at_the_first_match for first_failing {
  list.find(
    items: [5, 0],
    test: fn(item: Int) -> Bool {
      int.div_floor(dividend: 10, divisor: item) > 1
    },
  )
    == Some(value: 5)
}

pub fn is_enrolled(names: List<String>, name: String) -> Bool {
  list.contains(items: names, item: name)
}

example contains_present_and_absent for is_enrolled {
  is_enrolled(names: ["ann", "bob"], name: "bob")
    and not is_enrolled(names: ["ann", "bob"], name: "cy")
    and not is_enrolled(names: [], name: "ann")
}

pub fn passing_count(scores: List<Int>) -> Int {
  list.count(
    items: scores,
    test: fn(item: Int) -> Bool {
      item >= 60
    },
  )
}

example count_counts_the_matches for passing_count {
  passing_count(scores: [80, 50, 60, 10]) == 2
    and passing_count(scores: [10]) == 0
    and passing_count(scores: []) == 0
}

pub fn seat_of(names: List<String>, name: String) -> Option<Int> {
  list.index_of(items: names, item: name)
}

example index_of_first_occurrence for seat_of {
  seat_of(names: ["ann", "bob", "ann"], name: "ann") == Some(value: 0)
    and seat_of(names: ["ann", "bob"], name: "bob") == Some(value: 1)
    and seat_of(names: ["ann"], name: "cy") == None
    and seat_of(names: [], name: "ann") == None
}

pub fn worst(scores: List<Int>) -> Option<Int> {
  list.min(items: scores)
}

example min_of_an_empty_list_is_none for worst {
  worst(scores: []) == None
}

example min_of_lists for worst {
  worst(scores: [3, 9, 4]) == Some(value: 3)
    and list.min(items: ["b", "a"]) == Some(value: "a")
    and worst(scores: [1]) == Some(value: 1)
}
```

### Transforming and folding

```cernith
module guide.list_transform;

import std.int;
import std.list;

pub fn distinct_tags(tags: List<String>) -> List<String> {
  list.unique(items: tags)
}

example unique_keeps_the_first_of_each for distinct_tags {
  list.unique(items: [3, 1, 3, 2, 1]) == [3, 1, 2]
    and list.unique(items: [1, 1, 1]) == [1]
    and distinct_tags(tags: ["a", "b", "a"]) == ["a", "b"]
}

example unique_of_an_empty_list for distinct_tags {
  distinct_tags(tags: []) == []
}

pub fn evens(items: List<Int>) -> List<Int> {
  list.filter(
    items: items,
    keep: fn(item: Int) -> Bool {
      int.rem_euclid(dividend: item, divisor: 2) == 0
    },
  )
}

example filter_can_drop_everything for evens {
  evens(items: [1, 3]) == [] and evens(items: []) == []
}

example filter_keeps_order for evens {
  list.filter(
    items: [1, 2, 3, 4, 5],
    keep: fn(item: Int) -> Bool {
      item > 2
    },
  )
    == [3, 4, 5]
    and evens(items: [5, 4, 3, 2]) == [4, 2]
}

pub fn join_rows(rows: List<List<Int>>) -> List<Int> {
  list.flatten(lists: rows)
}

example flatten_in_order for join_rows {
  join_rows(rows: [[1, 2], [], [3]]) == [1, 2, 3]
}

example flatten_of_no_lists for join_rows {
  join_rows(rows: []) == [] and join_rows(rows: [[], []]) == []
}

pub fn mean_floor(items: List<Int>) -> Int {
  requires items.length > 0;

  int.div_floor(dividend: list.sum(items: items), divisor: items.length)
}

example sum_of_an_empty_list_is_zero for mean_floor {
  list.sum(items: []) == 0
}

example sum_of_lists for mean_floor {
  list.sum(items: [1, 2, 3]) == 6
    and list.sum(items: [5, -5]) == 0
    and mean_floor(items: [1, 2, 4]) == 2
}

pub fn product(items: List<Int>) -> Int {
  list.fold(
    items: items,
    initial: 1,
    combine: fn(total: Int, item: Int) -> Int {
      total * item
    },
  )
}

example fold_can_change_the_type for product {
  list.fold(
    items: ["ab", "c"],
    initial: 0,
    combine: fn(total: Int, item: String) -> Int {
      total + item.length
    },
  )
    == 3
}

example fold_from_the_left for product {
  list.fold(
    items: [1, 2, 3],
    initial: 10,
    combine: fn(total: Int, item: Int) -> Int {
      total - item
    },
  )
    == 4
    and product(items: [2, 3, 4]) == 24
}

example fold_of_an_empty_list_is_the_initial_value for product {
  list.fold(
    items: [],
    initial: 7,
    combine: fn(total: Int, item: Int) -> Int {
      total + item
    },
  )
    == 7
    and product(items: []) == 1
}

pub fn spread_out(items: List<Int>) -> List<Int> {
  list.flat_map(
    items: items,
    apply: fn(item: Int) -> List<Int> {
      [item, item * 10]
    },
  )
}

example flat_map_can_drop_items for spread_out {
  list.flat_map(
    items: [1, 2, 3],
    apply: fn(item: Int) -> List<Int> {
      if item == 2 {
        []
      } else {
        [item]
      }
    },
  )
    == [1, 3]
    and spread_out(items: []) == []
}

example flat_map_joins_the_results for spread_out {
  spread_out(items: [1, 2, 3]) == [1, 10, 2, 20, 3, 30]
}

pub fn squared(items: List<Int>) -> List<Int> {
  list.map(
    items: items,
    apply: fn(item: Int) -> Int {
      item * item
    },
  )
}

example map_keeps_length_and_order for squared {
  squared(items: [1, 2, 3]) == [1, 4, 9]
}

example map_of_an_empty_list for squared {
  squared(items: []) == []
}
```

### Ordering and combining

```cernith
module guide.list_order;

import std.list;

pub fn ascending(items: List<Int>) -> List<Int> {
  list.sort(items: items)
}

example sort_ascending for ascending {
  ascending(items: [3, 1, 2]) == [1, 2, 3]
    and list.sort(items: ["b", "a"]) == ["a", "b"]
    and ascending(items: [2, 1, 2]) == [1, 2, 2]
}

example sort_of_an_empty_list for ascending {
  ascending(items: []) == []
}

pub fn by_length(words: List<String>) -> List<String> {
  list.sort_by(
    items: words,
    less_than: fn(first: String, second: String) -> Bool {
      first.length < second.length
    },
  )
}

example sort_by_is_stable for by_length {
  by_length(words: ["ccc", "a", "bb", "d"]) == ["a", "d", "bb", "ccc"]
    and list.sort_by(
      items: [[2, 0], [1, 1], [2, 2], [1, 3]],
      less_than: fn(first: List<Int>, second: List<Int>) -> Bool {
        first[0] < second[0]
      },
    )
      == [[1, 1], [1, 3], [2, 0], [2, 2]]
}

example sort_by_takes_any_strict_order for by_length {
  list.sort_by(
    items: [1, 3, 2],
    less_than: fn(first: Int, second: Int) -> Bool {
      first > second
    },
  )
    == [3, 2, 1]
    and by_length(words: []) == []
}

example sort_by_with_a_non_strict_order_can_be_a_defect for by_length {
  defects(
    of: list.sort_by(
      items: [3, 1, 3, 2],
      less_than: fn(first: Int, second: Int) -> Bool {
        first <= second
      },
    ),
  )
    and not defects(
      of: list.sort_by(
        items: [3, 1, 2],
        less_than: fn(first: Int, second: Int) -> Bool {
          first <= second
        },
      ),
    )
    and defects(
      of: list.sort_by(
        items: [3, 1, 2],
        less_than: fn(first: Int, second: Int) -> Bool {
          true
        },
      ),
    )
}

pub fn newest_first(items: List<Int>) -> List<Int> {
  list.reverse(items: items)
}

example reverse_of_an_empty_list for newest_first {
  newest_first(items: []) == []
}

example reverse_order for newest_first {
  newest_first(items: [1, 2, 3]) == [3, 2, 1]
}

pub fn pair_up(
  names: List<String>,
  scores: List<Int>,
) -> List<list.Pair<String, Int>> {
  list.zip(first: names, second: scores)
}

example zip_fields_are_first_and_second for pair_up {
  pair_up(names: ["ann"], scores: [90])[0].first == "ann"
    and pair_up(names: ["ann"], scores: [90])[0].second == 90
}

example zip_pairs_up_to_the_shorter for pair_up {
  list.zip(first: [1, 2, 3], second: ["a", "b"])
    == [list.Pair(first: 1, second: "a"), list.Pair(first: 2, second: "b")]
    and pair_up(names: ["ann"], scores: [90, 80])
      == [list.Pair(first: "ann", second: 90)]
}

example zip_with_an_empty_list for pair_up {
  pair_up(names: [], scores: [1]) == []
    and pair_up(names: ["ann"], scores: []) == []
}
```

## Related

- [option](option.md): the functions for `Option<T>`; they handle the values returned by `list.find` and `list.first`
- [parallel](parallel.md): parallel versions of `list.map`, `list.filter`, `list.fold`, `list.any`, and `list.all`
- [map](map.md), [set](set.md): collections looked up by key
- [03 Types](../language/03-types.md): `List<T>` and the bounds `Eq` and `Ord`
- [09 Contracts](../language/09-contracts.md): `requires` and `ensures`
- [11 Effects](../language/11-effects.md): effect polymorphism
- [12 Errors and defects](../language/12-errors-and-defects.md): calls that violate preconditions, and defects
