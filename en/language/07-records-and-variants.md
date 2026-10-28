# Records and variants

You can define two kinds of compound types: **records**, which group named fields, and **variants**, whose values belong to exactly one of several cases. There are no tuples. Both records and variants are immutable and nominal: types with different names are distinct even when their structures match.

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

Use `where` to attach an invariant to a record. The condition refers to fields directly by name.

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

An invariant is a specification expression. Functions that have effects cannot be called in it ([09 Contracts](09-contracts.md)). A condition written only with comparisons of integers or floating-point numbers, `.length`, and `and`, `or`, and `not` has a simple form that the checker can readily use as an assumption. A call to a pure function of your own is accepted in the condition, but the checker cannot prove it when a record is built (warning `W0504`): the condition is then checked at run time. Prefer the simple form where it is enough.

## Variants

```text
[pub] variant name<type parameters> {
  case,
  case(field name: type, ...),
  ...
}
```

A value of a variant is exactly one of its cases. A case can have fields. A variant may be recursive or generic.

Refer to a case as `variant name.case name`. Construct a case with fields using a call; write a case without fields without parentheses. For a type in another module, use `module alias.type name.case name`. The prelude cases `Some`, `None`, `Ok`, and `Err` are the only exceptions: they need no type prefix.

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

### Or-patterns

An or-pattern joins alternatives with `|` so they share a single arm body. It can appear as an arm's pattern or within a constructor field pattern (`Some(value: Light.Red | Light.Yellow)`). The following rules apply.

- Every alternative binds the same names, each at the same type. A name that one alternative lacks or adds is `E0350`; a name with a different type in another alternative is `E0301`. The body sees each name once.
- An alternative is a literal or a case. `_` and a bare name match every value and would leave the other alternatives dead, so they are `E0351`.
- The alternatives are written in strictly increasing order: literals by value (`false` before `true`), cases in the order of the declaration of the type (`Red` before `Yellow`; `Some` before `None`; `Ok` before `Err`). A repeated alternative and a wrong order are `E0351`, and the message gives the order.
- The exhaustiveness check takes each alternative as one row.
- There are no pattern guards (`if` in an arm), and a `for` head takes no alternatives.

```cernith
module guide.or_patterns;

pub variant Light {
  Red,
  Yellow,
  Green,
}

pub variant Shape {
  Circle(radius: Int),
  Square(side: Int),
  Dot,
}

pub fn is_small(number: Int) -> Bool {
  match number {
    -1 | 0 | 1 => true,
    _ => false,
  }
}

example is_small_takes_literals for is_small {
  is_small(number: 0) and is_small(number: -1) and not is_small(number: 2)
}

pub fn must_stop(light: Light) -> Bool {
  match light {
    Light.Red | Light.Yellow => true,
    Light.Green => false,
  }
}

example must_stop_joins_two_cases for must_stop {
  must_stop(light: Light.Red)
    and must_stop(light: Light.Yellow)
    and not must_stop(light: Light.Green)
}

pub fn size(shape: Shape) -> Int {
  match shape {
    Shape.Circle(radius: n) | Shape.Square(side: n) => n,
    Shape.Dot => 0,
  }
}

example size_binds_the_same_name for size {
  size(shape: Shape.Circle(radius: 3)) == 3
    and size(shape: Shape.Square(side: 4)) == 4
    and size(shape: Shape.Dot) == 0
}

pub fn stop_in_option(light: Option<Light>) -> Bool {
  match light {
    Some(value: Light.Red | Light.Yellow) => true,
    Some(value: Light.Green) => false,
    None => false,
  }
}

example stop_in_option_uses_a_field_pattern for stop_in_option {
  stop_in_option(light: Some(value: Light.Yellow))
    and not stop_in_option(light: None)
}
```

`Shape.Dot` binds no name, so it cannot be joined with `Shape.Circle(radius: n)` (`E0350`).

```cernith-error E0350
module guide.or_patterns;

pub variant Shape {
  Circle(radius: Int),
  Dot,
}

pub fn size(shape: Shape) -> Int {
  match shape {
    Shape.Circle(radius: n) | Shape.Dot => n,
  }
}
```

The alternatives must follow the declaration. Here `Light.Yellow` is written before `Light.Red`.

```cernith-error E0351
module guide.or_patterns;

pub variant Light {
  Red,
  Yellow,
  Green,
}

pub fn must_stop(light: Light) -> Bool {
  match light {
    Light.Yellow | Light.Red => true,
    Light.Green => false,
  }
}
```

## Exhaustiveness

A `match` must cover all values of the scrutinee. If it does not, the error is `E0305` (non-exhaustive-match), and the diagnostic shows one example of an uncovered value, in the way it would be written in source (`Light.Yellow`, `Some(value: false)`).

- Variants, records, `Bool`, `Option`, and `Result` are checked completely, regardless of the depth of nesting.
- Literals of `Int` and `String` cannot cover the whole type. A `match` with only literal patterns needs a `_` or a name case.
- Each alternative of an or-pattern counts as one row. `Light.Red | Light.Yellow` alone does not cover `Light.Green` (`E0305`).

Missing cases are rejected at compile time, rather than left to fail at run time. When you add a variant case, the checker identifies matches that no longer cover every case.

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
