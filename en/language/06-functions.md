# Functions

Functions can be defined only at the top level (there is no syntax for defining a function inside a function). When a local function is needed, bind a function expression (lambda) to a `let`.

## Declaration

```text
[pub] fn name<type parameters>(parameters, ...) -> return type uses effects, ... {
  clauses...

  statements...
  final expression
}
```

- With `pub`, the function can be called from other modules. Without it, the function is available only within its module ([14 Modules and packages](14-modules-and-packages.md)).
- A parameter has the form `name: type`, and a trailing comma may or may not be added.
- The return type is required. A function that returns no value returns `Unit`.
- `uses` declares the effects that the function has. If there are no effects, it is not written ([11 Effects](11-effects.md)).
- At the start of the body, the clauses `requires`, `ensures`, and `decreases` can be written, in this order ([09 Contracts](09-contracts.md), [13 Termination](13-termination.md)). After the clauses comes one blank line, followed by the statements and the final expression that gives the value.

```cernith
module guide.functions;

pub fn rectangle_area(width: Int, height: Int) -> Int {
  requires width >= 0 and height >= 0;
  ensures result >= 0;

  width * height
}

example area_of_3_by_4 for rectangle_area {
  rectangle_area(width: 3, height: 4) == 12
}
```

## Parameters and labels

The name of a parameter becomes the label of a call as it is. The caller must pass all arguments with labels, in the same order as in the declaration (`E0304`, `E0310`; see [04 Expressions](04-expressions.md)). Therefore, **the names and order of the parameters are part of the published specification of the function**. Changing a name breaks every call.

The value of a parameter cannot be changed. Assigning to a parameter is `E0321`.

## Function values

Writing the name of a function in an expression position without calling it yields a **function value**. A function value can also be created with a function expression (lambda).

```text
fn(label: type, ...) -> return type uses effects { body }
```

The parameter types and the return type of a function expression cannot be omitted. The body has the same form as that of a function, and both `return` and clauses can be written.

A function type is written `fn(label: type, ...) -> return type uses effects`. **The labels of a function type are part of the type.** When a function value is called, the labels of the type on the side that sees the value are used. It does not matter if the function that was passed has different parameter names.

```cernith
module guide.function_values;

import std.list;

pub fn add_one(value: Int) -> Int {
  value + 1
}

pub fn apply_twice(apply: fn(item: Int) -> Int, start: Int) -> Int {
  apply(item: apply(item: start))
}

pub fn demo() -> Int {
  apply_twice(apply: add_one, start: 5)
}

example demo_is_7 for demo {
  demo() == 7
}

pub fn double_all(numbers: List<Int>) -> List<Int> {
  list.map(
    items: numbers,
    apply: fn(item: Int) -> Int {
      item * 2
    },
  )
}

example double_all_doubles for double_all {
  double_all(numbers: [1, 2, 3]) == [2, 4, 6]
}
```

The label used when calling `apply` inside `apply_twice` is `item`, which is written in the type of `apply`. It is not the parameter name `value` of the `add_one` that was passed. Calling with a label different from the label in the type is `E0310`.

```cernith-error E0310
module guide.function_values;

pub fn call(apply: fn(item: Int) -> Int) -> Int {
  apply(value: 1)
}
```

Compatibility between function types is contravariant in the parameters, covariant in the return value, and, for effects, “a function with fewer effects can be placed where more effects are allowed.” Parameter names are not compared.

A function expression cannot use a `var` of the enclosing function (`E0322`). Values bound with `let` and the values of parameters can be captured.

## Generics

Functions, records, and variants can have type parameters. A type parameter is a name that starts with an uppercase letter, written inside `<` `>` and separated by commas.

```text
pub fn first<T>(items: List<T>) -> Option<T>
pub record Box<T> { item: T, }
```

To restrict the types that a type parameter can take, a **bound** can be attached. There are only two bounds, `Eq` and `Ord`, and users cannot define their own.

| Bound | Operations available | Types that satisfy it |
| --- | --- | --- |
| `Eq` | `==`, `!=` | See “Equality and ordering” in [03 Types](03-types.md) |
| `Ord` | `<`, `<=`, `>`, `>=` in addition to `Eq`. Can also be a `Map` key or a `Set` element | Same as above |

Neither `==` nor `<` can be used on a value of a type parameter without a bound; doing so is `E0316`. Passing a type that does not satisfy the bound is also `E0316` (the check happens after the type arguments are determined, so the message names the concrete type). A bound name other than `Eq` or `Ord` is `E0318`.

```cernith
module guide.generics;

pub record Box<T> {
  item: T,
}

pub fn same<T: Eq>(first: T, second: T) -> Bool {
  first == second
}

example same_compares_equal_values for same {
  same(first: 1, second: 1) and not same(first: "a", second: "b")
}

pub fn unbox<T>(box: Box<T>) -> T {
  box.item
}

example unbox_returns_the_item for unbox {
  unbox(box: Box(item: 3)) == 3
}
```

```cernith-error E0316
module guide.generics;

pub fn same<T>(first: T, second: T) -> Bool {
  first == second
}
```

Type arguments are usually inferred. To state them explicitly, write `f::<Int>(...)`. Their number must equal the number of type parameters, excluding effect parameters (`E0315`). The notation is `::<` so that a `<` in an expression can be read as a comparison operator.

The `effect E` parameter, which expresses effect polymorphism, is handled separately from type parameters ([11 Effects](11-effects.md)).

## Type inference

Types must always be written in the following places.

- The parameters and return value of a function
- The fields of a record
- The declaration of a `var`
- The parameters and return value of a function expression

All other types are inferred locally. The information used for inference is, in this order:

1. The types of the arguments. In a call to a generic function, the type variables are determined by matching the parameter types with the argument types.
2. The **expected type**. The types of a function’s return value, a constructor, a list literal, `None`, `Ok`, and `Err` are determined by the position in which the expression is placed. For example, `fn f() -> Option<Int> { None }` needs no type annotation.

If a type variable remains undetermined by the end of the function, the error is `E0320` (cannot-infer). Fix it by writing the type arguments with `f::<T>(...)`, giving context to the right-hand side of a `let`, or writing the type, as in `var xs: List<Int> = [];`. There is no default type for numeric literals or the like. An integer literal is an `Int`, a decimal literal is a `Float64`, and `[]` is an empty list of the type that the context provides.

```cernith-error E0320
module guide.inference;

pub fn nothing<T>() -> List<T> {
  []
}

pub fn use_nothing() -> Int {
  let items = nothing();
  items.length
}
```

```cernith
module guide.inference;

pub fn empty_ints() -> List<Int> {
  []
}

example empty_ints_is_empty for empty_ints {
  empty_ints() == []
}

pub fn nothing<T>() -> List<T> {
  []
}

pub fn use_nothing() -> Int {
  nothing::<Int>().length
}

example use_nothing_is_zero for use_nothing {
  use_nothing() == 0
}
```

## Constants

```text
[pub] const name: type = expression;
```

A constant is an immutable value written at the top level of a module, and its type must always be written. The initializer is pure (it cannot call functions that have effects; `E0401`), and it is evaluated exactly once before the program starts, in dependency order. A cycle in which a constant depends on itself is `E0211`.

```cernith
module guide.constants;

pub const limit: Int = 10;

pub fn over_limit(value: Int) -> Bool {
  value > limit
}

example over_limit_compares for over_limit {
  over_limit(value: 11) and not over_limit(value: 10)
}
```

## Program entry point

A runnable program is a module that has a function named `main`.

```text
pub fn main(args: List<String>) -> Int uses io { ... }
```

`args` is the command-line arguments, and the return value is the exit status. If the program performs input and output, declare `uses io`. The form of `main` that receives authority (capabilities) is described in [17 Resources and capabilities](17-resources-and-capabilities.md). That is the only special rule for `main`; in every other respect it is an ordinary function.

## Related

- Rules for calling functions: [04 Expressions](04-expressions.md)
- Contracts: [09 Contracts](09-contracts.md)
- Recursion and termination: [13 Termination](13-termination.md)
- Functions implemented in other languages: [15 Foreign functions](15-foreign-functions.md)
