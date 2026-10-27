# std.fs

A module that reads and writes files and folders through capabilities. Write `import std.fs;` and call the functions as in `fs.read_text(access: ..., path: ...)`. Every function touches only paths under the **root** (a folder) of the capability received as `access:`.

There are two capabilities. `Fs` can read and write under its root, and `ReadOnlyFs` can only read. Both are requested by writing the root in the program’s `manifest` declaration, and are received as the second argument of `main`. They cannot be created anywhere else (error `E1301`). A function that uses files takes a capability as an argument and also declares `uses io` ([17 Resources and capabilities](../language/17-resources-and-capabilities.md)).

An open file is a **resource** and exists only inside a `using`. When the `using` ends, the file is closed automatically, whether the body succeeded or failed. A file cannot be left unclosed, and a file cannot be used after it has been closed.

## Types

```text
pub capability Fs {
  root: String,
}

pub capability ReadOnlyFs {
  root: String,
}

pub resource ReadFile release close_read;
pub resource WriteFile release close_write;

pub variant FsError {
  NotFound(path: String),
  Denied(path: String),
  CloseFailed(path: String),
  Failed(path: String, detail: String),
}
```

- `Fs` and `ReadOnlyFs` are capabilities that hold the root in `root`. `root` cannot be read directly (error `E0215`). They cannot be compared with `==` either. The root is obtained with `root_of` and `read_only_root_of`.
- `ReadFile` and `WriteFile` are resources that represent an open file. The functions named in `release` (`close_read`, `close_write`) are called at the end of the `using`.
- `FsError` is the reason an operation failed.

| Case | Meaning |
| --- | --- |
| `NotFound(path: ...)` | There is no file or folder at that path |
| `Denied(path: ...)` | A path that points outside the root ([Roots and paths](#roots-and-paths)), or a path for which the operation is not allowed |
| `CloseFailed(path: ...)` | Closing the file failed |
| `Failed(path: ..., detail: ...)` | Any other failure. `detail` holds a string giving the reason (for example, `"malformed-text"` when the file read by `read_text` is not UTF-8 text) |

For functions that take a path and can fail, such as `open_read`, `list_names`, and `remove`, `path` holds the path string that was passed in, unchanged. For a failure in an operation after the file has been opened (such as `read_all`), `path` holds the path of that file including the root (which differs from the path that was passed in unless the root is `"."`).

`std.fs` also defines the types `Entry` and `EntryKind`, which `fs.metadata` returns.

```text
pub record Entry {
  name: String,
  kind: EntryKind,
  size: Int,
  modified_ms: Int,
}

pub variant EntryKind {
  File,
  Folder,
  Link,
  Missing,
}
```

> Note: `std.fs` also defines the types `Change` and `ChangeKind`. They represent notifications of changes, but the functions of `std.fs` do not use them.

## Roots and paths

The root written in the `manifest` is the path of a folder as seen from the folder in which the program runs (the working directory). `"."` is the working directory itself.

```text
pub manifest Caps {
  data: fs.ReadOnlyFs(root: "data"),
  out: fs.Fs(root: "reports"),
}
```

The `path` passed to a function is a path relative to the root, with `/` as the separator ([path](path.md)). The function normalizes the path with `path.normalize` before joining it to the root. Therefore `dir/./b.txt` and `dir/b.txt` are the same file. A path for which `normalize` returns `None` is treated as one that **points outside the root** and is rejected with `Err(error: FsError.Denied(path: ...))`.

- An absolute path (`/etc/passwd`).
- A path that goes above the root (`../x`, `a/../../x`).

Using `..` in the middle of a path, as in `a/../b.txt`, is valid as long as it does not leave the root. The path is rejected without touching the file system, so nothing is revealed about what lies outside the root.

A capability can be **narrowed** with `narrow` and `read_only`. There is no function that widens one. A function that has received a capability still has no way to get outside the root.

## using and resources

The functions that open a file (`open_read`, `open_write`) return `Result<ReadFile, FsError>` or `Result<WriteFile, FsError>`. This resource is received with `using`.

```text
using file = fs.open_read(access: data, path: "notes.txt") {
  fs.read_all(file: file)
}
```

- `using name = acquire expression { body }` binds the file to `name` and runs the body if the acquire expression is `Ok`, and closes the file however the body ends (whether at the end of the body or by an exit through `?`). If the acquire expression is `Err`, that `Err` is the value of the whole `using`, and there is no closing step.
- The type of the acquire expression is `Result<resource, E>` and the type of the body is `Result<T, E>`, and they must have **the same `E`**. The whole `using` is a `Result<T, E>`. When the body needs to return an error of a different type, convert it to `FsError` outside the `using` with `result.map_error` (error `E1102`).
- If the body succeeds and the closing step fails, the value of the `using` is the failure of the closing step (`CloseFailed`). If the body had failed, the error of the body becomes the value of the `using`.
- `name` can be used only as an argument to a function. It cannot be returned, put in a list or a record, or taken out of the body (error `E1101`). Return the data read from the file as the value of the body.
- `return` cannot be used in the body of a `using` (error `E1107`). The value of the body is the value of the `using`.

Binding the acquire expression without putting it in a `using` is error `E1103`.

```cernith-error E1103
module guide.fs_open_outside_using;

import std.fs;

pub fn load(files: fs.ReadOnlyFs) -> Result<String, fs.FsError> uses io {
  let opened = fs.open_read(access: files, path: "a.txt");
  match opened {
    Ok(value: file) => fs.read_all(file: file),
    Err(error: error) => Err(error: error),
  }
}
```

If the error type of the body differs from `FsError`, it is error `E1102`.

```cernith-error E1102
module guide.fs_other_error_type;

import std.fs;

pub fn size(files: fs.ReadOnlyFs) -> Result<Int, String> uses io {
  using file = fs.open_read(access: files, path: "a.txt") {
    match fs.read_all(file: file) {
      Ok(value: text) => Ok(value: text.length),
      Err(error: _) => Err(error: "unreadable"),
    }
  }
}
```

Trying to take the file out of the body is error `E1101`.

```cernith-error E1101
module guide.fs_file_escapes;

import std.fs;

pub fn keep(
  files: fs.ReadOnlyFs,
) -> Result<List<fs.ReadFile>, fs.FsError> uses io {
  using file = fs.open_read(access: files, path: "a.txt") {
    Ok(value: [file])
  }
}
```

If you only need to read or write a whole file, the whole-file functions such as `read_text` and `write_text` write the `using` for you internally.

## Function list

Functions that read or write a whole file:

| Function | Return type | Meaning |
| --- | --- | --- |
| `fs.read_text(access, path)` | `Result<String, FsError>` | Reads the whole file as text |
| `fs.read_file_bytes(access, path)` | `Result<Bytes, FsError>` | Reads the whole file as a byte sequence |
| `fs.write_text(access, path, text)` | `Result<Unit, FsError>` | Replaces the contents of the file with text |
| `fs.write_file_bytes(access, path, bytes)` | `Result<Unit, FsError>` | Replaces the contents of the file with a byte sequence |
| `fs.append_text(access, path, text)` | `Result<Unit, FsError>` | Adds text at the end of the file (the file is made if it is missing) |
| `fs.append_bytes(access, path, bytes)` | `Result<Unit, FsError>` | Adds a byte sequence at the end of the file |
| `fs.replace_text(access, path, text)` | `Result<Unit, FsError>` | Replaces the contents of the file so that a crash leaves the old contents or the new, never a mixture |
| `fs.replace_bytes(access, path, bytes)` | `Result<Unit, FsError>` | The byte-sequence version of `replace_text` |

Functions that use an open file:

| Function | Return type | Meaning |
| --- | --- | --- |
| `fs.open_read(access, path)` | `Result<ReadFile, FsError>` | Opens for reading (used together with `using`) |
| `fs.open_write(access, path)` | `Result<WriteFile, FsError>` | Opens for writing (used together with `using`) |
| `fs.read_all(file)` | `Result<String, FsError>` | Reads the whole file as text |
| `fs.read_lines(file)` | `Result<List<String>, FsError>` | Reads the whole file as a list of lines |
| `fs.read_bytes(file)` | `Result<Bytes, FsError>` | Reads the whole file as a byte sequence |
| `fs.read_chunk(file, count)` | `Result<Bytes, FsError>` | Reads up to the next `count` bytes |
| `fs.write_all(file, text)` | `Result<Unit, FsError>` | Writes text |
| `fs.write_bytes(file, bytes)` | `Result<Unit, FsError>` | Writes a byte sequence |
| `fs.close_read(file)`, `fs.close_write(file)` | `Result<Unit, FsError>` | Closes the file (called by `using`) |

Folders and checking for existence:

| Function | Return type | Meaning |
| --- | --- | --- |
| `fs.list_names(access, path)` | `Result<List<String>, FsError>` | The list of names in a folder |
| `fs.is_present(access, path)` | `Bool` | Whether a file or folder exists at the path |
| `fs.is_folder(access, path)` | `Bool` | Whether the path is a folder |
| `fs.metadata(access, path)` | `Result<Entry, FsError>` | What the path is: its kind, size, and time of the last write |
| `fs.make_folder(access, path)` | `Result<Unit, FsError>` | Makes a folder and the folders above it that are missing |
| `fs.rename(access, from, to)` | `Result<Unit, FsError>` | Moves a file or folder to a new path |
| `fs.remove(access, path)` | `Result<Unit, FsError>` | Deletes a file |

Functions that create or inspect capabilities:

| Function | Return type | Meaning |
| --- | --- | --- |
| `fs.read_only(access)` | `ReadOnlyFs` | A read-only capability with the same root |
| `fs.narrow(access, folder)` | `Result<Fs, FsError>` | An `Fs` narrowed to a folder under the root |
| `fs.narrow_read_only(access, folder)` | `Result<ReadOnlyFs, FsError>` | A `ReadOnlyFs` narrowed to a folder under the root |
| `fs.root_of(access)` | `String` | The root of an `Fs` |
| `fs.read_only_root_of(access)` | `String` | The root of a `ReadOnlyFs` |
| `fs.simulated()`, `fs.simulated_read_only()` | `Fs`, `ReadOnlyFs` | Capabilities for tests |

Others:

| Function | Return type | Meaning |
| --- | --- | --- |
| `fs.lines_of(text)` | `List<String>` | Splits text into lines |
| `fs.describe(error)` | `String` | Turns an `FsError` into a string |

The functions that read (including `is_folder` and `metadata`) take a `ReadOnlyFs`, and the functions that write, as well as `append_text`, `append_bytes`, `replace_text`, `replace_bytes`, `make_folder`, `rename`, `remove`, and `narrow`, take an `Fs`. To read with an `Fs`, convert it with `fs.read_only(access: files)`. For a function that only reads, it is good form to write its argument as a `ReadOnlyFs`. Then the type shows that the function does not write.

```cernith-error E0301
module guide.fs_read_only_cannot_write;

import std.fs;

pub fn save(files: fs.ReadOnlyFs) -> Result<Unit, fs.FsError> uses io {
  fs.write_text(access: files, path: "a.txt", text: "x")
}
```

`uses io` is needed for the functions, among those that receive a capability, that query the world. `read_only`, `narrow`, `narrow_read_only`, `root_of`, `read_only_root_of`, `lines_of`, `describe`, `simulated`, and `simulated_read_only` are pure.

## Function details

### fs.read_text

```text
pub fn read_text(
  access: ReadOnlyFs,
  path: String,
) -> Result<String, FsError> uses io
```

Opens the file, reads the whole of it as UTF-8 text, and closes it. If there is no file, the result is `NotFound`; if the path is outside the root, it is `Denied`. If the file is not UTF-8 text, the result is `Failed(path: ..., detail: "malformed-text")`. An empty file gives `Ok(value: "")`.

### fs.read_file_bytes

```text
pub fn read_file_bytes(
  access: ReadOnlyFs,
  path: String,
) -> Result<Bytes, FsError> uses io
```

Reads the whole file as a byte sequence. It can read any file, including text files.

### fs.write_text

```text
pub fn write_text(
  access: Fs,
  path: String,
  text: String,
) -> Result<Unit, FsError> uses io
```

Opens the file for writing, writes `text`, and closes it. If there is no file, it creates one; if there is one, it replaces it, **discarding the previous contents** (this is not an append). If `text` is empty, the file becomes empty. If the path is outside the root, the result is `Denied`.

The result when the destination folder does not exist differs by execution environment. On the real file system, the result is `NotFound`. The simulated world creates the intermediate folders and writes. The examples in this document write only to places where the folder exists.

### fs.write_file_bytes

```text
pub fn write_file_bytes(
  access: Fs,
  path: String,
  bytes: Bytes,
) -> Result<Unit, FsError> uses io
```

The byte-sequence version of `write_text`. It replaces the contents of the file with `bytes`.

### fs.append_text and fs.append_bytes

```text
pub fn append_text(
  access: Fs,
  path: String,
  text: String,
) -> Result<Unit, FsError> uses io

pub fn append_bytes(
  access: Fs,
  path: String,
  bytes: Bytes,
) -> Result<Unit, FsError> uses io
```

Add `text` or `bytes` at the end of the file. The earlier contents are kept, and no newline is added. If there is no file, one is made; the folder above it must exist on the real file system (the result is `NotFound` if it does not; the simulated world creates the folders). If the path is outside the root, the result is `Denied`. Use them for a log that grows.

### fs.replace_text and fs.replace_bytes

```text
pub fn replace_text(
  access: Fs,
  path: String,
  text: String,
) -> Result<Unit, FsError> uses io

pub fn replace_bytes(
  access: Fs,
  path: String,
  bytes: Bytes,
) -> Result<Unit, FsError> uses io
```

Write `text` or `bytes` as the whole contents of the file so that a crash in the middle leaves the old contents or the new, never a mixture. The bytes go to a temporary file in the same folder (`.NAME.cernith-tmp`), are flushed, and the temporary file is renamed over the target. `write_text` and `write_file_bytes` empty the file first, so a crash can leave it empty or partly written. The target is created if it is missing. The result is `NotFound` if the folder does not exist, and `Denied` if the path is outside the root.

### fs.open_read and fs.open_write

```text
pub fn open_read(
  access: ReadOnlyFs,
  path: String,
) -> Result<ReadFile, FsError> uses io

pub fn open_write(
  access: Fs,
  path: String,
) -> Result<WriteFile, FsError> uses io
```

Opens a file. Use it as the acquire expression of a `using` ([using and resources](#using-and-resources)). Like `write_text`, `open_write` creates the file if there is none, and empties it at the moment of opening if there is one.

### fs.read_all, fs.read_lines, and fs.read_bytes

```text
pub fn read_all(file: ReadFile) -> Result<String, FsError> uses io
pub fn read_lines(file: ReadFile) -> Result<List<String>, FsError> uses io
pub fn read_bytes(file: ReadFile) -> Result<Bytes, FsError> uses io
```

Read the contents of an open `ReadFile` as text, as a list of lines, and as a byte sequence. `read_all` gives `Failed(..., detail: "malformed-text")` when the contents are not UTF-8 text. `read_lines` is the result of `read_all` split with `fs.lines_of`. These three always return the whole file, however much has been read before; only `read_chunk` keeps a read position (a `read_chunk` after a `read_all` continues from where the previous `read_chunk` stopped).

### fs.read_chunk

```text
pub fn read_chunk(
  file: ReadFile,
  count: Int,
) -> Result<Bytes, FsError> uses io {
  requires count >= 0;
  ...
}
```

Reads up to the next `count` bytes of the file. Each call continues from where the previous call stopped. Near the end of the file it returns fewer than `count` bytes, and after the end has been reached it returns an empty byte sequence. Use it to process a large file a little at a time without holding the whole file in memory.

**Precondition:** `count >= 0`. A call that does not satisfy it is a defect. If `count` is 0, the result is an empty byte sequence.

### fs.write_all and fs.write_bytes

```text
pub fn write_all(file: WriteFile, text: String) -> Result<Unit, FsError> uses io
pub fn write_bytes(file: WriteFile, bytes: Bytes) -> Result<Unit, FsError> uses io
```

Write to an open `WriteFile`. Successive calls are joined in the order written (writing `"one"` and `"two"` gives `"onetwo"`). No newline is added.

### fs.close_read and fs.close_write

```text
pub fn close_read(file: ReadFile) -> Result<Unit, FsError> uses io
pub fn close_write(file: WriteFile) -> Result<Unit, FsError> uses io
```

These are the functions given as the `release` of the resources, and they are called automatically when the `using` ends. The program does not need to call them. If closing fails, the result is `CloseFailed`.

### fs.list_names

```text
pub fn list_names(
  access: ReadOnlyFs,
  path: String,
) -> Result<List<String>, FsError> uses io
```

Returns a list of the **names** (names only, not paths) of the files and folders in the folder `path`. They are ordered by code point. The contents of subfolders are not included. The path that denotes the folder may also be `"."` (or `""`), which points to the root itself. If there is no such folder, the result is `NotFound`. If the path of a file is passed, the result is `Failed(path: ..., detail: "failed")`.

### fs.is_present

```text
pub fn is_present(access: ReadOnlyFs, path: String) -> Bool uses io
```

Returns `true` if there is a file or folder at the path. For a path outside the root, the result is `false`, not an `Err` of `Denied`. Because it does not return a `Result`, the reason for a failure is not available.

### fs.is_folder

```text
pub fn is_folder(access: ReadOnlyFs, path: String) -> Bool uses io
```

Returns `true` if the path is a folder, that is, if `list_names` succeeds on it. A file, a missing path, and a path outside the root give `false`.

### fs.metadata

```text
pub fn metadata(
  access: ReadOnlyFs,
  path: String,
) -> Result<Entry, FsError> uses io
```

Returns what the path is, without following a link. `name` is the last part of the path, `kind` is `File`, `Folder`, or `Link`, and `size` is the number of bytes of a file (0 for a folder and a link). `modified_ms` is the time of the last write, in milliseconds since 1970-01-01 UTC. The simulated world keeps no times and gives 0.

A path that is not there is not an error: the result is `Ok` with `kind: EntryKind.Missing`. A path outside the root is `Err(error: FsError.Denied(path: ...))`.

### fs.make_folder

```text
pub fn make_folder(access: Fs, path: String) -> Result<Unit, FsError> uses io
```

Makes the folder and the folders above it that are missing, like `mkdir -p`. A folder that is there already is not an error. On the real file system, the result is an `Err` (for example `NotFound` or `Failed`, depending on the system) if a part of the path is a file. The simulated world holds only files: a folder exists there while a file is below it, and `make_folder` succeeds without leaving a trace.

### fs.rename

```text
pub fn rename(
  access: Fs,
  from: String,
  to: String,
) -> Result<Unit, FsError> uses io
```

Moves a file, or a folder with what is below it, to the new path `to`. Both paths are checked against the root, and a path outside it is `Denied`. A file at the new path is replaced (in one step where the system can); a folder is not, and the result is an `Err` (for example `Denied`). The result is `NotFound` if `from` is not there; the `path` of the error is `from`.

### fs.remove

```text
pub fn remove(access: Fs, path: String) -> Result<Unit, FsError> uses io
```

Deletes a file. If there is no file, the result is `NotFound`. There is no function that deletes a folder, and the result of passing the path of a folder is `Denied` or `NotFound`, depending on the execution environment.

### fs.read_only

```text
pub fn read_only(access: Fs) -> ReadOnlyFs
```

Returns a read-only capability with the same root. It can do no more than the original `Fs`.

### fs.narrow and fs.narrow_read_only

```text
pub fn narrow(access: Fs, folder: String) -> Result<Fs, FsError>
pub fn narrow_read_only(
  access: ReadOnlyFs,
  folder: String,
) -> Result<ReadOnlyFs, FsError>
```

Returns a capability whose root is the folder `folder` under the root. `folder` is normalized with `path.normalize`, and if it points outside the root (becomes `None`), the result is `Err(error: FsError.Denied(path: folder))`. `"."` and `""` leave the root as it was. Whether the folder actually exists is not checked.

The root of the narrowed capability is the original root with `folder` appended (narrowing `"reports"` to `"2024"` gives `"reports/2024"`). The narrowed capability cannot get outside `folder`. It is used to pass a library function only the folder it needs, rather than all the files of the program.

### fs.root_of and fs.read_only_root_of

```text
pub fn root_of(access: Fs) -> String
pub fn read_only_root_of(access: ReadOnlyFs) -> String
```

Return the root of the capability.

### fs.lines_of

```text
pub fn lines_of(text: String) -> List<String>
```

Splits text into lines. The separator is `\n`, and a `\r` immediately before it is removed. A newline at the end of the last line does not start a new line. Empty text gives `[]`, and text consisting only of `"\n"` gives `[""]` (one empty line).

### fs.describe

```text
pub fn describe(error: FsError) -> String
```

| Error | String |
| --- | --- |
| `NotFound(path: "a")` | `"a: not found"` |
| `Denied(path: "b")` | `"b: denied"` |
| `CloseFailed(path: "c")` | `"c: closing failed"` |
| `Failed(path: "d", detail: "x")` | `"d: x"` |

### fs.simulated and fs.simulated_read_only

```text
pub fn simulated() -> Fs
pub fn simulated_read_only() -> ReadOnlyFs
```

Return capabilities for tests. The root is `"."`, the working directory of the simulated world. They can be called only inside an `example` or a `property`, and inside a function passed to `std.test.with_world` or `std.sim.with_sim`. Calling them anywhere else is error `E1302`.

The files of the world are given by writing `sim.file(path: ..., content: ...)` in the `setup` of `std.sim.with_sim`, and are checked after the run with `sim.file_content` and similar functions ([sim](sim.md)). The world of `std.test.with_world` has no files.

## Examples

### Whole-file functions and choosing between them

The following module is a program that counts the lines of `notes.txt` in the `data` folder and writes the result to the `reports` folder. Because the `manifest` makes the two folders separate capabilities, the types do not allow writing to `data` or reading `reports`.

In tests, `run` passes `fs.simulated()` and runs the body. `main` is given a record `Caps(...)` of the same shape as the `manifest`.

```cernith
module guide.reports;

import std.console;
import std.fs;
import std.int;
import std.list;
import std.sim;
import std.text as text_lib;

pub manifest Caps {
  console: console.Console,
  data: fs.ReadOnlyFs(root: "data"),
  out: fs.Fs(root: "reports"),
}

pub fn discard_summary(
  out: fs.Fs,
  name: String,
) -> Result<Unit, fs.FsError> uses io {
  fs.remove(access: out, path: name ++ ".summary")
}

example discard_summary_of_a_missing_file_is_not_found for discard_summary {
  run(
    setup: [],
    body: fn(files: fs.Fs) -> Result<Unit, fs.FsError> uses io {
      discard_summary(out: files, name: "notes")
    },
  ).value
    == Err(error: fs.FsError.NotFound(path: "notes.summary"))
}

example discard_summary_removes_the_file for discard_summary {
  match run(
    setup: [sim.file(path: "notes.summary", content: "3 lines")],
    body: fn(files: fs.Fs) -> Result<Unit, fs.FsError> uses io {
      discard_summary(out: files, name: "notes")
    },
  ) {
    outcome => outcome.value == Ok(value: ())
      and sim.file_paths(outcome: outcome) == [],
  }
}

pub fn has_file(data: fs.ReadOnlyFs, path: String) -> Bool uses io {
  fs.is_present(access: data, path: path)
}

example has_file_looks_for_the_path for has_file {
  run(
    setup: [sim.file(path: "d/notes.txt", content: "")],
    body: fn(files: fs.Fs) -> List<Bool> uses io {
      let data = fs.read_only(access: files);
      [
        has_file(data: data, path: "d/notes.txt"),
        has_file(data: data, path: "d/./notes.txt"),
        has_file(data: data, path: "d"),
        has_file(data: data, path: "."),
        has_file(data: data, path: "notes.txt"),
        has_file(data: data, path: "../d/notes.txt"),
        has_file(data: data, path: "/d/notes.txt"),
      ]
    },
  ).value
    == [true, true, true, true, false, false, false]
}

pub fn line_count(
  data: fs.ReadOnlyFs,
  path: String,
) -> Result<Int, fs.FsError> uses io {
  using file = fs.open_read(access: data, path: path) {
    let lines = fs.read_lines(file: file)?;
    Ok(value: lines.length)
  }
}

example line_count_counts_lines for line_count {
  run(
    setup: [
      sim.file(path: "notes.txt", content: "a\nb\nc\n"),
      sim.file(path: "empty.txt", content: ""),
      sim.file(path: "partial.txt", content: "a\nb"),
    ],
    body: fn(files: fs.Fs) -> List<Result<Int, fs.FsError>> uses io {
      let data = fs.read_only(access: files);
      [
        line_count(data: data, path: "notes.txt"),
        line_count(data: data, path: "empty.txt"),
        line_count(data: data, path: "partial.txt"),
      ]
    },
  ).value
    == [Ok(value: 3), Ok(value: 0), Ok(value: 2)]
}

example line_count_reports_a_missing_file_and_a_path_outside for line_count {
  run(
    setup: [sim.file(path: "notes.txt", content: "a\n")],
    body: fn(files: fs.Fs) -> List<Result<Int, fs.FsError>> uses io {
      let data = fs.read_only(access: files);
      [
        line_count(data: data, path: "nope.txt"),
        line_count(data: data, path: "../notes.txt"),
      ]
    },
  ).value
    == [
      Err(error: fs.FsError.NotFound(path: "nope.txt")),
      Err(error: fs.FsError.Denied(path: "../notes.txt")),
    ]
}

pub fn main(args: List<String>, caps: Caps) -> Int uses io {
  match summarize(data: caps.data, out: caps.out, name: "notes") {
    Ok(value: _) => 0,
    Err(error: error) => {
      discard console.write_error_line(
        access: caps.console,
        text: fs.describe(error: error),
      );
      1
    },
  }
}

example main_reports_a_missing_file_on_the_error_stream for main {
  match sim.with_sim(
    world: sim.quiet_world(),
    setup: [],
    run: fn() -> Int uses clock, io, random {
      main(
        args: [],
        caps: Caps(
          console: console.simulated(),
          data: fs.simulated_read_only(),
          out: fs.simulated(),
        ),
      )
    },
  ) {
    outcome => outcome.value == 1
      and sim.stderr_lines(outcome: outcome) == ["notes.txt: not found"]
      and sim.file_paths(outcome: outcome) == [],
  }
}

example main_writes_the_summary for main {
  match sim.with_sim(
    world: sim.quiet_world(),
    setup: [sim.file(path: "notes.txt", content: "a\nb\n")],
    run: fn() -> Int uses clock, io, random {
      main(
        args: [],
        caps: Caps(
          console: console.simulated(),
          data: fs.simulated_read_only(),
          out: fs.simulated(),
        ),
      )
    },
  ) {
    outcome => outcome.value == 0
      and sim.file_content(outcome: outcome, path: "notes.summary")
        == Some(value: "2 lines")
      and sim.stderr_lines(outcome: outcome) == [],
  }
}

fn run<T>(
  setup: List<String>,
  body: fn(files: fs.Fs) -> T uses io,
) -> sim.SimOutcome<T> {
  sim.with_sim(
    world: sim.quiet_world(),
    setup: setup,
    run: fn() -> T uses clock, io, random {
      body(files: fs.simulated())
    },
  )
}

pub fn summarize(
  data: fs.ReadOnlyFs,
  out: fs.Fs,
  name: String,
) -> Result<Unit, fs.FsError> uses io {
  let count = line_count(data: data, path: name ++ ".txt")?;
  fs.write_text(
    access: out,
    path: name ++ ".summary",
    text: int.to_text(value: count) ++ " lines",
  )
}

example summarize_writes_the_count for summarize {
  match run(
    setup: [sim.file(path: "notes.txt", content: "a\nb\n")],
    body: fn(files: fs.Fs) -> Result<Unit, fs.FsError> uses io {
      summarize(data: fs.read_only(access: files), out: files, name: "notes")
    },
  ) {
    outcome => outcome.value == Ok(value: ())
      and sim.file_content(outcome: outcome, path: "notes.summary")
        == Some(value: "2 lines")
      and sim.open_handles(outcome: outcome) == 0,
  }
}

pub fn summary_names(
  out: fs.ReadOnlyFs,
) -> Result<List<String>, fs.FsError> uses io {
  let names = fs.list_names(access: out, path: ".")?;
  Ok(
    value: list.filter(
      items: names,
      keep: fn(item: String) -> Bool {
        text_lib.ends_with(of: item, suffix: ".summary")
      },
    ),
  )
}

example summary_names_lists_only_the_summaries for summary_names {
  run(
    setup: [
      sim.file(path: "b.summary", content: ""),
      sim.file(path: "a.summary", content: ""),
      sim.file(path: "c.txt", content: ""),
      sim.file(path: "d/e.summary", content: ""),
    ],
    body: fn(files: fs.Fs) -> Result<List<String>, fs.FsError> uses io {
      summary_names(out: fs.read_only(access: files))
    },
  ).value
    == Ok(value: ["a.summary", "b.summary"])
    and run(
      setup: [],
      body: fn(files: fs.Fs) -> Result<List<String>, fs.FsError> uses io {
        summary_names(out: fs.read_only(access: files))
      },
    ).value
      == Ok(value: [])
}
```

### Using open files

Examples that combine `open_read` and `open_write` with `using` to process line by line, read in chunks, and read and write byte sequences.

```cernith
module guide.file_handles;

import std.bytes;
import std.fs;
import std.sim;

pub fn first_line(
  files: fs.ReadOnlyFs,
  path: String,
) -> Result<String, fs.FsError> uses io {
  using file = fs.open_read(access: files, path: path) {
    let lines = fs.read_lines(file: file)?;
    if lines.length == 0 {
      Ok(value: "")
    } else {
      Ok(value: lines[0])
    }
  }
}

example first_line_reads_the_first_line for first_line {
  run(
    setup: [
      sim.file(path: "a.txt", content: "one\ntwo\n"),
      sim.file(path: "empty.txt", content: ""),
      sim.file(path: "crlf.txt", content: "one\r\ntwo"),
    ],
    body: fn(files: fs.Fs) -> List<Result<String, fs.FsError>> uses io {
      let data = fs.read_only(access: files);
      [
        first_line(files: data, path: "a.txt"),
        first_line(files: data, path: "empty.txt"),
        first_line(files: data, path: "crlf.txt"),
        first_line(files: data, path: "missing.txt"),
      ]
    },
  ).value
    == [
      Ok(value: "one"),
      Ok(value: ""),
      Ok(value: "one"),
      Err(error: fs.FsError.NotFound(path: "missing.txt")),
    ]
}

pub fn head_bytes(
  files: fs.ReadOnlyFs,
  path: String,
  count: Int,
) -> Result<Bytes, fs.FsError> uses io {
  requires count >= 0;

  using file = fs.open_read(access: files, path: path) {
    fs.read_chunk(file: file, count: count)
  }
}

example head_bytes_of_a_negative_count_is_a_defect for head_bytes {
  defects(
    of: run(
      setup: [sim.file(path: "a.txt", content: "12345")],
      body: fn(files: fs.Fs) -> Result<Bytes, fs.FsError> uses io {
        head_bytes(files: fs.read_only(access: files), path: "a.txt", count: -1)
      },
    ).value,
  )
}

example head_bytes_reads_up_to_count_bytes for head_bytes {
  run(
    setup: [sim.file(path: "a.txt", content: "12345")],
    body: fn(files: fs.Fs) -> List<Result<Bytes, fs.FsError>> uses io {
      let data = fs.read_only(access: files);
      [
        head_bytes(files: data, path: "a.txt", count: 2),
        head_bytes(files: data, path: "a.txt", count: 0),
        head_bytes(files: data, path: "a.txt", count: 99),
      ]
    },
  ).value
    == [
      Ok(value: bytes.from_text(text: "12")),
      Ok(value: bytes.empty()),
      Ok(value: bytes.from_text(text: "12345")),
    ]
}

pub fn load(
  files: fs.ReadOnlyFs,
  path: String,
) -> Result<String, fs.FsError> uses io {
  using file = fs.open_read(access: files, path: path) {
    fs.read_all(file: file)
  }
}

example load_keeps_the_error_of_the_body_when_the_close_fails_too for load {
  run(
    setup: [
      sim.file_bytes(path: "raw.bin", content: bytes.new(count: 2, fill: 255)),
      sim.close_fails(path: "raw.bin"),
      sim.file(path: "ok.txt", content: "fine"),
      sim.close_fails(path: "ok.txt"),
    ],
    body: fn(files: fs.Fs) -> List<Result<String, fs.FsError>> uses io {
      let data = fs.read_only(access: files);
      [load(files: data, path: "raw.bin"), load(files: data, path: "ok.txt")]
    },
  ).value
    == [
      Err(error: fs.FsError.Failed(path: "raw.bin", detail: "malformed-text")),
      Err(error: fs.FsError.CloseFailed(path: "ok.txt")),
    ]
}

example load_reads_the_whole_text for load {
  run(
    setup: [
      sim.file(path: "a.txt", content: "héllo\n"),
      sim.file(path: "empty.txt", content: ""),
      sim.file_bytes(path: "raw.bin", content: bytes.new(count: 2, fill: 255)),
    ],
    body: fn(files: fs.Fs) -> List<Result<String, fs.FsError>> uses io {
      let data = fs.read_only(access: files);
      [
        load(files: data, path: "a.txt"),
        load(files: data, path: "empty.txt"),
        load(files: data, path: "raw.bin"),
      ]
    },
  ).value
    == [
      Ok(value: "héllo\n"),
      Ok(value: ""),
      Err(error: fs.FsError.Failed(path: "raw.bin", detail: "malformed-text")),
    ]
}

pub fn load_bytes(
  files: fs.ReadOnlyFs,
  path: String,
) -> Result<Bytes, fs.FsError> uses io {
  using file = fs.open_read(access: files, path: path) {
    fs.read_bytes(file: file)
  }
}

example load_bytes_reads_any_file for load_bytes {
  run(
    setup: [
      sim.file(path: "a.txt", content: "AB"),
      sim.file_bytes(path: "raw.bin", content: bytes.new(count: 2, fill: 255)),
    ],
    body: fn(files: fs.Fs) -> List<Result<Bytes, fs.FsError>> uses io {
      let data = fs.read_only(access: files);
      [
        load_bytes(files: data, path: "a.txt"),
        load_bytes(files: data, path: "raw.bin"),
      ]
    },
  ).value
    == [
      Ok(value: bytes.from_text(text: "AB")),
      Ok(value: bytes.new(count: 2, fill: 255)),
    ]
}

fn run<T>(
  setup: List<String>,
  body: fn(files: fs.Fs) -> T uses io,
) -> sim.SimOutcome<T> {
  sim.with_sim(
    world: sim.quiet_world(),
    setup: setup,
    run: fn() -> T uses clock, io, random {
      body(files: fs.simulated())
    },
  )
}

pub fn save_lines(
  files: fs.Fs,
  path: String,
  lines: List<String>,
) -> Result<Unit, fs.FsError> uses io {
  using file = fs.open_write(access: files, path: path) {
    for line in lines {
      fs.write_all(file: file, text: line ++ "\n")?;
    }
    Ok(value: ())
  }
}

example save_lines_of_no_lines_empties_the_file for save_lines {
  match run(
    setup: [sim.file(path: "out.txt", content: "old")],
    body: fn(files: fs.Fs) -> Result<Unit, fs.FsError> uses io {
      save_lines(files: files, path: "out.txt", lines: [])
    },
  ) {
    outcome => sim.file_content(outcome: outcome, path: "out.txt")
      == Some(value: ""),
  }
}

example save_lines_reports_a_failing_close for save_lines {
  run(
    setup: [sim.close_fails(path: "out.txt")],
    body: fn(files: fs.Fs) -> Result<Unit, fs.FsError> uses io {
      save_lines(files: files, path: "out.txt", lines: ["a"])
    },
  ).value
    == Err(error: fs.FsError.CloseFailed(path: "out.txt"))
}

example save_lines_writes_each_line for save_lines {
  match run(
    setup: [sim.file(path: "out.txt", content: "old content that is longer")],
    body: fn(files: fs.Fs) -> Result<Unit, fs.FsError> uses io {
      save_lines(files: files, path: "out.txt", lines: ["a", "b"])
    },
  ) {
    outcome => outcome.value == Ok(value: ())
      and sim.file_content(outcome: outcome, path: "out.txt")
        == Some(value: "a\nb\n")
      and sim.open_handles(outcome: outcome) == 0,
  }
}

pub fn store_bytes(
  files: fs.Fs,
  path: String,
  content: Bytes,
) -> Result<Unit, fs.FsError> uses io {
  using file = fs.open_write(access: files, path: path) {
    fs.write_bytes(file: file, bytes: content)
  }
}

example store_bytes_writes_them_as_they_are for store_bytes {
  match run(
    setup: [],
    body: fn(files: fs.Fs) -> Result<Unit, fs.FsError> uses io {
      store_bytes(
        files: files,
        path: "out.bin",
        content: bytes.new(count: 3, fill: 255),
      )
    },
  ) {
    outcome => outcome.value == Ok(value: ())
      and sim.file_content_bytes(outcome: outcome, path: "out.bin")
        == Some(value: bytes.new(count: 3, fill: 255)),
  }
}
```

### Staying within the root

Examples that read through a capability narrowed with `narrow`, check that a path pointing outside the root becomes `Denied`, and check the rules for narrowing. `explain` and `split_lines` are examples of `describe` and `lines_of`.

```cernith
module guide.confinement;

import std.bytes;
import std.fs;
import std.sim;

pub fn explain(error: fs.FsError) -> String {
  fs.describe(error: error)
}

example explain_names_each_failure for explain {
  explain(error: fs.FsError.NotFound(path: "a.txt")) == "a.txt: not found"
    and explain(error: fs.FsError.Denied(path: "../a")) == "../a: denied"
    and explain(error: fs.FsError.CloseFailed(path: "b")) == "b: closing failed"
    and explain(error: fs.FsError.Failed(path: "c", detail: "malformed-text"))
      == "c: malformed-text"
}

pub fn narrowed_read_only_root(files: fs.ReadOnlyFs, folder: String) -> String {
  match fs.narrow_read_only(access: files, folder: folder) {
    Ok(value: inner) => fs.read_only_root_of(access: inner),
    Err(error: error) => fs.describe(error: error),
  }
}

example narrowed_read_only_root_follows_the_same_rules for narrowed_read_only_root {
  narrowed_read_only_root(files: fs.simulated_read_only(), folder: "a/b")
    == "a/b"
    and narrowed_read_only_root(files: fs.simulated_read_only(), folder: "..")
      == "..: denied"
}

pub fn narrowed_root(files: fs.Fs, folder: String) -> String {
  match fs.narrow(access: files, folder: folder) {
    Ok(value: inner) => fs.root_of(access: inner),
    Err(error: error) => fs.describe(error: error),
  }
}

example narrowed_root_refuses_the_way_out for narrowed_root {
  narrowed_root(files: fs.simulated(), folder: "..") == "..: denied"
    and narrowed_root(files: fs.simulated(), folder: "../x") == "../x: denied"
    and narrowed_root(files: fs.simulated(), folder: "a/../../x")
      == "a/../../x: denied"
    and narrowed_root(files: fs.simulated(), folder: "/etc") == "/etc: denied"
}

example narrowed_root_stays_below_the_root for narrowed_root {
  narrowed_root(files: fs.simulated(), folder: "reports") == "reports"
    and narrowed_root(files: fs.simulated(), folder: "reports/./2024//")
      == "reports/2024"
    and narrowed_root(files: fs.simulated(), folder: "a/../b") == "b"
    and narrowed_root(files: fs.simulated(), folder: ".") == "."
    and narrowed_root(files: fs.simulated(), folder: "") == "."
}

pub fn read_in(
  files: fs.Fs,
  folder: String,
  path: String,
) -> Result<String, fs.FsError> uses io {
  let inner = fs.narrow(access: files, folder: folder)?;
  fs.read_text(access: fs.read_only(access: inner), path: path)
}

example read_in_reports_the_full_path_of_a_failing_read for read_in {
  run(
    setup: [
      sim.file_bytes(
        path: "reports/raw.bin",
        content: bytes.new(count: 2, fill: 255),
      ),
    ],
    body: fn(files: fs.Fs) -> Result<String, fs.FsError> uses io {
      read_in(files: files, folder: "reports", path: "raw.bin")
    },
  ).value
    == Err(
      error: fs.FsError.Failed(
        path: "reports/raw.bin",
        detail: "malformed-text",
      ),
    )
}

example read_in_sees_only_the_narrowed_folder for read_in {
  run(
    setup: [
      sim.file(path: "reports/2024/a.txt", content: "A"),
      sim.file(path: "secret.txt", content: "S"),
    ],
    body: fn(files: fs.Fs) -> List<Result<String, fs.FsError>> uses io {
      [
        read_in(files: files, folder: "reports", path: "2024/a.txt"),
        read_in(files: files, folder: "reports/2024", path: "a.txt"),
        read_in(files: files, folder: "reports", path: "../secret.txt"),
        read_in(files: files, folder: "reports", path: "/secret.txt"),
        read_in(files: files, folder: "reports", path: "secret.txt"),
        read_in(files: files, folder: "..", path: "secret.txt"),
      ]
    },
  ).value
    == [
      Ok(value: "A"),
      Ok(value: "A"),
      Err(error: fs.FsError.Denied(path: "../secret.txt")),
      Err(error: fs.FsError.Denied(path: "/secret.txt")),
      Err(error: fs.FsError.NotFound(path: "secret.txt")),
      Err(error: fs.FsError.Denied(path: "..")),
    ]
}

fn run<T>(
  setup: List<String>,
  body: fn(files: fs.Fs) -> T uses io,
) -> sim.SimOutcome<T> {
  sim.with_sim(
    world: sim.quiet_world(),
    setup: setup,
    run: fn() -> T uses clock, io, random {
      body(files: fs.simulated())
    },
  )
}

pub fn split_lines(text: String) -> List<String> {
  fs.lines_of(text: text)
}

example split_lines_splits_at_line_feeds for split_lines {
  split_lines(text: "a\nb\n") == ["a", "b"]
    and split_lines(text: "a\r\nb") == ["a", "b"]
    and split_lines(text: "a\n\nb") == ["a", "", "b"]
    and split_lines(text: "\n") == [""]
    and split_lines(text: "") == []
}
```

### Appending, replacing, renaming, and looking at a path

Examples that add to a log, replace a state file, move a file to an archive folder, and ask what a path is.

```cernith
module guide.fs_edits;

import std.bytes;
import std.fs;
import std.int;
import std.sim;

pub fn append_pair(
  files: fs.Fs,
  path: String,
) -> Result<Unit, fs.FsError> uses io {
  fs.append_bytes(
    access: files,
    path: path,
    bytes: bytes.new(count: 2, fill: 2),
  )
}

example append_bytes_adds_to_the_end for append_pair {
  match run(
    setup: [
      sim.file_bytes(path: "a.bin", content: bytes.new(count: 1, fill: 1)),
    ],
    body: fn(files: fs.Fs) -> Result<Unit, fs.FsError> uses io {
      append_pair(files: files, path: "a.bin")
    },
  ) {
    outcome => option_hex(
      found: sim.file_content_bytes(outcome: outcome, path: "a.bin"),
    )
      == "010202",
  }
}

pub fn archive(files: fs.Fs, name: String) -> Result<Unit, fs.FsError> uses io {
  fs.rename(access: files, from: name, to: "old/" ++ name)
}

example archive_moves_the_file for archive {
  match run(
    setup: [
      sim.file(path: "a.txt", content: "A"),
      sim.file(path: "old/a.txt", content: "older"),
    ],
    body: fn(files: fs.Fs) -> Result<Unit, fs.FsError> uses io {
      archive(files: files, name: "a.txt")
    },
  ) {
    outcome => outcome.value == Ok(value: ())
      and sim.file_paths(outcome: outcome) == ["old/a.txt"]
      and sim.file_content(outcome: outcome, path: "old/a.txt")
        == Some(value: "A"),
  }
}

example archive_reports_a_missing_file for archive {
  run(
    setup: [],
    body: fn(files: fs.Fs) -> Result<Unit, fs.FsError> uses io {
      archive(files: files, name: "nope.txt")
    },
  ).value
    == Err(error: fs.FsError.NotFound(path: "nope.txt"))
}

pub fn describe_path(files: fs.ReadOnlyFs, path: String) -> String uses io {
  match fs.metadata(access: files, path: path) {
    Ok(value: entry) => match entry.kind {
      fs.EntryKind.File => "file of "
        ++ int.to_text(value: entry.size)
        ++ " bytes",
      fs.EntryKind.Folder => "folder",
      fs.EntryKind.Link => "link",
      fs.EntryKind.Missing => "missing",
    },
    Err(error: error) => fs.describe(error: error),
  }
}

example describe_path_tells_what_a_path_is for describe_path {
  run(
    setup: [sim.file(path: "d/a.txt", content: "abc")],
    body: fn(files: fs.Fs) -> List<String> uses io {
      let data = fs.read_only(access: files);
      [
        describe_path(files: data, path: "d/a.txt"),
        describe_path(files: data, path: "d"),
        describe_path(files: data, path: "nope"),
        describe_path(files: data, path: "../d"),
      ]
    },
  ).value
    == ["file of 3 bytes", "folder", "missing", "../d: denied"]
}

pub fn journal(files: fs.Fs, line: String) -> Result<Unit, fs.FsError> uses io {
  fs.append_text(access: files, path: "journal.txt", text: line ++ "\n")
}

example journal_makes_the_file_and_adds_to_it for journal {
  match run(
    setup: [],
    body: fn(files: fs.Fs) -> Result<Unit, fs.FsError> uses io {
      journal(files: files, line: "start")?;
      journal(files: files, line: "stop")
    },
  ) {
    outcome => outcome.value == Ok(value: ())
      and sim.file_content(outcome: outcome, path: "journal.txt")
        == Some(value: "start\nstop\n"),
  }
}

fn option_hex(found: Option<Bytes>) -> String {
  match found {
    Some(value: data) => bytes.to_hex(bytes: data),
    None => "",
  }
}

pub fn prepare(files: fs.Fs, path: String) -> Result<Bool, fs.FsError> uses io {
  fs.make_folder(access: files, path: path)?;
  Ok(value: fs.is_folder(access: fs.read_only(access: files), path: path))
}

example prepare_makes_the_folder for prepare {
  run(
    setup: [sim.file(path: "x/y.txt", content: "")],
    body: fn(files: fs.Fs) -> List<Result<Bool, fs.FsError>> uses io {
      [prepare(files: files, path: "x"), prepare(files: files, path: "../x")]
    },
  ).value
    == [Ok(value: true), Err(error: fs.FsError.Denied(path: "../x"))]
}

fn run<T>(
  setup: List<String>,
  body: fn(files: fs.Fs) -> T uses io,
) -> sim.SimOutcome<T> {
  sim.with_sim(
    world: sim.quiet_world(),
    setup: setup,
    run: fn() -> T uses clock, io, random {
      body(files: fs.simulated())
    },
  )
}

pub fn save(
  files: fs.Fs,
  path: String,
  text: String,
) -> Result<Unit, fs.FsError> uses io {
  fs.replace_text(access: files, path: path, text: text)
}

example save_replaces_the_whole_content for save {
  match run(
    setup: [sim.file(path: "state.txt", content: "old and longer")],
    body: fn(files: fs.Fs) -> List<Result<Unit, fs.FsError>> uses io {
      [
        save(files: files, path: "state.txt", text: "new"),
        save(files: files, path: "../state.txt", text: "new"),
      ]
    },
  ) {
    outcome => outcome.value
      == [Ok(value: ()), Err(error: fs.FsError.Denied(path: "../state.txt"))]
      and sim.file_content(outcome: outcome, path: "state.txt")
        == Some(value: "new"),
  }
}
```

## Related

- [path](path.md): path normalization. A path for which `normalize` is `None` becomes `Denied`
- [sim](sim.md): builds a simulated world that has files, and inspects the files after a run
- [console](console.md), [env](env.md), [http](http.md): other capabilities
- [bytes](bytes.md): the functions on `Bytes`
- [17 Resources and capabilities](../language/17-resources-and-capabilities.md): the rules for capabilities, resources, and `using`
