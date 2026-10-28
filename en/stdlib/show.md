# std.show

This module provides `Show`, an interface for converting values to text, plus two text helpers. Import it with `import std.show;` and call `show.text(value: x)`.

A value is displayed as the Cernith code that constructs it, such as `Point(x: 1, y: 2)`, `Some(value: "a")`, or `[1, 2]`. You can paste this text into an `example`. Because `Show` is an interface ([22 Interfaces](../language/22-interfaces.md)), functions can require it as a bound: `fn describe<T: show.Show>(value: T) -> String`.

## Interface

```text
pub interface Show {
  fn text(value: Self) -> String;

  law stable(value: Self) {
    text(value: value) == text(value: value)
  }
}
```

The law requires each value to have a consistent text representation. `cernith test` checks it for every non-generic `Show` impl in the module under test.

## Types that have a Show impl

| Type | Text |
| --- | --- |
| `Bool` | `true`, `false` |
| `Int` | `-3` |
| `Float64` | `2.5`, `1.0` (as written by `float.to_text`) |
| `String` | the string between double quotes, written with `show.quote` |
| `Unit` | `()` |
| `List<T>` | `[1, 2]` (the texts of the elements, separated by `, `), when `T` has `Show` |
| `Option<T>` | `Some(value: 1)`, `None`, when `T` has `Show` |
| `Result<T, E>` | `Ok(value: 1)`, `Err(error: "bad")`, when `T` and `E` have `Show` |

A refined type uses the `impl` of its base type, so a `Percent` is written like an `Int`. A `distinct` type needs its own `impl`. The types that are not in the table, such as `Map` and `Set`, have no `Show` impl in this version.

## Derived impls

`impl show.Show for Point derive;` writes the method for a record or a variant whose fields all have `Show`. If a field does not, the error is `E1404` and the message names the first such field ([22 Interfaces](../language/22-interfaces.md)).

| Value | Text |
| --- | --- |
| A record | `Name(field: text, field: text)`, the fields in declaration order |
| A case of a variant with fields | `Type.Case(field: text, ...)` |
| A case without fields | `Type.Case` |

A generic type needs the bound on its parameters: `impl<T: show.Show> show.Show for Box<T> derive;`. A type that wants another text writes `text` itself.

## Function list

| Function | Return type | Meaning |
| --- | --- | --- |
| `show.text(value)` | `String` | The text of `value` (the method of `Show`) |
| `show.join_texts(items, separator)` | `String` | The texts of the items, joined by the separator |
| `show.quote(value)` | `String` | `value` between double quotes, with escapes |

## Function details

### show.join_texts

```text
pub fn join_texts<T: Show>(items: List<T>, separator: String) -> String
```

Writes each item with `text` and puts `separator` between them. For an empty list the result is the empty string. The text of the list itself, with brackets, is `show.text(value: items)`.

### show.quote

```text
pub fn quote(value: String) -> String
```

Puts `value` between double quotes. The backslash, the double quote, the newline, the tab, and the carriage return are written as `\\`, `\"`, `\n`, `\t`, and `\r`, as in a string literal. Other characters are kept.

## Example

```cernith
module guide.shows;

import std.show;

pub record Box<T> {
  item: T,
  tags: List<String>,
}

pub record Flag {
  on: Bool,
  ratio: Float64,
  note: Option<String>,
  outcome: Result<Int, String>,
}

pub type Percent = Int where self >= 0 and self <= 100;

pub record Point {
  x: Int,
  y: Int,
}

pub variant Shape {
  Dot(at: Point),
  Line(from: Point, to: Point),
  Empty,
}

impl<T: show.Show> show.Show for Box<T> derive;

impl show.Show for Flag derive;

impl show.Show for Point derive;

impl show.Show for Shape derive;

pub fn describe<T: show.Show>(value: T) -> String {
  show.text(value: value)
}

example a_case_is_written_with_its_type for describe {
  describe(value: Shape.Empty) == "Shape.Empty"
    and describe(value: Shape.Dot(at: Point(x: 0, y: 0)))
      == "Shape.Dot(at: Point(x: 0, y: 0))"
    and describe(
      value: Shape.Line(from: Point(x: 0, y: 0), to: Point(x: 3, y: 4)),
    )
      == "Shape.Line(from: Point(x: 0, y: 0), to: Point(x: 3, y: 4))"
}

example a_generic_record_needs_the_bound_of_its_parameter for describe {
  describe(value: Box(item: Some(value: Shape.Empty), tags: ["a\n", "q\"b"]))
    == "Box(item: Some(value: Shape.Empty), tags: [\"a\\n\", \"q\\\"b\"])"
}

example a_record_lists_its_fields for describe {
  describe(value: Point(x: 1, y: -2)) == "Point(x: 1, y: -2)"
}

example fields_of_every_kind for describe {
  describe(
    value: Flag(on: true, ratio: 2.5, note: None, outcome: Err(error: "bad")),
  )
    == "Flag(on: true, ratio: 2.5, note: None, outcome: Err(error: \"bad\"))"
}

example lists_options_and_results for describe {
  describe(value: [1, 2]) == "[1, 2]"
    and describe(value: [Some(value: 1), None]) == "[Some(value: 1), None]"
    and describe(value: [Ok(value: 1), Err(error: "bad")])
      == "[Ok(value: 1), Err(error: \"bad\")]"
}

example the_basic_types for describe {
  describe(value: true) == "true"
    and describe(value: -3) == "-3"
    and describe(value: 2.5) == "2.5"
    and describe(value: 1.0) == "1.0"
    and describe(value: "a\"b\n") == "\"a\\\"b\\n\""
    and describe(value: ()) == "()"
}

pub fn names(shapes: List<Shape>) -> String {
  show.join_texts(items: shapes, separator: "; ")
}

example join_texts_puts_the_separator_between for names {
  names(shapes: [Shape.Empty, Shape.Dot(at: Point(x: 1, y: 1))])
    == "Shape.Empty; Shape.Dot(at: Point(x: 1, y: 1))"
}

pub fn percent_text(percent: Percent) -> String {
  show.text(value: percent)
}

example a_refined_type_uses_the_text_of_its_base for percent_text {
  percent_text(percent: Percent(value: 7)) == "7"
}

pub fn quoted(text: String) -> String {
  show.quote(value: text)
}

example quote_escapes_four_things for quoted {
  quoted(text: "a\\b\"c\nd\te") == "\"a\\\\b\\\"c\\nd\\te\""
    and quoted(text: "plain") == "\"plain\""
}
```

## Related

- [22 Interfaces](../language/22-interfaces.md): interfaces, `impl`, `derive`, and bounds
- [codec](codec.md): `Encode` and `Decode`, the interfaces for data
- [int](int.md), [float](float.md): `int.to_text` and `float.to_text`
