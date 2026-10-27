# 最初のプログラム

Cernith のソースファイルは、1 つのモジュールを書いたもので、拡張子は `.cnth` です。ファイルは、`module` 行、`import` 行、定義（型・定数・関数・テスト）の順に並びます。

## 実行できる最小のプログラム

```cernith
module guide.hello;

import std.io;

pub fn main(args: List<String>) -> Int uses io {
  match io.write_line(text: "hello") {
    Ok(value: _) => 0,
    Err(error: _) => 1,
  }
}
```

- `module guide.hello;` は、このファイルがモジュール `guide.hello` であることを宣言します。
- `import std.io;` は、標準ライブラリのモジュール `std.io` を使うための宣言です。以後、`io.` を前置して `io.write_line(...)` のように呼びます（[14 モジュールとパッケージ](14-modules-and-packages.md)）。
- `pub fn main(args: List<String>) -> Int uses io { ... }` は、プログラムの入口です。コマンドライン引数の `List<String>` を受け取り、`Int` を返します。この `Int` がプログラムの終了ステータスです。`uses io` は、この関数が入出力を行うことの宣言です（[11 効果](11-effects.md)）。
- `io.write_line(text: "hello")` は、1 行を標準出力に書きます。**引数には必ずラベル（`text:`）を付けます。**
- `io.write_line` は成功か失敗かを `Result` で返します。`match` の各ケースは `パターン => 式,` の形で、最後の行にもカンマを付けます。`Ok(value: _)` の `_` は「値を使わない」という意味です。

## 契約とテストのあるプログラム

```cernith
module guide.first;

pub fn clamp(value: Int, low: Int, high: Int) -> Int {
  requires low <= high;
  ensures result >= low and result <= high;

  if value < low {
    low
  } else if value > high {
    high
  } else {
    value
  }
}

example clamps_high for clamp {
  clamp(value: 12, low: 0, high: 9) == 9
}

example keeps_inside for clamp {
  clamp(value: 5, low: 0, high: 9) == 5
}

property result_in_range for clamp(value: Int, low: Int, high: Int) {
  requires low <= high;

  let r = clamp(value: value, low: low, high: high);
  r >= low and r <= high
}
```

関数本体の構造は、次の順です。

1. **節**（`requires`、`ensures`、`decreases`）。それぞれ `;` で終わります。`requires` は呼び出し側が満たすべき条件、`ensures` は関数が保証する条件で、`result` は戻り値を指します（[09 契約](09-contracts.md)）。
2. 空行を 1 つ。
3. 文（`let`、`var`、代入、`for` など）。
4. 最後の式。**末尾に `;` を付けない最後の式が、関数の値になります。** `return` も使えますが、本体の最後の文にはできません（[05 文と束縛](05-statements.md)）。

関数の後ろに `example` と `property` を書きます。`example 名前 for 関数 { Bool 式 }` は具体的な入力についての事実を、`property 名前 for 関数(引数) { ... }` はすべての入力についての法則を述べます。処理系は、property の入力を自動で生成して試し、成り立たない入力が見つかれば、最小の反例を報告します（[16 テスト](16-testing.md)）。

## 処理系の使い方

処理系のコマンドは `cernith` です。ここでは、このドキュメントで使う最小限だけを示します。

| コマンド | することと終了コード |
| --- | --- |
| `cernith check ファイル` | 構文・名前・型・効果・網羅性・停止性などを検査する。エラーがなければ 0 |
| `cernith check --verify ファイル` | 上に加えて、契約と制約の証明をソルバーで試みる |
| `cernith test ファイル` | `example` と `property` を実行する。すべて通れば 0 |
| `cernith run ファイル` | `main` を実行する |
| `cernith fmt ファイル` | ファイルを正準形（[19 正準形](19-canonical-form.md)）に書き換える |
| `cernith api [モジュール] [--root フォルダ]` | フォルダ（既定は現在のフォルダ）にあるモジュールの公開インターフェースを、本体を除いて出力する。公開された定義ごとに、直前のコメント行、先頭行、`requires` と `ensures`、その定義を名指しする最初の example を示す。`--format json` で JSON になる。何も出力しなければ 2 で終了する |
| `cernith eval ファイル '式'` | ファイルのモジュールのスコープで式を評価し、型と値を JSON で出力する。値が得られれば 0、式が検査を通らなければ 1、欠陥なら 70、ステップ予算を使い切ると 71 で終了する |

診断は、既定では JSON で出力されます。`--format compact` を付けると、1 件につき 1 行になります。

```text
t.cnth:4:3 E0307 unqualified-integer-division: Integer division must name its rounding rule (int.div_floor, int.div_trunc, int.div_euclid). [fix (needs-review): ...]
```

形式は `ファイル:行:列 コード 名前: メッセージ` で、コードの意味は [20 診断コード](20-diagnostics.md) にまとめています。

`cernith test` の出力は、次の形です。

```text
passed   example  guide.first.clamps_high
passed   example  guide.first.keeps_inside
passed   property guide.first.result_in_range
3 tests: 3 passed
```

`cernith eval` を使うと、`example` を書かなくても、呼び出しの結果が分かります。次の例では、上の最初のプログラムを `first.cnth` として保存したモジュールを使います。

```text
$ cernith eval first.cnth 'clamp(value: 12, low: 0, high: 9)'
{
  "ok": true,
  "schema": "cernith-eval/0.1",
  "steps": 24,
  "type": "Int",
  "value": "9"
}
```

式は純粋でなければなりません。モジュールの名前は、非公開のものも含めてすべて使えます。`--session` を付けると、`eval` は標準入力から 1 行ずつ（式、`let 名前 = 式`、`import モジュール` のいずれか）読み、1 行ごとに JSON を 1 行出力します。

そのほかのコマンドを、1 行ずつ示します。

| コマンド | 内容 |
| --- | --- |
| `cernith profile ファイル` | `main` を参照インタープリタで実行し、関数ごとの呼び出し回数、評価ステップ数、割り当て数を数える。数え方は実行のたびに同じ。`--tests` ではテストを、`--format table` では表を出力する |
| `cernith debug ファイル` | `main` の実行を 1 回記録し、標準入力から 1 行ずつ読んだ質問（`summary`、`stack`、`locals`、`find-call`、`first-where`、`step` など）に、JSON を 1 行ずつ出力して答える |
| `cernith lsp` | エディタ向けの言語サーバー（LSP 3.17）。標準入出力で動き、診断、クイックフィックス、ホバー、定義、参照、シンボル、整形、名前変更を提供する |
| `cernith graph フォルダ` | パッケージのファイルと依存関係、およびインターフェースのフィンガープリントを、ビルドツール向けに JSON で出力する |
| `cernith outline [モジュール]`、`cernith show モジュール::名前` | パッケージの定義を、アドレスとハッシュ付きで一覧する。1 つの定義を出力する（JSON）。種別には `interface`、`impl`、`manifest` も含まれる |

## 次に読むもの

型と式の基本は [03 型](03-types.md) と [04 式](04-expressions.md)、Cernith に特有の規則（契約・効果・停止性）は [09 契約](09-contracts.md) 以降にあります。
