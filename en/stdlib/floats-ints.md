# std.floats and std.ints

These modules work with numeric arrays: `Floats` stores `Float64` values and `Ints` stores `Int` values. Import `std.floats` or `std.ints`, then call functions such as `floats.get(array, at)` and `ints.range(from, to)`. The type names `Floats` and `Ints` are in the prelude and need no import ([02 Lexical structure](../language/02-lexical-structure.md)).

The two modules have almost the same functions. Below, the shared functions are described together, and the functions that exist in only one module are shown separately.

## Floats and Ints

`Floats` and `Ints` are sequences that hold only numbers. Like `List<Float64>` and `List<Int>`, they are values, but they store their elements in an unboxed form, without allocating each element as a separate value.

- The elements of `Floats` are finite `Float64` values (a `Float64` is always finite to begin with; see [03 Types](../language/03-types.md)).
- The elements of `Ints` are `Int` values that fit in 64 bits, that is, integers from $-2^{63}$ to $2^{63}-1$. A value outside this range is rejected by a precondition (it is a defect). `Int` itself has no limit on the number of digits, but the values that can be stored in an `Ints` do have a limit.
- `==` and `!=` compare whether the lengths are equal and all elements are equal in order from the start. There are no ordering comparisons such as `<`.
- There are no literals. Arrays are created with functions such as `floats.from_list` and `ints.range`.

The differences from `List` are as follows.

| Operation | `List<T>` | `Floats` / `Ints` |
| --- | --- | --- |
| Reading an element | `xs[i]` | `floats.get(array, at)` |
| Length | `xs.length` | `floats.length(array)` |
| Traversing in order with `for x in ...` | Possible | Not possible. Use the range `0..floats.length(array: a)` as indexes, or convert to a list with `to_list` |
| Element type | Any | Finite `Float64` / 64-bit `Int` |

Both `Floats` and `Ints` can be converted to a `List` (`to_list`). To create one from a list, use `from_list`.

### Updating returns a new array

`floats.set(array, at, value)` returns a **new array** with the element at `at` replaced. The original array remains unchanged.

```text
a = floats.set(array: a, at: i, value: x);
```

Write it this way, by assigning the result back to a `var`. If nothing else refers to the array held by that `var`, the array is updated in place, so a loop that fills in elements one at a time takes work proportional to the number of elements. If the array is also referenced from elsewhere, it is copied at the first update. The visible result is the same in both cases.

### ints.fits and the 64-bit range

```text
pub spec fn fits(value: Int) -> Bool
```

`ints.fits(value)` is `true` when `value` is greater than or equal to $-2^{63}$ and less than or equal to $2^{63}-1$. Because it is a `spec fn`, it can be called only in contracts (`requires`, `ensures`, and loop invariants) and in `example` and `property`. Calling it in executable code is error `E0331` ([09 Contracts](../language/09-contracts.md)).

Functions that create or update an `Ints` require `fits` of the value passed in (see the precondition table below). Conversely, `ints.get` guarantees that its result satisfies `fits`, and `ints.to_list` guarantees that all elements satisfy `fits`.

`ints.sum` and `ints.dot` add up the elements of an `Ints`, but the result type is `Int`, which has no limit on the number of digits. Even if the total exceeds the 64-bit range, it is not a defect (see the example for `ints.dot`).

When the result of a `Floats` operation leaves the finite range (for example, `floats.sum` overflows), it is a defect, as with ordinary `Float64` operations.

## Function list

These are the functions found in both modules. `array` is a `Floats` in `floats` and an `Ints` in `ints`. `T` is `Float64` in `floats` and `Int` in `ints`.

| Function | Return type | Meaning |
| --- | --- | --- |
| `new(count, fill)` | Array | An array of `count` elements, all equal to `fill` |
| `from_list(items)` | Array | Creates an array from a `List<T>` |
| `to_list(array)` | `List<T>` | Converts to a `List<T>` |
| `length(array)` | `Int` | The number of elements |
| `get(array, at)` | `T` | The element at index `at` (starting from 0) |
| `set(array, at, value)` | Array | A new array with the element at `at` set to `value` |
| `slice(array, from, to)` | Array | The part from index `from` up to but not including `to` |
| `concat(first, second)` | Array | The array `first` followed by `second` |
| `fill(array, from, to, value)` | Array | A new array with the indexes from `from` up to but not including `to` set to `value` |
| `reverse(array)` | Array | The array in reverse order |
| `sum(array)` | `T` (`Int` in `ints`) | The sum of the elements; 0 if empty |
| `dot(first, second)` | `T` (`Int` in `ints`) | The dot product (the sum of the products of corresponding elements); 0 if empty |
| `max(array)` | `Option<T>` | The largest element; `None` if empty |
| `min(array)` | `Option<T>` | The smallest element; `None` if empty |

These functions exist only in `std.floats`.

| Function | Return type | Meaning |
| --- | --- | --- |
| `floats.map(array, apply)` | `Floats` | An array with `apply` applied to each element |
| `floats.from_function(count, make)` | `Floats` | An array built by calling `make(index: i)` for each index from `0` to `count - 1` |
| `floats.cumulative_sum(array)` | `Floats` | The cumulative sum; the element at index `i` is the sum from the start through the `i`-th element |

These functions exist only in `std.ints`.

| Function | Return type | Meaning |
| --- | --- | --- |
| `ints.range(from, to)` | `Ints` | An array that increases by 1 from `from` to `to - 1` (`to` is not included) |
| `ints.fits(value)` | `Bool` | Whether `value` is within the 64-bit range (`spec fn`) |

The functions passed to `floats.map` and `floats.from_function` are effect-polymorphic. If the function passed is pure, the call is pure; if you pass a function that is `uses io`, the caller must also declare `uses io` ([11 Effects](../language/11-effects.md)).

## Preconditions

The functions that are not in the following table (`length`, `to_list`, `concat`, `reverse`, `sum`, `max`, `min`, `map`, `cumulative_sum`) have no preconditions. A call that violates a precondition listed here is a defect.

| Function | Precondition |
| --- | --- |
| `floats.new(count, fill)` | `0 <= count <= 1073741823` |
| `ints.new(count, fill)` | `0 <= count <= 1073741823`, `ints.fits(value: fill)` |
| `ints.from_list(items)` | All elements satisfy `ints.fits` |
| `get(array, at)` | `0 <= at < length(array)` |
| `floats.set(array, at, value)` | `0 <= at < length(array)` |
| `ints.set(array, at, value)` | `0 <= at < length(array)`, `ints.fits(value: value)` |
| `slice(array, from, to)` | `0 <= from <= to <= length(array)` |
| `floats.fill(array, from, to, value)` | `0 <= from <= to <= length(array)` |
| `ints.fill(array, from, to, value)` | `0 <= from <= to <= length(array)`, `ints.fits(value: value)` |
| `dot(first, second)` | `length(array: first) == length(array: second)` |
| `floats.from_function(count, make)` | `0 <= count <= 1073741823` |
| `ints.range(from, to)` | `from <= to`, `to - from <= 1073741823`, `ints.fits(value: from)`, `ints.fits(value: to)` |

The upper limit on the number of elements, 1,073,741,823, is $2^{30}-1$.

`get` uses a precondition to match the behavior of `List` indexing, `xs[i]`: an out-of-range index is a defect, and the checker reports potentially invalid accesses as `E0506`. If you do not know whether an index is valid, compare it with `floats.length(array: a)` before accessing the array.

## Function details

### new

```text
pub fn new(count: Int, fill: Float64) -> Floats {
  requires 0 <= count and count <= 1073741823;
  ensures length(array: result) == count;
  ...
}

pub fn new(count: Int, fill: Int) -> Ints {
  requires 0 <= count and count <= 1073741823 and fits(value: fill);
  ensures length(array: result) == count;
  ...
}
```

Creates an array of `count` elements, all equal to `fill`. If `count` is `0`, the result is an empty array.

### from_list and to_list

```text
pub fn from_list(items: List<Float64>) -> Floats {
  ensures length(array: result) == items.length;
  ...
}

pub fn from_list(items: List<Int>) -> Ints {
  requires forall i in 0..items.length: fits(value: items[i]);
  ensures length(array: result) == items.length;
  ...
}

pub fn to_list(array: Floats) -> List<Float64> {
  ensures result.length == length(array: array);
  ...
}

pub fn to_list(array: Ints) -> List<Int> {
  ensures result.length == length(array: array)
    and (forall i in 0..result.length: fits(value: result[i]));
  ...
}
```

These convert to and from a `List`. The order of the elements does not change.

### length, get, and set

```text
pub fn length(array: Floats) -> Int {
  ensures result >= 0;
  ...
}

pub fn get(array: Floats, at: Int) -> Float64 {
  requires 0 <= at and at < length(array: array);
  ...
}

pub fn set(array: Floats, at: Int, value: Float64) -> Floats {
  requires 0 <= at and at < length(array: array);
  ensures length(array: result) == length(array: array);
  ...
}
```

In `ints`, the types are `Ints` and `Int`; `get` gains `ensures fits(value: result);`, and the `requires` of `set` gains `and fits(value: value)`. `set` returns a new array and leaves the original array unchanged. The result of `set` has the same length as the original array.

### slice and concat

```text
pub fn slice(array: Floats, from: Int, to: Int) -> Floats {
  requires 0 <= from and from <= to and to <= length(array: array);
  ensures length(array: result) == to - from;
  ...
}

pub fn concat(first: Floats, second: Floats) -> Floats {
  ensures length(array: result) == length(array: first) + length(array: second);
  ...
}
```

`slice` extracts the part from index `from` up to but not including `to`. If `from == to`, the result is an empty array. `concat` joins two arrays. Either one (or both) may be empty.

### fill

```text
pub fn fill(array: Floats, from: Int, to: Int, value: Float64) -> Floats {
  requires 0 <= from and from <= to and to <= length(array: array);
  ensures length(array: result) == length(array: array);
  ...
}
```

Returns a new array in which the elements from index `from` up to but not including `to` are set to `value`. If `from == to`, nothing changes.

### reverse

```text
pub fn reverse(array: Floats) -> Floats {
  ensures length(array: result) == length(array: array)
    and (
      forall i in 0..length(array: result): get(array: result, at: i)
        == get(array: array, at: length(array: array) - 1 - i)
    );
  ...
}
```

The array with its elements in reverse order.

### sum, dot, max, and min

```text
pub fn sum(array: Floats) -> Float64
pub fn dot(first: Floats, second: Floats) -> Float64 {
  requires length(array: first) == length(array: second);
  ...
}

pub fn max(array: Floats) -> Option<Float64> {
  ensures match result {
    None => length(array: array) == 0,
    Some(value: _) => length(array: array) > 0,
  };
  ...
}
```

In `ints`, the return values of `sum` and `dot` are `Int`. `min` is the minimum counterpart of `max` and has an `ensures` of the same form.

`sum` and `dot` add from the start in order. Because the result of rounding in `Floats` can vary with the order of addition, this order defines the result. The `sum` of an empty array is `0.0` (`0` in `ints`). The `dot` of two empty arrays is also `0.0` (`0`). `max` and `min` return `None` for an empty array (this is not made a precondition).

### floats.map

```text
pub fn map<effect E>(
  array: Floats,
  apply: fn(value: Float64) -> Float64 uses E,
) -> Floats uses E {
  ensures length(array: result) == length(array: array);
  ...
}
```

Returns an array with `apply` applied to each element. The label of the argument of `apply` is `value`.

### floats.from_function

```text
pub fn from_function<effect E>(
  count: Int,
  make: fn(index: Int) -> Float64 uses E,
) -> Floats uses E {
  requires 0 <= count and count <= 1073741823;
  ensures length(array: result) == count;
  ...
}
```

Calls `make(index: i)` in order from index `0` to `count - 1` and returns an array of the results. The label of the argument of `make` is `index`.

### floats.cumulative_sum

```text
pub fn cumulative_sum(array: Floats) -> Floats {
  ensures length(array: result) == length(array: array)
    and (
      forall i in 0..length(array: result): get(array: result, at: i)
        == (
          if i == 0 {
            get(array: array, at: 0)
          } else {
            get(array: result, at: i - 1) + get(array: array, at: i)
          }
        )
    );
  ...
}
```

A cumulative sum. For `[1.0, 2.0, 3.5]`, the result is `[1.0, 3.0, 6.5]`. An empty array gives an empty array.

### ints.range

```text
pub fn range(from: Int, to: Int) -> Ints {
  requires from <= to
    and to - from <= 1073741823
    and fits(value: from)
    and fits(value: to);
  ensures length(array: result) == to - from;
  ...
}
```

The array `from`, `from + 1`, …, `to - 1`. If `from == to`, the result is an empty array. Unlike the range expression in `for i in from..to`, it is built as an `Ints` value.

## Example: Floats

```cernith
module guide.float_arrays;

import std.float;
import std.floats;

pub fn samples() -> Floats {
  floats.from_list(items: [3.5, 1.0, 2.5])
}

example arrays_with_the_same_elements_are_equal for samples {
  samples() == floats.from_list(items: [3.5, 1.0, 2.5])
    and samples() != floats.from_list(items: [3.5, 1.0])
    and floats.new(count: 0, fill: 1.0) == floats.from_list(items: [])
}

example concat_joins_two_arrays for samples {
  floats.to_list(
    array: floats.concat(
      first: samples(),
      second: floats.from_list(items: [4.0]),
    ),
  )
    == [3.5, 1.0, 2.5, 4.0]
    and floats.length(
      array: floats.concat(
        first: floats.new(count: 0, fill: 0.0),
        second: floats.new(count: 0, fill: 0.0),
      ),
    )
      == 0
}

example cumulative_sum_adds_up for samples {
  floats.to_list(array: floats.cumulative_sum(array: samples()))
    == [3.5, 4.5, 7.0]
    and floats.length(
      array: floats.cumulative_sum(array: floats.from_list(items: [])),
    )
      == 0
}

example dot_multiplies_and_adds for samples {
  floats.dot(first: samples(), second: floats.from_list(items: [2.0, 4.0, 2.0]))
    == 16.0
    and floats.dot(
      first: floats.from_list(items: []),
      second: floats.from_list(items: []),
    )
      == 0.0
}

example dot_needs_equal_lengths for samples {
  defects(
    of: floats.dot(first: samples(), second: floats.from_list(items: [1.0])),
  )
}

example fill_outside_the_array_is_a_defect for samples {
  defects(of: floats.fill(array: samples(), from: 1, to: 4, value: 0.0))
    and defects(of: floats.fill(array: samples(), from: 2, to: 1, value: 0.0))
}

example fill_sets_a_range for samples {
  floats.to_list(
    array: floats.fill(array: samples(), from: 1, to: 3, value: 0.0),
  )
    == [3.5, 0.0, 0.0]
    and floats.fill(array: samples(), from: 2, to: 2, value: 0.0) == samples()
}

example from_function_calls_make_for_each_index for samples {
  floats.to_list(
    array: floats.from_function(
      count: 3,
      make: fn(index: Int) -> Float64 {
        float.from_int(value: index) + 0.5
      },
    ),
  )
    == [0.5, 1.5, 2.5]
}

example from_function_rejects_a_negative_count for samples {
  defects(
    of: floats.from_function(
      count: -1,
      make: fn(index: Int) -> Float64 {
        1.0
      },
    ),
  )
}

example from_list_keeps_the_elements for samples {
  floats.to_list(array: samples()) == [3.5, 1.0, 2.5]
    and floats.length(array: floats.from_list(items: [])) == 0
}

example get_outside_the_array_is_a_defect for samples {
  defects(of: floats.get(array: samples(), at: 3))
    and defects(of: floats.get(array: samples(), at: -1))
}

example get_reads_one_element for samples {
  floats.get(array: samples(), at: 1) == 1.0
}

example length_counts_the_elements for samples {
  floats.length(array: samples()) == 3
}

example map_applies_to_each_element for samples {
  floats.to_list(
    array: floats.map(
      array: samples(),
      apply: fn(value: Float64) -> Float64 {
        value * 2.0
      },
    ),
  )
    == [7.0, 2.0, 5.0]
}

example max_is_the_largest for samples {
  floats.max(array: samples()) == Some(value: 3.5)
    and floats.max(array: floats.new(count: 0, fill: 0.0)) == None
}

example min_is_the_smallest for samples {
  floats.min(array: samples()) == Some(value: 1.0)
    and floats.min(array: floats.new(count: 0, fill: 0.0)) == None
}

example new_makes_equal_elements for samples {
  floats.to_list(array: floats.new(count: 2, fill: 1.5)) == [1.5, 1.5]
}

example new_rejects_a_bad_count for samples {
  defects(of: floats.new(count: -1, fill: 0.0))
    and defects(of: floats.new(count: 1073741824, fill: 0.0))
}

example reverse_turns_the_array_around for samples {
  floats.to_list(array: floats.reverse(array: samples())) == [2.5, 1.0, 3.5]
    and floats.length(array: floats.reverse(array: floats.from_list(items: [])))
      == 0
}

example set_gives_a_new_array for samples {
  floats.to_list(array: floats.set(array: samples(), at: 0, value: 9.0))
    == [9.0, 1.0, 2.5]
    and floats.get(array: samples(), at: 0) == 3.5
}

example set_outside_the_array_is_a_defect for samples {
  defects(of: floats.set(array: samples(), at: 3, value: 0.0))
}

example slice_outside_the_array_is_a_defect for samples {
  defects(of: floats.slice(array: samples(), from: 2, to: 4))
    and defects(of: floats.slice(array: samples(), from: 2, to: 1))
    and defects(of: floats.slice(array: samples(), from: -1, to: 1))
}

example slice_takes_a_range for samples {
  floats.to_list(array: floats.slice(array: samples(), from: 1, to: 3))
    == [1.0, 2.5]
    and floats.length(array: floats.slice(array: samples(), from: 2, to: 2))
      == 0
}

example sum_adds_from_the_left for samples {
  floats.sum(array: samples()) == 7.0
    and floats.sum(array: floats.new(count: 0, fill: 1.0)) == 0.0
}

example sum_beyond_the_range_is_a_defect for samples {
  defects(of: floats.sum(array: floats.from_list(items: [1.0e308, 1.0e308])))
}

example to_list_gives_the_elements for samples {
  floats.to_list(array: floats.new(count: 2, fill: 0.5)) == [0.5, 0.5]
}
```

## Example: Ints

```cernith
module guide.int_arrays;

import std.ints;

pub fn counts() -> Ints {
  ints.from_list(items: [30, 10, 20])
}

example arrays_with_the_same_elements_are_equal for counts {
  counts() == ints.from_list(items: [30, 10, 20])
    and counts() != ints.from_list(items: [30, 10])
    and ints.new(count: 0, fill: 1) == ints.from_list(items: [])
}

example concat_joins_two_arrays for counts {
  ints.to_list(
    array: ints.concat(first: counts(), second: ints.from_list(items: [40])),
  )
    == [30, 10, 20, 40]
    and ints.length(
      array: ints.concat(
        first: ints.new(count: 0, fill: 0),
        second: ints.new(count: 0, fill: 0),
      ),
    )
      == 0
}

example dot_can_exceed_64_bits for counts {
  ints.dot(
    first: ints.from_list(items: [4611686018427387904]),
    second: ints.from_list(items: [4]),
  )
    == 18446744073709551616
}

example dot_multiplies_and_adds for counts {
  ints.dot(first: counts(), second: ints.from_list(items: [1, 2, 3])) == 110
    and ints.dot(
      first: ints.from_list(items: []),
      second: ints.from_list(items: []),
    )
      == 0
}

example dot_needs_equal_lengths for counts {
  defects(of: ints.dot(first: counts(), second: ints.from_list(items: [1])))
}

example fill_outside_the_array_is_a_defect for counts {
  defects(of: ints.fill(array: counts(), from: 1, to: 4, value: 0))
    and defects(of: ints.fill(array: counts(), from: 2, to: 1, value: 0))
    and defects(
      of: ints.fill(
        array: counts(),
        from: 0,
        to: 1,
        value: 9223372036854775808,
      ),
    )
}

example fill_sets_a_range for counts {
  ints.to_list(array: ints.fill(array: counts(), from: 1, to: 3, value: 0))
    == [30, 0, 0]
    and ints.fill(array: counts(), from: 2, to: 2, value: 0) == counts()
}

example from_list_keeps_the_elements for counts {
  ints.to_list(array: counts()) == [30, 10, 20]
    and ints.length(array: ints.from_list(items: [])) == 0
}

example from_list_rejects_a_value_outside_the_range for counts {
  defects(of: ints.from_list(items: [1, 9223372036854775808]))
}

example get_outside_the_array_is_a_defect for counts {
  defects(of: ints.get(array: counts(), at: 3))
    and defects(of: ints.get(array: counts(), at: -1))
}

example get_reads_one_element for counts {
  ints.get(array: counts(), at: 1) == 10
}

example length_counts_the_elements for counts {
  ints.length(array: counts()) == 3
}

example max_is_the_largest for counts {
  ints.max(array: counts()) == Some(value: 30)
    and ints.max(array: ints.new(count: 0, fill: 0)) == None
}

example min_is_the_smallest for counts {
  ints.min(array: counts()) == Some(value: 10)
    and ints.min(array: ints.new(count: 0, fill: 0)) == None
}

example new_makes_equal_elements for counts {
  ints.to_list(array: ints.new(count: 2, fill: 15)) == [15, 15]
}

example new_rejects_a_bad_count_or_fill for counts {
  defects(of: ints.new(count: -1, fill: 0))
    and defects(of: ints.new(count: 1073741824, fill: 0))
    and defects(of: ints.new(count: 1, fill: 9223372036854775808))
}

example range_counts_up_by_one for counts {
  ints.to_list(array: ints.range(from: 2, to: 5)) == [2, 3, 4]
    and ints.to_list(array: ints.range(from: -1, to: 1)) == [-1, 0]
    and ints.length(array: ints.range(from: 3, to: 3)) == 0
}

example range_rejects_a_bad_range for counts {
  defects(of: ints.range(from: 3, to: 2))
    and defects(of: ints.range(from: 0, to: 1073741824))
    and defects(of: ints.range(from: 0, to: 9223372036854775808))
}

example reverse_turns_the_array_around for counts {
  ints.to_list(array: ints.reverse(array: counts())) == [20, 10, 30]
    and ints.length(array: ints.reverse(array: ints.from_list(items: []))) == 0
}

example set_gives_a_new_array for counts {
  ints.to_list(array: ints.set(array: counts(), at: 0, value: 9)) == [9, 10, 20]
    and ints.get(array: counts(), at: 0) == 30
}

example set_rejects_a_bad_index_or_value for counts {
  defects(of: ints.set(array: counts(), at: 3, value: 0))
    and defects(
      of: ints.set(array: counts(), at: 0, value: 9223372036854775808),
    )
}

example slice_outside_the_array_is_a_defect for counts {
  defects(of: ints.slice(array: counts(), from: 2, to: 4))
    and defects(of: ints.slice(array: counts(), from: 2, to: 1))
    and defects(of: ints.slice(array: counts(), from: -1, to: 1))
}

example slice_takes_a_range for counts {
  ints.to_list(array: ints.slice(array: counts(), from: 1, to: 3)) == [10, 20]
    and ints.length(array: ints.slice(array: counts(), from: 2, to: 2)) == 0
}

example sum_adds_from_the_left for counts {
  ints.sum(array: counts()) == 60
    and ints.sum(array: ints.new(count: 0, fill: 1)) == 0
}

example sum_can_exceed_64_bits for counts {
  ints.sum(
    array: ints.from_list(items: [9223372036854775807, 9223372036854775807]),
  )
    == 18446744073709551614
}

example to_list_gives_the_elements for counts {
  ints.to_list(array: ints.new(count: 2, fill: 7)) == [7, 7]
}

pub fn only_if_it_fits(value: Int) -> Int {
  requires ints.fits(value: value);

  value
}

example a_value_outside_the_range_is_a_defect for only_if_it_fits {
  defects(of: only_if_it_fits(value: 9223372036854775808))
}

example fits_is_the_64_bit_range for only_if_it_fits {
  ints.fits(value: 9223372036854775807)
    and ints.fits(value: -9223372036854775807 - 1)
    and not ints.fits(value: 9223372036854775808)
    and not ints.fits(value: -9223372036854775807 - 2)
}
```

## Related

- [list](list.md): functions for `List`
- [bytes](bytes.md): the byte sequence `Bytes`. Like `Floats` and `Ints`, it is an unboxed sequence that can be compared as a value
- [03 Types](../language/03-types.md): `Floats`, `Ints`, and the finiteness of `Float64`
- [09 Contracts](../language/09-contracts.md): `requires`, `ensures`, and `spec fn`
