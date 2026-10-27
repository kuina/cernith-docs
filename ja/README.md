# Cernith 言語リファレンス

Cernith（セルニス）は、くいなちゃんによって作られた、静的に検査されるプログラミング言語です。型・効果・契約（事前条件と事後条件）が言語の一部で、プログラムを実行する前に、チェッカーが名前・型・効果・網羅性・停止性を検査し、契約の証明を試みます。ソースファイルの拡張子は `.cnth` です。

このリポジトリは、Cernith 2.0 の言語仕様と、標準ライブラリ `std` のコアモジュールを日本語で説明します。

```cernith
module guide.overview;

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
```

## 言語の特徴

- **引数はすべてラベル付きで渡す。** `clamp(value: 12, low: 0, high: 9)` のように、呼び出し側を読むだけで各引数の意味が分かります。ラベルを省くと、チェッカーが拒否します。
- **暗黙の変換も、暗黙の省略もない。** `Int` と `Float64` は自動では変換されず、`null` はなく、`match` はすべてのケースを網羅しなければならず、値を返す関数の結果を黙って捨てることはできません。
- **失敗は値で表す。** 例外はありません。起こりうる失敗は `Result` や `Option` として型に現れ、`?` で呼び出し元へ渡します。プログラムの誤り（契約違反、範囲外の添字など）は「欠陥」と呼び、プログラムはそこで停止します。
- **契約と制約付き型を言語が持つ。** `requires` / `ensures` と、`type Percent = Int where self >= 0 and self <= 100;` のような制約付き型で、関数や値が満たすべき条件を書きます。チェッカーはソルバーで証明を試み、証明できなかった条件は実行時に検査されます。
- **副作用は関数の型に宣言する。** `uses io, clock` のように効果を宣言し、宣言のない関数は純粋です。
- **停止性を検査する。** `while` ループには減少量（`decreases`）、再帰には構造再帰または減少量が必要です。止まらない可能性のある関数は `uses diverge` と宣言します。
- **テストを言語に含む。** 関数のすぐ後に `example`（具体例）と `property`（すべての入力についての法則）を書き、処理系が実行します。
- **書式は一つだけ。** 空白や宣言の順序まで含めて、正準形が一つに決まっています。

## 目次

### 言語仕様（[language/](language/)）

| ページ | 内容 |
| --- | --- |
| [01 最初のプログラム](language/01-first-program.md) | モジュールの形と、検査・テスト・実行のしかた |
| [02 字句](language/02-lexical-structure.md) | 識別子、リテラル、コメント、キーワード、プレリュード |
| [03 型](language/03-types.md) | 組み込み型と、型の基本方針 |
| [04 式](language/04-expressions.md) | 演算子、呼び出し、`if` / `match`、`?` |
| [05 文と束縛](language/05-statements.md) | `let` / `var`、名前の規則、ループ |
| [06 関数](language/06-functions.md) | 関数、関数値、ジェネリクス、型推論 |
| [07 レコードとバリアント](language/07-records-and-variants.md) | ユーザー定義型、パターン、網羅性 |
| [08 制約付き型と distinct 型](language/08-refined-and-distinct-types.md) | 値の範囲を守る型と、意味を区別する型 |
| [09 契約](language/09-contracts.md) | `requires`、`ensures`、`where`、仕様式 |
| [10 証明義務と保証レベル](language/10-verification.md) | チェッカーが何を証明し、何を実行時に回すか |
| [11 効果](language/11-effects.md) | `uses`、効果パラメータ |
| [12 エラーと欠陥](language/12-errors-and-defects.md) | `Result`、`?`、欠陥、終了ステータス |
| [13 停止性](language/13-termination.md) | ループと再帰の停止の根拠 |
| [14 モジュールとパッケージ](language/14-modules-and-packages.md) | `import`、可視性、パッケージ |
| [15 foreign 関数](language/15-foreign-functions.md) | 他の言語で実装された関数 |
| [16 テスト](language/16-testing.md) | `example`、`property`、シミュレート世界 |
| [17 リソースとケイパビリティ](language/17-resources-and-capabilities.md) | `using`、`capability`、`manifest` |
| [18 並行処理](language/18-concurrency.md) | `parallel`、`scope`、`await`、チャネル |
| [19 正準形](language/19-canonical-form.md) | 書式の規則 |
| [20 診断コード](language/20-diagnostics.md) | チェッカーが報告するコードの一覧 |
| [21 文法](language/21-grammar.md) | 構文の定義と演算子の優先順位 |
| [22 インターフェース](language/22-interfaces.md) | `interface`、`impl`、境界、法則、`derive` |
| [23 単位](language/23-units-of-measure.md) | `unit`、`Float64<m / s>` のような量の型、単位を区別するエラー |

### 標準ライブラリ（[stdlib/](stdlib/)）

[stdlib/README.md](stdlib/README.md) に規約とモジュールの一覧があります。

### サンプル（[../examples/](../examples/README.md)）

[examples/](../examples/README.md) に、Cernith の書き方を示す小さなプログラムがあります。どれもコンパイラで検査済みです（コメントは英語のみです）。

## 表記

- コード例は、実際にチェッカーで検査し、テストを実行して確かめたものです。拒否される例には、報告される診断コード（`E0304` など）を添えます。
- `module guide.…;` で始まるコード例は、そのまま一つのファイルとして検査できる完全なモジュールです。
- 関数の引数は `名前: 型` の形で書き、呼び出しでは `名前: 値` の形で渡します。説明では、関数を `module.name(label, label)` のように、ラベルだけを並べて表すことがあります。
- 診断コードの頭文字は `E` がエラー、`W` が警告です。一覧は [20 診断コード](language/20-diagnostics.md) にあります。
