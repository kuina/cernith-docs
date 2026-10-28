# std.text

This module provides operations on `String`. Import it with `import std.text;` and call functions such as `text.split(of: ..., separator: ...)`. It also defines `text.SplitPair`, a record for the result of a split.

A `String` is a sequence of Unicode scalar values ([03 Types](../language/03-types.md)). In this module, all positions, lengths, and counts are measured in scalar values. They are not byte counts, not counts of UTF-16 units, and not the unit a person perceives as one character (a grapheme). For example, the length of `"a😀b"` is 3, and `text.index_of(of: "a😀b", part: "b")` is `Some(value: 2)`. `"e\u{301}"`, which uses a combining character (`e` and U+0301), has length 2 even though it looks like one character.

There is no dedicated type for characters. A single character is a `String` of length 1. Every “character” in this module means a `String` of length 1.

All functions are pure and have no effects. A call that violates a precondition (`requires`) is a defect ([12 Errors and defects](../language/12-errors-and-defects.md)). The functions that have preconditions are `code_point`, `from_code_point`, `slice`, `count`, `split`, `split_once`, `replace`, `repeat`, `pad_start`, and `pad_end`. The other functions never cause a defect, whatever the arguments.

## Function list

| Function | Return type | Meaning |
| --- | --- | --- |
| `text.is_empty(of)` | `Bool` | `true` if the length is 0 |
| `text.chars(of)` | `List<String>` | A list of strings of length 1, one per scalar value |
| `text.code_point(of)` | `Int` | The code point of a string of length 1 |
| `text.from_code_point(value)` | `String` | Makes a string of length 1 from a code point |
| `text.slice(of, from, to)` | `String` | The part from position `from` (inclusive) to `to` (exclusive) |
| `text.index_of(of, part)` | `Option<Int>` | The position where `part` first appears |
| `text.contains(of, part)` | `Bool` | `true` if it contains `part` |
| `text.starts_with(of, prefix)` | `Bool` | `true` if it starts with `prefix` |
| `text.ends_with(of, suffix)` | `Bool` | `true` if it ends with `suffix` |
| `text.count(of, part)` | `Int` | The number of times `part` appears without overlapping |
| `text.split(of, separator)` | `List<String>` | The list of parts separated by `separator` |
| `text.split_once(of, separator)` | `Option<SplitPair>` | Splits into the two parts before and after the first `separator` |
| `text.join(parts, separator)` | `String` | Concatenates with `separator` in between |
| `text.lines(of)` | `List<String>` | The list of lines |
| `text.replace(of, target, replacement)` | `String` | Replaces every `target` with `replacement` |
| `text.repeat(of, count)` | `String` | Repeats `count` times |
| `text.pad_start(of, width, fill)` | `String` | Adds `fill` on the left until the length is `width` |
| `text.pad_end(of, width, fill)` | `String` | Adds `fill` on the right until the length is `width` |
| `text.trim(of)` | `String` | Removes whitespace from both ends |
| `text.trim_start(of)` | `String` | Removes whitespace from the start |
| `text.trim_end(of)` | `String` | Removes whitespace from the end |
| `text.is_space(of)` | `Bool` | `true` if it is a single whitespace character |
| `text.to_lower(of)` | `String` | Converts ASCII uppercase letters to lowercase |
| `text.to_upper(of)` | `String` | Converts ASCII lowercase letters to uppercase |

There is one type:

```text
pub record SplitPair {
  before: String,
  after: String,
}
```

It is the result of `text.split_once`, and holds the part before the separator (`before`) and the part after it (`after`). Because there are no tuples, it is a record for bundling two values ([03 Types](../language/03-types.md)).

In the examples that follow, the right-hand side of `→` is the value of the call. `→ defect` means that the call violates a precondition and is a defect. These are verified by the `example` items in “Examples” at the end.

## Length and characters

### text.is_empty

```text
pub fn is_empty(of: String) -> Bool
```

`true` only for a string of length 0 (`""`). A string consisting only of whitespace is not empty.

- `text.is_empty(of: "")` → `true`
- `text.is_empty(of: " ")` → `false`

### text.chars

```text
pub fn chars(of: String) -> List<String>
```

Splits a string into scalar values. The length of the result is `of.length`, and the element at position `i` is `text.slice(of: of, from: i, to: i + 1)`. `text.join(parts: text.chars(of: of), separator: "")` gives back `of`.

- `text.chars(of: "ab")` → `["a", "b"]`
- `text.chars(of: "")` → `[]`
- `text.chars(of: "a😀")` → `["a", "😀"]`
- `text.chars(of: "e\u{301}").length` → `2`

### text.code_point

```text
pub fn code_point(of: String) -> Int
```

Returns the Unicode code point of a string of length 1 as an `Int`.

Precondition: `of.length == 1`. Passing a string of length 0, or of length 2 or more, is a defect.

Postcondition: the result is between 0 and 1114111 (U+10FFFF) inclusive, and is not a surrogate (55296 through 57343, U+D800 through U+DFFF). A `String` is a sequence of scalar values, so a string of surrogates cannot be made in the first place.

- `text.code_point(of: "A")` → `65`
- `text.code_point(of: "😀")` → `128512`
- `text.code_point(of: "\n")` → `10`
- `text.code_point(of: "")` → defect
- `text.code_point(of: "ab")` → defect

### text.from_code_point

```text
pub fn from_code_point(value: Int) -> String
```

Makes a string of length 1 from a code point. It is the inverse of `text.code_point`.

Precondition: `value >= 0 and value <= 1114111 and not (value >= 55296 and value <= 57343)`. Passing a value outside the range, or the value of a surrogate, is a defect.

Postcondition: the length of the result is 1.

The precondition makes the caller responsible for supplying a valid scalar value, since other values cannot be represented in a `String`. This function therefore does not return an `Option`.

- `text.from_code_point(value: 97)` → `"a"`
- `text.from_code_point(value: 128512)` → `"😀"`
- `text.from_code_point(value: 0).length` → `1`
- `text.from_code_point(value: 55295).length` → `1` (just before the surrogates)
- `text.from_code_point(value: 57344).length` → `1` (just after the surrogates)
- `text.from_code_point(value: 1114111).length` → `1` (maximum)
- `text.from_code_point(value: 55296)` → defect
- `text.from_code_point(value: 57343)` → defect
- `text.from_code_point(value: 1114112)` → defect
- `text.from_code_point(value: -1)` → defect

## Substrings and searching

### text.slice

```text
pub fn slice(of: String, from: Int, to: Int) -> String
```

Returns the part from position `from` up to just before `to`. Positions are counts of scalar values from 0, where `from` is included and `to` is not.

Precondition: `0 <= from and from <= to and to <= of.length`. A negative position, or a position beyond the end, is a defect. There are no positions counted from the end.

Postcondition: the length of the result is `to - from`.

- `text.slice(of: "a😀bc", from: 1, to: 3)` → `"😀b"`
- `text.slice(of: "abc", from: 0, to: 3)` → `"abc"`
- `text.slice(of: "abc", from: 0, to: 0)` → `""`
- `text.slice(of: "abc", from: 3, to: 3)` → `""`
- `text.slice(of: "abc", from: 2, to: 1)` → defect
- `text.slice(of: "abc", from: -1, to: 2)` → defect
- `text.slice(of: "abc", from: 1, to: 4)` → defect

### text.index_of

```text
pub fn index_of(of: String, part: String) -> Option<Int>
```

Returns the position where `part` first appears as `Some(value: position)`. If it does not appear, the result is `None`. Positions are counted in scalar values.

Edge cases: if `part` is an empty string, the result is `Some(value: 0)` even when `of` is empty. If `part` is longer than `of`, the result is `None`.

Postcondition: for `Some(value: position)`, `position >= 0`, `position + part.length <= of.length`, and `text.slice(of: of, from: position, to: position + part.length) == part`.

A missing match is an ordinary result, represented by `Option` rather than a sentinel such as `-1`.

- `text.index_of(of: "banana", part: "an")` → `Some(value: 1)`
- `text.index_of(of: "banana", part: "x")` → `None`
- `text.index_of(of: "a😀b", part: "b")` → `Some(value: 2)`
- `text.index_of(of: "abc", part: "")` → `Some(value: 0)`
- `text.index_of(of: "", part: "")` → `Some(value: 0)`
- `text.index_of(of: "a", part: "ab")` → `None`

### text.contains, text.starts_with, and text.ends_with

```text
pub fn contains(of: String, part: String) -> Bool
pub fn starts_with(of: String, prefix: String) -> Bool
pub fn ends_with(of: String, suffix: String) -> Bool
```

Each is `true` when the string contains `part`, starts with `prefix`, or ends with `suffix`, respectively. `contains` is the same as `text.index_of` returning `Some`. `starts_with` is the same as `index_of` returning `Some(value: 0)`.

Edge cases: if `part`, `prefix`, or `suffix` is an empty string, the result is `true` whatever `of` is (even if it is empty). If the argument is longer than `of`, the result is `false`.

- `text.contains(of: "banana", part: "nan")` → `true`
- `text.contains(of: "banana", part: "nab")` → `false`
- `text.contains(of: "abc", part: "")` → `true`
- `text.contains(of: "", part: "a")` → `false`
- `text.starts_with(of: "report.txt", prefix: "rep")` → `true`
- `text.starts_with(of: "report.txt", prefix: "ort")` → `false`
- `text.starts_with(of: "a", prefix: "")` → `true`
- `text.starts_with(of: "a", prefix: "ab")` → `false`
- `text.ends_with(of: "report.txt", suffix: ".txt")` → `true`
- `text.ends_with(of: "report.txt", suffix: ".md")` → `false`
- `text.ends_with(of: "a", suffix: "")` → `true`
- `text.ends_with(of: "a", suffix: "ba")` → `false`

### text.count

```text
pub fn count(of: String, part: String) -> Int
```

Returns the number of times `part` appears in `of`. It searches from the left, and after a match it resumes the search from just after the matched part, so overlapping occurrences are not counted.

Precondition: `not text.is_empty(of: part)`. Passing an empty `part` is a defect (an empty string appears at every position, so the count is not defined).

Postcondition: the result is 0 or greater.

- `text.count(of: "a-b-c", part: "-")` → `2`
- `text.count(of: "aaa", part: "aa")` → `1`
- `text.count(of: "aaaa", part: "aa")` → `2`
- `text.count(of: "abc", part: "x")` → `0`
- `text.count(of: "", part: "a")` → `0`
- `text.count(of: "a😀a", part: "a")` → `2`
- `text.count(of: "abc", part: "")` → defect

## Splitting and joining

### text.split

```text
pub fn split(of: String, separator: String) -> List<String>
```

Returns the list of parts separated by `separator`. It searches from the left, and after a match it resumes the search from just after the matched part, so overlapping occurrences are not used as separators. If the text before or after a separator is empty, that part remains in the result as an empty string.

Precondition: `not text.is_empty(of: separator)`. Passing an empty `separator` is a defect. To split into single characters, use `text.chars`.

Postcondition: the length of the result is 1 or more, and `text.join(parts: result, separator: separator)` gives back `of`.

Edge cases: if `of` is an empty string, the result is a list of one empty string (`[""]`), not an empty list. If separators are consecutive, an empty string is placed between them. If a separator is at the start or the end, an empty string is placed at the start or the end.

- `text.split(of: "a,b,,c", separator: ",")` → `["a", "b", "", "c"]`
- `text.split(of: "abc", separator: ",")` → `["abc"]`
- `text.split(of: "", separator: ",")` → `[""]`
- `text.split(of: ",", separator: ",")` → `["", ""]`
- `text.split(of: ",,", separator: ",")` → `["", "", ""]`
- `text.split(of: "a,", separator: ",")` → `["a", ""]`
- `text.split(of: "aaa", separator: "aa")` → `["", "a"]`
- `text.split(of: "a--b---c", separator: "--")` → `["a", "b", "-c"]`
- `text.split(of: "abc", separator: "")` → defect

### text.split_once

```text
pub fn split_once(of: String, separator: String) -> Option<SplitPair>
```

Splits into two parts, before and after the first occurrence of `separator`. It returns `Some(value: SplitPair(before: ..., after: ...))`, and if `separator` does not appear, the result is `None`. The second and later occurrences of `separator` remain in `after` as they are.

Precondition: `not text.is_empty(of: separator)`. Passing an empty `separator` is a defect.

Edge cases: `before` and `after` can be empty strings. If `of` is an empty string, the result is `None`.

Use it when you want to split into two at the first separator, as in `key=value`. Even if the value contains `=`, it remains in `after` as it is.

- `text.split_once(of: "k=v=w", separator: "=")` → `Some(value: text.SplitPair(before: "k", after: "v=w"))`
- `text.split_once(of: "kv", separator: "=")` → `None`
- `text.split_once(of: "", separator: "=")` → `None`
- `text.split_once(of: "=", separator: "=")` → `Some(value: text.SplitPair(before: "", after: ""))`
- `text.split_once(of: "k=", separator: "=")` → `Some(value: text.SplitPair(before: "k", after: ""))`
- `text.split_once(of: "a==b", separator: "==")` → `Some(value: text.SplitPair(before: "a", after: "b"))`
- `text.split_once(of: "abc", separator: "")` → defect

### text.join

```text
pub fn join(parts: List<String>, separator: String) -> String
```

Concatenates the elements of `parts` with `separator` between them. `separator` goes only between elements, not at the start or the end. It is the inverse of `text.split`. There is no precondition.

Edge cases: if `parts` is an empty list, the result is `""`, and if it has one element, the result is that element. `separator` may be an empty string.

- `text.join(parts: ["a", "b", "c"], separator: ", ")` → `"a, b, c"`
- `text.join(parts: [], separator: ",")` → `""`
- `text.join(parts: ["x"], separator: ",")` → `"x"`
- `text.join(parts: ["", ""], separator: "-")` → `"-"`
- `text.join(parts: ["a", "b"], separator: "")` → `"ab"`

### text.lines

```text
pub fn lines(of: String) -> List<String>
```

Splits a string into lines in three steps:

1. Apply `text.split` with `"\n"`.
2. If the last part is an empty string, remove that one part.
3. If a part ends with `"\r"`, remove that one character from its end.

Even if the last line ends with `"\n"`, no empty line is produced after it. Step 3 looks only at the end of each line, so a `"\r"` at the end of a last line that is not followed by `"\n"` is also removed. Only one is removed, and a `"\r"` in the middle of a line is kept.

Unlike `text.split`, an empty string has zero lines (`text.lines(of: "")` is `[]`).

- `text.lines(of: "a\nb\n")` → `["a", "b"]`
- `text.lines(of: "a\r\nb")` → `["a", "b"]`
- `text.lines(of: "a")` → `["a"]`
- `text.lines(of: "")` → `[]`
- `text.lines(of: "\n")` → `[""]`
- `text.lines(of: "\n\n")` → `["", ""]`
- `text.lines(of: "a\n\nb")` → `["a", "", "b"]`
- `text.lines(of: "a\n\n")` → `["a", ""]`
- `text.lines(of: "a\r")` → `["a"]`
- `text.lines(of: "\r")` → `[""]`
- `text.lines(of: "a\rb")` → `["a\rb"]`
- `text.lines(of: "a\r\r\n")` → `["a\r"]`
- `text.lines(of: "a\n\r")` → `["a", ""]`
- `text.lines(of: " ")` → `[" "]`

### text.replace

```text
pub fn replace(of: String, target: String, replacement: String) -> String
```

Replaces every `target` in `of` with `replacement`. By the same rule as `text.split`, it replaces only non-overlapping occurrences, from the left. It does not search again inside the replaced result. If `target` does not appear, the result is `of` unchanged.

Precondition: `not text.is_empty(of: target)`. Passing an empty `target` is a defect.

- `text.replace(of: "a-b-c", target: "-", replacement: "+")` → `"a+b+c"`
- `text.replace(of: "aaa", target: "aa", replacement: "b")` → `"ba"`
- `text.replace(of: "aaaa", target: "aa", replacement: "")` → `""`
- `text.replace(of: "aaa", target: "a", replacement: "aa")` → `"aaaaaa"`
- `text.replace(of: "abc", target: "x", replacement: "y")` → `"abc"`
- `text.replace(of: "", target: "a", replacement: "b")` → `""`
- `text.replace(of: "abc", target: "", replacement: "x")` → defect

## Formatting

### text.repeat

```text
pub fn repeat(of: String, count: Int) -> String
```

Concatenates `of` `count` times.

Precondition: `count >= 0`. A negative count is a defect.

Postcondition: the length of the result is `of.length * count`.

- `text.repeat(of: "ab", count: 3)` → `"ababab"`
- `text.repeat(of: "ab", count: 0)` → `""`
- `text.repeat(of: "", count: 5)` → `""`
- `text.repeat(of: "a", count: -1)` → defect

### text.pad_start and text.pad_end

```text
pub fn pad_start(of: String, width: Int, fill: String) -> String
pub fn pad_end(of: String, width: Int, fill: String) -> String
```

Adds `fill` until the length of `of` is `width`: on the left for `pad_start` and on the right for `pad_end`. If the length of `of` is already `width` or more, the result is `of` unchanged. It never truncates. `width` may be negative (the result is then `of` unchanged).

Precondition: `fill.length == 1`. A `fill` of length 0, or of length 2 or more, is a defect.

Postcondition: if `of.length >= width`, the result is `of`, and if `of.length < width`, the length of the result is `width`.

- `text.pad_start(of: "7", width: 3, fill: "0")` → `"007"`
- `text.pad_start(of: "1234", width: 3, fill: "0")` → `"1234"`
- `text.pad_start(of: "ab", width: 2, fill: "0")` → `"ab"`
- `text.pad_start(of: "", width: 2, fill: "0")` → `"00"`
- `text.pad_start(of: "ab", width: -3, fill: "0")` → `"ab"`
- `text.pad_start(of: "ab", width: 5, fill: "😀")` → `"😀😀😀ab"`
- `text.pad_start(of: "a", width: 3, fill: "")` → defect
- `text.pad_end(of: "ab", width: 5, fill: ".")` → `"ab..."`
- `text.pad_end(of: "abcdef", width: 3, fill: ".")` → `"abcdef"`
- `text.pad_end(of: "", width: 0, fill: ".")` → `""`
- `text.pad_end(of: "a", width: 3, fill: "ab")` → defect

### text.trim, text.trim_start, and text.trim_end

```text
pub fn trim(of: String) -> String
pub fn trim_start(of: String) -> String
pub fn trim_end(of: String) -> String
```

Remove whitespace: `trim` from both ends, `trim_start` from the start, and `trim_end` from the end. Whitespace in the middle of the string remains. A string consisting only of whitespace becomes `""`.

The whitespace removed is only the following four characters. They are the same as the characters for which `text.is_space` returns `true`.

| Character | Code point |
| --- | --- |
| Space | U+0020 |
| Tab `\t` | U+0009 |
| Line feed `\n` | U+000A |
| Carriage return `\r` | U+000D |

Other Unicode whitespace, such as the vertical tab (U+000B), the form feed (U+000C), the no-break space (U+00A0), and the ideographic space (U+3000), is not removed.

- `text.trim(of: "  a b \t\n")` → `"a b"`
- `text.trim(of: "abc")` → `"abc"`
- `text.trim(of: "   ")` → `""`
- `text.trim(of: "")` → `""`
- `text.trim(of: "\u{a0}a\u{a0}")` → `"\u{a0}a\u{a0}"`
- `text.trim(of: "\u{b}a")` → `"\u{b}a"`
- `text.trim_start(of: " \n a b  ")` → `"a b  "`
- `text.trim_start(of: "   ")` → `""`
- `text.trim_start(of: "x")` → `"x"`
- `text.trim_end(of: "  a b \n")` → `"  a b"`
- `text.trim_end(of: "   ")` → `""`
- `text.trim_end(of: "x")` → `"x"`

### text.is_space

```text
pub fn is_space(of: String) -> Bool
```

`true` only when `of` is a single whitespace character (one of `" "`, `"\t"`, `"\n"`, and `"\r"`). An empty string, or a string of two or more characters, gives `false`. It is the criterion for the characters that `trim`, `trim_start`, and `trim_end` remove.

- `text.is_space(of: " ")` → `true`
- `text.is_space(of: "\t")` → `true`
- `text.is_space(of: "\n")` → `true`
- `text.is_space(of: "\r")` → `true`
- `text.is_space(of: "a")` → `false`
- `text.is_space(of: "")` → `false`
- `text.is_space(of: "  ")` → `false`
- `text.is_space(of: "\u{3000}")` → `false`

### text.to_lower and text.to_upper

```text
pub fn to_lower(of: String) -> String
pub fn to_upper(of: String) -> String
```

`to_lower` converts the ASCII uppercase letters `A` through `Z` to lowercase, and `to_upper` converts the ASCII lowercase letters `a` through `z` to uppercase. All other characters are unchanged. There is no locale setting and no Unicode case mapping. `É` and `é`, the full-width `Ａ` and `ａ`, `ß`, and similar characters are left as they are. The length of the result is the same as that of `of`.

To compare ASCII strings without regard to case, convert both with `to_lower` and then compare them with `==`.

- `text.to_lower(of: "Hello, WORLD 42")` → `"hello, world 42"`
- `text.to_lower(of: "É")` → `"É"`
- `text.to_lower(of: "ＡÉß")` → `"ＡÉß"`
- `text.to_lower(of: "")` → `""`
- `text.to_upper(of: "Hello, world 42")` → `"HELLO, WORLD 42"`
- `text.to_upper(of: "é")` → `"é"`
- `text.to_upper(of: "ß")` → `"ß"`
- `text.to_upper(of: "")` → `""`

## Examples

Examples for each function. A call that violates a precondition is confirmed to be a defect with `defects(of: ...)` ([16 Testing](../language/16-testing.md)). An `example` must be attached to a function in the same module, so each module here contains a small function to which the examples are attached.

Examples of positions and searching.

```cernith
module guide.text_positions;

import std.text;

pub fn last_character(of: String) -> Option<String> {
  if of.length == 0 {
    None
  } else {
    Some(value: text.slice(of: of, from: of.length - 1, to: of.length))
  }
}

example chars_split_into_code_points for last_character {
  text.chars(of: "ab") == ["a", "b"]
    and text.chars(of: "") == []
    and text.chars(of: "a😀") == ["a", "😀"]
    and text.chars(of: "é").length == 2
}

example code_point_names_a_character for last_character {
  text.code_point(of: "A") == 65
    and text.code_point(of: "😀") == 128512
    and text.code_point(of: "\n") == 10
}

example code_point_of_the_wrong_length_is_a_defect for last_character {
  defects(of: text.code_point(of: ""))
    and defects(of: text.code_point(of: "ab"))
}

example contains_present_and_absent for last_character {
  text.contains(of: "banana", part: "nan")
    and not text.contains(of: "banana", part: "nab")
    and text.contains(of: "abc", part: "")
    and not text.contains(of: "", part: "a")
}

example count_counts_separate_occurrences for last_character {
  text.count(of: "a-b-c", part: "-") == 2
    and text.count(of: "aaa", part: "aa") == 1
    and text.count(of: "aaaa", part: "aa") == 2
    and text.count(of: "abc", part: "x") == 0
    and text.count(of: "", part: "a") == 0
    and text.count(of: "a😀a", part: "a") == 2
}

example count_of_an_empty_part_is_a_defect for last_character {
  defects(of: text.count(of: "abc", part: ""))
}

example ends_with_cases for last_character {
  text.ends_with(of: "report.txt", suffix: ".txt")
    and not text.ends_with(of: "report.txt", suffix: ".md")
    and text.ends_with(of: "a", suffix: "")
    and not text.ends_with(of: "a", suffix: "ba")
}

example from_code_point_makes_a_character for last_character {
  text.from_code_point(value: 97) == "a"
    and text.from_code_point(value: 128512) == "😀"
    and text.from_code_point(value: 0).length == 1
    and text.from_code_point(value: 55295).length == 1
    and text.from_code_point(value: 57344).length == 1
    and text.from_code_point(value: 1114111).length == 1
}

example from_code_point_outside_the_scalar_values_is_a_defect for last_character {
  defects(of: text.from_code_point(value: 55296))
    and defects(of: text.from_code_point(value: 57343))
    and defects(of: text.from_code_point(value: 1114112))
    and defects(of: text.from_code_point(value: -1))
}

example index_of_first_occurrence for last_character {
  text.index_of(of: "banana", part: "an") == Some(value: 1)
    and text.index_of(of: "banana", part: "x") == None
    and text.index_of(of: "a😀b", part: "b") == Some(value: 2)
    and text.index_of(of: "abc", part: "") == Some(value: 0)
    and text.index_of(of: "", part: "") == Some(value: 0)
    and text.index_of(of: "a", part: "ab") == None
}

example is_empty_cases for last_character {
  text.is_empty(of: "") and not text.is_empty(of: " ")
}

example last_character_cases for last_character {
  last_character(of: "abc") == Some(value: "c")
    and last_character(of: "a😀") == Some(value: "😀")
    and last_character(of: "") == None
}

example slice_counts_code_points for last_character {
  text.slice(of: "a😀bc", from: 1, to: 3) == "😀b"
    and text.slice(of: "abc", from: 0, to: 3) == "abc"
    and text.slice(of: "abc", from: 0, to: 0) == ""
    and text.slice(of: "abc", from: 3, to: 3) == ""
}

example slice_out_of_range_is_a_defect for last_character {
  defects(of: text.slice(of: "abc", from: 2, to: 1))
    and defects(of: text.slice(of: "abc", from: -1, to: 2))
    and defects(of: text.slice(of: "abc", from: 1, to: 4))
}

example starts_with_cases for last_character {
  text.starts_with(of: "report.txt", prefix: "rep")
    and not text.starts_with(of: "report.txt", prefix: "ort")
    and text.starts_with(of: "a", prefix: "")
    and not text.starts_with(of: "a", prefix: "ab")
}
```

Examples of splitting, joining, and replacing.

```cernith
module guide.text_split;

import std.text;

pub fn parse_setting(line: String) -> Option<text.SplitPair> {
  text.split_once(of: text.trim(of: line), separator: "=")
}

example join_with_separator for parse_setting {
  text.join(parts: ["a", "b", "c"], separator: ", ") == "a, b, c"
    and text.join(parts: [], separator: ",") == ""
    and text.join(parts: ["x"], separator: ",") == "x"
    and text.join(parts: ["", ""], separator: "-") == "-"
    and text.join(parts: ["a", "b"], separator: "") == "ab"
}

example lines_drop_the_last_line_end for parse_setting {
  text.lines(of: "a\nb\n") == ["a", "b"]
    and text.lines(of: "a\r\nb") == ["a", "b"]
    and text.lines(of: "a") == ["a"]
    and text.lines(of: "") == []
    and text.lines(of: " ") == [" "]
}

example lines_edge_cases for parse_setting {
  text.lines(of: "\n") == [""]
    and text.lines(of: "\n\n") == ["", ""]
    and text.lines(of: "a\n\nb") == ["a", "", "b"]
    and text.lines(of: "a\n\n") == ["a", ""]
    and text.lines(of: "a\r") == ["a"]
    and text.lines(of: "\r") == [""]
    and text.lines(of: "a\rb") == ["a\rb"]
    and text.lines(of: "a\r\r\n") == ["a\r"]
    and text.lines(of: "a\n\r") == ["a", ""]
}

example parse_setting_reads_key_and_value for parse_setting {
  parse_setting(line: "  mode = fast \n")
    == Some(value: text.SplitPair(before: "mode ", after: " fast"))
    and parse_setting(line: "key=a=b")
      == Some(value: text.SplitPair(before: "key", after: "a=b"))
    and parse_setting(line: "novalue") == None
}

example replace_every_occurrence for parse_setting {
  text.replace(of: "a-b-c", target: "-", replacement: "+") == "a+b+c"
    and text.replace(of: "aaa", target: "aa", replacement: "b") == "ba"
    and text.replace(of: "aaaa", target: "aa", replacement: "") == ""
    and text.replace(of: "aaa", target: "a", replacement: "aa") == "aaaaaa"
    and text.replace(of: "abc", target: "x", replacement: "y") == "abc"
    and text.replace(of: "", target: "a", replacement: "b") == ""
}

example replace_of_an_empty_target_is_a_defect for parse_setting {
  defects(of: text.replace(of: "abc", target: "", replacement: "x"))
}

example split_cases for parse_setting {
  text.split(of: "a,b,,c", separator: ",") == ["a", "b", "", "c"]
    and text.split(of: "abc", separator: ",") == ["abc"]
    and text.split(of: "", separator: ",") == [""]
    and text.split(of: ",", separator: ",") == ["", ""]
    and text.split(of: ",,", separator: ",") == ["", "", ""]
    and text.split(of: "a,", separator: ",") == ["a", ""]
    and text.split(of: "aaa", separator: "aa") == ["", "a"]
    and text.split(of: "a--b---c", separator: "--") == ["a", "b", "-c"]
}

example split_of_an_empty_separator_is_a_defect for parse_setting {
  defects(of: text.split(of: "abc", separator: ""))
}

example split_once_at_the_first_separator for parse_setting {
  text.split_once(of: "k=v=w", separator: "=")
    == Some(value: text.SplitPair(before: "k", after: "v=w"))
    and text.split_once(of: "kv", separator: "=") == None
    and text.split_once(of: "", separator: "=") == None
    and text.split_once(of: "=", separator: "=")
      == Some(value: text.SplitPair(before: "", after: ""))
    and text.split_once(of: "k=", separator: "=")
      == Some(value: text.SplitPair(before: "k", after: ""))
    and text.split_once(of: "a==b", separator: "==")
      == Some(value: text.SplitPair(before: "a", after: "b"))
}

example split_once_of_an_empty_separator_is_a_defect for parse_setting {
  defects(of: text.split_once(of: "abc", separator: ""))
}
```

Examples of formatting.

```cernith
module guide.text_format;

import std.text;

pub fn column(name: String, width: Int) -> String {
  text.pad_end(
    of: text.to_upper(of: text.trim(of: name)),
    width: width,
    fill: ".",
  )
}

example column_cases for column {
  column(name: " id ", width: 5) == "ID..."
    and column(name: "name", width: 2) == "NAME"
}

example is_space_names_ascii_whitespace for column {
  text.is_space(of: " ")
    and text.is_space(of: "\t")
    and text.is_space(of: "\n")
    and text.is_space(of: "\r")
    and not text.is_space(of: "a")
    and not text.is_space(of: "")
    and not text.is_space(of: "  ")
    and not text.is_space(of: "　")
}

example pad_end_fills_on_the_right for column {
  text.pad_end(of: "ab", width: 5, fill: ".") == "ab..."
    and text.pad_end(of: "abcdef", width: 3, fill: ".") == "abcdef"
    and text.pad_end(of: "", width: 0, fill: ".") == ""
}

example pad_end_with_a_bad_fill_is_a_defect for column {
  defects(of: text.pad_end(of: "a", width: 3, fill: "ab"))
}

example pad_start_fills_on_the_left for column {
  text.pad_start(of: "7", width: 3, fill: "0") == "007"
    and text.pad_start(of: "1234", width: 3, fill: "0") == "1234"
    and text.pad_start(of: "ab", width: 2, fill: "0") == "ab"
    and text.pad_start(of: "", width: 2, fill: "0") == "00"
    and text.pad_start(of: "ab", width: -3, fill: "0") == "ab"
    and text.pad_start(of: "ab", width: 5, fill: "😀") == "😀😀😀ab"
}

example pad_start_with_a_bad_fill_is_a_defect for column {
  defects(of: text.pad_start(of: "a", width: 3, fill: ""))
}

example repeat_a_negative_number_of_times_is_a_defect for column {
  defects(of: text.repeat(of: "a", count: -1))
}

example repeat_copies for column {
  text.repeat(of: "ab", count: 3) == "ababab"
    and text.repeat(of: "ab", count: 0) == ""
    and text.repeat(of: "", count: 5) == ""
}

example to_lower_changes_ascii_letters for column {
  text.to_lower(of: "Hello, WORLD 42") == "hello, world 42"
    and text.to_lower(of: "É") == "É"
    and text.to_lower(of: "ＡÉß") == "ＡÉß"
    and text.to_lower(of: "") == ""
}

example to_upper_changes_ascii_letters for column {
  text.to_upper(of: "Hello, world 42") == "HELLO, WORLD 42"
    and text.to_upper(of: "é") == "é"
    and text.to_upper(of: "ß") == "ß"
    and text.to_upper(of: "") == ""
}

example trim_end_removes_trailing_space for column {
  text.trim_end(of: "  a b \n") == "  a b"
    and text.trim_end(of: "   ") == ""
    and text.trim_end(of: "x") == "x"
}

example trim_removes_ascii_whitespace for column {
  text.trim(of: "  a b \t\n") == "a b"
    and text.trim(of: "abc") == "abc"
    and text.trim(of: "   ") == ""
    and text.trim(of: "") == ""
    and text.trim(of: " a ") == " a "
    and text.trim(of: "\u{b}a") == "\u{b}a"
}

example trim_start_removes_leading_space for column {
  text.trim_start(of: " \n a b  ") == "a b  "
    and text.trim_start(of: "   ") == ""
    and text.trim_start(of: "x") == "x"
}
```

## Related

- [03 Types](../language/03-types.md): `String` is a sequence of Unicode scalar values
- [12 Errors and defects](../language/12-errors-and-defects.md): a violation of a precondition is a defect
- [list](list.md): the functions for `List<T>`
- [bytes](bytes.md): conversion between `String` and UTF-8 byte sequences (`bytes.from_text`, `bytes.to_text`)
