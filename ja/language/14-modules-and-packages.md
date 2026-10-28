# モジュールとパッケージ

## モジュール

**モジュール**は 1 つのソースファイルです。最初のコード行は `module a.b.c;` です。モジュールの名前は、パッケージのソースルート（`src/`）からの、ファイルのパスと一致しなければなりません。`/` を `.` に変え、拡張子を除きます。`src/a/b/c.cnth` は `module a.b.c;` です。一致しないと `E0218` です。名前の各部分は snake_case で、最初の部分 `std` は標準ライブラリのために予約されています（`std` 以下に自分のモジュールを作ると `E0219`）。

モジュールの中身は、`import` 行、宣言（型・定数・関数）、関数に付く `example` と `property` の順です。それ以外は書けません。

## import

```text
import std.list;                  // list.take(items: xs, count: 2) のように使う
import geometry.shapes as shapes; // shapes.Rect、shapes.area(...) のように使う
```

- `import` にはモジュールの完全な名前を書きます。最後の部分、または `as` の後ろに指定した名前が**別名**になります。ほかのモジュールの名前を使うときは、必ずこの別名を付けます（`list.take(...)`、`shapes.Rect`、`shapes.Length(value: 2)`）。
- ワイルドカードの `import`、別名なしで名前を取り込む `import`、再エクスポートはありません。
- `import` は 1 行に 1 つで、正準形ではパスの順に並べます。
- 同じ別名を 2 回使うと `E0207` です。別名を、パラメータやローカル変数の名前にはできません（`E0202`）。
- 使われない `import` は `E0209` です。型の中だけで使うモジュール（`i: shapes.Length`）も、使っていることになります。
- 存在しないモジュールの `import` は `E0206` です。
- **`import` の循環は禁止です**（`E0210`）。その結果、異なるモジュールの関数どうしの再帰は書けず、停止性の検査（[13 停止性](13-termination.md)）は、モジュールごとに行えます。

```cernith-error E0207
module guide.imports;

import std.int;
import std.int;

pub fn magnitude(value: Int) -> Int {
  int.abs(value: value)
}
```

## 可視性

宣言は、`pub` を付けるとほかのモジュールに公開され、付けなければ定義元のモジュール内でのみ使えます。可視性はこの 2 段階で、`pub(...)` のような指定はありません。

| 宣言 | `pub` のとき | `pub` でないとき |
| --- | --- | --- |
| 関数、`spec fn` | ほかのモジュールから呼べる。シグネチャと契約が仕様になる | モジュールの中だけ |
| 定数 | 読める | モジュールの中だけ |
| レコード | フィールドが読め、`R(...)` と `R.with(...)` が使える（型は透明） | モジュールの中だけ |
| バリアント | すべてのケースを使い、`match` できる | モジュールの中だけ |
| 制約付き型 | 外部からも `T(value: n)` と `T.check(value: n)` が使える。生成時には条件が適用される。値は土台の型として使える | モジュールの中だけ |
| `distinct` 型 | **抽象**。型の名前は使えるが、`T(value: ...)` も `.value` も、外では使えない（`E0215`） | モジュールの中だけ |
| インターフェース | 名前をほかのモジュールで境界として使え、メソッドをそこから呼べる（[22 インターフェース](22-interfaces.md)） | モジュールの中だけ |
| `impl` | `pub` は付けられない。そのインターフェースと型が見える場所ならどこでも見える | |
| `example`、`property` | 公開されない | |

`pub` でないものを他のモジュールから使うと、`E0208`（not-exported）です。

### 抽象型

`distinct` 型の内部表現は、定義元のモジュールに隠されます。値を直接生成したり、中身を読み出したりできるのは、そのモジュールだけです。ほかのモジュールでは、公開された関数を通じて操作します。レコードの内部表現を隠したい場合は、非公開のレコードを土台にした `distinct` 型を公開します。

```text
type AccountData = record ...;           // 非公開
pub type Account = distinct AccountData; // 外からは抽象
pub fn open_account(owner: String) -> Account { ... }
```

`pub` の `distinct` 型の土台の型は、非公開でも構いません。

### 公開された仕様に非公開の名前を入れない

`pub` の関数の引数と戻り値、`pub` のレコードやバリアントのフィールド、`pub` の定数には、公開された型を使う必要があります。`pub` の関数の契約から呼べる `spec fn` と、参照できる定数も、`pub` のものに限られます。違反すると `E0216`（private-in-public）になります。呼び出し側が、満たすべき条件の内容をすべて確認できるようにするためです。

```cernith-error E0216
module guide.visibility;

record Inner {
  value: Int,
}

pub fn make() -> Inner {
  Inner(value: 1)
}
```

### テストの置き場所

`example` と `property` は、**同じモジュールにある関数**に付けます（`example e for f` の `f` は、そのモジュールで宣言された関数）。別のモジュールの関数のテストは、呼ぶ側のモジュールに、普通の関数として書きます。

## 定数

定数は純粋で、停止します。プログラムの開始前に、依存の順に一度だけ評価されます。定数が自分自身に（ほかの定数や関数を通して）依存する循環は `E0211` です。独立な定数の順序は、モジュールのパス、名前の順と決まっていて、ファイルシステムの順序に依存しません。

## パッケージ

**パッケージ**は、ルートに `cernith.toml` を持ち、ソースを `src/` の下に置いたディレクトリです。プログラムは、1 つのパッケージ、標準ライブラリ `std`、パスで指定した依存パッケージからできます。パッケージのレジストリやバージョンはありません。

```toml
name = "inventory"
entry = "app.main"      # main を持つモジュール（ライブラリなら省略）

[dependencies]
geometry = { path = "../geometry" }
```

- `name` は必須で、snake_case の名前です（`std` は不可）。
- `entry` は、`cernith run` が実行する `main` を持つモジュールです。
- 未知のキーを指定すると `E0224` になります。
- `targets` は、パッケージが対応するバックエンド（`python`、`javascript`、`wasm`、`native`）のリストです。省略すると、すべてに対応します。ホストやブラウザー向けの Wasm のビルドは、どれも `wasm` と数えます。`cernith build` は、リストにないバックエンドを拒否し（`E0229`）、`cernith difftest` は、宣言したバックエンドだけで実行します。パッケージが対応できるのは、すべての依存が対応しているバックエンドだけです（`E0228`）。
- `minimum-level` と `[budget]` もキーとして認識されますが、指定した制限はまだ適用されません。

パッケージのディレクトリを、そのままコマンドに渡せます（`cernith check app/`、`cernith test app/`、`cernith run app/`）。

### 依存

`[dependencies]` のキーは、依存先の `cernith.toml` の `name` と同じでなければなりません（`E0225`）。**依存先のモジュール名は、すべてその名前で始まります。** `../geometry/src/geometry/shapes.cnth` は `module geometry.shapes;` です。使う側は、普通のモジュールと同じように `import geometry.shapes as shapes;` と書きます。可視性の規則は、上の通りです。

`cernith.lock` には、間接的な依存も含めた全依存パッケージの、マニフェストと `.cnth` ファイルの SHA-256 ハッシュが記録されます。生成には `cernith lock` を使います。ファイルがない場合、依存の内容とハッシュが一致しない場合、依存が記録されていない場合は `E0226` になります。変更内容を確認してから `cernith lock` を実行してください。

次は、`geometry` を使う `app` パッケージの例です。

```text
geometry/
  cernith.toml          name = "geometry"
  src/geometry/shapes.cnth
app/
  cernith.toml          name = "app" / entry = "app.main" / [dependencies] geometry = { path = "../geometry" }
  src/app/main.cnth
```

```text
module geometry.shapes;

pub type Length = Int where self >= 0;

pub record Rect {
  width: Length,
  height: Length,
}

pub type Side = distinct Int;

pub fn area(rect: Rect) -> Int {
  rect.width * rect.height
}

pub fn make_side(value: Int) -> Side {
  requires value >= 0;

  Side(value: value)
}

pub fn side_length(side: Side) -> Int {
  side.value
}
```

```text
module app.main;

import geometry.shapes as shapes;

pub fn main(args: List<String>) -> Int {
  let rect = shapes.Rect(
    width: shapes.Length(value: 2),
    height: shapes.Length(value: 5),
  );
  shapes.area(rect: rect) + shapes.side_length(side: shapes.make_side(value: 4))
}
```

`shapes.Rect(...)` と `shapes.Length(value: 2)` は、`Rect` が透明な `pub` のレコード、`Length` が開いた制約付き型なので、外から作れます。`shapes.Side(value: 4)` は、`Side` が抽象型なので `E0215` です。`geometry` の `pub` でない関数（たとえば `shapes.cnth` に加えた非公開の `fn hidden()`）を呼ぶと `E0208` です。

## インターフェースとフィンガープリント

モジュールの**インターフェース**は、公開された宣言から本体を除いたものです。（これは、境界である `interface` 宣言とは別のものです。[22 インターフェース](22-interfaces.md) を参照してください。）関数はシグネチャと契約（`pub spec fn` は本体も）、レコード・バリアント・制約付き型は宣言全体、`distinct` 型は存在だけ、定数は名前と型です。モジュールの**フィンガープリント**は、そのインターフェースと、`import` しているモジュールのフィンガープリントのハッシュです。

- 関数の本体、非公開の宣言、`example`、`distinct` 型の中身を変えても、フィンガープリントは変わりません。
- 公開されたシグネチャ、契約、レコードやバリアント、制約付き型の条件を変えると、そのモジュールと、それを（推移的に）`import` するすべてのモジュールのフィンガープリントが変わります。

処理系は、ソースと `import` 先のフィンガープリントが変わらなかったモジュールの検査結果を再利用できます。結果は、完全な検査と必ず一致します。

## 診断コード

| コード | 名前 | 意味 |
| --- | --- | --- |
| `E0206` | unknown-module | 存在しないモジュールの `import` |
| `E0207` | duplicate-definition | 名前の二重定義、`import` の重複 |
| `E0208` | not-exported | 他のモジュールの非公開の名前を使った |
| `E0209` | unused-import | 使われない `import` |
| `E0210` | import-cycle | `import` の循環 |
| `E0211` | constant-cycle | 定数が自分自身に依存する |
| `E0215` | abstract-type | `distinct` 型の構築・`.value` を定義モジュールの外で使った |
| `E0216` | private-in-public | 公開されたシグネチャ・フィールド・契約に非公開の名前がある |
| `E0218` | module-path-mismatch | モジュール名とファイルのパスが一致しない |
| `E0219` | reserved-module-name | `std` 以下のモジュール名 |
| `E0224` | invalid-manifest | `cernith.toml` が使えない |
| `E0225` | dependency-not-found | 依存が見つからない、名前が違う、2 か所にある |
| `E0226` | lock-out-of-date | `cernith.lock` がない、または依存と合わない |
| `E0227` | dependency-cycle | パッケージが循環して依存している |
| `E0228` | dependency-lacks-target | パッケージが対応するバックエンドに、依存先が対応していない |
| `E0229` | target-not-declared | パッケージが `targets` で宣言していないバックエンドでビルドしようとした |

## 関連

- `distinct` 型と制約付き型：[08 制約付き型と distinct 型](08-refined-and-distinct-types.md)
- 他の言語で実装された関数：[15 foreign 関数](15-foreign-functions.md)
