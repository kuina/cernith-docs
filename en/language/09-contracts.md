# Contracts

A **contract** states, in the same language as the code, a condition that a function, a loop, or a type must satisfy. If a contract is broken, the program stops with a defect ([12 Errors and defects](12-errors-and-defects.md)). The checker attempts to prove that the contract is always kept ([10 Proof obligations and assurance levels](10-verification.md)).

## Kinds of clauses

| Clause | Where it is written | Meaning | Who is blamed when it is broken |
| --- | --- | --- | --- |
| `requires P;` | Function, property | `P` holds when the function is called | The caller |
| `ensures Q;` | Function | `Q` holds when the function returns. `result` is the return value | The function itself |
| `decreases M;` | Function, loop | A non-negative `Int` that becomes smaller with every recursion or loop iteration ([13 Termination](13-termination.md)) | The function |
| `invariant I;` | Loop | `I` holds when the loop is entered and at the end of each iteration | The function |
| `where P` | Refined type, record | Every value of the type satisfies `P` (`self` is the value; for a record, the field names are visible) | The place where the value was created |
| `assert P;` | Statement | `P` holds at that point. From then on, `P` is assumed to hold when checking | The function |

The “blame” appears in diagnostics and defect reports. A call that violates `requires` is a mistake of the **caller**. A return value that violates `ensures` is a mistake of the **function**.

## Position and order

Clauses are written at the top of the body of a function or loop, in the order `requires`, `ensures`, `decreases` (for a loop, `invariant`, `decreases`). Writing several clauses of the same kind means that all of them must hold (logical conjunction). Put one blank line between the clauses and the statements that follow. A program whose clauses are in a different order is reordered in the canonical form ([19 Canonical form](19-canonical-form.md)).

There is no `old(...)`. Parameters are never modified, and a local `var` cannot be observed from the caller, so there is no need to distinguish the value at function entry from the value at exit. To refer, inside a loop invariant, to the value a `var` had before the loop was entered, bind it with `let` before the loop.

```cernith
module guide.contracts;

pub fn clamp(value: Int, low: Int, high: Int) -> Int {
  requires low <= high;
  ensures result >= low and result <= high;

  if value < low {
    low
  } else if value > high {
    high
  } else {
    value
  }
}

example clamp_above for clamp {
  clamp(value: 12, low: 0, high: 9) == 9
}

example clamp_inside for clamp {
  clamp(value: 5, low: 0, high: 9) == 5
}

example clamp_rejects_a_reversed_range for clamp {
  defects(of: clamp(value: 5, low: 9, high: 0))
}
```

`defects(of: expression)` is an expression that can be used only inside tests; it is `true` when evaluating the expression stops with a defect ([16 Testing](16-testing.md)).

When there is a call that violates a precondition, `cernith check --verify` reports `E0501` (precondition-violated) and shows an example of a violating input. A body that can violate a postcondition gets `E0502`.

```cernith-error E0501
module guide.contracts;

pub fn square_root_floor(value: Int) -> Int {
  requires value >= 0;

  value
}

pub fn caller(x: Int) -> Int {
  square_root_floor(value: x)
}
```

```cernith-error E0502
module guide.contracts;

pub fn absolute(value: Int) -> Int {
  ensures result >= 0;

  value
}
```

## Specification expressions

The conditions of a contract are **specification expressions**. They are ordinary expressions of type `Bool` (of type `Int` in `decreases`), but they can use the following elements, which cannot be written in ordinary expressions.

| Element | Meaning |
| --- | --- |
| `result` | In `ensures`, the return value of the function |
| `self` | In a type’s `where`, the value being checked |
| `implies` | Implication. `a implies b` is true if `a` is false |
| `forall`, `exists` | Quantifiers |
| Calls to a `spec fn` | Functions that exist only for specifications |

A specification expression must be **pure**. A function with effects cannot be called (`E0402`). A lambda cannot be written either.

Expressions are written in three kinds of places, and the elements available differ.

| Context | Place | What can be used |
| --- | --- | --- |
| Code | Function bodies, loop bodies, constants | Ordinary expressions |
| Specification | `requires`, `ensures`, `decreases`, `invariant`, `where`, bodies of `spec fn` | Code, plus quantifiers, calls to a `spec fn`, `result`, and `self` |
| Test | Bodies of `example` and `property` | Both code and specification |

Writing a quantifier or a call to a `spec fn` in code is an error (`E0332`, `E0331`). Writing `result` outside `ensures`, or `self` outside `where`, is `E0313`. The syntax for specifications does not exist at run time.

### Quantifiers

```text
forall i in 0..n: p        // p for every i with 0 <= i < n
forall x in items: p       // p for every element of the list
exists i in a..=b: p       // there exists an i with a <= i <= b for which p holds
forall v: Int: p           // p for every integer v (proof-only)
```

A quantifier that ranges over an integer range or the elements of a list is **executable**. Both a range and a list are finite, so every case can actually be examined. A quantifier that ranges over a whole type (`forall v: Int: ...`) is **proof-only**. It is not evaluated at run time, and if it cannot be proved it is reported as “unverified.” It is never silently treated as having succeeded.

The body of a quantifier extends as far to the right as possible. When placing a quantifier inside an operator expression, enclose it in parentheses.

### spec fn

A `spec fn` is a function that can be called only inside specifications. It is pure, it terminates, and it does not appear in ordinary code. It is used to give a name to a condition that is used repeatedly.

```cernith
module guide.specs;

pub spec fn all_positive(items: List<Int>) -> Bool {
  forall i in 0..items.length: items[i] > 0
}

pub fn sum_positive(items: List<Int>) -> Int {
  requires all_positive(items: items);
  ensures result >= 0;

  var total: Int = 0;
  for index in 0..items.length {
    invariant total >= 0;

    total = total + items[index];
  }
  total
}

example sum_positive_adds for sum_positive {
  sum_positive(items: [1, 2, 3]) == 6
}
```

```cernith-error E0331
module guide.specs;

pub spec fn is_positive(value: Int) -> Bool {
  value > 0
}

pub fn check(value: Int) -> Bool {
  is_positive(value: value)
}
```

```cernith-error E0332
module guide.specs;

pub fn all_positive(items: List<Int>) -> Bool {
  forall i in 0..items.length: items[i] > 0
}
```

```cernith-error E0402
module guide.specs;

import std.clock;

pub fn after_epoch(value: Int) -> Int {
  requires clock.now_ms() > 0;

  value
}
```

## Writing the conditions a contract expresses

A postcondition can use `match` and `implies`.

```cernith
module guide.postconditions;

pub fn first_or_none(items: List<Int>) -> Option<Int> {
  ensures match result {
    None => items.length == 0,
    Some(value: _) => items.length > 0,
  };

  if items.length == 0 {
    None
  } else {
    Some(value: items[0])
  }
}

example first_or_none_reports_emptiness for first_or_none {
  first_or_none(items: []) == None
    and first_or_none(items: [4]) == Some(value: 4)
}

pub fn pass_through(value: Int) -> Int {
  ensures value > 0 implies result > 0;

  value
}

example pass_through_keeps_the_value for pass_through {
  pass_through(value: 3) == 3
}
```

The postcondition of `first_or_none` states a property of the function, that `None` is returned only when the input is empty, which goes beyond what the type can express.

## Contracts at run time

The implementation may check contracts at run time.

- A condition that the checker has **proved** is not checked at run time.
- A condition that cannot be proved and is executable is checked at run time. If it is violated, the result is a defect, and the program stops with a report of it.
- A **proof-only** condition (a quantifier over a whole type) is not checked at run time.
- When running `cernith test`, all executable conditions are checked.

Checking a contract does not change the result of the program. A check either passes or stops the program.

## Related

- Kinds of proof obligations and assurance levels: [10 Proof obligations and assurance levels](10-verification.md)
- Defects: [12 Errors and defects](12-errors-and-defects.md)
- Conditions attached to types (`where`): [08 Refined types and distinct types](08-refined-and-distinct-types.md)
