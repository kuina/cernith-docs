# Records and variants

There are two compound types that users can define: **records** (a set of named fields) and **variants** (exactly one of several cases). There are no tuples. Both are immutable and nominal types. Even with the same structure, types with different names are different types.

## Records

```text
[pub] record name<type parameters> {
  field name: type,
  ...
} [where condition;]
```

A field has the form `name: type,`, one per line, and the last field also ends with a comma. The order of the fields is significant.

| Expression | Meaning |
| --- | --- |
| `Rect(width: 3, height: 4)` | Construction. All fields are passed, in declaration order, with labels |
| `r.width` | Reading a field |
| `Rect.with(base: r, width: 5)` | A copy of `r` with only the specified fields replaced. The labels are listed in declaration order |

Records cannot be modified. There is no assignment such as `r.width = 5;`; a new value with the field replaced is created with `with`. If the labels in a construction are missing, extra, or out of order, the error is `E0310`, and reading a field that does not exist is `E0317`.

```cernith
module guide.rectangles;

pub record Rect {
  width: Int,
  height: Int,
}

pub fn widen(rect: Rect) -> Rect {
  Rect.with(base: rect, width: rect.width + 1)
}

example widen_keeps_height for widen {
  widen(rect: Rect(width: 1, height: 2)) == Rect(width: 2, height: 2)
}
```

```cernith-error E0310
module guide.rectangles;

pub record Rect {
  width: Int,
  height: Int,
}

pub fn square(side: Int) -> Rect {
  Rect(height: side, width: side)
}
```

### Record invariants

An invariant can be attached to a record with `where`. Inside the condition, the names of the fields are used as they are.

```cernith
module guide.grids;

pub record Grid {
  width: Int,
  height: Int,
  cells: List<Int>,
} where width >= 0 and cells.length == width * height;

pub fn empty_grid() -> Grid {
  Grid(width: 0, height: 0, cells: [])
}

example empty_grid_has_no_cells for empty_grid {
  empty_grid().cells.length == 0
}
```

Every construction `Grid(...)` and every `Grid.with(...)` gives rise to the proof obligation that “the invariant holds.” If it can be proved, no check is needed; if there is a counterexample, `cernith check --verify` reports the error `E0503`; if neither can be determined, a run-time check is made ([10 Proof obligations and assurance levels](10-verification.md)). In the following `blank`, `cells` is always empty, so the invariant is violated for inputs where `width * height` is not 0. `cernith check --verify` finds this violation.

```cernith-error E0503
module guide.grids;

pub record Grid {
  width: Int,
  height: Int,
  cells: List<Int>,
} where width >= 0 and cells.length == width * height;

pub fn blank(width: Int, height: Int) -> Grid {
  requires width >= 0 and height >= 0;

  Grid(
    width: width,
    height: height,
    cells: [],
  )
}
```

An invariant is a specification expression. Functions that have effects cannot be called in it ([09 Contracts](09-contracts.md)). A condition written only with comparisons of integers or floating-point numbers, `.length`, and `and`, `or`, and `not` has a simple form that the checker can readily use as an assumption.

## Variants

```text
[pub] variant name<type parameters> {
  case,
  case(field name: type, ...),
  ...
}
```

A value of a variant is exactly one of its cases. A case can have fields. A variant may be recursive or generic.

A case is written in the form `variant name.case name`. A case with fields is constructed in the form of a call, and a case without fields is written without parentheses. For a type in another module, it is `module alias.type name.case name`. Only `Some`, `None`, `Ok`, and `Err` are cases from the prelude, so they are written without a type name.

```cernith
module guide.expressions;

pub variant Expr {
  Add(left: Expr, right: Expr),
  Number(value: Int),
  Zero,
}

pub fn evaluate(expression: Expr) -> Int {
  match expression {
    Expr.Add(left: left, right: right) => evaluate(expression: left)
      + evaluate(expression: right),
    Expr.Number(value: value) => value,
    Expr.Zero => 0,
  }
}

example evaluates_a_sum for evaluate {
  evaluate(
    expression: Expr.Add(
      left: Expr.Number(value: 2),
      right: Expr.Add(left: Expr.Number(value: 3), right: Expr.Zero),
    ),
  )
    == 5
}
```

Writing the name of a case with fields without calling it yields a **function value** that has the labels of those fields. For example, `Msg.Typed` is `fn(text: String) -> Msg`. It can be passed as a function as it is, as in `list.map(items: texts, apply: Msg.Typed)` (the same applies to `Some`, `Ok`, and `Err`). Using a case with fields without a call where a value of the variant is expected (for example `Msg.Typed` where a `Msg` is needed), or adding `()` to a case without fields, is `E0301`.

> Note: A recursive variant or record cannot have, as a field, a function that takes a value of its own recursive group as an argument (`E0602`). If it could, recursion through functions would become invisible to the termination check ([13 Termination](13-termination.md)).

## Patterns

The following forms can be written as the pattern of a `match` case.

| Pattern | Meaning |
| --- | --- |
| `_` | Matches anything and binds no value |
| Name | Matches anything and binds the value to that name |
| Literal | `0`, `"x"`, `true`, and so on. Matches when the values are equal |
| `Type.Case(field: pattern, ...)` | A variant case. List **all fields in declaration order**, using `_` for those that are not used |
| `Type.Case` | A case without fields |
| `Type(field: pattern, ...)` | A record. All fields are likewise listed |
| `Some(value: p)`, `None`, `Ok(value: p)`, `Err(error: p)` | Cases of `Option` and `Result` |

Patterns can be nested. There are no list or tuple patterns (`[]`, `[x, ...]`). Case analysis on lists is written with `if xs.length == 0 { ... }` and `xs[0]`. Writing a case name without the type name is error `E0201`, and the message shows the correct form (`Light.Red`).

```cernith
module guide.patterns;

pub record Point {
  x: Int,
  y: Int,
}

pub fn describe(point: Point) -> String {
  match point {
    Point(x: 0, y: 0) => "origin",
    Point(x: _, y: 0) => "on the x axis",
    Point(x: 0, y: _) => "on the y axis",
    Point(x: _, y: _) => "elsewhere",
  }
}

example describes_points for describe {
  describe(point: Point(x: 0, y: 0)) == "origin"
    and describe(point: Point(x: 3, y: 0)) == "on the x axis"
    and describe(point: Point(x: 0, y: 2)) == "on the y axis"
    and describe(point: Point(x: 1, y: 1)) == "elsewhere"
}
```

```cernith-error E0201
module guide.patterns;

pub variant Light {
  Green,
  Red,
}

pub fn go(light: Light) -> Bool {
  match light {
    Green => true,
    Red => false,
  }
}
```

## Exhaustiveness

A `match` must cover all values of the scrutinee. If it does not, the error is `E0305` (non-exhaustive-match), and the diagnostic shows one example of an uncovered value, in the way it would be written in source (`Light.Yellow`, `Some(value: false)`).

- Variants, records, `Bool`, `Option`, and `Result` are checked completely, regardless of the depth of nesting.
- Literals of `Int` and `String` cannot cover the whole type. A `match` with only literal patterns needs a `_` or a name case.

There is no treatment in which an uncovered case fails at run time; it is rejected at compile time (check time). As a result, when a case is added to a variant, every place that matches on that variant is found by the check.

```cernith-error E0305
module guide.exhaustive;

pub variant Light {
  Green,
  Red,
  Yellow,
}

pub fn go(light: Light) -> Bool {
  match light {
    Light.Green => true,
    Light.Red => false,
  }
}
```

```cernith
module guide.exhaustive;

pub fn name_of(number: Int) -> String {
  match number {
    0 => "zero",
    1 => "one",
    _ => "many",
  }
}

example name_of_names_small_numbers for name_of {
  name_of(number: 0) == "zero" and name_of(number: 9) == "many"
}
```

## Related

- Refined types and `distinct` types: [08 Refined types and distinct types](08-refined-and-distinct-types.md)
- Handling `Option` and `Result`: [12 Errors and defects](12-errors-and-defects.md)
- Generics: [06 Functions](06-functions.md)
