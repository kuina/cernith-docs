# 診断コード

チェッカーの診断には、**コード**（`E0304` などの 5 文字）、コードごとに一意の**名前**（`missing-argument-label` など）、メッセージ、発生箇所が含まれます。コードが `E` で始まる場合はエラー、`W` で始まる場合は警告です。エラーがあると、`cernith check` は 0 以外の終了コードで終了します。

このページでは、言語の規則に関する診断コードを、意味と関連する章へのリンクとともに一覧にしています。中間表現（`E09xx`）、編集 API（`E10xx`）、ホスト関数（`E1304`）のコードは含みません。

```text
file:line:col E0304 missing-argument-label: Arguments must be named: expected a, b. [fix (safe): Add the parameter names as labels.]
```

- 診断には、**修正**が付くことがあります。修正には `safe`（意味を変えずに適用できる）と `needs-review`（意図の確認が必要）の区別があります。
- 証明義務に由来する診断（`E05xx`）には、状態（`refuted`、`unknown`）と責任（`blame caller` など）、反例の入力が添えられます（[10 証明義務と保証レベル](10-verification.md)）。
- `cernith explain コード` は、コードの説明、直し方、例を表示します。
- 同じ原因から連鎖して起こる診断は抑えられ、原因ごとに 1 つだけ報告されます。

コードの番号は、百の位がおおよその分野を表します：`E01xx` 書式、`E02xx` 名前とモジュール（テストの `E0222`・`E0223` を含む）、`E03xx` 型、`E04xx` 効果、`E05xx` 契約と証明義務（停止性の `E0509` を含む）、`E06xx` 停止性、`E07xx` 未完成の箇所、`E11xx` リソース、`E12xx` 並行処理、`E13xx` ケイパビリティ、`E14xx` インターフェースと遅延列。警告（`W`）も同じ付け方で、`W0102`（書式）、`W0504`・`W0801`（契約と証明義務）、`W0701`（未完成の箇所）、`W1203`（並行処理）があります。

## 字句・構文・書式

| コード | 名前 | 意味 | 章 |
| --- | --- | --- | --- |
| `E0001` | syntax-error | テキストが文法として読めない（字句の誤りを含む） | [02](02-lexical-structure.md) |
| `E0101` | non-canonical-form | テキストは正しいが、正準形でない（リリースビルド） | [19](19-canonical-form.md) |
| `W0102` | non-canonical-text | テキストは正しいが、正準形でない（`check`）。正準形が添えられる | [19](19-canonical-form.md) |

## 名前の解決

| コード | 名前 | 意味 | 章 |
| --- | --- | --- | --- |
| `E0201` | unresolved-name | 名前・型・メンバー・ケースが定義されていない | [04](04-expressions.md) |
| `E0202` | name-shadows-module | パラメータや変数が、`import` したモジュールの別名と同じ名前 | [05](05-statements.md) |
| `E0203` | shadowing | 1 つの関数の中で、名前を 2 回束縛した | [05](05-statements.md) |
| `E0204` | unused-binding | 束縛した名前が一度も読まれない | [05](05-statements.md) |
| `E0205` | unknown-example-target | `example` や `property` が、そのモジュールにない関数を指している | [16](16-testing.md) |

## モジュールとパッケージ

| コード | 名前 | 意味 | 章 |
| --- | --- | --- | --- |
| `E0206` | unknown-module | 存在しないモジュールを `import` した | [14](14-modules-and-packages.md) |
| `E0207` | duplicate-definition | 名前を 2 回定義した、または `import` した | [14](14-modules-and-packages.md) |
| `E0208` | not-exported | 他のモジュールの非公開の関数・定数・型を使った | [14](14-modules-and-packages.md) |
| `E0209` | unused-import | 使われない `import` | [14](14-modules-and-packages.md) |
| `E0210` | import-cycle | モジュールが互いに `import` している（循環） | [14](14-modules-and-packages.md) |
| `E0211` | constant-cycle | 定数が自分自身に依存している | [14](14-modules-and-packages.md) |
| `E0215` | abstract-type | `distinct` 型を、定義したモジュールの外で構築・展開した | [08](08-refined-and-distinct-types.md) |
| `E0216` | private-in-public | 公開されたシグネチャ・フィールド・契約に、非公開の型や名前がある | [14](14-modules-and-packages.md)、[23](23-units-of-measure.md) |
| `E0218` | module-path-mismatch | モジュール名がファイルのパスと一致しない | [14](14-modules-and-packages.md) |
| `E0219` | reserved-module-name | `std` 以下のモジュール名を使った | [14](14-modules-and-packages.md) |
| `E0224` | invalid-manifest | パッケージの `cernith.toml` が使えない | [14](14-modules-and-packages.md) |
| `E0225` | dependency-not-found | 依存が見つからない、名前が違う、2 か所にある | [14](14-modules-and-packages.md) |
| `E0226` | lock-out-of-date | `cernith.lock` がない、または依存と合わない | [14](14-modules-and-packages.md) |
| `E0227` | dependency-cycle | パッケージが循環して依存している | [14](14-modules-and-packages.md) |
| `E0228` | dependency-lacks-target | パッケージが対応するバックエンドに、依存先が対応していない | [14](14-modules-and-packages.md) |
| `E0229` | target-not-declared | パッケージが `targets` で宣言していないバックエンドでビルドしようとした | [14](14-modules-and-packages.md) |

## foreign 関数

| コード | 名前 | 意味 | 章 |
| --- | --- | --- | --- |
| `E0212` | bad-foreign-declaration | `foreign` の行が、未知のバックエンド・重複・不正なシンボルを指す | [15](15-foreign-functions.md) |
| `E0213` | foreign-type-not-supported | foreign の境界を越えられない型 | [15](15-foreign-functions.md)、[23](23-units-of-measure.md) |
| `E0220` | missing-foreign-target | foreign 関数に、ビルドするバックエンド向けの実装がない | [15](15-foreign-functions.md) |

## テスト

| コード | 名前 | 意味 | 章 |
| --- | --- | --- | --- |
| `E0222` | test-construct-in-code | `defects(of: ...)` を `example`・`property` の外で使った | [16](16-testing.md) |
| `E0223` | reserved-test-name | `regression_` で始まる名前が、プロパティを呼んでいない | [16](16-testing.md) |

## 型

| コード | 名前 | 意味 | 章 |
| --- | --- | --- | --- |
| `E0301` | type-mismatch | 期待された型と違う型の値（`distinct` 型が関わるときは `E0303`） | [03](03-types.md) |
| `E0302` | implicit-conversion | `Int` と `Float64` を混ぜた | [03](03-types.md)、[23](23-units-of-measure.md) |
| `E0303` | incompatible-identities | `distinct` 型を、別の型の代わりに使った（または逆） | [08](08-refined-and-distinct-types.md) |
| `E0304` | missing-argument-label | 呼び出しの引数にラベルがない | [04](04-expressions.md) |
| `E0305` | non-exhaustive-match | `match` がすべての場合を網羅していない | [07](07-records-and-variants.md) |
| `E0306` | unused-result | `Unit` でない値を暗黙に捨てた | [05](05-statements.md) |
| `E0307` | unqualified-integer-division | `Int` どうしに `/` を使った | [03](03-types.md) |
| `E0309` | error-type-mismatch | `?` で、エラーの型が合わない、または `Result`/`Option` を返さない関数で使った | [12](12-errors-and-defects.md) |
| `E0310` | argument-labels-mismatch | ラベルが足りない、未知、または宣言の順序と違う（呼び出し、コンストラクタ、`with`、パターン） | [04](04-expressions.md) |
| `E0311` | var-needs-type | `var` に型注釈がない | [05](05-statements.md) |
| `E0312` | return-in-tail-position | `return` が関数本体の最後の文 | [05](05-statements.md) |
| `E0313` | misplaced-spec-keyword | `result` や `self` を、使える節の外で使った | [09](09-contracts.md) |
| `E0314` | type-alias | 型の別名を宣言した | [03](03-types.md) |
| `E0315` | wrong-type-arguments | 型引数の個数が違う | [06](06-functions.md) |
| `E0316` | bound-not-satisfied | `Eq`、`Ord`、またはインターフェースが必要な型が、それを満たさない | [06](06-functions.md)、[22](22-interfaces.md)、[08](08-refined-and-distinct-types.md) |
| `E0317` | no-such-field | 存在しないフィールド（制約付き型の `.value` を含む） | [07](07-records-and-variants.md) |
| `E0318` | unknown-bound | `Eq`・`Ord` でも、見えているインターフェースでもない境界 | [06](06-functions.md)、[22](22-interfaces.md) |
| `E0319` | not-a-value | 型やモジュールを値として使った | [04](04-expressions.md) |
| `E0320` | cannot-infer | 引数や文脈から型を推論できない | [06](06-functions.md)、[08](08-refined-and-distinct-types.md) |
| `E0321` | assign-to-non-var | `let` やパラメータに代入した | [05](05-statements.md) |
| `E0322` | closure-captures-var | 関数式が、周りの関数の `var` を使った | [05](05-statements.md) |
| `E0323` | let-has-type | `let` に型注釈を書いた | [05](05-statements.md) |
| `E0331` | spec-function-in-code | `spec fn` や property を、実行できるコードから呼んだ | [09](09-contracts.md) |
| `E0332` | spec-construct-in-code | 仕様の中の量化子・ラムダ、またはコードの中の量化子 | [09](09-contracts.md) |
| `E0350` | or-pattern-bindings-differ | or パターンの選択肢が、同じ名前を束縛していない | [07](07-records-and-variants.md) |
| `E0351` | or-pattern-alternatives | or パターンの選択肢が `_` や名前である、重複している、または宣言順に並んでいない | [07](07-records-and-variants.md) |
| `E0360` | unit-mismatch | ある単位（または単位なし）の数が、別の単位の期待される場所にある | [23](23-units-of-measure.md) |
| `E0361` | unit-constructor-argument | 単位のコンストラクタを、`Int` でも `Float64` でもないものに適用した | [23](23-units-of-measure.md) |
| `E0362` | unit-not-canonical | 量の型の単位が、正準形で書かれていない | [23](23-units-of-measure.md) |

## 効果

| コード | 名前 | 意味 | 章 |
| --- | --- | --- | --- |
| `E0401` | undeclared-effect | 宣言していない効果を使った | [11](11-effects.md) |
| `E0402` | effect-in-spec | 仕様の中で効果を使った、または効果を宣言した `spec fn` | [11](11-effects.md) |
| `E0403` | effect-in-test | `example` や `property` が効果のあるコードを呼んだ | [11](11-effects.md) |
| `E0404` | effect-mismatch | 関数値が、許される以上の効果を持つ | [11](11-effects.md) |
| `E0405` | unknown-effect | 存在しない効果の名前 | [11](11-effects.md) |

## 契約と証明義務

| コード | 名前 | 意味 | 章 |
| --- | --- | --- | --- |
| `E0501` | precondition-violated | 呼び出しが、呼ばれる関数の `requires` に違反しうる | [10](10-verification.md) |
| `E0502` | postcondition-violated | 本体が `ensures` に違反しうる | [10](10-verification.md) |
| `E0503` | restriction-violated | 生成が、`where` の条件に違反しうる | [10](10-verification.md) |
| `E0504` | assertion-violated | `assert` が失敗しうる | [10](10-verification.md) |
| `E0505` | invariant-violated | ループ不変条件が、入るとき、または 1 回の繰り返しの後に破れうる | [10](10-verification.md) |
| `E0506` | index-out-of-range | 添字が範囲外になりうる | [10](10-verification.md) |
| `E0507` | division-by-zero | `Float64` の割り算の除数が 0 になりうる | [10](10-verification.md) |
| `E0508` | float-overflow | `Float64` の演算が、有限でない値を作りうる（既定では `pub` の定義ではエラー、それ以外は警告にとどまる。[10](10-verification.md) を参照） | [10](10-verification.md) |
| `W0504` | obligation-unproved | ソルバーが証明義務を決められなかった | [10](10-verification.md) |
| `W0801` | vacuous-contract | 契約が自明に成り立つ（空虚） | [10](10-verification.md) |

## 停止性

| コード | 名前 | 意味 | 章 |
| --- | --- | --- | --- |
| `E0509` | measure-not-decreasing | `decreases` の減少量が、再帰呼び出しで小さくならない | [13](13-termination.md) |
| `E0601` | termination-unproved | ループや再帰のグループに、停止の根拠がない | [13](13-termination.md) |
| `E0602` | non-positive-recursive-type | 再帰的な型が、自分の再帰グループの値を取る関数を持つ | [13](13-termination.md) |

## 未完成の箇所

| コード | 名前 | 意味 | 章 |
| --- | --- | --- | --- |
| `E0702` | hole-in-release | リリースビルドに `hole` が残っている | [02](02-lexical-structure.md) |
| `W0701` | hole-present | 関数に `hole` がある | [02](02-lexical-structure.md) |

## リソースとケイパビリティ

| コード | 名前 | 意味 | 章 |
| --- | --- | --- | --- |
| `E1101` | resource-escapes-scope | リソース（第二級の値）を、戻した・格納した・コピーした・クロージャが取り込んだ、または呼び出しの引数以外の場所で使った | [17](17-resources-and-capabilities.md) |
| `E1102` | release-error-not-covered | `using` の本体のエラー型が、取得・解放のエラー型と違う | [17](17-resources-and-capabilities.md) |
| `E1103` | resource-outside-using | リソースの取得を `using` の外で束縛した | [17](17-resources-and-capabilities.md) |
| `E1105` | bad-resource-declaration | リソースの `release` 関数がない、またはシグネチャが違う | [17](17-resources-and-capabilities.md) |
| `E1106` | resource-as-type-argument | リソース型を型引数やフィールドにした | [17](17-resources-and-capabilities.md) |
| `E1107` | return-inside-using | `using` の本体の中で `return` した | [17](17-resources-and-capabilities.md) |
| `E1301` | capability-outside-manifest | ケイパビリティを、`manifest` と定義モジュールの外で作った | [17](17-resources-and-capabilities.md) |
| `E1302` | simulated-outside-test | テスト用のケイパビリティを、テストの外で作った | [17](17-resources-and-capabilities.md) |
| `E1303` | bad-manifest | `manifest` が 2 つある、フィールドがケイパビリティでない、`main` が受け取らない | [17](17-resources-and-capabilities.md) |
| `E1305` | capability-unavailable-on-target | `manifest` が挙げたケイパビリティを、ビルドするバックエンドの世界が提供しない | [17](17-resources-and-capabilities.md) |
| `E1306` | internal-function | 標準ライブラリの内部用の関数を、プログラムが呼んだ | [17](17-resources-and-capabilities.md) |

## 並行処理

| コード | 名前 | 意味 | 章 |
| --- | --- | --- | --- |
| `E1201` | task-not-awaited | タスクをすべての経路で `await` していない | [18](18-concurrency.md) |
| `E1202` | task-escapes-scope | タスク・スコープ・チャネルの端が、スコープの値になった、またはタスクをコピー・格納・受け渡し・取り込みした | [18](18-concurrency.md) |
| `E1203` | missing-await | `async` の関数の呼び出しに `await` がない | [18](18-concurrency.md) |
| `E1204` | task-awaited-twice | タスクを 2 回以上 `await` した | [18](18-concurrency.md) |
| `E1205` | async-through-effect-parameter | 効果 `async` を効果パラメータに通した | [18](18-concurrency.md) |
| `E1206` | await-of-nothing | 中断できないものの前に `await` を書いた | [18](18-concurrency.md) |
| `E1207` | await-in-using | `using` の本体が中断点を含む | [18](18-concurrency.md) |
| `W1203` | wait-cannot-complete | チャネルから受信するが、送信しない | [18](18-concurrency.md) |

## インターフェースと遅延列

| コード | 名前 | 意味 | 章 |
| --- | --- | --- | --- |
| `E1401` | invalid-interface | インターフェースの宣言が規則に反する（`Self` のないメソッド、型パラメータを持つメソッド、インターフェース自体の型パラメータ） | [22](22-interfaces.md) |
| `E1402` | incoherent-impl | `impl` が、インターフェースと型コンストラクタごとにインスタンスは 1 つ、という規則に反する | [22](22-interfaces.md) |
| `E1403` | impl-does-not-match | `impl` がインターフェースと合っていない | [22](22-interfaces.md) |
| `E1404` | cannot-derive | `impl` が `derive` を求めたが、コンパイラが導出できない | [22](22-interfaces.md) |
| `E1405` | interface-is-not-a-type | 型が必要な場所にインターフェースを書いた | [22](22-interfaces.md) |
| `E1406` | loop-over-seq-clause | `Seq` に対する `for` に `invariant` または `decreases` の節がある | [05](05-statements.md) |

## 関連

- 規則の詳細は、各コードの「章」の列にあるページにあります。
