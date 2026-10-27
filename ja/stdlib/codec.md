# std.codec

フォーマットから見たデータの形と、型の値をその形に変え、また戻す 2 つのインターフェース、`Encode` と `Decode` のモジュールです。使うには `import std.codec;` と書き、メソッドを `codec.encode(value: x)` や `codec.decode::<T>(value: v)` のように呼びます。

形は `Value` です。フォーマット（ツールやライブラリのテキスト形式）は、`Value` と自分のテキストとの変換を、一度だけ書きます。すると、`Encode` と `Decode` の `impl` を持つ型は、すべてそのフォーマットを持ちます。`impl codec.Encode for T derive;` は、レコードやバリアントの `impl` を書きます。2 つのインターフェースは、[22 インターフェース](../language/22-interfaces.md) で説明しています。

## Value

`Value` はバリアントです。ケースは、モジュール名と型名を付けて書きます：`codec.Value.Int(value: 1)`。

| ケース | 保持するもの |
| --- | --- |
| `Null` | 値がないこと |
| `Bool(value)` | `Bool` |
| `Int(value)` | `Int` |
| `Float(value)` | `Float64` |
| `Text(value)` | `String` |
| `List(items)` | `List<Value>` |
| `Record(fields)` | `List<Field>`。レコードのフィールドを宣言の順に並べたもの |
| `Case(name, fields)` | バリアントのケースの名前と、その `List<Field>` |

`Field` は、`name: String` と `value: Value` のレコードです。`DecodeError` は、`path: String` と `message: String` のレコードです。`CaseParts` は、`name: String` と `fields: List<Field>` のレコードです。

## インターフェース

```text
pub interface Encode {
  fn encode(value: Self) -> Value;
}

pub interface Decode {
  fn decode(value: Value) -> Result<Self, DecodeError>;
}
```

`Decode` では、`Self` が戻り値にしか現れないので、呼び出しは型を書きます：`codec.decode::<Point>(value: v)`。書かなければ、型が文脈から決まらなければならず、決まらなければ `E0320` です（[22 インターフェース](../language/22-interfaces.md)）。

## impl がある型

| 型 | `Encode` | `Decode` |
| --- | --- | --- |
| `Bool`、`Int`、`Float64`、`String`、`Unit` | あり | あり |
| `List<T>` | `T` が持つとき | `T` が持つとき |
| `Option<T>` | `T` が持つとき | `T` が持つとき |
| `Result<T, E>` | `T` と `E` が持つとき | なし |

## 形

| 値 | 変換先 |
| --- | --- |
| `Bool`、`Int`、`Float64`、`String` | `Bool`、`Int`、`Float`、`Text` |
| `List<T>` | `List`。各要素は、その要素の変換結果 |
| レコード | `Record`。フィールドごとに 1 つの `Field`。宣言の順 |
| バリアントのケース | `Case`。ケースの名前と、ケースのフィールドごとに 1 つの `Field` |
| `None` と `()` | `Null` |
| `Some(value: x)` | `x` の変換結果 |
| `Ok(value: x)` と `Err(error: x)` | フィールド `value` を持つ `Ok` という名前の `Case`、フィールド `error` を持つ `Err` という名前の `Case` |

- `Some(value: None)` は `None` と同じに変換されるので、読み戻したときに区別できません。区別が必要なフォーマットは、レコードを使えます。
- 数は広がりません。`Float64` を読むには `Float` が必要で、`Int` の値はエラーです。
- `Option<T>` を読むと、`Null` は `None` に、`T` として読めるほかの値は `Some(value: x)` になります。

## 導出される impl

`impl codec.Encode for Point derive;` と `impl codec.Decode for Point derive;` は、フィールドがすべてそのインターフェースを持つレコードまたはバリアントのメソッドを書きます。ジェネリックな型には、パラメータに境界が必要です。インターフェースを持たないフィールドがあると `E1404` です（[22 インターフェース](../language/22-interfaces.md)）。

導出される `decode` は、各フィールドを名前で読み、最初のエラーを返します。`DecodeError` の `path` は、エラーの場所を表します。`.marks[1]` は、フィールド `marks` の添字 1 の要素です。`message` は、何が期待され、何が見つかったかを述べます。

| 問題 | `path` | `message` |
| --- | --- | --- |
| フィールドがない | （空） | `there is no field name` |
| 要素の型が違う | `.marks[1]` | `expected an Int, found a String` |
| ケースの名前が未知 | （空） | `Shape has no case Circle` |

ほかの形が必要な型は、2 つのメソッドを自分で書きます。次の例は、導出できない `distinct` 型でそうしています。`Meters` を、ただの `Int` として書きます。

## 関数一覧

| 関数 | 戻り値の型 | 意味 |
| --- | --- | --- |
| `codec.encode(value)` | `Value` | `Encode` のメソッド |
| `codec.decode::<T>(value)` | `Result<T, DecodeError>` | `Decode` のメソッド |
| `codec.field::<T>(fields, name)` | `Result<T, DecodeError>` | フィールド `name` を `T` として読む。エラーの `path` に名前を加える |
| `codec.record_fields(value, type_name)` | `Result<List<Field>, DecodeError>` | `Record` のフィールド。ほかの値ならエラー |
| `codec.case_parts(value, type_name)` | `Result<CaseParts, DecodeError>` | `Case` の名前とフィールド。ほかの値ならエラー |
| `codec.unknown_case(type_name, name)` | `DecodeError` | バリアントにないケース名のエラー |
| `codec.describe(value)` | `String` | `Value` の種類の短い説明。メッセージ用 |

最後の 5 つは、導出される `decode` の部品です。手で書く `decode` でも使えるように、公開されています。

## 関数の詳細

### codec.field

```text
pub fn field<T: Decode>(fields: List<Field>, name: String) -> Result<T, DecodeError>
```

`name` という名前の最初のフィールドを探し、その値を `T` として読みます。読むのに失敗すると、エラーの `path` は、`.name` に値の中の `path` を続けたものです。そのようなフィールドがなければ、エラーは空の `path` と `there is no field name` です。型を書きます：`codec.field::<Int>(fields: fields, name: "x")`。

### codec.record_fields と codec.case_parts

```text
pub fn record_fields(value: Value, type_name: String) -> Result<List<Field>, DecodeError>
pub fn case_parts(value: Value, type_name: String) -> Result<CaseParts, DecodeError>
```

`Record` からフィールドを、`Case` から名前とフィールドを取り出します。ほかの値には、何が期待され、何が見つかったかを述べるエラーを返します。たとえば `expected a record Point, found null` や `expected a case of Shape, found null` です。`type_name` は、このメッセージにだけ使われます。

### codec.unknown_case

```text
pub fn unknown_case(type_name: String, name: String) -> DecodeError
```

エラー `Shape has no case Circle`（`type_name: "Shape"`、`name: "Circle"` のとき）を、空の `path` で返します。

### codec.describe

```text
pub fn describe(value: Value) -> String
```

`null`、`a Bool`、`an Int`、`a Float64`、`a String`、`a list`、`a record` のどれか、または `the case ` にケースの名前を続けたものを返します。

## 例

導出される `impl` と、`distinct` 型のために手で書いた 1 組です。

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

`codec.record_fields` と `codec.field` を使って手で書いた `Point` の `impl` です。

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

## 関連

- [22 インターフェース](../language/22-interfaces.md)：インターフェース、`impl`、`derive`、境界
- [show](show.md)：テキストのためのインターフェース `Show`
- [07 レコードとバリアント](../language/07-records-and-variants.md)
