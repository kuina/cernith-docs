# Expressions

Expressions include literals, names, calls, field reads, indexing, operators, `if`, `match`, function expressions (lambdas), lists, and `?`. This chapter describes the grammar and typing rules, except for `?`, which is covered in [12 Errors and defects](12-errors-and-defects.md).

## Calls and labels

In a function call, **every argument carries a label, and the arguments are listed in the same order as in the declaration**. A label is the name of a parameter.

```text
function name(label1: expression1, label2: expression2, ...)
```

The label is the parameter name used inside the function, so each argument's purpose is clear at the call site.

```cernith
module guide.labels;

pub fn area(width: Int, height: Int) -> Int {
  width * height
}

example area_of_3_by_4 for area {
  area(width: 3, height: 4) == 12
}
```

Omitting a label is error `E0304` (missing-argument-label), and labels that do not match the declaration (a different name, a missing label, an extra label, or a different order) are error `E0310` (argument-labels-mismatch).

```cernith-error E0304
module guide.labels;

pub fn area(width: Int, height: Int) -> Int {
  width * height
}

pub fn use_area() -> Int {
  area(3, 4)
}
```

```cernith-error E0310
module guide.labels;

pub fn area(width: Int, height: Int) -> Int {
  width * height
}

pub fn use_area() -> Int {
  area(height: 4, width: 3)
}
```

The same label rules apply to functions, record and variant construction (`Rect(width: 3, height: 4)`), `Type.with(...)`, and patterns ([07 Records and variants](07-records-and-variants.md)). Indexing, `xs[i]`, is the only call form that does not require labels.

When a call is made through a value of a function type, the labels are those of the **type as seen from the caller** ([06 Functions](06-functions.md)).

## Fields, indexing, and length

| Expression | Meaning |
| --- | --- |
| `r.name` | The field `name` of the record `r` |
| `xs[i]` | The `i`-th element of the list `xs` (the first is 0). Out of range is a defect |
| `xs.length`, `s.length` | The length of a `List` or `String` (an `Int` that is zero or greater) |
| `.name` or `[i]` following `f()` | The same notation can follow a call result (`make().x`) |

## Operators

| Operator | Operands | Result | Notes |
| --- | --- | --- | --- |
| `+`, `-`, `*` | Two `Int`s, or two `Float64`s | The same type | Mixing `Int` and `Float64` is `E0302` |
| `/` | Two `Float64`s | `Float64` | Between `Int`s it is `E0307` |
| Unary `-` | `Int` or `Float64` | The same type | |
| `==`, `!=` | Two values of the same type that supports equality comparison | `Bool` | Bound `Eq` ([03 Types](03-types.md)) |
| `<`, `<=`, `>`, `>=` | Two values of the same ordered type | `Bool` | Bound `Ord` |
| `and`, `or`, `not`, `implies` | `Bool` | `Bool` | `and`, `or`, and `implies` use short-circuit evaluation |
| `++` | Two `String`s, or two `List`s of the same type | The same type | Concatenation |

`+` cannot concatenate a `String` or a `List`. Concatenation is `++`.

### Precedence

Operators are listed from highest to lowest precedence. Operators in the same row have equal precedence.

| Strength | Operators | Associativity |
| --- | --- | --- |
| 1 (strongest) | Postfix: `.name`, `(...)`, `[i]`, `?` | Left |
| 2 | Unary `-`, `await` | |
| 3 | `*`, `/` | Left |
| 4 | `+`, `-` | Left |
| 5 | `++` | Left |
| 6 | `==`, `!=`, `<`, `<=`, `>`, `>=` | Cannot be chained |
| 7 | `not` | Prefix |
| 8 | `and` | Left |
| 9 | `or` | Left |
| 10 (weakest) | `implies` | Right |

`?` written directly after the operand of `await` belongs to the `await`: `await e?` is `(await e)?` ([18 Concurrency](18-concurrency.md)). Comparison operators cannot be chained. `a < b < c` is a syntax error (`E0001`). Because `not` binds more weakly than comparison, `not x == y` is read as `not (x == y)`. Because `and` binds more strongly than `or`, `a and b or c` is read as `(a and b) or c` (canonical form adds these parentheses; see [19 Canonical form](19-canonical-form.md)).

```cernith
module guide.operators;

pub fn arithmetic() -> Int {
  1 + 2 * 3 - 4
}

example arithmetic_is_3 for arithmetic {
  arithmetic() == 3
}

pub fn concatenate() -> List<Int> {
  [1] ++ [2, 3] ++ []
}

example concatenate_joins for concatenate {
  concatenate() == [1, 2, 3]
}

pub fn logic(a: Bool, b: Bool, c: Bool) -> Bool {
  (a and b) or (c and not a == b)
}

example logic_follows_precedence for logic {
  logic(a: true, b: true, c: false) and not logic(a: true, b: false, c: false)
}
```

```cernith-error E0001
module guide.operators;

pub fn between(a: Int, b: Int, c: Int) -> Bool {
  a < b < c
}
```

### Short-circuit evaluation

`a and b` skips `b` when `a` is false. `a or b` skips `b` when `a` is true. `a implies b` skips `b` when `a` is false. Thus, the right operand of `and` or `implies` can assume the left operand is true; the right operand of `or` can assume it is false. The verifier (`cernith check --verify`) uses these same assumptions, so a condition on the left can guard an index access on the right:

```cernith
module guide.guards;

pub fn first_is_positive(xs: List<Int>) -> Bool {
  xs.length > 0 and xs[0] > 0
}

example empty_list_is_not_positive for first_is_positive {
  not first_is_positive(xs: []) and first_is_positive(xs: [1])
}
```

## `if` and `match` expressions

`if` and `match` are expressions. They have a value, and when placed at the end of a block, they become the value of that block.

```text
if condition { expression } else { expression }
if condition { expression } else if condition { expression } else { expression }
match expression {
  pattern => expression,
  pattern => { block },
}
```

- The condition is a `Bool` and is not enclosed in parentheses.
- All branches must produce compatible types, taking subtyping into account. Incompatible branches produce `E0301` (`E0302` for `Int` and `Float64`, or `E0303` when a `distinct` type is involved).
- An `if` without an `else` has type `Unit`. A branch that returns a value cannot be written without an `else`.
- Each case of a `match` has the form `pattern => expression,` or `pattern => { block },`, and **the last case also ends with a comma**. A `match` must be exhaustive over all possible values of the scrutinee ([07 Records and variants](07-records-and-variants.md)).

If the value of an `if` without `else` is used, the types do not match and the result is an error.

```cernith-error E0301
module guide.branches;

pub fn maybe(flag: Bool) -> Int {
  let value = if flag { 1 };
  value
}
```

```cernith
module guide.branches;

pub fn describe(sign: Int) -> String {
  if sign < 0 {
    "negative"
  } else if sign == 0 {
    "zero"
  } else {
    "positive"
  }
}

example describes_signs for describe {
  describe(sign: -3) == "negative"
    and describe(sign: 0) == "zero"
    and describe(sign: 8) == "positive"
}

pub fn or_zero(value: Option<Int>) -> Int {
  match value {
    Some(value: number) => number,
    None => 0,
  }
}

example or_zero_defaults for or_zero {
  or_zero(value: Some(value: 5)) == 5 and or_zero(value: None) == 0
}
```

`if`, `match`, `for`, and `while` can be written as expressions only **at the start of an expression**. They cannot be written without parentheses as the operand of an operator, or as the base expression followed by `.name` or `[i]`. `1 + if ...` is a syntax error (`E0001`), and it can be written by enclosing the `if` in parentheses. In the position of a list element, a call argument, or a field value, parentheses are not needed.

```cernith-error E0001
module guide.branches;

pub fn plus(flag: Bool) -> Int {
  1 + if flag { 1 } else { 2 }
}
```

```cernith
module guide.branches;

pub fn plus(flag: Bool) -> Int {
  1
    + (
      if flag {
        1
      } else {
        2
      }
    )
}

example plus_adds_the_branch for plus {
  plus(flag: true) == 2 and plus(flag: false) == 3
}
```

This rule lets the grammar distinguish an `if` statement followed by a line starting with `-x` from an `if` expression from which `x` is subtracted.

## Lists

`[a, b, c]` creates a `List`. The types of the elements must agree (the result is the common supertype, taking subtyping into account). The empty list `[]` can be used only when its type is determined by the context (give it a type, as in `var xs: List<Int> = [];`; see type inference in [06 Functions](06-functions.md)).

The ranges `a..b` (`a` inclusive to `b` exclusive) and `a..=b` (`a` to `b`, both inclusive) can be written only inside `for` loops and quantifiers (`forall`, `exists`). They cannot be used as values.

## Function expressions (lambdas)

```text
fn(label: type, ...) -> return type { body }
```

The types of the arguments and the return value cannot be omitted. The body has the same form as the body of a function. If effects are used, add `uses`. A function expression cannot read or modify a `var` of the enclosing function (error `E0322`). Details are in [06 Functions](06-functions.md).

## Related

- Statements and bindings: [05 Statements and bindings](05-statements.md)
- Defining and calling functions: [06 Functions](06-functions.md)
- Grammar of operators: [21 Grammar](21-grammar.md)
