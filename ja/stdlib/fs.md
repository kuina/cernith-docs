# std.fs

ファイルとフォルダを、ケイパビリティを通して読み書きするモジュールです。`import std.fs;` と書き、`fs.read_text(access: ..., path: ...)` のように呼びます。どの関数も、`access:` で受け取ったケイパビリティの**ルート**（フォルダ）の下のパスにしか触れません。

使うケイパビリティは 2 つです。`Fs` はルートの下を読み書きでき、`ReadOnlyFs` は読むだけです。どちらも、プログラムの `manifest` 宣言にルートを書いて求め、`main` の第 2 引数として受け取ります。ほかの場所では作れません（エラー `E1301`）。ファイルを使う関数は、ケイパビリティを引数に取り、`uses io` も宣言します（[17 リソースとケイパビリティ](../language/17-resources-and-capabilities.md)）。

開いたファイルは**リソース**で、`using` の中でだけ存在します。`using` が終わると、成功しても失敗しても自動的に閉じられます。閉じ忘れることも、閉じたあとのファイルを使うこともできません。

## 型

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

- `Fs` と `ReadOnlyFs` は、ルートを `root` に持つケイパビリティです。`root` を直接読むことはできません（エラー `E0215`）。`==` で比べることもできません。ルートは、`root_of` と `read_only_root_of` で得られます。
- `ReadFile` と `WriteFile` は、開いたファイルを表すリソースです。`release` に書かれた関数（`close_read`、`close_write`）が、`using` の終わりに呼ばれます。
- `FsError` は、操作が失敗した理由です。

| ケース | 意味 |
| --- | --- |
| `NotFound(path: ...)` | そのパスのファイルやフォルダがない |
| `Denied(path: ...)` | ルートの外を指すパス（[ルートとパス](#ルートとパス)）、またはその操作が許されないパス |
| `CloseFailed(path: ...)` | ファイルを閉じるのに失敗した |
| `Failed(path: ..., detail: ...)` | ほかの失敗。`detail` に理由の文字列が入る（例：`read_text` で読んだファイルが UTF-8 のテキストでないときは `"malformed-text"`） |

`open_read`、`list_names`、`remove` などパスを受け取って失敗する関数では、`path` に渡したパスの文字列がそのまま入ります。ファイルを開いたあとの操作（`read_all` など）の失敗では、ルートを含む、そのファイルのパスが入ります（ルートが `"."` でなければ、渡したパスとは異なります）。

`std.fs` は、`fs.metadata` が返す型 `Entry` と `EntryKind` も定義しています。

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

> 補足：`std.fs` は、`Change` と `ChangeKind` という型も定義しています。変更の通知を表す型ですが、`std.fs` の関数はこれらを使いません。

## ルートとパス

`manifest` に書くルートは、プログラムを実行するフォルダ（作業ディレクトリ）から見たフォルダのパスです。`"."` は作業ディレクトリそのものです。

```text
pub manifest Caps {
  data: fs.ReadOnlyFs(root: "data"),
  out: fs.Fs(root: "reports"),
}
```

関数に渡す `path` は、ルートからの相対パスで、区切りは `/` です（[path](path.md)）。関数は、パスを `path.normalize` で整えてからルートにつなぎます。したがって、`dir/./b.txt` と `dir/b.txt` は同じファイルです。`normalize` が `None` を返すパスは、**ルートの外を指すもの**として、`Err(error: FsError.Denied(path: ...))` で拒否されます。

- 絶対パス（`/etc/passwd`）。
- ルートより上に出るパス（`../x`、`a/../../x`）。

`a/../b.txt` のように、途中で `..` を使ってもルートの外に出なければ有効です。ファイルシステムには触れずに拒否されるので、ルートの外に何があるかは分かりません。

ケイパビリティは、`narrow` と `read_only` で**狭める**ことができます。広げる関数はありません。ルートの外へ出る手段がないことは、ケイパビリティを受け取った関数の中でも変わりません。

## using とリソース

ファイルを開く関数（`open_read`、`open_write`）は、`Result<ReadFile, FsError>` や `Result<WriteFile, FsError>` を返します。このリソースは、`using` で受け取ります。

```text
using file = fs.open_read(access: data, path: "notes.txt") {
  fs.read_all(file: file)
}
```

- `using 名前 = 開く式 { 本体 }` は、開く式が `Ok` ならファイルを `名前` に束縛して本体を実行し、本体がどう終わっても（本体の最後、`?` による脱出、いずれの場合も）ファイルを閉じます。開く式が `Err` なら、その `Err` が `using` 全体の値で、閉じる処理はありません。
- 開く式の型は `Result<リソース, E>`、本体の型は `Result<T, E>` で、**同じ `E`** でなければなりません。`using` 全体は `Result<T, E>` です。本体が別の型のエラーを返したいときは、`using` の外で `result.map_error` で `FsError` に変換します（エラー `E1102`）。
- 本体が成功して、閉じる処理が失敗したときは、`using` の値は閉じる処理の失敗（`CloseFailed`）です。本体が失敗していたときは、本体のエラーが `using` の値になります。
- `名前` は、関数の引数としてだけ使えます。戻り値にしたり、リストやレコードに入れたり、本体の外に持ち出したりはできません（エラー `E1101`）。ファイルから読んだデータを、本体の値として返します。
- `using` の本体では `return` を使えません（エラー `E1107`）。本体の値が `using` の値です。

開く式を `using` に入れずに束縛すると、エラー `E1103` です。

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

本体のエラーの型が `FsError` と違うと、エラー `E1102` です。

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

ファイルを本体の外へ出そうとすると、エラー `E1101` です。

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

ファイルを丸ごと読み書きするだけなら、`read_text`、`write_text` などの一括の関数が、`using` を内部で書いてくれます。

## 関数一覧

ファイルを丸ごと読み書きする関数：

| 関数 | 戻り値の型 | 意味 |
| --- | --- | --- |
| `fs.read_text(access, path)` | `Result<String, FsError>` | ファイルの全体をテキストで読む |
| `fs.read_file_bytes(access, path)` | `Result<Bytes, FsError>` | ファイルの全体をバイト列で読む |
| `fs.write_text(access, path, text)` | `Result<Unit, FsError>` | ファイルの内容を、テキストで置き換える |
| `fs.write_file_bytes(access, path, bytes)` | `Result<Unit, FsError>` | ファイルの内容を、バイト列で置き換える |
| `fs.append_text(access, path, text)` | `Result<Unit, FsError>` | ファイルの末尾にテキストを追加する（ファイルがなければ作る） |
| `fs.append_bytes(access, path, bytes)` | `Result<Unit, FsError>` | ファイルの末尾にバイト列を追加する |
| `fs.replace_text(access, path, text)` | `Result<Unit, FsError>` | 途中でクラッシュしても、古い内容か新しい内容のどちらかが残り、混ざらないように、ファイルの内容を置き換える |
| `fs.replace_bytes(access, path, bytes)` | `Result<Unit, FsError>` | `replace_text` のバイト列版 |

開いたファイルを使う関数：

| 関数 | 戻り値の型 | 意味 |
| --- | --- | --- |
| `fs.open_read(access, path)` | `Result<ReadFile, FsError>` | 読むために開く（`using` と一緒に使う） |
| `fs.open_write(access, path)` | `Result<WriteFile, FsError>` | 書くために開く（`using` と一緒に使う） |
| `fs.read_all(file)` | `Result<String, FsError>` | ファイルの全体をテキストで読む |
| `fs.read_lines(file)` | `Result<List<String>, FsError>` | 全体を行のリストで読む |
| `fs.read_bytes(file)` | `Result<Bytes, FsError>` | 全体をバイト列で読む |
| `fs.read_chunk(file, count)` | `Result<Bytes, FsError>` | 次の `count` バイトまで読む |
| `fs.write_all(file, text)` | `Result<Unit, FsError>` | テキストを書く |
| `fs.write_bytes(file, bytes)` | `Result<Unit, FsError>` | バイト列を書く |
| `fs.close_read(file)`、`fs.close_write(file)` | `Result<Unit, FsError>` | ファイルを閉じる（`using` が呼ぶ） |

フォルダと存在の確認：

| 関数 | 戻り値の型 | 意味 |
| --- | --- | --- |
| `fs.list_names(access, path)` | `Result<List<String>, FsError>` | フォルダの中の名前の一覧 |
| `fs.is_present(access, path)` | `Bool` | パスにファイルかフォルダがあるか |
| `fs.is_folder(access, path)` | `Bool` | パスがフォルダか |
| `fs.metadata(access, path)` | `Result<Entry, FsError>` | パスが何であるか（種類、サイズ、最後に書き込んだ時刻） |
| `fs.make_folder(access, path)` | `Result<Unit, FsError>` | フォルダと、ないときはその上のフォルダを作る |
| `fs.rename(access, from, to)` | `Result<Unit, FsError>` | ファイルかフォルダを新しいパスへ移す |
| `fs.remove(access, path)` | `Result<Unit, FsError>` | ファイルを削除する |

ケイパビリティを作る・調べる関数：

| 関数 | 戻り値の型 | 意味 |
| --- | --- | --- |
| `fs.read_only(access)` | `ReadOnlyFs` | 同じルートで読むだけのケイパビリティ |
| `fs.narrow(access, folder)` | `Result<Fs, FsError>` | ルートの下のフォルダに狭めた `Fs` |
| `fs.narrow_read_only(access, folder)` | `Result<ReadOnlyFs, FsError>` | ルートの下のフォルダに狭めた `ReadOnlyFs` |
| `fs.root_of(access)` | `String` | `Fs` のルート |
| `fs.read_only_root_of(access)` | `String` | `ReadOnlyFs` のルート |
| `fs.simulated()`、`fs.simulated_read_only()` | `Fs`、`ReadOnlyFs` | テスト用のケイパビリティ |

そのほか：

| 関数 | 戻り値の型 | 意味 |
| --- | --- | --- |
| `fs.lines_of(text)` | `List<String>` | テキストを行に分ける |
| `fs.describe(error)` | `String` | `FsError` を文字列にする |

読む関数（`is_folder` と `metadata` を含む）は `ReadOnlyFs` を、書く関数と `append_text`、`append_bytes`、`replace_text`、`replace_bytes`、`make_folder`、`rename`、`remove`、`narrow` は `Fs` を受け取ります。`Fs` で読むには、`fs.read_only(access: files)` で変えます。読むだけの関数の引数は、`ReadOnlyFs` と書くのがよい形です。そうすると、その関数が書き込まないことが、型から分かります。

```cernith-error E0301
module guide.fs_read_only_cannot_write;

import std.fs;

pub fn save(files: fs.ReadOnlyFs) -> Result<Unit, fs.FsError> uses io {
  fs.write_text(access: files, path: "a.txt", text: "x")
}
```

`uses io` が必要なのは、ケイパビリティを受け取る関数のうち、世界に問い合わせる関数です。`read_only`、`narrow`、`narrow_read_only`、`root_of`、`read_only_root_of`、`lines_of`、`describe`、`simulated`、`simulated_read_only` は純粋です。

## 関数の詳細

### fs.read_text

```text
pub fn read_text(
  access: ReadOnlyFs,
  path: String,
) -> Result<String, FsError> uses io
```

ファイルを開き、全体を UTF-8 のテキストとして読み、閉じます。ファイルがなければ `NotFound`、パスがルートの外なら `Denied` です。ファイルが UTF-8 のテキストでなければ、`Failed(path: ..., detail: "malformed-text")` です。空のファイルは `Ok(value: "")` です。

### fs.read_file_bytes

```text
pub fn read_file_bytes(
  access: ReadOnlyFs,
  path: String,
) -> Result<Bytes, FsError> uses io
```

ファイルの全体を、バイト列として読みます。テキストのファイルを含め、どんなファイルでも読めます。

### fs.write_text

```text
pub fn write_text(
  access: Fs,
  path: String,
  text: String,
) -> Result<Unit, FsError> uses io
```

ファイルを書くために開き、`text` を書き、閉じます。ファイルがなければ作り、あれば**前の内容を捨てて**置き換えます（追記ではありません）。`text` が空なら、ファイルは空になります。パスがルートの外なら `Denied` です。

書き込み先のフォルダがないときの結果は、実行環境によって異なります。実際のファイルシステムでは、`NotFound` になります。シミュレート世界は、途中のフォルダを作って書きます。この文書の例は、フォルダのある場所にだけ書いています。

### fs.write_file_bytes

```text
pub fn write_file_bytes(
  access: Fs,
  path: String,
  bytes: Bytes,
) -> Result<Unit, FsError> uses io
```

`write_text` のバイト列版です。ファイルの内容を `bytes` で置き換えます。

### fs.append_text と fs.append_bytes

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

ファイルの末尾に `text` か `bytes` を追加します。前の内容は残り、改行は追加されません。ファイルがなければ作ります。実際のファイルシステムでは、その上のフォルダが存在している必要があります（なければ結果は `NotFound` です。シミュレート世界はフォルダを作ります）。パスがルートの外なら `Denied` です。増えていくログに使います。

### fs.replace_text と fs.replace_bytes

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

`text` か `bytes` をファイルの内容全体として書きます。途中でクラッシュしても、古い内容か新しい内容のどちらかが残り、混ざることはありません。バイト列は同じフォルダの一時ファイル（`.NAME.cernith-tmp`）に書かれ、フラッシュされてから、その一時ファイルが対象の上にリネームされます。`write_text` と `write_file_bytes` は先にファイルを空にするので、クラッシュすると空のまま、あるいは途中まで書かれたままになることがあります。対象がなければ作られます。フォルダがなければ結果は `NotFound`、パスがルートの外なら `Denied` です。

### fs.open_read と fs.open_write

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

ファイルを開きます。`using` の開く式として使います（[using とリソース](#using-とリソース)）。`open_write` は、`write_text` と同じく、ファイルがなければ作り、あれば開いた時点で空にします。

### fs.read_all、fs.read_lines、fs.read_bytes

```text
pub fn read_all(file: ReadFile) -> Result<String, FsError> uses io
pub fn read_lines(file: ReadFile) -> Result<List<String>, FsError> uses io
pub fn read_bytes(file: ReadFile) -> Result<Bytes, FsError> uses io
```

開いた `ReadFile` の内容を、テキスト、行のリスト、バイト列で読みます。`read_all` は UTF-8 のテキストでないとき `Failed(..., detail: "malformed-text")` です。`read_lines` は、`read_all` した結果を `fs.lines_of` で分けたものです。この 3 つは、それまでに何を読んでいても、常にファイルの全体を返します。読み位置を持つのは `read_chunk` だけです（`read_all` のあとの `read_chunk` は、前回の `read_chunk` の続きから読みます）。

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

ファイルの次の `count` バイトまでを読みます。呼ぶたびに、前回の続きから読みます。ファイルの終わり近くでは `count` より短く、終わりに達したあとは空のバイト列を返します。大きなファイルを、全体をメモリに置かずに少しずつ処理するときに使います。

**事前条件：** `count >= 0`。満たさない呼び出しは欠陥です。`count` が 0 なら、空のバイト列です。

### fs.write_all と fs.write_bytes

```text
pub fn write_all(file: WriteFile, text: String) -> Result<Unit, FsError> uses io
pub fn write_bytes(file: WriteFile, bytes: Bytes) -> Result<Unit, FsError> uses io
```

開いた `WriteFile` に書きます。続けて呼ぶと、書いた順につながります（`"one"` と `"two"` を書くと `"onetwo"`）。改行は足されません。

### fs.close_read と fs.close_write

```text
pub fn close_read(file: ReadFile) -> Result<Unit, FsError> uses io
pub fn close_write(file: WriteFile) -> Result<Unit, FsError> uses io
```

リソースの `release` に指定された関数で、`using` が終わるときに自動的に呼ばれます。プログラムが呼ぶ必要はありません。閉じるのに失敗すると `CloseFailed` です。

### fs.list_names

```text
pub fn list_names(
  access: ReadOnlyFs,
  path: String,
) -> Result<List<String>, FsError> uses io
```

フォルダ `path` の中にあるファイルとフォルダの**名前**（パスではなく、名前だけ）のリストを返します。コードポイント順に並びます。サブフォルダの中身は含みません。フォルダを表すパスは、ルート自身を指す `"."`（または `""`）でもかまいません。フォルダがなければ `NotFound` です。ファイルのパスを渡すと、`Failed(path: ..., detail: "failed")` です。

### fs.is_present

```text
pub fn is_present(access: ReadOnlyFs, path: String) -> Bool uses io
```

パスにファイルかフォルダがあれば `true` です。ルートの外のパスは、`Denied` の `Err` ではなく `false` です。`Result` を返さないので、失敗の理由は分かりません。

### fs.is_folder

```text
pub fn is_folder(access: ReadOnlyFs, path: String) -> Bool uses io
```

パスがフォルダなら、つまりそのパスで `list_names` が成功するなら、`true` を返します。ファイル、存在しないパス、ルートの外のパスは `false` です。

### fs.metadata

```text
pub fn metadata(
  access: ReadOnlyFs,
  path: String,
) -> Result<Entry, FsError> uses io
```

パスが何であるかを、リンクをたどらずに返します。`name` はパスの最後の部分、`kind` は `File`、`Folder`、`Link` のいずれか、`size` はファイルのバイト数（フォルダとリンクは 0）です。`modified_ms` は最後に書き込んだ時刻で、1970-01-01 UTC からのミリ秒です。シミュレート世界は時刻を持たず、0 を返します。

存在しないパスはエラーではなく、`kind: EntryKind.Missing` の `Ok` になります。ルートの外のパスは `Err(error: FsError.Denied(path: ...))` です。

### fs.make_folder

```text
pub fn make_folder(access: Fs, path: String) -> Result<Unit, FsError> uses io
```

フォルダと、ないときはその上のフォルダを、`mkdir -p` のように作ります。すでにあるフォルダはエラーではありません。実際のファイルシステムでは、パスの途中がファイルだと、結果は `Err` です（システムによって `NotFound` や `Failed` など）。シミュレート世界はファイルだけを持ちます。フォルダは、その下にファイルがある間だけ存在し、`make_folder` は何も残さずに成功します。

### fs.rename

```text
pub fn rename(
  access: Fs,
  from: String,
  to: String,
) -> Result<Unit, FsError> uses io
```

ファイルを、またはフォルダをその下にあるものごと、新しいパス `to` へ移します。2 つのパスはどちらもルートに対して検査され、外に出るパスは `Denied` です。新しいパスにあるファイルは置き換えられます（システムが可能なら 1 ステップで）。フォルダは置き換えられず、結果は `Err` です（たとえば `Denied`）。`from` がなければ結果は `NotFound` で、エラーの `path` は `from` です。

### fs.remove

```text
pub fn remove(access: Fs, path: String) -> Result<Unit, FsError> uses io
```

ファイルを削除します。ファイルがなければ `NotFound` です。フォルダを削除する関数はなく、フォルダのパスを渡したときの結果は、実行環境によって `Denied` または `NotFound` です。

### fs.read_only

```text
pub fn read_only(access: Fs) -> ReadOnlyFs
```

同じルートの、読むだけのケイパビリティを返します。元の `Fs` より多くのことはできません。

### fs.narrow と fs.narrow_read_only

```text
pub fn narrow(access: Fs, folder: String) -> Result<Fs, FsError>
pub fn narrow_read_only(
  access: ReadOnlyFs,
  folder: String,
) -> Result<ReadOnlyFs, FsError>
```

ルートの下のフォルダ `folder` をルートとするケイパビリティを返します。`folder` は `path.normalize` で整えられ、ルートの外を指す（`None` になる）なら `Err(error: FsError.Denied(path: folder))` です。`"."` や `""` は、元のルートのままです。フォルダが実際にあるかどうかは確かめません。

狭めたケイパビリティのルートは、元のルートに `folder` をつなげたものです（`"reports"` を `"2024"` に狭めると `"reports/2024"`）。狭めたあとのケイパビリティは、`folder` の外に出られません。ライブラリの関数に、プログラムのすべてのファイルでなく、必要なフォルダだけを渡すために使います。

### fs.root_of と fs.read_only_root_of

```text
pub fn root_of(access: Fs) -> String
pub fn read_only_root_of(access: ReadOnlyFs) -> String
```

ケイパビリティのルートを返します。

### fs.lines_of

```text
pub fn lines_of(text: String) -> List<String>
```

テキストを行に分けます。区切りは `\n` で、直前の `\r` は取り除きます。最後の行の終わりの改行は、新しい行を始めません。空のテキストは `[]` で、`"\n"` だけのテキストは `[""]`（空の行が 1 つ）です。

### fs.describe

```text
pub fn describe(error: FsError) -> String
```

| エラー | 文字列 |
| --- | --- |
| `NotFound(path: "a")` | `"a: not found"` |
| `Denied(path: "b")` | `"b: denied"` |
| `CloseFailed(path: "c")` | `"c: closing failed"` |
| `Failed(path: "d", detail: "x")` | `"d: x"` |

### fs.simulated と fs.simulated_read_only

```text
pub fn simulated() -> Fs
pub fn simulated_read_only() -> ReadOnlyFs
```

テストのためのケイパビリティを返します。ルートは `"."` で、シミュレート世界の作業ディレクトリです。`example` と `property` の中、および `std.test.with_world` や `std.sim.with_sim` に渡す関数の中でだけ呼べます。ほかの場所で呼ぶとエラー `E1302` です。

世界のファイルは、`std.sim.with_sim` の `setup` に `sim.file(path: ..., content: ...)` を書いて与え、実行後に `sim.file_content` などで確かめます（[sim](sim.md)）。`std.test.with_world` の世界にはファイルがありません。

## 例

### 一括の関数と使い分け

次のモジュールは、`data` フォルダの `notes.txt` の行数を数えて、`reports` フォルダに書き出すプログラムです。`manifest` で 2 つのフォルダを別々のケイパビリティにしているので、`data` に書くことも、`reports` を読むことも、型の上でできません。

テストでは、`run` が `fs.simulated()` を渡して本体を実行します。`main` には `manifest` と同じ形のレコード `Caps(...)` を渡します。

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

### 開いたファイルを使う

`open_read` と `open_write` を `using` と組み合わせて、行ごとの処理、チャンクでの読み込み、バイト列の読み書きをする例です。

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

### ルートからはみ出さない

`narrow` で狭めたケイパビリティを通して読み、ルートの外を指すパスが `Denied` になることと、狭めるときの規則を確かめる例です。`explain` と `split_lines` は、`describe` と `lines_of` の例です。

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

### 追記、置き換え、名前の変更、パスの種類の確認

ログに追記し、状態ファイルを置き換え、ファイルをアーカイブのフォルダへ移し、パスが何かを尋ねる例です。

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

## 関連

- [path](path.md)：パスの正規化。`normalize` が `None` のパスは `Denied`
- [sim](sim.md)：ファイルを持つシミュレート世界を作り、実行後のファイルを調べる
- [console](console.md)、[env](env.md)、[http](http.md)：ほかのケイパビリティ
- [bytes](bytes.md)：`Bytes` の関数
- [17 リソースとケイパビリティ](../language/17-resources-and-capabilities.md)：ケイパビリティ、リソース、`using` の規則
