# Refined types and distinct types

There are two ways to create a new type on top of an existing one.

| Kind | Declaration | Purpose |
| --- | --- | --- |
| Refined type | `type name = base type where condition;` | Restricts the range of values. The operations of the base type can be used as they are |
| Distinct type | `type name = distinct base type;` | Prevents values with different meanings from being mixed up. The operations of the base type cannot be used |

A type alias (`type UserId = Int;`) cannot be written (`E0314`). These two are the only ways to create a type.

## Refined types

```text
[pub] type name = base type where condition;
```

The condition is a `Bool` expression, and `self` stands for the value being checked. Every value of the type satisfies the condition.

```cernith
module guide.percent;

pub type Percent = Int where self >= 0 and self <= 100;

pub fn bump(percent: Percent) -> Int {
  percent + 1
}

example bump_adds_one for bump {
  bump(percent: Percent(value: 7)) == 8
}

pub fn full() -> Percent {
  Percent(value: 100)
}

example full_is_100 for full {
  full() == 100
}

pub fn parse_percent(number: Int) -> Option<Percent> {
  Percent.check(value: number)
}

example check_accepts_and_rejects for parse_percent {
  parse_percent(number: 50) == Some(value: Percent(value: 50))
    and parse_percent(number: 101) == None
}
```

### Constructing values

| Expression | Meaning |
| --- | --- |
| `Percent(value: n)` | Creates a `Percent` from `n`. If the condition is not satisfied, the result is a defect |
| `Percent.check(value: n)` | `Some(value: ...)` if the condition is satisfied, `None` otherwise. It never causes a defect |

Which one to use depends on whether the input can be trusted. Use `Percent(value: n)` for a value that is known to satisfy the condition, and use `Percent.check(value: n)` for a value that comes from outside as input, handling the `None` case.

An integer or floating-point literal does not become a value of a refined type on its own. `var p: Percent = 50` and `fn f() -> Percent { 50 }` are errors (`E0301`); write `Percent(value: 50)` instead. The diagnostic suggests this rewrite.

```cernith-error E0301
module guide.percent;

pub type Percent = Int where self >= 0 and self <= 100;

pub fn half() -> Percent {
  50
}
```

`Percent(value: n)` produces a **proof obligation**. If the checker can prove that `n` satisfies the condition, the runtime check is omitted. If `cernith check --verify` finds a counterexample, the result is `E0503` (restriction-violated); if the checker cannot decide, the condition is checked at run time ([10 Proof obligations and assurance levels](10-verification.md)).

```cernith-error E0503
module guide.percent;

pub type Percent = Int where self >= 0 and self <= 100;

pub fn make(number: Int) -> Percent {
  Percent(value: number)
}
```

If the input is narrowed with a precondition, the obligation is proved.

```cernith
module guide.percent;

pub type Percent = Int where self >= 0 and self <= 100;

pub fn make(number: Int) -> Percent {
  requires number >= 0 and number <= 100;

  Percent(value: number)
}

example make_keeps_the_number for make {
  make(number: 5) == 5
}
```

### Using values

A `Percent` can be used as it is wherever an `Int` is expected. No unwrapping operation is needed for `percent + 1` or `percent > 0`. `.value` cannot be written (`E0317`). The result is an `Int`, because the result of `+ 1` is not necessarily in the range 0 to 100.

## Distinct types

```text
[pub] type name = distinct base type;
```

A `distinct` type is a separate type that has the same representation as its base type. It is handled as follows.

| Expression | Meaning |
| --- | --- |
| `UserId(value: 7)` | Creates a value from a value of the base type |
| `id.value` | Extracts the value of the base type |
| `==`, `!=` | Can be used between values of the same `distinct` type |
| `<` and the like | Can be used between values of the same `distinct` type if the base type has an ordering |

No other operations (such as arithmetic) are available, and a `distinct` type cannot be mixed with its base type or with another `distinct` type (`E0303`, incompatible-identities). To use a value in a computation, extract it with `.value` and wrap the result in the constructor again.

```cernith
module guide.ids;

pub type OrderId = distinct Int;

pub type UserId = distinct Int;

pub fn earlier(first: UserId, second: UserId) -> Bool {
  first < second
}

example earlier_orders for earlier {
  earlier(first: UserId(value: 1), second: UserId(value: 2))
}

pub fn next_id(id: UserId) -> UserId {
  UserId(value: id.value + 1)
}

example next_id_adds_one for next_id {
  next_id(id: UserId(value: 7)).value == 8
}

pub fn same_user(first: UserId, second: UserId) -> Bool {
  first == second
}

example same_user_compares for same_user {
  same_user(first: UserId(value: 1), second: UserId(value: 1))
}
```

```cernith-error E0303
module guide.ids;

pub type OrderId = distinct Int;

pub type UserId = distinct Int;

pub fn to_order(id: UserId) -> OrderId {
  id
}
```

```cernith-error E0303
module guide.ids;

pub type UserId = distinct Int;

pub fn next_id(id: UserId) -> UserId {
  id + 1
}
```

Both exist to prevent values that have the same representation but different meanings, such as a user ID and an order ID that are both `Int`, from being mixed up.

From other modules, a `distinct` type is **abstract**. The name of the type can be used, but neither `UserId(value: ...)` nor `.value` can be used outside the module that defines it (`E0215`). The defining module publishes the functions that create and read values ([14 Modules and packages](14-modules-and-packages.md)). A refined type, in contrast, can be constructed by anyone (the condition acts as the guard).

A refined type can also be used as the base of a `distinct` type.

```cernith
module guide.counts;

pub type Cents = Int where self >= 0;

pub type Count = distinct Cents;

pub fn three() -> Count {
  Count(value: Cents(value: 3))
}

example three_is_three for three {
  three().value == 3
}
```

## Subtyping

Subtyping in Cernith is made of a single rule: **a refined type is a subtype of its base type** (`Percent` is a subtype of `Int`; transitively, so is a refined type whose base is another refined type).

This relation lifts as follows.

- Covariantly through an immutable constructor: `List<Percent>` is a subtype of `List<Int>`. The same holds for `Option`, `Result`, records, and variants with type parameters.
- Through function types: parameters are contravariant, return values are covariant, and a function with fewer effects is the subtype. Parameter names are not compared.

There are no other relations. `Int` is not a subtype of `Float64`, a `distinct` type is not a subtype of its base type, and a base type is not a subtype of a refined type (in the opposite direction, a constructor or `check` is needed).

A value of a subtype can be placed as it is wherever a value of the base type is expected. This is not a conversion; neither the representation nor the meaning of the value changes. Where two types meet, such as the two branches of an `if`, the cases of a `match`, the elements of a list, or the operands of a comparison, the type is the supertype if one is above the other. If neither is, the program is rejected.

```cernith
module guide.subtypes;

pub type Percent = Int where self >= 0 and self <= 100;

pub fn ints_of(percents: List<Percent>) -> List<Int> {
  percents
}

example ints_of_keeps_elements for ints_of {
  ints_of(percents: [Percent(value: 1)]) == [1]
}

pub fn pick(percent: Percent, use_percent: Bool) -> Int {
  if use_percent {
    percent
  } else {
    1000
  }
}

example pick_joins_to_int for pick {
  pick(percent: Percent(value: 5), use_percent: true) == 5
    and pick(percent: Percent(value: 5), use_percent: false) == 1000
}
```

## Related

- Record invariants (`where`): [07 Records and variants](07-records-and-variants.md)
- Contracts and specification expressions: [09 Contracts](09-contracts.md)
- Proof obligations: [10 Proof obligations and assurance levels](10-verification.md)
