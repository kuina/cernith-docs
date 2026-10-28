# std.path

ファイルパスの文字列を組み立て、分解、正規化するモジュールです。`import std.path;` で読み込み、`path.join(parts: ...)` のように呼び出します。すべての関数は純粋で、ファイルシステムにはアクセスしません。

`std.path` のパスは、**`/` で区切った相対パス**を表す `String` です。専用の型はありません。`a/b/c.txt` のように書き、先頭が `/` のパス（絶対パス）や、`\` を区切りにしたパスは扱いません（`\` は普通の文字です）。ファイルを開く [fs](fs.md) の関数は、パスを `path.normalize` で整えてから使います。

## 関数一覧

| 関数 | 戻り値の型 | 意味 | 例 |
| --- | --- | --- | --- |
| `path.join(parts)` | `String` | 空でない部分を `/` でつなぐ | `["a", "", "b"]` → `"a/b"` |
| `path.normalize(path)` | `Option<String>` | `.`、`..`、重複した `/` を取り除く | `"a/./b//c/../d"` → `Some(value: "a/b/d")` |
| `path.file_name(path)` | `String` | 最後の `/` のあとの部分 | `"a/b/c.txt"` → `"c.txt"` |
| `path.extension(path)` | `String` | ファイル名の拡張子（`.` を除く）。なければ `""` | `"a/b.tar.gz"` → `"gz"` |
| `path.parent(path)` | `String` | 最後の部分を除いたパス。なければ `"."` | `"a/b/c.txt"` → `"a/b"` |

どの関数にも事前条件はなく、任意の文字列を渡せます。

## 関数の詳細

### path.normalize

```text
pub fn normalize(path: String) -> Option<String>
```

次の規則に従ってパスを正規化します。

- 空の部分（`a//b` の `//`）と `.` の部分は取り除きます。
- `..` の部分は、直前の部分を取り消します。`a/b/../c` は `a/c` です。
- 結果が空になるとき（`""`、`"."`、`"./"`、`"a/.."` など）は、`Some(value: ".")` です。
- 末尾の `/` は取り除きます。

次の場合は `None` を返します。

- 絶対パス（先頭が `/`）。
- 始点より上に出るパス。`../x` や `a/../../x` のように、取り消す部分が残っていないところに `..` が来るもの。

**ルート外を指すパスは、正規化された相対パスとして表せない**ため、`None` を返します。`Result` を返す `fs` の関数は、このパスを `FsError.Denied` として拒否します。`fs.is_present` は `false` を返します（[fs](fs.md)）。

### path.join

```text
pub fn join(parts: List<String>) -> String
```

`parts` から空文字列を除き、`/` で連結します。空のリストや空文字列だけのリストでは `""` を返します。`join` 自体は正規化しないため、`"a/"` と `"b"` をつなぐと `"a//b"` になり、`..` や絶対パスもそのまま残ります。正規化するには、結果を `normalize` に渡してください。

### path.file_name

```text
pub fn file_name(path: String) -> String
```

`/` で区切った最後の部分を返します。`/` を含まなければ、`path` 全体です。`path` が `/` で終わっていれば（最後の部分が空なので）`""` です。

### path.extension

```text
pub fn extension(path: String) -> String
```

`file_name(path)` の最後の `.` よりあとの部分を、`.` を除いて返します。次の場合は `""` です。

- ファイル名に `.` がない（`noext`）。
- ファイル名が `.` で始まる、または `.` で終わる（`.hidden`、`x.`）。

ファイル名の中に `.` が複数あっても、最後の `.` より後ろだけを拡張子とします（`b.tar.gz` なら `gz`）。ただし、ファイル名の先頭が `.` のときは、`.hidden.txt` のように途中にも `.` があっても、拡張子は `""` です。

### path.parent

```text
pub fn parent(path: String) -> String
```

`/` で区切った最後の部分を除いたパスを返します。`/` を含まないパス（と空文字列）では、`"."` です。`normalize` はしないので、`"a//b"` の親は `"a/"` です。

## 例

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

## 関連

- [fs](fs.md)：パスを `normalize` してからルートの下のファイルを開く
- [text](text.md)：文字列の関数
