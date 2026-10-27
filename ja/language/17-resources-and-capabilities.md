# リソースとケイパビリティ

ファイルやソケットのような外部の資源を、プログラムが安全に扱うための仕組みです。次の 4 つの原則でできています。

1. プログラムはメモリを管理しません。ポインタ、確保と解放、デストラクタはありません。
2. **リソース**（ファイル、ソケット、ウィンドウ）は、それを解放するスコープの中にだけ存在し、スコープの外に出られません。
3. **権限は値です。** 外の世界に作用する操作には、プログラムの `manifest` から受け取った**ケイパビリティ**が必要です。
4. プログラムが書いていないタイミングで、暗黙に実行されるコードはありません。

## 第二級の値

リソースの値（と、`Scope`・タスク・チャネルの端。[18 並行処理](18-concurrency.md)）は**第二級**です。第二級の名前は、**関数呼び出しの引数**として、同じ型のパラメータに渡す使い方しかできません（レコードやバリアントのコンストラクタ、演算子の引数にはできません）。ラムダは第二級の名前を取り込めません（`E1101`）。代わりに、その名前をラムダの引数として渡します。

戻り値にしたり、レコード・バリアント・リスト・`Option`・`Result` に入れたり、`var` に代入したり、スコープの外で `let` に束縛したりすることはできません（`E1101`、resource-escapes-scope）。関数のパラメータが第二級の型なら、関数の本体も同じ規則で検査されるので、呼ばれた関数がリソースを持ち出すこともできません。

## リソース型

```text
pub resource ReadFile release close_read;

fn close_read(file: ReadFile) -> Result<Unit, FsError> uses io;
```

`resource 名前 release 関数;` は、`distinct` 型に似た、整数のハンドルの上の不透明な型を宣言します。値を作る・開くのは、その型を定義したモジュールだけです。`release` に指定する関数は、同じモジュールにあり、そのリソース型のパラメータを 1 つ取り、`Result<Unit, E>` を返さなければなりません（`E1105`）。

リソースを得る関数は、`Result<リソース型, E>` を返します。リソース型は `Eq` でも `Ord` でもなく、型引数にもフィールドにもできません（`E1106`）。

## using

```text
using 名前 = 取得の式 { 本体 }
```

取得の式は `Result<R, E>`（`R` はリソース型）、本体は同じ `E` の `Result<T, E>` で、`using` 全体の型は `Result<T, E>` です。`using` は式で、入れ子にもできます。`名前` は本体の中だけで使える、第二級の不変の束縛です。

評価は次の順です。

1. 取得の式を評価します。`Err(e)` なら、それが `using` の値です。何も取得していないので、解放もしません。
2. `Ok(r)` なら、`名前` を `r` に束縛して本体を評価します。
3. 本体がどう終わっても（最後まで到達、`?` による脱出、キャンセル）、**解放関数が、ちょうど 1 回**実行されます。`return` は本体の中に書けません（`E1107`）。本体の値が `using` の値です。
4. 本体が `Ok` で、解放が `Err(e2)` を返せば、`using` の値は `Err(e2)` です。本体が `Err(e1)` で解放も失敗したときは、`using` の値は `Err(e1)` で、解放の `e2` は**抑制された**エラーとして記録され、値にはなりません。
5. 欠陥はプログラムを終わらせます。解放関数は**実行されません**（状態が信頼できないため）。

次の例は、ファイルを開いて行数を数え、ファイルを閉じる呼び出しを書かずに済ませます。

```cernith
module guide.lines;

import std.fs;

pub fn line_count(
  files: fs.ReadOnlyFs,
  path: String,
) -> Result<Int, fs.FsError> uses io {
  using file = fs.open_read(access: files, path: path) {
    let lines = fs.read_lines(file: file)?;
    Ok(value: lines.length)
  }
}
```

リソースを取得する呼び出しを、`using` の外で `let` に束縛したり、`?` や `match` で取り出したりすると、`E1103`（resource-outside-using）です。`using` の本体のエラーの型が、取得・解放のエラーの型と違うと `E1102` で、`result.map_error` で揃えます。

```cernith-error E1103
module guide.lines;

import std.fs;

pub fn bad(files: fs.ReadOnlyFs) -> Result<Int, fs.FsError> uses io {
  let file = fs.open_read(access: files, path: "a")?;
  let text = fs.read_all(file: file)?;
  Ok(value: text.length)
}
```

```cernith-error E1101
module guide.lines;

import std.fs;

pub fn bad(files: fs.ReadOnlyFs) -> Result<Int, fs.FsError> uses io {
  using file = fs.open_read(access: files, path: "a") {
    let all = [file];
    Ok(value: all.length)
  }
}
```

```cernith-error E1102
module guide.lines;

import std.fs;

pub variant Other {
  Oops,
}

pub fn bad(files: fs.ReadOnlyFs) -> Result<Int, fs.FsError> uses io {
  using file = fs.open_read(access: files, path: "a") {
    discard fs.read_all(file: file);
    Err(error: Other.Oops)
  }
}
```

```cernith-error E1107
module guide.lines;

import std.fs;

pub fn bad(files: fs.ReadOnlyFs) -> Result<Int, fs.FsError> uses io {
  using file = fs.open_read(access: files, path: "a") {
    discard fs.read_all(file: file);
    return Ok(value: 1);
  }
}
```

## ケイパビリティ

**ケイパビリティ**は、外の世界への操作の権限を表す値です。

```text
pub capability Fs {
  root: String,
};

pub capability Console;
```

- `capability 名前 { フィールド: 型, ... };`（権限の内容を持たなければ `capability 名前;`）は、不透明で、第一級の（自由に受け渡せる）型を宣言します。フィールドは**権限の内容**（grant）で、定義したモジュールだけが読めます。ケイパビリティは `Eq` ではありません。
- ケイパビリティを作れるのは、`manifest` と、定義したモジュールだけです。ほかの場所で作ると、`E1301` です。
- ケイパビリティを狭めた**派生**のケイパビリティは、ライブラリの関数が元のケイパビリティから作ります（`fs.read_only(access: files)`、`fs.narrow(access: files, folder: "reports")`）。元より広い権限を与える関数はありません。
- ケイパビリティに対する操作は、ケイパビリティをラベル付きのパラメータとして取り、**かつ `uses io` を宣言します**。ケイパビリティを持っているだけでは効果を持ちません。操作を呼ぶ関数が `uses io` を宣言していなければ `E0401` です（[11 効果](11-effects.md)）。パラメータにケイパビリティを持たない関数は、その資源に触れません。
- `clock` と `random` は、ケイパビリティを使わない効果のままです。

### 標準ライブラリのケイパビリティ

| ケイパビリティ | モジュール | `manifest` に書く権限の内容 | 権限 |
| --- | --- | --- | --- |
| `Console` | [std.console](../stdlib/console.md) | なし | 標準入出力 |
| `Fs`、`ReadOnlyFs` | [std.fs](../stdlib/fs.md) | `root`：フォルダ | 根の下のファイル（読み書き、または読み取り専用） |
| `Env` | [std.env](../stdlib/env.md) | なし | 環境変数の読み取り |
| `Http` | [std.http](../stdlib/http.md) | `hosts`：接続してよいホスト名（`"*"` はテスト用） | そのホストへの HTTP リクエスト |
| `Process` | [std.process](../stdlib/process.md) | `programs`（それぞれの名前、絶対パス、引数の規則）、`work_root`、`environment`、`max_output_bytes`、`max_seconds` | 列挙されたプログラムを起動し、その出力を読む |

`Process` は、この一覧で最大の権限です。起動するプログラムは Cernith が検査したものではないからです。その権限の内容は許可リストです。呼び出しはプログラムを名前で指定し、パスは決して渡しません。渡せる引数は、そのプログラムの規則が許すものだけです。子プロセスが見る環境変数は、権限の内容に挙げたものだけです。どの呼び出しにも、時間の上限と出力の上限があります。`cernith audit` は、権限の内容全体を出力します（[std.process](../stdlib/process.md)）。

### manifest

```text
pub manifest Caps {
  files: fs.Fs(root: "."),
  console: console.Console,
}

pub fn main(args: List<String>, caps: Caps) -> Int uses io { ... }
```

`manifest 名前 { フィールド: ケイパビリティの型(権限の内容), ... }` は、プログラムが必要とするすべての権限の宣言で、フィールドがケイパビリティの値になるレコードです。権限の内容は、ソースに書きます。プログラムには `manifest` が高々 1 つあり、`main` はそれを 2 番目のパラメータとして受け取ります。そうでないと `E1303` です。ビルドするバックエンドの世界が提供しないケイパビリティを `manifest` が挙げていると、そのバックエンドではビルドできず、`E1305` です（たとえば、ブラウザーのページのファイルは専用領域 `@opfs` にだけあり、作業ディレクトリにはなく、`Process` も提供されません）。`std.io`、`std.clock`、`std.random` の関数は、ケイパビリティを取らず、`uses io`（`clock`、`random`）を宣言すれば呼べます。`main(args)` の形（`manifest` を使わない）でも、`manifest` を使う形でも同じです。

```cernith
module guide.report;

import std.console;
import std.fs;
import std.int;
import std.sim;

pub manifest Caps {
  files: fs.Fs(root: "."),
  console: console.Console,
}

pub fn line_count(
  files: fs.ReadOnlyFs,
  path: String,
) -> Result<Int, fs.FsError> uses io {
  using file = fs.open_read(access: files, path: path) {
    let lines = fs.read_lines(file: file)?;
    Ok(value: lines.length)
  }
}

example counts_the_lines_of_a_simulated_file for line_count {
  sim.with_sim(
    world: sim.quiet_world(),
    setup: [sim.file(path: "notes.txt", content: "a\nb\nc\n")],
    run: fn() -> Result<Int, fs.FsError> uses io {
      line_count(files: fs.simulated_read_only(), path: "notes.txt")
    },
  ).value
    == Ok(value: 3)
}

pub fn main(args: List<String>, caps: Caps) -> Int uses io {
  match line_count(files: fs.read_only(access: caps.files), path: "notes.txt") {
    Ok(value: count) => {
      discard console.write_line(
        access: caps.console,
        text: int.to_text(value: count),
      );
      0
    },
    Err(error: _) => 1,
  }
}
```

処理系の `audit` コマンドは、`manifest` を、すべての foreign 関数と並べて、信頼の境界として一覧にします。

### テストでの権限

テストでは、ケイパビリティを `fs.simulated()`、`fs.simulated_read_only()`、`console.simulated()`、`process.simulated_with(...)` のような関数で作れます。これらはシミュレート世界を相手にし、`example` と `property` の本文、および `std.sim.with_sim` と `std.test.with_world` に渡す `run:` のラムダの中だけで呼べます。ほかの場所（普通の補助関数の中など）で呼ぶと `E1302` です。補助関数には、ケイパビリティを引数で渡します。`manifest` のレコードをテストで使うときは、`Caps(files: fs.simulated(), console: console.simulated())` のように、フィールドごとに組み立てます（[std.sim](../stdlib/sim.md)）。

```cernith-error E1302
module guide.report;

import std.fs;

pub fn make() -> fs.ReadOnlyFs {
  fs.simulated_read_only()
}
```

### ライブラリが持つ義務

外の資源に触れる標準ライブラリの関数は、仕様、`example`、シミュレート世界の中での対応物を持ちます。資源を得る関数は、`Result<リソース, E>` を返し、`E` は閉じたバリアントです。ファイル、コンソール、環境変数、ネットワーク、ほかのプログラム（`std.fs`、`std.console`、`std.env`、`std.http`、`std.process`、およびパスのための `std.path`）が、この仕組みの上に作られています。`Fs` と `ReadOnlyFs` は根のディレクトリを持ち、すべてのパスは、根の外に出られないように正規化されます。根から出るパスは `Denied` です。標準ライブラリが内部で使う関数（`std.sys.request`、`std.async.erase`、`std.async.reveal` など）は、ケイパビリティなしで資源に触れたり、値の型を変えたりできるので、プログラムから呼ぶと `E1306` です。ケイパビリティを取る `std.fs`、`std.env`、`std.http`、`std.console`、`std.process` などの関数を使います。

## 診断コード

| コード | 名前 | 意味 |
| --- | --- | --- |
| `E1101` | resource-escapes-scope | 第二級の値を、戻した・格納した・取り込ませた・呼び出しの引数以外で使った |
| `E1102` | release-error-not-covered | `using` の本体のエラー型が、取得・解放のエラー型と違う |
| `E1103` | resource-outside-using | リソースの取得を `using` の外で束縛した |
| `E1105` | bad-resource-declaration | `release` の関数がない、または署名が違う |
| `E1106` | resource-as-type-argument | リソース型を型引数やフィールドにした |
| `E1107` | return-inside-using | `using` の本体の中で `return` した |
| `E1301` | capability-outside-manifest | ケイパビリティを、`manifest` と定義モジュールの外で作った |
| `E1302` | simulated-outside-test | テスト用のケイパビリティをテストの外で作った |
| `E1303` | bad-manifest | `manifest` が 2 つある、フィールドがケイパビリティでない、`main` が受け取らない |
| `E1305` | capability-unavailable-on-target | `manifest` が挙げたケイパビリティを、ビルドするバックエンドの世界が提供しない |
| `E1306` | internal-function | 標準ライブラリの内部用の関数を、プログラムが呼んだ |

## 関連

- 効果：[11 効果](11-effects.md)
- 並行処理（タスクとリソース）：[18 並行処理](18-concurrency.md)
- 標準ライブラリ：[std.fs](../stdlib/fs.md)、[std.console](../stdlib/console.md)、[std.http](../stdlib/http.md)、[std.process](../stdlib/process.md)、[std.sim](../stdlib/sim.md)
