# std.codec

This module provides a common data representation and two interfaces for converting to and from it: `Encode` and `Decode`. Import it with `import std.codec;` and call the methods as `codec.encode(value: x)` and `codec.decode::<T>(value: v)`.

The common representation is `Value`. A text format implemented by a tool or library needs only one conversion between its text and `Value`. Any type with `Encode` and `Decode` impls can then use that format. For records and variants, `impl codec.Encode for T derive;` generates the impl. Both interfaces are described in [22 Interfaces](../language/22-interfaces.md).

## Value

`Value` is a variant. The cases are written with the module name and the type name: `codec.Value.Int(value: 1)`.

| Case | Holds |
| --- | --- |
| `Null` | No value |
| `Bool(value)` | A `Bool` |
| `Int(value)` | An `Int` |
| `Float(value)` | A `Float64` |
| `Text(value)` | A `String` |
| `List(items)` | A `List<Value>` |
| `Record(fields)` | A `List<Field>`, the fields of a record in declaration order |
| `Case(name, fields)` | The name of a case of a variant and its `List<Field>` |

`Field` is a record of `name: String` and `value: Value`. `DecodeError` is a record of `path: String` and `message: String`. `CaseParts` is a record of `name: String` and `fields: List<Field>`.

## Interfaces

```text
pub interface Encode {
  fn encode(value: Self) -> Value;
}

pub interface Decode {
  fn decode(value: Value) -> Result<Self, DecodeError>;
}
```

In `Decode`, `Self` occurs only in the result, so a call names the type: `codec.decode::<Point>(value: v)`. Without it the type must come from the context, or the error is `E0320` ([22 Interfaces](../language/22-interfaces.md)).

## Types that have impls

| Type | `Encode` | `Decode` |
| --- | --- | --- |
| `Bool`, `Int`, `Float64`, `String`, `Unit` | Yes | Yes |
| `List<T>` | Yes, when `T` has it | Yes, when `T` has it |
| `Option<T>` | Yes, when `T` has it | Yes, when `T` has it |
| `Result<T, E>` | Yes, when `T` and `E` have it | No |

## Shapes

| Value | Encoded as |
| --- | --- |
| `Bool`, `Int`, `Float64`, `String` | `Bool`, `Int`, `Float`, `Text` |
| `List<T>` | `List`, with the encoding of each element |
| A record | `Record`, with one `Field` for each field, in declaration order |
| A case of a variant | `Case`, with the name of the case and one `Field` for each field of the case |
| `None` and `()` | `Null` |
| `Some(value: x)` | The encoding of `x` |
| `Ok(value: x)` and `Err(error: x)` | `Case` named `Ok` with the field `value`, and `Case` named `Err` with the field `error` |

- `Some(value: None)` is encoded like `None`, so it cannot be told apart after decoding. A format that needs the difference can use a record.
- Numeric types are not converted automatically. Decoding a `Float64` requires a `Float`; an `Int` value produces an error.
- Decoding an `Option<T>` gives `None` for `Null`, and `Some(value: x)` for any other value that decodes as a `T`.

## Derived impls

`impl codec.Encode for Point derive;` and `impl codec.Decode for Point derive;` write the methods for a record or a variant whose fields all have the interface. A generic type needs the bound on its parameters. If a field does not have the interface, the error is `E1404` ([22 Interfaces](../language/22-interfaces.md)).

The derived `decode` reads each field by name and returns the first error. The `path` of a `DecodeError` says where the error is: `.marks[1]` is the item at index 1 of the field `marks`. The `message` says what was expected and what was found.

| Problem | `path` | `message` |
| --- | --- | --- |
| A field is missing | (empty) | `there is no field name` |
| An item has the wrong type | `.marks[1]` | `expected an Int, found a String` |
| The case name is unknown | (empty) | `Shape has no case Circle` |

To use a different representation, implement both methods explicitly. The example below encodes a `Meters` value as a plain `Int`. `Meters` is a `distinct` type, for which impls cannot be derived.

## Function list

| Function | Return type | Meaning |
| --- | --- | --- |
| `codec.encode(value)` | `Value` | The method of `Encode` |
| `codec.decode::<T>(value)` | `Result<T, DecodeError>` | The method of `Decode` |
| `codec.field::<T>(fields, name)` | `Result<T, DecodeError>` | The field `name` decoded as a `T`, with the name added to the path of an error |
| `codec.record_fields(value, type_name)` | `Result<List<Field>, DecodeError>` | The fields of a `Record`, or an error for another value |
| `codec.case_parts(value, type_name)` | `Result<CaseParts, DecodeError>` | The name and fields of a `Case`, or an error for another value |
| `codec.unknown_case(type_name, name)` | `DecodeError` | The error for a case name that the variant does not have |
| `codec.describe(value)` | `String` | A short description of the kind of a `Value`, for messages |

The last five functions are the helpers that the derived `decode` is built from. They are public so that a hand-written `decode` can use them too.

## Function details

### codec.field

```text
pub fn field<T: Decode>(fields: List<Field>, name: String) -> Result<T, DecodeError>
```

Finds the first field called `name` and decodes its value as a `T`. If the decoding fails, the `path` of the error is `.name` followed by the path inside the value. If there is no such field, the error is `there is no field name` with an empty path. Write the type: `codec.field::<Int>(fields: fields, name: "x")`.

### codec.record_fields and codec.case_parts

```text
pub fn record_fields(value: Value, type_name: String) -> Result<List<Field>, DecodeError>
pub fn case_parts(value: Value, type_name: String) -> Result<CaseParts, DecodeError>
```

Take the fields out of a `Record`, or the name and fields out of a `Case`. For any other value the error says what was expected and what was found, for example `expected a record Point, found null` or `expected a case of Shape, found null`. `type_name` is used only in that message.

### codec.unknown_case

```text
pub fn unknown_case(type_name: String, name: String) -> DecodeError
```

The error `Shape has no case Circle` (for `type_name: "Shape"` and `name: "Circle"`), with an empty path.

### codec.describe

```text
pub fn describe(value: Value) -> String
```

Returns `null`, `a Bool`, `an Int`, `a Float64`, `a String`, `a list`, `a record`, or `the case ` followed by the name of the case.

## Examples

Derived impls, and a hand-written pair for a `distinct` type:

```cernith
module guide.wire;

import std.codec;

pub type Meters = distinct Int;

pub record Point {
  x: Int,
  y: Int,
}

pub variant Shape {
  Dot(at: Point),
  Empty,
}

pub record Tagged {
  name: Option<String>,
  marks: List<Int>,
}

impl codec.Decode for Meters {
  fn decode(value: codec.Value) -> Result<Meters, codec.DecodeError> {
    match codec.decode::<Int>(value: value) {
      Ok(value: number) => Ok(value: Meters(value: number)),
      Err(error: failure) => Err(error: failure),
    }
  }
}

impl codec.Decode for Point derive;

impl codec.Decode for Shape derive;

impl codec.Decode for Tagged derive;

impl codec.Encode for Meters {
  fn encode(value: Meters) -> codec.Value {
    codec.encode(value: value.value)
  }
}

impl codec.Encode for Point derive;

impl codec.Encode for Shape derive;

impl codec.Encode for Tagged derive;

pub fn decode_meters(value: codec.Value) -> Result<Meters, codec.DecodeError> {
  codec.decode::<Meters>(value: value)
}

example meters_are_read_from_ints for decode_meters {
  decode_meters(value: codec.Value.Int(value: 5)) == Ok(value: Meters(value: 5))
    and decode_meters(value: codec.Value.Null)
      == Err(
        error: codec.DecodeError(
          path: "",
          message: "expected an Int, found null",
        ),
      )
}

pub fn decode_shape(value: codec.Value) -> Result<Shape, codec.DecodeError> {
  codec.decode::<Shape>(value: value)
}

example decode_names_an_unknown_case for decode_shape {
  decode_shape(value: codec.Value.Case(name: "Circle", fields: []))
    == Err(
      error: codec.DecodeError(path: "", message: "Shape has no case Circle"),
    )
}

example decode_reads_what_encode_wrote for decode_shape {
  decode_shape(value: encode_shape(shape: Shape.Dot(at: Point(x: 1, y: 2))))
    == Ok(value: Shape.Dot(at: Point(x: 1, y: 2)))
}

pub fn decode_tagged(value: codec.Value) -> Result<Tagged, codec.DecodeError> {
  codec.decode::<Tagged>(value: value)
}

example decode_names_the_path_of_a_bad_item for decode_tagged {
  decode_tagged(
    value: codec.Value.Record(
      fields: [
        codec.Field(name: "name", value: codec.Value.Null),
        codec.Field(
          name: "marks",
          value: codec.Value.List(
            items: [codec.Value.Int(value: 1), codec.Value.Text(value: "x")],
          ),
        ),
      ],
    ),
  )
    == Err(
      error: codec.DecodeError(
        path: ".marks[1]",
        message: "expected an Int, found a String",
      ),
    )
}

example decode_reports_a_missing_field for decode_tagged {
  decode_tagged(value: codec.Value.Record(fields: []))
    == Err(
      error: codec.DecodeError(path: "", message: "there is no field name"),
    )
}

pub fn encode_meters(meters: Meters) -> codec.Value {
  codec.encode(value: meters)
}

example meters_are_written_as_ints for encode_meters {
  encode_meters(meters: Meters(value: 5)) == codec.Value.Int(value: 5)
}

pub fn encode_point(point: Point) -> codec.Value {
  codec.encode(value: point)
}

example a_record_is_a_record_of_its_fields for encode_point {
  encode_point(point: Point(x: 1, y: 2))
    == codec.Value.Record(
      fields: [
        codec.Field(name: "x", value: codec.Value.Int(value: 1)),
        codec.Field(name: "y", value: codec.Value.Int(value: 2)),
      ],
    )
}

pub fn encode_shape(shape: Shape) -> codec.Value {
  codec.encode(value: shape)
}

example a_case_has_its_name_and_fields for encode_shape {
  encode_shape(shape: Shape.Empty)
    == codec.Value.Case(name: "Empty", fields: [])
    and encode_shape(shape: Shape.Dot(at: Point(x: 0, y: 5)))
      == codec.Value.Case(
        name: "Dot",
        fields: [
          codec.Field(
            name: "at",
            value: encode_point(point: Point(x: 0, y: 5)),
          ),
        ],
      )
}

pub fn encode_tagged(tagged: Tagged) -> codec.Value {
  codec.encode(value: tagged)
}

example none_is_null_and_some_is_the_value for encode_tagged {
  encode_tagged(tagged: Tagged(name: None, marks: []))
    == codec.Value.Record(
      fields: [
        codec.Field(name: "name", value: codec.Value.Null),
        codec.Field(name: "marks", value: codec.Value.List(items: [])),
      ],
    )
    and encode_tagged(tagged: Tagged(name: Some(value: "a"), marks: [7]))
      == codec.Value.Record(
        fields: [
          codec.Field(name: "name", value: codec.Value.Text(value: "a")),
          codec.Field(
            name: "marks",
            value: codec.Value.List(items: [codec.Value.Int(value: 7)]),
          ),
        ],
      )
}
```

A hand-written `Point` impl, using `codec.record_fields` and `codec.field`:

```cernith
module guide.handwritten;

import std.codec;

pub record Point {
  x: Int,
  y: Int,
}

impl codec.Decode for Point {
  fn decode(value: codec.Value) -> Result<Point, codec.DecodeError> {
    let fields = codec.record_fields(value: value, type_name: "Point")?;
    let x = codec.field::<Int>(fields: fields, name: "x")?;
    let y = codec.field::<Int>(fields: fields, name: "y")?;
    Ok(value: Point(x: x, y: y))
  }
}

impl codec.Encode for Point {
  fn encode(value: Point) -> codec.Value {
    codec.Value.Record(
      fields: [
        codec.Field(name: "x", value: codec.encode(value: value.x)),
        codec.Field(name: "y", value: codec.encode(value: value.y)),
      ],
    )
  }
}

pub fn read_point(value: codec.Value) -> Result<Point, codec.DecodeError> {
  codec.decode::<Point>(value: value)
}

example read_point_reads_the_fields for read_point {
  read_point(value: codec.encode(value: Point(x: 3, y: 4)))
    == Ok(value: Point(x: 3, y: 4))
}

example read_point_reports_what_is_wrong for read_point {
  read_point(value: codec.Value.Null)
    == Err(
      error: codec.DecodeError(
        path: "",
        message: "expected a record Point, found null",
      ),
    )
    and read_point(
      value: codec.Value.Record(
        fields: [codec.Field(name: "x", value: codec.Value.Int(value: 1))],
      ),
    )
      == Err(error: codec.DecodeError(path: "", message: "there is no field y"))
}
```

## Related

- [22 Interfaces](../language/22-interfaces.md): interfaces, `impl`, `derive`, and bounds
- [show](show.md): `Show`, the interface for text
- [07 Records and variants](../language/07-records-and-variants.md)
