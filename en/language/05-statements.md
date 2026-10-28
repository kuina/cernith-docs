# Statements and bindings

A function or loop body is a block containing zero or more statements and an optional final expression. The final expression has no trailing `;` and supplies the block's value. Without a final expression, the block evaluates to `()` (of type `Unit`).

## Kinds of statements

| Statement | Form | Meaning |
| --- | --- | --- |
| `let` | `let name = expression;` | Binds an immutable name. The type is the type of the expression and cannot be written |
| `var` | `var name: type = expression;` | Declares a mutable local variable. The type is required |
| Assignment | `name = expression;` | Replaces the value of a `var` |
| Expression statement | `f(...);` | Executes a call. The result must be `Unit` |
| `discard` | `discard expression;` | Makes explicit that a value is thrown away |
| `return` | `return expression;` | Ends the function immediately and returns a value |
| `assert` | `assert expression;` | Asserts that a condition holds |
| `if` / `match` / `for` / `while` | | Control structures. When written as statements, no `;` is added |

A statement ends with `;`. `if`, `match`, `for`, and `while` are exceptions: a statement that has a block does not take a `;`.

## let and var

`let` creates an immutable binding. Its type is inferred from the right-hand side; adding a type annotation produces `E0323`. `var` creates a mutable local variable and requires a type annotation; omitting it produces `E0311`. Use `name = expression;` to change its value. Assigning to a `let` binding or a parameter produces `E0321`.

```cernith
module guide.bindings;

pub fn sum(items: List<Int>) -> Int {
  var total: Int = 0;
  for item in items {
    total = total + item;
  }
  total
}

example sum_adds_up for sum {
  sum(items: [1, 2, 3]) == 6 and sum(items: []) == 0
}
```

A `var` is a local variable that can be used only inside the function in which it is declared. A modification is never observable from outside the function. What can be passed to another function is a value, not the variable itself. A function expression (lambda) can neither read nor write a `var` of the enclosing function (`E0322`).

```cernith-error E0323
module guide.bindings;

pub fn f() -> Int {
  let a: Int = 1;
  a
}
```

```cernith-error E0311
module guide.bindings;

pub fn f() -> Int {
  var a = 1;
  a
}
```

```cernith-error E0321
module guide.bindings;

pub fn f() -> Int {
  let a = 1;
  a = 2;
  a
}
```

```cernith-error E0322
module guide.bindings;

pub fn f() -> Int {
  var n: Int = 0;
  let get = fn() -> Int {
    n + 1
  };
  get()
}
```

> Note: A type cannot be written on `let` so that type information is not split across two places. To state a type, declare the variable with `var`, which takes a type, or make the right-hand side an expression whose type is evident.

## Rules for names

Three rules keep names unambiguous within a function:

**Within one function, a name that is already in scope cannot be bound again.** If a name introduced by `let`, `var`, a parameter, or a pattern is the same as a name that is already bound, the error is `E0203` (shadowing). An outer name cannot be hidden (shadowed). This ensures that the same name never refers to different things in the middle of a function. Names whose scopes do not overlap, such as the variables of two consecutive `for` loops or the names in the two branches of an `if`, may be the same.

**A name that is bound must be read at least once.** A `let`, `var`, or pattern name that is not used is `E0204` (unused-binding). To fix it, do one of the following: remove the binding, throw away only the result with `discard`, or change the name of the pattern or loop variable to `_`. Only parameters may go unused.

**A parameter or variable cannot have the same name as the alias of an imported module.** When `import std.int;` is present, a variable named `int` cannot be created (`E0202`). Import the module under a different alias (`import std.int as int_lib;`).

```cernith-error E0203
module guide.names;

pub fn f(a: Int) -> Int {
  let a = 2;
  a
}
```

```cernith-error E0204
module guide.names;

pub fn f() -> Int {
  let unused = 1;
  2
}
```

```cernith-error E0202
module guide.names;

import std.int;

pub fn f(int: Int) -> Int {
  int
}
```

## Expression statements and discard

An expression statement calls a function as `f(...);`. If the result is not `Unit`, discarding it implicitly produces `E0306` (unused-result). Add `discard` when you intend to ignore the result.

```cernith
module guide.discards;

import std.io;

pub fn log_and_double(value: Int) -> Int uses io {
  discard io.write_line(text: "doubling");
  value * 2
}
```

`io.write_line` returns a `Result`. Writing it as a statement without `discard` would let a failure go unnoticed, so it is rejected.

```cernith-error E0306
module guide.discards;

import std.io;

pub fn log_and_double(value: Int) -> Int uses io {
  io.write_line(text: "doubling");
  value * 2
}
```

`_` cannot be used as the name of a `let`. To receive a result without using the value, use `discard`.

## return

`return expression;` ends the function immediately and returns that value. It is used to leave from the middle of a block, in particular from inside a loop. **It cannot be written as the last statement of a function body** (`E0312`). In that case, make the value the final expression.

```cernith
module guide.returns;

pub fn first_negative(items: List<Int>) -> Option<Int> {
  for item in items {
    if item < 0 {
      return Some(value: item);
    }
  }
  None
}

example finds_the_first for first_negative {
  first_negative(items: [3, -1, -2]) == Some(value: -1)
    and first_negative(items: [3, 4]) == None
}
```

```cernith-error E0312
module guide.returns;

pub fn one() -> Int {
  return 1;
}
```

## assert

`assert expression;` asserts that the expression holds at that point. If it does not hold, it is a defect. In verification, the statements after an `assert` are examined on the assumption that the expression holds ([10 Proof obligations and assurance levels](10-verification.md)).

## Loops

### for

```text
for name in list { ... }
for i in a..b { ... }       // from a up to but excluding b
for i in a..=b { ... }      // from a to b inclusive
```

`for` traverses the elements of a list, or a range of `Int`s (a range whose end is not after its start, such as `1..0`, is empty), in order, and gives a name to each element (`_` can also be written, and so can a pattern that always matches, such as a record pattern `Point(x: a, y: b)`; a pattern that might not match is `E0305`). Lists and ranges are finite, so a `for` over them always ends and needs no justification for termination. A `Map` is traversed through the list of `entries` in `std.map`. A `String` cannot be traversed directly; turn it into a `List<String>` with `text.chars`.

For an element that is not used, make the name `_`.

You can also iterate over a `std.seq.Seq` ([std.seq](../stdlib/seq.md)). A `Seq` is lazy and carries an upper bound on its length, `bound`. Each iteration advances the sequence by one step, up to `bound` iterations in total. This guarantees termination without an explicit measure, and an early `return` leaves the remaining elements unevaluated.

```text
for name in seq { ... }     // at most seq.bound turns
```

A `for` over a `Seq` takes no `invariant` or `decreases` clause (`E1406`): the body cannot name the hidden cursor, and the bound is the measure. Loop over a list or a range if the loop needs a clause.

```cernith
module guide.seq_loops;

import std.seq;

pub fn first_above(limit: Int) -> Option<Int> {
  for number in seq.iterate(
    seed: 1,
    next: fn(value: Int) -> Int {
      value * 2
    },
    bound: 1000,
  ) {
    if number > limit {
      return Some(value: number);
    }
  }
  None
}

example first_above_stops_early for first_above {
  first_above(limit: 100) == Some(value: 128)
}

pub fn sum_of_squares(count: Int) -> Int {
  var total: Int = 0;
  for number in seq.range(from: 1, to: count + 1) {
    total = total + number * number;
  }
  total
}

example sum_of_squares_adds_up for sum_of_squares {
  sum_of_squares(count: 3) == 14 and sum_of_squares(count: 0) == 0
}
```

```cernith-error E1406
module guide.seq_loops;

import std.seq;

pub fn total() -> Int {
  var total: Int = 0;
  for number in seq.range(from: 0, to: 3) {
    invariant total >= 0;

    total = total + number;
  }
  total
}
```

### while

```text
while condition {
  invariant condition;     // optional
  decreases expression;    // required
  ...
}
```

A `while` requires a `decreases` clause. Write an `Int` expression that is zero or greater and becomes smaller each time the loop repeats; this shows why the loop stops. `invariant` is a condition that holds before and after each iteration, and it can be omitted ([13 Termination](13-termination.md), [09 Contracts](09-contracts.md)). The clauses are written at the start of the block.

```cernith
module guide.loops;

pub fn count_items(items: List<Int>) -> Int {
  var count: Int = 0;
  for _ in items {
    count = count + 1;
  }
  count
}

example count_items_counts for count_items {
  count_items(items: [5, 6, 7]) == 3
}

pub fn sum_below(n: Int) -> Int {
  requires n >= 0;
  ensures result >= 0;

  var total: Int = 0;
  var i: Int = 0;
  while i < n {
    invariant total >= 0 and i >= 0;
    decreases n - i;

    total = total + i;
    i = i + 1;
  }
  total
}

example sum_below_adds_the_numbers_below for sum_below {
  sum_below(n: 4) == 6
}
```

```cernith-error E0601
module guide.loops;

pub fn countdown(n: Int) -> Int {
  var i: Int = n;
  while i > 0 {
    i = i - 1;
  }
  i
}
```

Loops have no `break` or `continue`. To leave in the middle, use `return` inside a function.

## Using if and match as statements

An `if` can be used as a statement without an `else`. In that case its type is `Unit`, and the value of its body cannot be used.

```cernith
module guide.conditionals;

pub fn bump_if(flag: Bool, start: Int) -> Int {
  var n: Int = start;
  if flag {
    n = n + 1;
  }
  n
}

example bump_only_when_flagged for bump_if {
  bump_if(flag: true, start: 1) == 2 and bump_if(flag: false, start: 1) == 1
}
```

The rules for using `if` and `match` as expressions are in [04 Expressions](04-expressions.md).

## Related

- Defining functions: [06 Functions](06-functions.md)
- Termination of loops: [13 Termination](13-termination.md)
