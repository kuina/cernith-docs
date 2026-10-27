# std.show

値をテキストにするインターフェース `Show` と、テキストを扱う 2 つの関数のモジュールです。使うには `import std.show;` と書き、メソッドを `show.text(value: x)` のように呼びます。

値のテキストは、その値を作る Cernith のコードです：`Point(x: 1, y: 2)`、`Some(value: "a")`、`[1, 2]`。読む人は、これをそのまま `example` に貼り付けられます。`Show` はインターフェースなので（[22 インターフェース](../language/22-interfaces.md)）、関数は `fn describe<T: show.Show>(value: T) -> String` のように、境界で求められます。

## インターフェース

```text
pub interface Show {
  fn text(value: Self) -> String;

  law stable(value: Self) {
    text(value: value) == text(value: value)
  }
}
```

法則は、値のテキストが 1 つであることを述べます。`cernith test` は、テストするモジュールにある `Show` の `impl` のうち、型パラメータを持たないものすべてについて、この法則を実行します。

## Show の impl がある型

| 型 | テキスト |
| --- | --- |
| `Bool` | `true`、`false` |
| `Int` | `-3` |
| `Float64` | `2.5`、`1.0`（`float.to_text` が書くとおり） |
| `String` | 二重引用符で囲んだ文字列。`show.quote` で書く |
| `Unit` | `()` |
| `List<T>` | `[1, 2]`（要素のテキストを `, ` で区切る）。`T` が `Show` を持つとき |
| `Option<T>` | `Some(value: 1)`、`None`。`T` が `Show` を持つとき |
| `Result<T, E>` | `Ok(value: 1)`、`Err(error: "bad")`。`T` と `E` が `Show` を持つとき |

制約付き型は、土台の型の `impl` を使うので、`Percent` は `Int` と同じように書かれます。`distinct` 型には、自分の `impl` が必要です。表にない型（`Map` や `Set` など）には、このバージョンでは `Show` の `impl` がありません。

## 導出される impl

`impl show.Show for Point derive;` は、フィールドがすべて `Show` を持つレコードまたはバリアントのメソッドを書きます。持たないフィールドがあると `E1404` で、メッセージが、その最初のフィールドを示します（[22 インターフェース](../language/22-interfaces.md)）。

| 値 | テキスト |
| --- | --- |
| レコード | `名前(フィールド: テキスト, フィールド: テキスト)`。フィールドは宣言の順 |
| フィールドのあるバリアントのケース | `型.ケース(フィールド: テキスト, ...)` |
| フィールドのないケース | `型.ケース` |

ジェネリックな型には、パラメータに境界が必要です：`impl<T: show.Show> show.Show for Box<T> derive;`。ほかのテキストにしたい型は、`text` を自分で書きます。

## 関数一覧

| 関数 | 戻り値の型 | 意味 |
| --- | --- | --- |
| `show.text(value)` | `String` | `value` のテキスト（`Show` のメソッド） |
| `show.join_texts(items, separator)` | `String` | 要素のテキストを、区切りでつないだもの |
| `show.quote(value)` | `String` | `value` を二重引用符で囲み、エスケープしたもの |

## 関数の詳細

### show.join_texts

```text
pub fn join_texts<T: Show>(items: List<T>, separator: String) -> String
```

各要素を `text` で書き、間に `separator` を入れます。リストが空なら、結果は空文字列です。角括弧を含むリスト自身のテキストは、`show.text(value: items)` です。

### show.quote

```text
pub fn quote(value: String) -> String
```

`value` を二重引用符で囲みます。バックスラッシュ、二重引用符、改行、タブ、復帰は、文字列リテラルと同じく `\\`、`\"`、`\n`、`\t`、`\r` と書かれます。ほかの文字はそのままです。

## 例

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

## 関連

- [22 インターフェース](../language/22-interfaces.md)：インターフェース、`impl`、`derive`、境界
- [codec](codec.md)：データのためのインターフェース `Encode` と `Decode`
- [int](int.md)、[float](float.md)：`int.to_text` と `float.to_text`
