# std.path

This module provides functions that handle file paths as strings. Write `import std.path;` and call its functions as in `path.join(parts: ...)`. Every function is pure and does not touch the file system. They only assemble, decompose, and tidy path strings.

A path in `std.path` is a `String` that represents a **relative path separated by `/`**. There is no dedicated type. Write it as in `a/b/c.txt`. Paths that begin with `/` (absolute paths) and paths that use `\` as the separator are not handled (`\` is an ordinary character). The functions of [fs](fs.md), which open files, tidy the path with `path.normalize` before using it.

## Function list

| Function | Return type | Meaning | Example |
| --- | --- | --- | --- |
| `path.join(parts)` | `String` | Joins the non-empty parts with `/` | `["a", "", "b"]` → `"a/b"` |
| `path.normalize(path)` | `Option<String>` | Removes `.`, `..`, and duplicate `/` | `"a/./b//c/../d"` → `Some(value: "a/b/d")` |
| `path.file_name(path)` | `String` | The part after the last `/` | `"a/b/c.txt"` → `"c.txt"` |
| `path.extension(path)` | `String` | The extension of the file name (without the `.`), or `""` if there is none | `"a/b.tar.gz"` → `"gz"` |
| `path.parent(path)` | `String` | The path without its last part, or `"."` if there is none | `"a/b/c.txt"` → `"a/b"` |

No function has a precondition, and any string can be passed.

## Function details

### path.normalize

```text
pub fn normalize(path: String) -> Option<String>
```

Puts a path into the standard form.

- Empty parts (the `//` in `a//b`) and `.` parts are removed.
- A `..` part cancels the part before it. `a/b/../c` becomes `a/c`.
- When the result would be empty (`""`, `"."`, `"./"`, `"a/.."`, and so on), the result is `Some(value: ".")`.
- A trailing `/` is removed.

It returns `None` in the following cases.

- An absolute path (one that begins with `/`).
- A path that goes above the starting point: one with a `..` where no part remains to be canceled, such as `../x` or `a/../../x`.

It returns `None` because **a path that points outside the root cannot be represented as a normalized result**. The functions of `fs` that return a `Result` reject a path for which `normalize` returns `None` as `FsError.Denied`; `fs.is_present` returns `false` for it ([fs](fs.md)).

### path.join

```text
pub fn join(parts: List<String>) -> String
```

Joins those elements of `parts` that are not empty strings with `/`. An empty list, or a list of only empty strings, gives `""`. `join` does not tidy the result. Joining `"a/"` and `"b"` gives `"a//b"`, and `..` and absolute paths are kept as they are. To tidy the result, pass the result of `join` to `normalize`.

### path.file_name

```text
pub fn file_name(path: String) -> String
```

Returns the last part when the path is separated by `/`. If the path does not contain `/`, the result is the whole `path`. If `path` ends with `/`, the result is `""` (because the last part is empty).

### path.extension

```text
pub fn extension(path: String) -> String
```

Returns the part of `file_name(path)` after its last `.`, without the `.`. The result is `""` in the following cases.

- The file name has no `.` (`noext`).
- The file name begins with `.` or ends with `.` (`.hidden`, `x.`).

Even if the file name contains several `.`, only the part after the last `.` is the extension (`gz` for `b.tar.gz`). However, when the file name begins with `.`, the extension is `""` even if there is another `.` later, as in `.hidden.txt`.

### path.parent

```text
pub fn parent(path: String) -> String
```

Returns the path without its last part, when the path is separated by `/`. For a path that does not contain `/` (and for the empty string), the result is `"."`. It does not `normalize`, so the parent of `"a//b"` is `"a/"`.

## Example

```cernith
module guide.paths;

import std.path;

pub fn extension_of(target: String) -> String {
  path.extension(path: target)
}

example extension_of_follows_the_last_dot for extension_of {
  extension_of(target: "a/b.tar.gz") == "gz"
    and extension_of(target: "c.txt") == "txt"
    and extension_of(target: "noext") == ""
    and extension_of(target: ".hidden") == ""
    and extension_of(target: "x.") == ""
    and extension_of(target: ".hidden.txt") == ""
    and extension_of(target: "") == ""
}

pub fn folder_of(target: String) -> String {
  path.parent(path: target)
}

example folder_of_drops_the_last_part for folder_of {
  folder_of(target: "a/b/c.txt") == "a/b"
    and folder_of(target: "a/b") == "a"
    and folder_of(target: "c.txt") == "."
    and folder_of(target: "") == "."
}

pub fn joined(parts: List<String>) -> String {
  path.join(parts: parts)
}

example joined_skips_empty_parts for joined {
  joined(parts: ["a", "", "b"]) == "a/b"
    and joined(parts: ["a"]) == "a"
    and joined(parts: []) == ""
    and joined(parts: ["", ""]) == ""
    and joined(parts: ["a/", "b"]) == "a//b"
}

pub fn name_of(target: String) -> String {
  path.file_name(path: target)
}

example name_of_is_the_last_part for name_of {
  name_of(target: "a/b/c.txt") == "c.txt"
    and name_of(target: "c.txt") == "c.txt"
    and name_of(target: "a/b/") == ""
    and name_of(target: "") == ""
}

pub fn tidy(target: String) -> Option<String> {
  path.normalize(path: target)
}

example tidy_refuses_what_leaves_the_start for tidy {
  tidy(target: "../x") == None
    and tidy(target: "a/../../x") == None
    and tidy(target: "..") == None
    and tidy(target: "/etc/passwd") == None
    and tidy(target: "/") == None
}

example tidy_resolves_dots for tidy {
  tidy(target: "a/./b//c/../d") == Some(value: "a/b/d")
    and tidy(target: "a/b/") == Some(value: "a/b")
    and tidy(target: "") == Some(value: ".")
    and tidy(target: "./") == Some(value: ".")
    and tidy(target: "a/..") == Some(value: ".")
}
```

## Related

- [fs](fs.md): normalizes the path with `normalize` and then opens a file under the root
- [text](text.md): functions on strings
