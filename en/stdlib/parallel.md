# std.parallel

This module provides versions of [list](list.md)'s `map`, `filter`, `fold`, `any`, and `all` that can run on multiple workers. They return the same results, but require pure callbacks so that calls for different elements can run concurrently. Import it with `import std.parallel;` and call functions such as `parallel.map(items: ..., apply: ...)`.

## Function list

| Function | Return type | Meaning | Corresponding function in `std.list` |
| --- | --- | --- | --- |
| `parallel.all(items, test)` | `Bool` | `test` is `true` for every element | `list.all` |
| `parallel.any(items, test)` | `Bool` | There is an element for which `test` is `true` | `list.any` |
| `parallel.filter(items, keep)` | `List<T>` | Only the elements for which `keep` is `true` (the order is preserved) | `list.filter` |
| `parallel.fold(items, initial, combine)` | `T` | Combines the elements into one with `combine` | `list.fold` (but the type of `combine` differs; see below) |
| `parallel.map(items, apply)` | `List<B>` | Applies `apply` to each element | `list.map` |

The other functions of `std.list` (`find`, `count`, `flat_map`, and so on) have no parallel version.

## Rules

### The result is the same as that of the same-named function in `std.list`

By definition, the result of each function of `parallel` is the result of the same-named function of `std.list` called with the same arguments. The order of the result is the order of `items`. Whatever order the calls for the individual elements are made in, the order of the result does not change.

A `std.list` call cannot always be replaced directly with its `parallel` counterpart. The callbacks must meet two additional requirements.

### The functions you pass have no effects

The types of the functions that the functions of `parallel` receive (`test`, `keep`, `apply`, and `combine`) have no `uses`. They are not effect-polymorphic (`effect E`) as in the same-named functions of `std.list`. Passing a function that has effects (such as `uses io`) is error `E0404` ([11 Effects](../language/11-effects.md)).

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

Without effects, the result for one element is never affected by the result for another element. The order of the calls therefore has no influence on the result, and the execution environment may run the calls for the individual elements side by side on several workers. Depending on the execution environment, they actually run in parallel; in other environments, they run sequentially. The result is the same either way.

The functions of `parallel` are themselves pure, and the caller does not need to declare any effect (the `async` effect is not needed either; see [18 Concurrency](../language/18-concurrency.md)). The signatures of the functions of `parallel` have no `uses`.

If a function that is passed becomes a defect, the program stops with the defect, just as when the function is called sequentially.

### In fold, `combine` is associative and `initial` is its identity

```text
pub fn fold<T>(
  items: List<T>,
  initial: T,
  combine: fn(total: T, item: T) -> T,
) -> T
```

`parallel.fold` has a different type from `list.fold`. In `list.fold`, the type `A` of the result of the fold may differ from the element type `T`, but in `parallel.fold`, `initial`, `total`, `item`, and the result are all of the same type `T`.

`combine` and `initial` have the following conditions.

- `combine` is associative. For any `a`, `b`, and `c`, `combine(total: combine(total: a, item: b), item: c)` and `combine(total: a, item: combine(total: b, item: c))` are equal.
- `initial` is an identity element of `combine`. For any `x`, both `combine(total: initial, item: x)` and `combine(total: x, item: initial)` are equal to `x`.

When these conditions hold, the result is equal to the result of the sequential `list.fold` however the elements are divided into groups and combined. The result of `parallel.fold` is guaranteed to be equal to that of `list.fold` only when these conditions hold.

Examples that satisfy the conditions are `+` with `0`, `*` with `1`, and the string `++` with `""`. Examples that do not are `total - item` (not associative) and `+` with `initial: 10` (`10` is not an identity element).

The library cannot verify these properties of `combine` and `initial`; the caller must ensure they hold. For integer addition, you can test them with a `property`, as in `fold_of_a_split_is_the_fold_of_the_halves` below.

## Function details

### parallel.map

```text
pub fn map<A, B>(items: List<A>, apply: fn(item: A) -> B) -> List<B>
```

A list of the results of applying `apply` to each element, in the same order.

- Precondition: none.
- Postcondition: `result.length == items.length`.

### parallel.filter

```text
pub fn filter<T>(items: List<T>, keep: fn(item: T) -> Bool) -> List<T>
```

Keeps only the elements for which `keep` returns `true`, in the original order.

- Precondition: none.
- Postcondition: `result.length <= items.length`.

### parallel.fold

```text
pub fn fold<T>(
  items: List<T>,
  initial: T,
  combine: fn(total: T, item: T) -> T,
) -> T
```

Combines the elements into one with `combine`. The conditions are as in the section above. For an empty list, `initial` is returned.

- Precondition: none. However, `combine` and `initial` have the conditions above.

### parallel.any and parallel.all

```text
pub fn any<T>(items: List<T>, test: fn(item: T) -> Bool) -> Bool
pub fn all<T>(items: List<T>, test: fn(item: T) -> Bool) -> Bool
```

`parallel.any` is `true` if there is even one element for which `test` is `true`, and `parallel.all` is `true` if `test` is `true` for every element. For an empty list, `parallel.any` is `false` and `parallel.all` is `true`. The result is the same as that of `list.any` and `list.all`, and even if `test` would become a defect for an element after the element that decides the result, it does not become a defect.

- Precondition: none.

## Example

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

## Related

- [list](list.md): the sequential `map`, `filter`, `fold`, `any`, and `all` (effect-polymorphic)
- [11 Effects](../language/11-effects.md): pure functions and effects
- [18 Concurrency](../language/18-concurrency.md): `async` and tasks
