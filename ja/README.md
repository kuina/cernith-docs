# Cernith 言語リファレンス

Cernith（セルニス）は、くいなちゃんが開発したプログラミング言語です。型・効果・契約（事前条件と事後条件）を静的に検査します。実行前にチェッカーが名前・型・効果・網羅性・停止性を確認し、契約の証明を試みます。ソースファイルの拡張子は `.cnth` です。

このドキュメントでは、Cernith 2.0 の言語仕様と、標準ライブラリ `std` のコアモジュールを日本語で解説します。

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

- **すべての引数にラベルを付けます。** `clamp(value: 12, low: 0, high: 9)` のように、呼び出し箇所から各引数の役割が分かります。ラベルを省略するとエラーになります。
- **変換や省略は明示します。** `Int` と `Float64` は自動変換されません。`null` はなく、`match` はすべてのケースを網羅する必要があります。関数の戻り値を暗黙に捨てることもできません。
- **失敗は値で表します。** 例外はありません。起こりうる失敗は `Result` や `Option` で表し、`?` で呼び出し元に伝えられます。契約違反や範囲外の添字など、プログラム自体の誤りは「欠陥」と呼び、その場でプログラムが停止します。
- **契約と制約付き型で条件を記述できます。** `requires` / `ensures` や、`type Percent = Int where self >= 0 and self <= 100;` のような制約付き型で、関数や値に求める条件を表します。チェッカーはソルバーによる証明を試み、証明できなかった条件は実行時に検査します。
- **副作用は関数の型に宣言します。** `uses io, clock` のように効果を列挙し、効果を宣言しない関数は純粋として扱います。
- **停止性を検査します。** `while` ループには減少量（`decreases`）、再帰には構造再帰または減少量が必要です。停止しない可能性のある関数は、`uses diverge` を宣言します。
- **テストが言語に組み込まれています。** 対象の関数の直後に、具体的なケースを検査する `example` や、すべての入力で成り立つはずの法則を表す `property` を書きます。これらはテストランナーが実行します。
- **書式は統一されています。** 空白や宣言の順序まで含めて、一意の正準形が定められています。

## 目次

### 言語仕様（[language/](language/)）

| ページ | 内容 |
| --- | --- |
| [01 最初のプログラム](language/01-first-program.md) | モジュールの形と、検査・テスト・実行のしかた |
| [02 字句](language/02-lexical-structure.md) | 識別子、リテラル、コメント、キーワード、プレリュード |
| [03 型](language/03-types.md) | 組み込み型と型の基本規則 |
| [04 式](language/04-expressions.md) | 演算子、呼び出し、`if` / `match`、`?` |
| [05 文と束縛](language/05-statements.md) | `let` / `var`、名前の規則、ループ |
| [06 関数](language/06-functions.md) | 関数、関数値、ジェネリクス、型推論 |
| [07 レコードとバリアント](language/07-records-and-variants.md) | ユーザー定義型、パターン、網羅性 |
| [08 制約付き型と distinct 型](language/08-refined-and-distinct-types.md) | 値の範囲の制限と、意味の異なる値の区別 |
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
| [23 単位](language/23-units-of-measure.md) | `unit`、`Float64<m / s>` などの量の型、単位の不一致によるエラー |

### 標準ライブラリ（[stdlib/](stdlib/)）

規約とモジュールの一覧は、[stdlib/README.md](stdlib/README.md) を参照してください。

### サンプル（[../examples/](../examples/README.md)）

[examples/](../examples/README.md) には、Cernith の書き方を示す、コンパイラで検査済みのプログラムを収録しています。コメントは英語です。

## 表記

- コード例はチェッカーで検査し、テストも実行済みです。意図的にエラーになる例には、想定される診断コード（`E0304` など）を記載しています。
- `module guide.…;` で始まる例は、1 つのファイルに保存してそのまま検査できる完全なモジュールです。
- 関数の引数は `名前: 型` の形で書き、呼び出しでは `名前: 値` の形で渡します。説明では、関数を `module.name(label, label)` のように、ラベルだけを並べて表すことがあります。
- 診断コードの頭文字は `E` がエラー、`W` が警告です。一覧は [20 診断コード](language/20-diagnostics.md) にあります。
