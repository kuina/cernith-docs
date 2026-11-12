# Termination

**A function that does not declare the effect `diverge` must always terminate.** The checker confirms termination with the following rules, and rejects the program if it cannot. A function that declares `uses diverge` promises nothing about termination, and a function that calls it must also declare `diverge` (the rule for effects; [11 Effects](11-effects.md)). This chain continues up to `main`.

A function with no loops or recursion terminates as long as the functions it calls terminate. The following cases need explicit justification.

## Loops

- **`for`** traverses a finite, immutable list or range, so it always terminates. No justification is needed.
- **`while`** requires a `decreases` clause. Write an `Int` expression that is non-negative when the loop is entered and after each iteration, and that becomes strictly smaller with each iteration.

```text
while low < high {
  invariant 0 <= low and low <= high;
  decreases high - low;
  ...
}
```

Without a justification, the result is `E0601` (termination-unproved). If the decreasing measure does not actually decrease, `cernith check --verify` reports `E0509` (measure-not-decreasing) and shows a counterexample input.

```cernith-error E0601
module guide.termination;

pub fn countdown(n: Int) -> Int {
  var i: Int = n;
  while i > 0 {
    i = i - 1;
  }
  i
}
```

An `E0601` diagnostic suggests a `decreases` clause, such as `decreases b - a;` for `while a < b`. This is only a suggestion: the checker never inserts or infers a decreasing measure automatically.

## Recursion

A function that calls itself, or a group of functions that call one another, needs one of the following.

1. **Structural recursion** (single functions only). One of the parameters is matched with `match`, and the recursive call passes a variable bound inside a constructor pattern in that parameter’s position. A variable bound by a further `match` on such a variable also qualifies. The argument of the call is then a proper part of the original value. If an arm’s pattern is just a name (`other => f(chain: other)`), that name refers to the whole parameter and does not count. A part produced by a library function such as `list.drop` does not count; it must be a part obtained from a pattern.
2. **`decreases`.** Write `decreases M;` on every function of the group. For each call to a function in the group, the decreasing measure of the callee must be non-negative and smaller than the measure the caller had at its entry. If several `decreases` clauses are written, the measure is in **lexicographic order** of the clauses in the order written (the functions in the group must have the same number of components).
3. **`uses diverge`.**

Mutually calling functions have no structural-recursion form. Every function of the group must have a decreasing measure.

```cernith
module guide.termination;

pub variant Chain {
  Link(size: Int, rest: Chain),
  End,
}

pub fn countdown(outer: Int, inner: Int) -> Int {
  requires outer >= 0 and inner >= 0;
  decreases outer;
  decreases inner;

  if inner > 0 {
    countdown(outer: outer, inner: inner - 1)
  } else if outer > 0 {
    countdown(outer: outer - 1, inner: 5)
  } else {
    0
  }
}

example countdown_reaches_zero for countdown {
  countdown(outer: 2, inner: 3) == 0
}

pub fn factorial(n: Int) -> Int {
  requires n >= 0;
  decreases n;

  if n == 0 {
    1
  } else {
    n * factorial(n: n - 1)
  }
}

example factorial_of_5 for factorial {
  factorial(n: 5) == 120
}

pub fn is_even(n: Int) -> Bool {
  requires n >= 0;
  decreases n;

  if n == 0 {
    true
  } else {
    is_odd(n: n - 1)
  }
}

example parity for is_even {
  is_even(n: 4) and not is_even(n: 3) and is_odd(n: 3)
}

pub fn is_odd(n: Int) -> Bool {
  requires n >= 0;
  decreases n;

  if n == 0 {
    false
  } else {
    is_even(n: n - 1)
  }
}

pub fn total_size(chain: Chain) -> Int {
  match chain {
    Chain.End => 0,
    Chain.Link(size: size, rest: rest) => size + total_size(chain: rest),
  }
}

example total_size_adds_the_links for total_size {
  total_size(
    chain: Chain.Link(size: 2, rest: Chain.Link(size: 3, rest: Chain.End)),
  )
    == 5
}
```

`total_size` uses structural recursion. It passes the `rest` obtained from `match`, so no `decreases` is needed. In `factorial`, `n` becomes smaller by one with each call. `is_even` and `is_odd` call each other, so both have a decreasing measure. `countdown` has a lexicographic measure, because either `outer` decreases, or `inner` decreases while `outer` stays the same.

A recursion whose decreasing measure does not decrease is `E0509` (reported by `cernith check --verify`).

```cernith-error E0509
module guide.termination;

pub fn grow(n: Int) -> Int {
  requires n >= 0;
  decreases n;

  if n <= 0 {
    0
  } else {
    grow(n: n + 1)
  }
}
```

A recursion without a decreasing measure is `E0601`.

```cernith-error E0601
module guide.termination;

pub fn count_down(n: Int) -> Int {
  if n <= 0 {
    0
  } else {
    count_down(n: n - 1)
  }
}
```

## Recursion cannot hide from the call graph

The checker builds a call graph from the calls in function bodies (a call inside a lambda is counted as belonging to the enclosing function) and finds the strongly connected components of mutually calling functions. This analysis requires recursion to appear in the graph. The following rules are intended to prevent recursion that would be hidden from it.

- A function expression (lambda) cannot refer, inside its own body, to the name it is bound to. Only named functions can recurse, and their calls become edges of the call graph. **Known limitation:** passing a named function as a value does not add an edge to the call graph. For example, if the function being defined is `spin`, passing it in `apply(g: spin, value: n)` allows recursion through that value to go undetected. The checker accepts this recursion without a termination argument.
- **A recursive record or variant cannot have, as a field, a function that takes a value of its own recursive group as an argument** (`E0602`). If it could, recursion with no edge in the call graph (a fixed-point combinator) could be written, as in `record W { call: fn(w: W) -> Int }`. A function that does not take its own type as an argument can be a field.

```cernith-error E0602
module guide.termination;

pub record Wrapper {
  call: fn(w: Wrapper) -> Int,
}
```

- **It must be possible to construct a value of a declared type** (`E0603`). A `distinct` or refined type whose chain of base types leads back to itself, a record that contains itself, and a variant in which every case contains itself are rejected. No value could be constructed, and the checks would also run in a cycle. Recursive structures that can terminate are allowed, such as recursion through a `List` or an `Option`, or a variant with a case that does not contain the type itself.

```cernith-error E0603
module guide.no_values;

pub record Endless {
  next: Endless,
}
```

## Processing that does not end

A loop that keeps reading until the input runs out has no decreasing measure. Declare `uses diverge` on the function. However, reading into a finite list and iterating over it with `for` preserves termination better. `main` may declare `diverge`.

Termination says nothing about running time. Even a function that terminates may be stopped for exceeding the limit on execution steps ([12 Errors and defects](12-errors-and-defects.md)).

```cernith
module guide.diverging;

import std.int;

pub fn collatz_steps(n: Int) -> Int uses diverge {
  requires n >= 1;

  if n == 1 {
    0
  } else if int.rem_euclid(dividend: n, divisor: 2) == 0 {
    1 + collatz_steps(n: int.div_floor(dividend: n, divisor: 2))
  } else {
    1 + collatz_steps(n: 3 * n + 1)
  }
}

example collatz_of_6 for collatz_steps {
  collatz_steps(n: 6) == 8
}
```

A function that calls `collatz_steps` must declare `uses diverge` (`E0401`).

## Limits

Decreasing measures must be `Int` expressions, optionally combined lexicographically. Other well-founded orders are not supported. A function that needs such an order to justify termination, as some graph algorithms do, must declare `diverge`.

## Diagnostic codes

| Code | Name | Meaning |
| --- | --- | --- |
| `E0601` | termination-unproved | A `while` loop or a recursive group without a justification |
| `E0602` | non-positive-recursive-type | A recursive type has a function that takes a value of its own recursive group |
| `E0603` | type-without-values | A declared type has no values |
| `E0509` | measure-not-decreasing | The decreasing measure does not become smaller (with a counterexample input) |
| `W0504` | obligation-unproved | A `term` obligation that the solver could not decide |

## Related

- Contract clauses: [09 Contracts](09-contracts.md)
- The `term` proof obligation: [10 Proof obligations and assurance levels](10-verification.md)
