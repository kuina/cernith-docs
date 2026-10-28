# std.bytes

This module creates, reads, and transforms `Bytes`, a sequence of integers from 0 to 255. `Bytes` is a prelude type with no literal syntax ([03 Types](../language/03-types.md)). Import the module with `import std.bytes;` and call its functions as in `bytes.length(bytes: data)`. The module alias and argument label are both spelled `bytes`, but serve different roles.

`Bytes` is immutable. `==` compares its length and elements; ordering comparisons are not supported. Functions such as `bytes.set` and `bytes.fill` return a modified copy rather than changing the original sequence. Positions are zero-based.

All functions are pure and have no effects. A call that does not satisfy a precondition (`requires`) is a defect ([12 Errors and defects](../language/12-errors-and-defects.md)). The functions that turn data coming from outside (strings, lists) into `Bytes` (`from_list`, `from_hex`, `from_base64`), and `to_text`, which turns `Bytes` back into a string, return `None` rather than a defect for input that cannot be converted.

## Function list

| Function | Return type | Meaning |
| --- | --- | --- |
| `bytes.empty()` | `Bytes` | A sequence of length 0 |
| `bytes.new(count, fill)` | `Bytes` | A sequence of `count` copies of the value `fill` |
| `bytes.from_list(items)` | `Option<Bytes>` | Builds from a `List<Int>`. `None` if any element is outside 0 to 255 |
| `bytes.to_list(bytes)` | `List<Int>` | The list of elements |
| `bytes.length(bytes)` | `Int` | The number of bytes |
| `bytes.get(bytes, index)` | `Int` | The byte at position `index` |
| `bytes.set(bytes, index, value)` | `Bytes` | A new sequence with the byte at `index` set to `value` |
| `bytes.fill(bytes, from, to, value)` | `Bytes` | A new sequence with a range filled with `value` |
| `bytes.slice(bytes, from, to)` | `Bytes` | The part from position `from` up to but not including `to` |
| `bytes.concat(first, second)` | `Bytes` | The two sequences joined |
| `bytes.concat_all(parts)` | `Bytes` | All the sequences in a list joined |
| `bytes.find(bytes, pattern)` | `Option<Int>` | The position where `pattern` first appears |
| `bytes.index_of(bytes, value)` | `Option<Int>` | The position where the byte `value` first appears |
| `bytes.starts_with(bytes, prefix)` | `Bool` | `true` if the sequence starts with `prefix` |
| `bytes.read_uint(bytes, at, width, big_endian)` | `Int` | Reads `width` bytes from position `at` as an unsigned integer |
| `bytes.read_int(bytes, at, width, big_endian)` | `Int` | Reads the same range as a two’s-complement signed integer |
| `bytes.uint_bytes(value, width, big_endian)` | `List<Int>` | Turns an unsigned integer into `width` bytes |
| `bytes.int_bytes(value, width, big_endian)` | `List<Int>` | Turns a signed integer into `width` two’s-complement bytes |
| `bytes.from_text(text)` | `Bytes` | The UTF-8 encoding of a string |
| `bytes.to_text(bytes)` | `Option<String>` | Decodes as UTF-8 into a string. `None` if invalid |
| `bytes.to_hex(bytes)` | `String` | A hexadecimal string |
| `bytes.from_hex(text)` | `Option<Bytes>` | Builds from a hexadecimal string. `None` if invalid |
| `bytes.to_base64(bytes)` | `String` | A Base64 string |
| `bytes.from_base64(text)` | `Option<Bytes>` | Builds from a Base64 string. `None` if invalid |

In the examples that follow, the right side of `→` is the value of the call. `→ defect` means that the call violates a precondition and is a defect. In the examples, a `Bytes` is written as a list such as `[1, 2]`; this is shorthand for “the `Bytes` built with `bytes.from_list(items: [1, 2])`.” These are checked by the `example`s in “Examples” at the end.

## Creating and reading

### bytes.empty

```text
pub fn empty() -> Bytes
```

A sequence of length 0.

- `bytes.length(bytes: bytes.empty())` → `0`

### bytes.new

```text
pub fn new(count: Int, fill: Int) -> Bytes
```

Creates a sequence of `count` copies of the value `fill`.

Precondition: `0 <= count and count <= 1073741823 and fill >= 0 and fill <= 255`. A negative count, a count above 1073741823 (2^30 - 1), and a `fill` outside 0 to 255 are defects.

Postcondition: the length of the result is `count`.

- `bytes.to_list(bytes: bytes.new(count: 3, fill: 7))` → `[7, 7, 7]`
- `bytes.length(bytes: bytes.new(count: 0, fill: 0))` → `0`
- `bytes.new(count: -1, fill: 0)` → defect
- `bytes.new(count: 1073741824, fill: 0)` → defect
- `bytes.new(count: 1, fill: 256)` → defect

### bytes.from_list and bytes.to_list

```text
pub fn from_list(items: List<Int>) -> Option<Bytes>
pub fn to_list(bytes: Bytes) -> List<Int>
```

`from_list` turns a `List<Int>` into `Bytes`. It returns `Some` when every element is from 0 to 255, and `None` if even one element is out of range (256 or more, or negative). An empty list gives `Some` with an empty sequence. `to_list` is the reverse: it turns the elements into a `List<Int>`. The length of the result is the same as `bytes.length`, and each element is from 0 to 255.

`from_list` returns an `Option` because a `List<Int>` can contain integers outside the byte range. The caller can handle these as `None` rather than having to rule them out in advance.

- `bytes.from_list(items: [0, 255])` → `Some(value: ...)` (a `Bytes` whose elements are `[0, 255]`)
- `bytes.from_list(items: [])` → `Some(value: bytes.empty())`
- `bytes.from_list(items: [256])` → `None`
- `bytes.from_list(items: [-1])` → `None`
- `bytes.to_list(bytes: bytes.from_text(text: "A"))` → `[65]`

### bytes.length

```text
pub fn length(bytes: Bytes) -> Int
```

Returns the number of bytes. For data created from a `String` with `bytes.from_text`, this may differ from the number of characters.

- `bytes.length(bytes: bytes.from_text(text: "é"))` → `2`

### bytes.get

```text
pub fn get(bytes: Bytes, index: Int) -> Int
```

Returns the byte at position `index` as an `Int` from 0 to 255.

Precondition: `index >= 0 and index < bytes.length(bytes: bytes)`. A position outside the range is a defect.

- `bytes.get(bytes: bytes.from_text(text: "ab"), index: 1)` → `98`
- `bytes.get(bytes: bytes.from_text(text: "a"), index: 1)` → defect
- `bytes.get(bytes: bytes.from_text(text: "a"), index: -1)` → defect

## Transforming

### bytes.set

```text
pub fn set(bytes: Bytes, index: Int, value: Int) -> Bytes
```

Returns a new sequence with the byte at position `index` set to `value`. The original sequence does not change.

Precondition: `index >= 0 and index < bytes.length(bytes: bytes)`, and `value >= 0 and value <= 255`. It is a defect if the position is out of range or `value` is outside 0 to 255.

Postcondition: the length of the result is the same as that of the original sequence.

To update a sequence held in a variable, assign the result to the same variable, as in `data = bytes.set(bytes: data, index: i, value: v);` ([05 Statements and bindings](../language/05-statements.md)).

- `bytes.to_list(bytes: bytes.set(bytes: bytes.new(count: 3, fill: 0), index: 1, value: 9))` → `[0, 9, 0]`
- `bytes.set(bytes: bytes.new(count: 1, fill: 0), index: 1, value: 0)` → defect
- `bytes.set(bytes: bytes.new(count: 1, fill: 0), index: 0, value: 256)` → defect
- `bytes.set(bytes: bytes.new(count: 1, fill: 0), index: 0, value: -1)` → defect

The `set_keeps_the_original` example checks that the original sequence is unchanged.

### bytes.fill

```text
pub fn fill(bytes: Bytes, from: Int, to: Int, value: Int) -> Bytes
```

Returns a new sequence with the positions from `from` up to just before `to` set to `value`. `from` is included and `to` is not.

Precondition: `0 <= from and from <= to and to <= bytes.length(bytes: bytes)`, and `value >= 0 and value <= 255`.

Postcondition: the length of the result is the same as that of the original sequence.

Edge case: if `from == to`, nothing changes.

- `bytes.to_list(bytes: bytes.fill(bytes: bytes.new(count: 4, fill: 1), from: 1, to: 3, value: 9))` → `[1, 9, 9, 1]`
- `bytes.to_list(bytes: bytes.fill(bytes: bytes.new(count: 2, fill: 1), from: 1, to: 1, value: 9))` → `[1, 1]`
- `bytes.fill(bytes: bytes.new(count: 1, fill: 0), from: 0, to: 2, value: 0)` → defect
- `bytes.fill(bytes: bytes.new(count: 1, fill: 0), from: 0, to: 1, value: 256)` → defect

### bytes.slice

```text
pub fn slice(bytes: Bytes, from: Int, to: Int) -> Bytes
```

Returns the part from position `from` up to just before `to`. `from` is included and `to` is not.

Precondition: `0 <= from and from <= to and to <= bytes.length(bytes: bytes)`.

Postcondition: the length of the result is `to - from`.

- `bytes.to_list(bytes: bytes.slice(bytes: bytes.from_text(text: "abcd"), from: 1, to: 3))` → `[98, 99]`
- `bytes.length(bytes: bytes.slice(bytes: bytes.from_text(text: "ab"), from: 2, to: 2))` → `0`
- `bytes.slice(bytes: bytes.from_text(text: "a"), from: 1, to: 0)` → defect
- `bytes.slice(bytes: bytes.from_text(text: "a"), from: 0, to: 2)` → defect

### bytes.concat and bytes.concat_all

```text
pub fn concat(first: Bytes, second: Bytes) -> Bytes
pub fn concat_all(parts: List<Bytes>) -> Bytes
```

`concat` joins two sequences, and `concat_all` joins all the sequences in a list, in order. The length of the result of `concat` is the sum of the two lengths. There is no precondition. Passing an empty list to `concat_all` gives an empty sequence.

- `bytes.to_list(bytes: bytes.concat(first: bytes.from_text(text: "a"), second: bytes.from_text(text: "bc")))` → `[97, 98, 99]`
- `bytes.to_list(bytes: bytes.concat_all(parts: [bytes.from_text(text: "a"), bytes.empty(), bytes.from_text(text: "bc")]))` → `[97, 98, 99]`
- `bytes.length(bytes: bytes.concat_all(parts: []))` → `0`

## Searching

### bytes.find, bytes.index_of, and bytes.starts_with

```text
pub fn find(bytes: Bytes, pattern: Bytes) -> Option<Int>
pub fn index_of(bytes: Bytes, value: Int) -> Option<Int>
pub fn starts_with(bytes: Bytes, prefix: Bytes) -> Bool
```

`find` returns the position where the sequence `pattern` first appears, or `None` if it does not appear. `index_of` returns the position where the single byte `value` first appears, or `None` if it does not appear. `starts_with` is `true` if `bytes` starts with `prefix`. There is no precondition.

Edge cases: if `pattern` or `prefix` is an empty sequence, `find` gives `Some(value: 0)` and `starts_with` gives `true`, even when `bytes` is empty. If `pattern` is longer than `bytes`, the result is `None`. Passing a `value` outside 0 to 255 to `index_of` is not a defect; it equals no byte, so the result is `None`.

- `bytes.find(bytes: bytes.from_text(text: "abcabc"), pattern: bytes.from_text(text: "ca"))` → `Some(value: 2)`
- `bytes.find(bytes: bytes.from_text(text: "abc"), pattern: bytes.from_text(text: "x"))` → `None`
- `bytes.find(bytes: bytes.from_text(text: "abc"), pattern: bytes.empty())` → `Some(value: 0)`
- `bytes.find(bytes: bytes.empty(), pattern: bytes.empty())` → `Some(value: 0)`
- `bytes.find(bytes: bytes.from_text(text: "a"), pattern: bytes.from_text(text: "ab"))` → `None`
- `bytes.index_of(bytes: bytes.from_text(text: "abca"), value: 99)` → `Some(value: 2)`
- `bytes.index_of(bytes: bytes.from_text(text: "ab"), value: 0)` → `None`
- `bytes.index_of(bytes: bytes.from_text(text: "ab"), value: 256)` → `None`
- `bytes.index_of(bytes: bytes.empty(), value: 0)` → `None`
- `bytes.starts_with(bytes: bytes.from_text(text: "abc"), prefix: bytes.from_text(text: "ab"))` → `true`
- `bytes.starts_with(bytes: bytes.from_text(text: "abc"), prefix: bytes.from_text(text: "bc"))` → `false`
- `bytes.starts_with(bytes: bytes.empty(), prefix: bytes.empty())` → `true`

## Reading and writing integers

When an integer is represented by several bytes, the width and the byte order are specified by arguments.

- `width`: the number of bytes, from 1 to 8.
- `big_endian`: if `true`, the most significant byte comes first (big-endian); if `false`, the least significant byte comes first (little-endian).

Signed integers are represented in two’s complement. The range that can be represented in `width` bytes is as follows.

| Kind | Range | `width: 1` | `width: 8` |
| --- | --- | --- | --- |
| Unsigned | At least 0 and less than 256^width | 0 to 255 | 0 to 18446744073709551615 |
| Signed | At least -128 × 256^(width-1) and less than 128 × 256^(width-1) | -128 to 127 | -9223372036854775808 to 9223372036854775807 |

### bytes.read_uint

```text
pub fn read_uint(bytes: Bytes, at: Int, width: Int, big_endian: Bool) -> Int
```

Reads `width` bytes from position `at` as an unsigned integer.

Precondition: `width >= 1 and width <= 8 and at >= 0 and at + width <= bytes.length(bytes: bytes)`. It is a defect if the width is out of range or the range to read extends past the end of the sequence.

Postcondition: the result is at least 0 and less than 256^width.

Reading the byte sequence `[1, 2, 3, 4]` with width 4 gives `1 × 256^3 + 2 × 256^2 + 3 × 256 + 4 = 16909060` with `big_endian: true`, and `4 × 256^3 + 3 × 256^2 + 2 × 256 + 1 = 67305985` with `big_endian: false`.

- `bytes.read_uint(bytes: [1, 2, 3, 4], at: 0, width: 4, big_endian: true)` → `16909060`
- `bytes.read_uint(bytes: [1, 2, 3, 4], at: 0, width: 4, big_endian: false)` → `67305985`
- `bytes.read_uint(bytes: [1, 2, 3], at: 1, width: 2, big_endian: true)` → `515`
- `bytes.read_uint(bytes: [9, 255, 255, 255, 255, 255, 255, 255, 255], at: 1, width: 8, big_endian: true)` → `18446744073709551615`
- `bytes.read_uint(bytes: [1, 2, 3, 4, 5, 6, 7, 8], at: 0, width: 8, big_endian: false)` → `578437695752307201`
- `bytes.read_uint(bytes: [1, 2], at: 1, width: 2, big_endian: true)` → defect
- `bytes.read_uint(bytes: [1, 2], at: -1, width: 1, big_endian: true)` → defect
- `bytes.read_uint(bytes: [1, 2], at: 0, width: 0, big_endian: true)` → defect

### bytes.read_int

```text
pub fn read_int(bytes: Bytes, at: Int, width: Int, big_endian: Bool) -> Int
```

Reads the same range as `read_uint` as a two’s-complement signed integer. If the most significant bit is 1, the value is negative. The precondition is the same as that of `read_uint`.

- `bytes.read_int(bytes: [254, 255], at: 0, width: 2, big_endian: false)` → `-2`
- `bytes.read_int(bytes: [255, 254], at: 0, width: 2, big_endian: true)` → `-2`
- `bytes.read_int(bytes: [128], at: 0, width: 1, big_endian: true)` → `-128`
- `bytes.read_int(bytes: [127], at: 0, width: 1, big_endian: true)` → `127`
- `bytes.read_int(bytes: [128, 0, 0, 0, 0, 0, 0, 0], at: 0, width: 8, big_endian: true)` → `-9223372036854775808`
- `bytes.read_int(bytes: [1, 2], at: 0, width: 9, big_endian: true)` → defect

### bytes.uint_bytes

```text
pub fn uint_bytes(value: Int, width: Int, big_endian: Bool) -> List<Int>
```

Turns the unsigned integer `value` into a list of `width` bytes. This is the reverse of `read_uint`.

Precondition: `width >= 1 and width <= 8 and value >= 0 and value < 256^width`. A negative value, or a value that does not fit in `width` bytes, is a defect.

Postcondition: the length of the result is `width`, and each element is from 0 to 255.

To get a `Bytes`, pass the result to `bytes.from_list` (if the precondition is satisfied, the result is always `Some`).

- `bytes.uint_bytes(value: 16909060, width: 4, big_endian: true)` → `[1, 2, 3, 4]`
- `bytes.uint_bytes(value: 16909060, width: 4, big_endian: false)` → `[4, 3, 2, 1]`
- `bytes.uint_bytes(value: 258, width: 2, big_endian: false)` → `[2, 1]`
- `bytes.uint_bytes(value: 0, width: 3, big_endian: true)` → `[0, 0, 0]`
- `bytes.uint_bytes(value: 18446744073709551615, width: 8, big_endian: true)` → `[255, 255, 255, 255, 255, 255, 255, 255]`
- `bytes.uint_bytes(value: 256, width: 1, big_endian: true)` → defect
- `bytes.uint_bytes(value: -1, width: 1, big_endian: true)` → defect
- `bytes.uint_bytes(value: 0, width: 9, big_endian: true)` → defect

### bytes.int_bytes

```text
pub fn int_bytes(value: Int, width: Int, big_endian: Bool) -> List<Int>
```

Turns the signed integer `value` into a list of `width` two’s-complement bytes. This is the reverse of `read_int`.

Precondition: `width >= 1 and width <= 8`, and `value` is at least `-128 × 256^(width-1)` and less than `128 × 256^(width-1)`. A value outside the range is a defect.

Postcondition: the length of the result is `width`.

- `bytes.int_bytes(value: -2, width: 2, big_endian: false)` → `[254, 255]`
- `bytes.int_bytes(value: -2, width: 2, big_endian: true)` → `[255, 254]`
- `bytes.int_bytes(value: 127, width: 1, big_endian: true)` → `[127]`
- `bytes.int_bytes(value: -128, width: 1, big_endian: true)` → `[128]`
- `bytes.int_bytes(value: -1, width: 3, big_endian: false)` → `[255, 255, 255]`
- `bytes.int_bytes(value: -9223372036854775808, width: 8, big_endian: true)` → `[128, 0, 0, 0, 0, 0, 0, 0]`
- `bytes.int_bytes(value: 128, width: 1, big_endian: true)` → defect
- `bytes.int_bytes(value: -129, width: 1, big_endian: true)` → defect
- `bytes.int_bytes(value: 0, width: 0, big_endian: true)` → defect

## Conversion to and from strings

### bytes.from_text and bytes.to_text

```text
pub fn from_text(text: String) -> Bytes
pub fn to_text(bytes: Bytes) -> Option<String>
```

`from_text` turns a string into a UTF-8 byte sequence. Any `String` can be converted. A scalar value becomes one to four bytes, so the length of the result does not necessarily match the number of characters.

`to_text` reads the byte sequence as UTF-8 and turns it into a string. If the sequence is not valid UTF-8, the result is `None`. The following are considered invalid.

- A bad leading byte (192, 193, 245 or more, and so on)
- Missing continuation bytes, a byte that is not a continuation byte where one is expected, and a continuation byte without a leading byte
- An overlong encoding, that is, a value written in a longer form than necessary (`[192, 128]`, `[224, 128, 128]`, and so on)
- An encoding of a surrogate (U+D800 to U+DFFF)
- An encoding of a value above U+10FFFF

An empty sequence gives `Some(value: "")`. The byte 0 is read as U+0000, and a byte order mark (`[239, 187, 191]`, U+FEFF) is not removed but remains as a character. For any `s`, `to_text(from_text(s))` is `Some(value: s)`.

`to_text` returns an `Option` because not every byte sequence represents a valid sequence of Unicode scalar values. Invalid UTF-8 produces `None`; it is not repaired with replacement characters.

- `bytes.to_list(bytes: bytes.from_text(text: "aé€😀"))` → `[97, 195, 169, 226, 130, 172, 240, 159, 152, 128]`
- `bytes.to_list(bytes: bytes.from_text(text: ""))` → `[]`
- `bytes.to_text(bytes: bytes.from_text(text: "aé€😀"))` → `Some(value: "aé€😀")`
- `bytes.to_text(bytes: bytes.empty())` → `Some(value: "")`
- `bytes.to_text(bytes: [0])` → `Some(value: "\u{0}")`
- `bytes.to_text(bytes: [239, 187, 191])` → `Some(value: "\u{feff}")`
- `bytes.to_text(bytes: [244, 143, 191, 191])` → `Some(value: "\u{10ffff}")`
- `bytes.to_text(bytes: [255])` → `None`
- `bytes.to_text(bytes: [97, 128])` → `None`
- `bytes.to_text(bytes: [195])` → `None`
- `bytes.to_text(bytes: [226, 130])` → `None`
- `bytes.to_text(bytes: [192, 128])` → `None`
- `bytes.to_text(bytes: [224, 128, 128])` → `None`
- `bytes.to_text(bytes: [240, 128, 128, 128])` → `None`
- `bytes.to_text(bytes: [237, 160, 128])` → `None`
- `bytes.to_text(bytes: [244, 144, 128, 128])` → `None`
- `bytes.to_text(bytes: [245, 128, 128, 128])` → `None`

### bytes.to_hex and bytes.from_hex

```text
pub fn to_hex(bytes: Bytes) -> String
pub fn from_hex(text: String) -> Option<Bytes>
```

`to_hex` writes each byte as two lowercase hexadecimal digits and concatenates them. The length of the result is twice the number of bytes.

`from_hex` is the reverse. It reads two digits at a time and accepts the digits `0` to `9` and the letters `a` to `f` and `A` to `F`. The following inputs give `None`.

- A length that is odd
- A character that is not a hexadecimal digit (including a `0x` prefix, whitespace, separators, and non-ASCII characters)

An empty string gives `Some(value: bytes.empty())`.

- `bytes.to_hex(bytes: [0, 255, 122])` → `"00ff7a"`
- `bytes.to_hex(bytes: bytes.empty())` → `""`
- `bytes.from_hex(text: "00ff7A")` → `Some(value: ...)` (a `Bytes` whose elements are `[0, 255, 122]`)
- `bytes.from_hex(text: "FF0a")` → `Some(value: ...)` (a `Bytes` whose elements are `[255, 10]`)
- `bytes.from_hex(text: "")` → `Some(value: bytes.empty())`
- `bytes.from_hex(text: "0")` → `None`
- `bytes.from_hex(text: "abc")` → `None`
- `bytes.from_hex(text: "zz")` → `None`
- `bytes.from_hex(text: "0x")` → `None`
- `bytes.from_hex(text: "ab cd")` → `None`
- `bytes.from_hex(text: "éé")` → `None`

### bytes.to_base64 and bytes.from_base64

```text
pub fn to_base64(bytes: Bytes) -> String
pub fn from_base64(text: String) -> Option<Bytes>
```

This is standard Base64 (the characters are `A` to `Z`, `a` to `z`, `0` to `9`, `+`, and `/`). Three bytes become four characters, and a final remainder is padded to four characters with `=`.

`to_base64` always writes this form (with `=` padding).

`from_base64` accepts **only the canonical form**. The following inputs give `None`.

- A length that is not a multiple of 4 (including the form with the `=` padding omitted)
- A character that is neither a Base64 character nor `=` (including whitespace, newlines, and the URL-oriented `-` and `_`)
- An `=` in any position other than the last one or two of the final four characters
- Leftover bits in the padding part that are not 0 (a form that `to_base64` never writes)

An empty string gives `Some(value: bytes.empty())`.

- `bytes.to_base64(bytes: bytes.from_text(text: "hi"))` → `"aGk="`
- `bytes.to_base64(bytes: bytes.from_text(text: "h"))` → `"aA=="`
- `bytes.to_base64(bytes: bytes.from_text(text: "hij"))` → `"aGlq"`
- `bytes.to_base64(bytes: bytes.empty())` → `""`
- `bytes.from_base64(text: "aGk=")` → `Some(value: bytes.from_text(text: "hi"))`
- `bytes.from_base64(text: "aA==")` → `Some(value: bytes.from_text(text: "h"))`
- `bytes.from_base64(text: "aGlq")` → `Some(value: bytes.from_text(text: "hij"))`
- `bytes.from_base64(text: "+/+/")` → `Some(value: ...)` (a `Bytes` whose elements are `[251, 255, 191]`)
- `bytes.from_base64(text: "")` → `Some(value: bytes.empty())`
- `bytes.from_base64(text: "aGk")` → `None`
- `bytes.from_base64(text: "a")` → `None`
- `bytes.from_base64(text: "aG=k")` → `None`
- `bytes.from_base64(text: "aGl=")` → `None`
- `bytes.from_base64(text: "aB==")` → `None`
- `bytes.from_base64(text: "====")` → `None`
- `bytes.from_base64(text: "aGk=aGlq")` → `None`
- `bytes.from_base64(text: "aGk=aGk=")` → `None`
- `bytes.from_base64(text: "!!!!")` → `None`
- `bytes.from_base64(text: "a-_=")` → `None`
- `bytes.from_base64(text: "aGk=\n")` → `None`

## Examples

Examples for each function. A call that violates a precondition is checked to be a defect with `defects(of: ...)` ([16 Testing](../language/16-testing.md)). An `example` must be attached to a function of the same module, so each module has a small function that the examples are attached to. `bytes_of` is a helper function that builds a `Bytes` from a list.

Examples of creating, reading, transforming, and searching.

```cernith
module guide.bytes_basics;

import std.bytes;

pub fn bytes_of(items: List<Int>) -> Bytes {
  match bytes.from_list(items: items) {
    Some(value: found) => found,
    None => bytes.empty(),
  }
}

example bytes_of_builds_from_a_list for bytes_of {
  bytes.to_list(bytes: bytes_of(items: [1, 2])) == [1, 2]
    and bytes.length(bytes: bytes_of(items: [])) == 0
}

example concat_all_joins_every_part for bytes_of {
  bytes.to_list(
    bytes: bytes.concat_all(
      parts: [
        bytes.from_text(text: "a"),
        bytes.empty(),
        bytes.from_text(text: "bc"),
      ],
    ),
  )
    == [97, 98, 99]
    and bytes.length(bytes: bytes.concat_all(parts: [])) == 0
}

example concat_joins_two_sequences for bytes_of {
  bytes.to_list(
    bytes: bytes.concat(
      first: bytes.from_text(text: "a"),
      second: bytes.from_text(text: "bc"),
    ),
  )
    == [97, 98, 99]
}

example empty_has_no_bytes for bytes_of {
  bytes.length(bytes: bytes.empty()) == 0
}

example fill_out_of_range_is_a_defect for bytes_of {
  defects(
    of: bytes.fill(
      bytes: bytes.new(count: 1, fill: 0),
      from: 0,
      to: 2,
      value: 0,
    ),
  )
    and defects(
      of: bytes.fill(
        bytes: bytes.new(count: 1, fill: 0),
        from: 0,
        to: 1,
        value: 256,
      ),
    )
}

example fill_sets_a_range for bytes_of {
  bytes.to_list(
    bytes: bytes.fill(
      bytes: bytes.new(count: 4, fill: 1),
      from: 1,
      to: 3,
      value: 9,
    ),
  )
    == [1, 9, 9, 1]
    and bytes.to_list(
      bytes: bytes.fill(
        bytes: bytes.new(count: 2, fill: 1),
        from: 1,
        to: 1,
        value: 9,
      ),
    )
      == [1, 1]
}

example find_gives_the_first_position for bytes_of {
  bytes.find(
    bytes: bytes.from_text(text: "abcabc"),
    pattern: bytes.from_text(text: "ca"),
  )
    == Some(value: 2)
    and bytes.find(
      bytes: bytes.from_text(text: "abc"),
      pattern: bytes.from_text(text: "x"),
    )
      == None
    and bytes.find(bytes: bytes.from_text(text: "abc"), pattern: bytes.empty())
      == Some(value: 0)
    and bytes.find(bytes: bytes.empty(), pattern: bytes.empty())
      == Some(value: 0)
    and bytes.find(
      bytes: bytes.from_text(text: "a"),
      pattern: bytes.from_text(text: "ab"),
    )
      == None
}

example from_list_accepts_only_byte_values for bytes_of {
  bytes.from_list(items: [0, 255]) == Some(value: bytes_of(items: [0, 255]))
    and bytes.from_list(items: []) == Some(value: bytes.empty())
    and bytes.from_list(items: [256]) == None
    and bytes.from_list(items: [-1]) == None
}

example get_out_of_range_is_a_defect for bytes_of {
  defects(of: bytes.get(bytes: bytes.from_text(text: "a"), index: 1))
    and defects(of: bytes.get(bytes: bytes.from_text(text: "a"), index: -1))
}

example get_reads_one_byte for bytes_of {
  bytes.get(bytes: bytes.from_text(text: "ab"), index: 1) == 98
}

example index_of_gives_the_first_position for bytes_of {
  bytes.index_of(bytes: bytes.from_text(text: "abca"), value: 99)
    == Some(value: 2)
    and bytes.index_of(bytes: bytes.from_text(text: "ab"), value: 0) == None
    and bytes.index_of(bytes: bytes.from_text(text: "ab"), value: 256) == None
    and bytes.index_of(bytes: bytes.empty(), value: 0) == None
}

example length_counts_bytes_not_characters for bytes_of {
  bytes.length(bytes: bytes.from_text(text: "é")) == 2
}

example new_makes_equal_bytes for bytes_of {
  bytes.to_list(bytes: bytes.new(count: 3, fill: 7)) == [7, 7, 7]
    and bytes.length(bytes: bytes.new(count: 0, fill: 0)) == 0
}

example new_out_of_range_is_a_defect for bytes_of {
  defects(of: bytes.new(count: -1, fill: 0))
    and defects(of: bytes.new(count: 1073741824, fill: 0))
    and defects(of: bytes.new(count: 1, fill: 256))
}

example set_out_of_range_is_a_defect for bytes_of {
  defects(
    of: bytes.set(bytes: bytes.new(count: 1, fill: 0), index: 1, value: 0),
  )
    and defects(
      of: bytes.set(bytes: bytes.new(count: 1, fill: 0), index: 0, value: 256),
    )
    and defects(
      of: bytes.set(bytes: bytes.new(count: 1, fill: 0), index: 0, value: -1),
    )
}

example slice_out_of_range_is_a_defect for bytes_of {
  defects(of: bytes.slice(bytes: bytes.from_text(text: "a"), from: 1, to: 0))
    and defects(
      of: bytes.slice(bytes: bytes.from_text(text: "a"), from: 0, to: 2),
    )
}

example slice_takes_a_range for bytes_of {
  bytes.to_list(
    bytes: bytes.slice(bytes: bytes.from_text(text: "abcd"), from: 1, to: 3),
  )
    == [98, 99]
    and bytes.length(
      bytes: bytes.slice(bytes: bytes.from_text(text: "ab"), from: 2, to: 2),
    )
      == 0
}

example starts_with_compares_the_front for bytes_of {
  bytes.starts_with(
    bytes: bytes.from_text(text: "abc"),
    prefix: bytes.from_text(text: "ab"),
  )
    and not bytes.starts_with(
      bytes: bytes.from_text(text: "abc"),
      prefix: bytes.from_text(text: "bc"),
    )
    and bytes.starts_with(bytes: bytes.empty(), prefix: bytes.empty())
}

example to_list_gives_the_values for bytes_of {
  bytes.to_list(bytes: bytes.from_text(text: "A")) == [65]
}

pub fn set_keeps_the_original() -> Bool {
  let before = bytes.new(count: 3, fill: 0);
  let after = bytes.set(bytes: before, index: 1, value: 9);
  bytes.to_list(bytes: before) == [0, 0, 0]
    and bytes.to_list(bytes: after) == [0, 9, 0]
}

example set_gives_a_new_sequence for set_keeps_the_original {
  set_keeps_the_original()
}
```

Examples of reading and writing integers. `encode_header` builds a header consisting of a one-byte version number followed by a two-byte length (big-endian), and `read_length` reads that length back.

```cernith
module guide.bytes_numbers;

import std.bytes;

pub fn bytes_of(items: List<Int>) -> Bytes {
  match bytes.from_list(items: items) {
    Some(value: found) => found,
    None => bytes.empty(),
  }
}

pub fn encode_header(version: Int, length: Int) -> List<Int> {
  requires version >= 0 and version <= 255;
  requires length >= 0 and length <= 65535;

  bytes.uint_bytes(value: version, width: 1, big_endian: true)
    ++ bytes.uint_bytes(value: length, width: 2, big_endian: true)
}

example encode_header_cases for encode_header {
  encode_header(version: 1, length: 258) == [1, 1, 2]
    and encode_header(version: 0, length: 0) == [0, 0, 0]
}

example encode_header_out_of_range_is_a_defect for encode_header {
  defects(of: encode_header(version: 256, length: 0))
    and defects(of: encode_header(version: 0, length: 65536))
}

example int_bytes_out_of_range_is_a_defect for encode_header {
  defects(of: bytes.int_bytes(value: 128, width: 1, big_endian: true))
    and defects(of: bytes.int_bytes(value: -129, width: 1, big_endian: true))
    and defects(of: bytes.int_bytes(value: 0, width: 0, big_endian: true))
}

example int_bytes_writes_twos_complement for encode_header {
  bytes.int_bytes(value: -2, width: 2, big_endian: false) == [254, 255]
    and bytes.int_bytes(value: -2, width: 2, big_endian: true) == [255, 254]
    and bytes.int_bytes(value: 127, width: 1, big_endian: true) == [127]
    and bytes.int_bytes(value: -128, width: 1, big_endian: true) == [128]
    and bytes.int_bytes(value: -1, width: 3, big_endian: false)
      == [255, 255, 255]
    and bytes.int_bytes(value: -9223372036854775808, width: 8, big_endian: true)
      == [128, 0, 0, 0, 0, 0, 0, 0]
}

example uint_bytes_out_of_range_is_a_defect for encode_header {
  defects(of: bytes.uint_bytes(value: 256, width: 1, big_endian: true))
    and defects(of: bytes.uint_bytes(value: -1, width: 1, big_endian: true))
    and defects(of: bytes.uint_bytes(value: 0, width: 9, big_endian: true))
}

example uint_bytes_writes_either_order for encode_header {
  bytes.uint_bytes(value: 16909060, width: 4, big_endian: true) == [1, 2, 3, 4]
    and bytes.uint_bytes(value: 16909060, width: 4, big_endian: false)
      == [4, 3, 2, 1]
    and bytes.uint_bytes(value: 258, width: 2, big_endian: false) == [2, 1]
    and bytes.uint_bytes(value: 0, width: 3, big_endian: true) == [0, 0, 0]
    and bytes.uint_bytes(
      value: 18446744073709551615,
      width: 8,
      big_endian: true,
    )
      == [255, 255, 255, 255, 255, 255, 255, 255]
}

pub fn read_length(frame: Bytes) -> Int {
  requires bytes.length(bytes: frame) >= 3;

  bytes.read_uint(bytes: frame, at: 1, width: 2, big_endian: true)
}

example read_int_reads_twos_complement for read_length {
  bytes.read_int(
    bytes: bytes_of(items: [254, 255]),
    at: 0,
    width: 2,
    big_endian: false,
  )
    == -2
    and bytes.read_int(
      bytes: bytes_of(items: [255, 254]),
      at: 0,
      width: 2,
      big_endian: true,
    )
      == -2
    and bytes.read_int(
      bytes: bytes_of(items: [128]),
      at: 0,
      width: 1,
      big_endian: true,
    )
      == -128
    and bytes.read_int(
      bytes: bytes_of(items: [127]),
      at: 0,
      width: 1,
      big_endian: true,
    )
      == 127
    and bytes.read_int(
      bytes: bytes_of(items: [128, 0, 0, 0, 0, 0, 0, 0]),
      at: 0,
      width: 8,
      big_endian: true,
    )
      == -9223372036854775808
}

example read_int_with_a_bad_width_is_a_defect for read_length {
  defects(
    of: bytes.read_int(
      bytes: bytes_of(items: [1, 2]),
      at: 0,
      width: 9,
      big_endian: true,
    ),
  )
}

example read_length_reads_the_header for read_length {
  read_length(frame: bytes_of(items: [1, 1, 2])) == 258
    and read_length(
      frame: bytes_of(items: encode_header(version: 7, length: 65535)),
    )
      == 65535
}

example read_uint_out_of_range_is_a_defect for read_length {
  defects(
    of: bytes.read_uint(
      bytes: bytes_of(items: [1, 2]),
      at: 1,
      width: 2,
      big_endian: true,
    ),
  )
    and defects(
      of: bytes.read_uint(
        bytes: bytes_of(items: [1, 2]),
        at: -1,
        width: 1,
        big_endian: true,
      ),
    )
    and defects(
      of: bytes.read_uint(
        bytes: bytes_of(items: [1, 2]),
        at: 0,
        width: 0,
        big_endian: true,
      ),
    )
}

example read_uint_reads_either_order for read_length {
  bytes.read_uint(
    bytes: bytes_of(items: [1, 2, 3, 4]),
    at: 0,
    width: 4,
    big_endian: true,
  )
    == 16909060
    and bytes.read_uint(
      bytes: bytes_of(items: [1, 2, 3, 4]),
      at: 0,
      width: 4,
      big_endian: false,
    )
      == 67305985
    and bytes.read_uint(
      bytes: bytes_of(items: [1, 2, 3]),
      at: 1,
      width: 2,
      big_endian: true,
    )
      == 515
    and bytes.read_uint(
      bytes: bytes_of(items: [9, 255, 255, 255, 255, 255, 255, 255, 255]),
      at: 1,
      width: 8,
      big_endian: true,
    )
      == 18446744073709551615
    and bytes.read_uint(
      bytes: bytes_of(items: [1, 2, 3, 4, 5, 6, 7, 8]),
      at: 0,
      width: 8,
      big_endian: false,
    )
      == 578437695752307201
}
```

Examples of conversion to and from strings.

```cernith
module guide.bytes_text;

import std.bytes;

pub fn bytes_of(items: List<Int>) -> Bytes {
  match bytes.from_list(items: items) {
    Some(value: found) => found,
    None => bytes.empty(),
  }
}

pub fn fingerprint(text: String) -> String {
  bytes.to_hex(bytes: bytes.from_text(text: text))
}

example fingerprint_is_the_hex_of_utf_8 for fingerprint {
  fingerprint(text: "hi") == "6869"
    and fingerprint(text: "é") == "c3a9"
    and fingerprint(text: "") == ""
}

example from_base64_reads_canonical_text_only for fingerprint {
  bytes.from_base64(text: "aGk=") == Some(value: bytes.from_text(text: "hi"))
    and bytes.from_base64(text: "aA==")
      == Some(value: bytes.from_text(text: "h"))
    and bytes.from_base64(text: "aGlq")
      == Some(value: bytes.from_text(text: "hij"))
    and bytes.from_base64(text: "+/+/")
      == Some(value: bytes_of(items: [251, 255, 191]))
    and bytes.from_base64(text: "") == Some(value: bytes.empty())
}

example from_base64_refuses_everything_else for fingerprint {
  bytes.from_base64(text: "aGk") == None
    and bytes.from_base64(text: "a") == None
    and bytes.from_base64(text: "aG=k") == None
    and bytes.from_base64(text: "aGl=") == None
    and bytes.from_base64(text: "aB==") == None
    and bytes.from_base64(text: "====") == None
    and bytes.from_base64(text: "aGk=aGlq") == None
    and bytes.from_base64(text: "aGk=aGk=") == None
    and bytes.from_base64(text: "!!!!") == None
    and bytes.from_base64(text: "a-_=") == None
    and bytes.from_base64(text: "aGk=\n") == None
}

example from_hex_reads_both_cases_and_refuses_the_rest for fingerprint {
  bytes.from_hex(text: "00ff7A") == Some(value: bytes_of(items: [0, 255, 122]))
    and bytes.from_hex(text: "FF0a") == Some(value: bytes_of(items: [255, 10]))
    and bytes.from_hex(text: "") == Some(value: bytes.empty())
    and bytes.from_hex(text: "0") == None
    and bytes.from_hex(text: "abc") == None
    and bytes.from_hex(text: "zz") == None
    and bytes.from_hex(text: "0x") == None
    and bytes.from_hex(text: "ab cd") == None
    and bytes.from_hex(text: "éé") == None
}

example from_text_writes_utf_8 for fingerprint {
  bytes.to_list(bytes: bytes.from_text(text: "aé€😀"))
    == [97, 195, 169, 226, 130, 172, 240, 159, 152, 128]
    and bytes.to_list(bytes: bytes.from_text(text: "")) == []
}

example to_base64_pads_to_four_characters for fingerprint {
  bytes.to_base64(bytes: bytes.from_text(text: "hi")) == "aGk="
    and bytes.to_base64(bytes: bytes.from_text(text: "h")) == "aA=="
    and bytes.to_base64(bytes: bytes.from_text(text: "hij")) == "aGlq"
    and bytes.to_base64(bytes: bytes.empty()) == ""
}

example to_hex_writes_two_lower_case_digits_per_byte for fingerprint {
  bytes.to_hex(bytes: bytes_of(items: [0, 255, 122])) == "00ff7a"
    and bytes.to_hex(bytes: bytes.empty()) == ""
}

example to_text_reads_utf_8 for fingerprint {
  bytes.to_text(bytes: bytes.from_text(text: "aé€😀")) == Some(value: "aé€😀")
    and bytes.to_text(bytes: bytes.empty()) == Some(value: "")
    and bytes.to_text(bytes: bytes_of(items: [0])) == Some(value: "\u{0}")
    and bytes.to_text(bytes: bytes_of(items: [239, 187, 191]))
      == Some(value: "\u{feff}")
    and bytes.to_text(bytes: bytes_of(items: [244, 143, 191, 191]))
      == Some(value: "􏿿")
}

example to_text_refuses_what_is_not_utf_8 for fingerprint {
  bytes.to_text(bytes: bytes_of(items: [255])) == None
    and bytes.to_text(bytes: bytes_of(items: [97, 128])) == None
    and bytes.to_text(bytes: bytes_of(items: [195])) == None
    and bytes.to_text(bytes: bytes_of(items: [226, 130])) == None
    and bytes.to_text(bytes: bytes_of(items: [192, 128])) == None
    and bytes.to_text(bytes: bytes_of(items: [224, 128, 128])) == None
    and bytes.to_text(bytes: bytes_of(items: [240, 128, 128, 128])) == None
    and bytes.to_text(bytes: bytes_of(items: [237, 160, 128])) == None
    and bytes.to_text(bytes: bytes_of(items: [244, 144, 128, 128])) == None
    and bytes.to_text(bytes: bytes_of(items: [245, 128, 128, 128])) == None
}
```

## Related

- [03 Types](../language/03-types.md): `Bytes` is a sequence stored more compactly than a `List`
- [12 Errors and defects](../language/12-errors-and-defects.md): a precondition violation is a defect
- [text](text.md): the functions on `String` (a `String` is a sequence of Unicode scalar values)
- [floats and ints](floats-ints.md): `Floats` and `Ints`
- [word32 and word64](word32-word64.md): fixed-width integer operations
