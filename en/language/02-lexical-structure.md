# Lexical structure

Source files are written in UTF-8 without a BOM. The lexer cuts out tokens from left to right, taking the longest match. Whitespace only separates tokens and carries no meaning. However, a tab character outside a string literal and a carriage return (CRLF line ends) are lexical errors (`E0001`): indent with spaces and end lines with LF. The syntax does not depend on formatting (line breaks and indentation), so any formatting yields the same syntax tree as far as the grammar is concerned. The rules for formatting are defined separately ([19 Canonical form](19-canonical-form.md)).

All lexical errors are reported with the code `E0001` (syntax-error).

## Identifiers

| Kind | Form | Where it is used |
| --- | --- | --- |
| Lowercase name | `[a-z][a-z0-9]*(_[a-z0-9]+)*` | Values, functions, fields, parameters, labels, modules, variables |
| Uppercase name | `[A-Z][A-Za-z0-9]*` | Types, records, variants and their cases, effect parameters |

A lowercase name is in snake_case and may use only lowercase ASCII letters and digits. A `_` cannot appear at the start or end of a name, nor twice in a row. An uppercase name is in PascalCase and cannot contain `_`. This distinction makes it possible to tell from the name alone whether it denotes a value or a type.

```cernith-error E0001
module guide.names;

pub fn My_fn() -> Int {
  1
}
```

A keyword (described below) cannot be used as a name.

```cernith-error E0001
module guide.names;

pub fn f(unit: Int) -> Int {
  unit
}
```

The name `_` (a single underscore) stands for an unused value in patterns and loop variables ([05 Statements and bindings](05-statements.md), [07 Records and variants](07-records-and-variants.md)).

## Literals

| Kind | Notation | Examples |
| --- | --- | --- |
| Integer (`Int`) | `0`, or a string of decimal digits that does not start with `0` | `0`, `42`, `12345678901234567890` |
| Floating-point number (`Float64`) | An integer part, `.`, a fractional part of one or more digits, and an optional exponent | `1.5`, `0.25`, `1.0e21`, `2.5E-3` |
| String (`String`) | Enclosed in `"` | `"héllo"` |
| Boolean (`Bool`) | `true`, `false` | |
| Unit (`Unit`) | `()` | |

- Integers are written in decimal only. Digit separators, leading `0`s, and suffixes cannot be used. A negative number is written with the unary `-` (`-7`). `Int` has no upper limit on the number of digits.
- A floating-point number requires `.` and at least one digit of the fractional part. `1e5`, `1.`, and `.5` are not literals. The exponent is `e` or `E` followed by an optional sign and a string of digits. A literal whose value is outside the range of `Float64` (for example `1.0e400`) is `E0001`.
- Within a string, the escapes `\\`, `\"`, `\n`, `\t`, `\r`, and `\u{hex digits}` can be used. Any other use of `\` is an error. A line break cannot be written directly inside a string literal (a line break is `\n`). Any other Unicode scalar value can be written as it is.

```cernith
module guide.literals;

pub fn answer() -> Int {
  42
}

example answer_is_42 for answer {
  answer() == 42
}

pub fn big() -> Int {
  12345678901234567890
}

example big_is_big for big {
  big() == 12345678901234567890
}

pub fn greeting() -> String {
  "tab:\t quote:\" backslash:\\ héllo"
}

example greeting_length for greeting {
  greeting().length == 31
}

pub fn nothing() -> Unit {
  ()
}

example nothing_is_unit for nothing {
  nothing() == ()
}

pub fn ratio() -> Float64 {
  1500.0
}

example ratio_is_1500 for ratio {
  ratio() == 1500.0
}
```

Each of the following examples is rejected as a lexical error.

```cernith-error E0001
module guide.literals;

pub fn f() -> Float64 {
  1e5
}
```

```cernith-error E0001
module guide.literals;

pub fn f() -> Int {
  007
}
```

```cernith-error E0001
module guide.literals;

pub fn f() -> String {
  "a\qb"
}
```

> Note: Floating-point numbers handle only finite values. An operation that would produce infinity or NaN is a defect ([03 Types](03-types.md)).

## Comments

A comment runs from `//` to the end of the line. There are no block comments. A comment has no meaning and does not affect the behavior of the program.

A comment can be written immediately before a definition, an `import`, a statement, a clause, a field, a case of a variant, or a case of a `match`; at the end of a block; and at the end of the file. It cannot be written in the middle of an expression or in the middle of a parameter list. A comment written at the end of a line of code is moved, in canonical form, to the line immediately before the element that follows ([19 Canonical form](19-canonical-form.md)).

```cernith
module guide.comments;

// double it.
pub fn double(value: Int) -> Int {
  // write it with addition.
  value + value
}

example doubles for double {
  double(value: 4) == 8
}
```

## Keywords

The following words are keywords and cannot be used as names.

```text
module import as pub type distinct where record variant const fn spec
foreign unit uses requires ensures decreases invariant let var return
discard assert if else match for in while example property and or not
implies forall exists true false self effect resource release using
await capability manifest
```

`unit` declares a unit of measure ([23 Units of measure](23-units-of-measure.md)). `interface`, `impl`, `law`, and `derive` are not keywords; they have a meaning only where an item, an interface member, or the end of an `impl` header is expected. `self` can be written only inside the `where` clause of a type ([09 Contracts](09-contracts.md)). `async` and `scope` are not keywords. `async` is used as the name of an effect (`uses async`), and `scope` as the name of a function in `std.async` ([18 Concurrency](18-concurrency.md)).

`result` is not a keyword either. It denotes the return value of the function only inside an `ensures` clause. Anywhere else it is an ordinary name (a local variable may be called `result`; however, `result` inside an `ensures` clause is always the return value).

## Symbols

```text
( ) [ ] { } , : ; . .. ..= :: -> => = == != < <= > >= ++ + - * / ? | ^
```

If a `>` that closes type arguments is immediately followed by `=`, the two are read as `>=`. Put a space between `>` and `=`, as in `var xs: List<Int> = [];`. In canonical form, a space is always inserted.

```cernith-error E0001
module guide.wart;

pub fn f() -> Int {
  var xs: List<Int>= [];
  xs.length
}
```

## Prelude

The following names are always available without an `import`. They are not keywords.

| Kind | Names |
| --- | --- |
| Types | `Int`, `Float64`, `Bool`, `String`, `Unit`, `List`, `Option`, `Result`, `Bytes`, `Floats`, `Ints` |
| Variant cases | `Ok`, `Err`, `Some`, `None` |
| Built-in function | `hole` |

`hole(reason: "explanation")` is an expression that marks an unfinished place. It can be written in a position of any type. The checker notifies it with the warning `W0701` (hole-present), and if execution reaches it, it becomes a defect ([12 Errors and defects](12-errors-and-defects.md)).

The prelude contains only the types that literals, `?`, `match`, and operators refer to directly. Other types, such as `Map` and `Set`, are used from modules of the standard library, with the module name attached (`map.Map<String, Int>`).

## Related

- The diagnostic for a violation of the naming rules for identifiers is under `E0001` in [20 Diagnostic codes](20-diagnostics.md).
- The list of types is in [03 Types](03-types.md).
