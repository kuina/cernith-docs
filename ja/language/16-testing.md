# テスト

テストは言語の一部です。テストする関数と同じモジュールの、その関数のすぐ後に書き、`cernith test` が実行します。テストには `example` と `property` の 2 種類があります。

```cernith
module guide.testing;

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

example clamp_below for clamp {
  clamp(value: -5, low: 0, high: 10) == 0
}

property clamp_is_idempotent for clamp(value: Int, low: Int, high: Int) {
  requires low <= high;

  let once = clamp(value: value, low: low, high: high);
  clamp(value: once, low: low, high: high) == once
}
```

- **example** は、1 つの `Bool` 式です。1 回だけ評価されます。具体的な入力についての事実を述べます。
- **property** は、パラメータと、省略できる `requires` 節、`Bool` で終わるブロックを持ちます。パラメータの型のすべての入力について成り立つ法則を述べます。処理系が入力を生成して試し、偽になる入力があれば、最小の反例を報告します。
- `for 関数` で指す関数は、**同じモジュールで宣言された関数**でなければなりません（`E0205`）。テストの名前はモジュールの中で一意です。テストは公開されません。
- テストは、`test` 文脈で評価されます。純粋なコード、`diverge` のコード、`spec fn`、仕様に使える要素をすべて使えます（[09 契約](09-contracts.md)）。効果のあるコードは、シミュレート世界の中で試します（後述）。

## example の実行

`cernith test` は、式を、すべての契約を検査しながら評価します。結果は次のどれかです。

| 状態 | 意味 |
| --- | --- |
| `passed` | 式が `true` |
| `failed` | 式が `false` |
| `error` | 評価が欠陥で止まった、または予算（後述）を使い切った。報告が付く。**欠陥で止まった example は、成功ではない** |

偶然に事前条件を破るテストは、関数の失敗ではなく、テストの `error` です。欠陥になることを確かめたいテストは、`defects(of: ...)` で明示します。

## defects(of: ...)

欠陥はコードの中では捕まえられません（[12 エラーと欠陥](12-errors-and-defects.md)）。「この呼び出しは事前条件に違反する」と述べたいテストは、次のように書きます。

```cernith
module guide.testing_defects;

pub fn halve(value: Int) -> Int {
  requires value >= 0;
  ensures result >= 0;

  value
}

example halve_of_a_negative_is_a_defect for halve {
  defects(of: halve(value: -1)) and not defects(of: halve(value: 6))
}
```

`defects(of: 式)` は、式の評価が欠陥で止まれば `true`、返れば `false` の `Bool` です。`example` と `property` の中だけで書け、ほかの場所で使うと `E0222` です。欠陥の分類や責任は調べられません。`defects` の中の呼び出しには、証明義務は作られません（欠陥になるのが目的だからです）。資源の枯渇は捕まえません。

## property の実行

プロパティの 1 回ごと（1 ケース）に、パラメータの値が生成され、`requires` が評価され、ブロックが評価されます。

| ケースの結果 | 意味 |
| --- | --- |
| `pass` | ブロックが `true` |
| `fail` | ブロックが `false`、または欠陥で止まった（欠陥の報告が付く） |
| `discard` | `requires` が偽。捨てられる |

**100 ケース通る**（`--cases N` で変更）か、最初の失敗で止まります。捨てたケースが、その 10 倍（既定の 100 では 1000 個）を超えると、`gave-up` です。意味のある個数の入力を試せなかったことを表し、成功ではありません。

```cernith-snippet
module guide.testing_gave_up;

pub fn identity(value: Int) -> Int {
  value
}

property rarely_applies for identity(value: Int) {
  requires value == 123456789;

  identity(value: value) == value
}
```

上の property は `gave-up` になります（`cernith test` の出力は、後に示します）。`requires` で入力を絞りすぎずに、「すべての入力について成り立つ」形に書きます。

### 入力の生成

入力は型から決まります。利用者が生成器を書くことはありません。

| 型 | 値 |
| --- | --- |
| `Int` | `[-size, size]` の一様な値。15% の確率で境界の値（`0`、`±1`、`±2`、`10`、`100`、`255`、`256`、`2^31-1`、`-2^31`、`2^53`、`2^63`、`-2^63`） |
| `Float64` | 有限の値だけ。0.25 の倍数か `[-size, size]` の一様な値。15% の確率で境界の値（`0.0`、`±1.0`、`0.5`、`1e-9`、`1e9`、`±1e100`） |
| `Bool`、`Unit` | 両方の値、`()` |
| `String` | 長さ `size / 4` まで。ASCII でない文字、空白、カンマ、`=` を含む小さな文字集合 |
| `List<T>` | 長さ `size / 4` まで。要素は半分の大きさで生成 |
| `Option<T>`、`Result<T, E>` | 各ケース。`None` は 4 回に 1 回 |
| レコード | 各フィールド。不変条件（`where`）は、制約付き型と同じく、満たす値だけを選ぶ。400 回で満たす値が見つからなければ、プロパティが `error` |
| バリアント | ランダムなケースと生成したフィールド。深さ 4 を超えると、フィールドのないケースがあればそれを選ぶ（再帰的な値が有限になる） |
| `distinct` 型 | 土台の値を包む |
| 制約付き型 | 土台の値から、条件を満たすものを選ぶ。いくつかの尺度（0 の近く、遠く）で試す。400 回で見つからなければ、プロパティが `error` |
| 関数型 | 生成できない。プロパティは `error` |

`size` は、ケース番号が増えるにつれて 3 から大きくなります（`2 * case + 3`、最大 200）。初めのケースは小さな値です。

### 再現性

実行は決定的です。乱数の種は、グローバルな種（`--seed`、既定は 0）、モジュールのパス、プロパティの名前のハッシュで決まります。同じプログラム、種、ケース数なら、どのマシンでも同じ入力と同じ結果です。種は報告に含まれます。

### 縮小

失敗した入力は**縮小**されます。パラメータを順に、より単純な候補に置き換えて、それでも失敗する最初の候補を採用することを、置き換えられなくなるか 2000 個の候補を試すまで繰り返します（欠陥で止まるのも失敗として数えます）。

| 型 | 単純な候補（この順） |
| --- | --- |
| `Int` | `0`、負なら `-v`、`v / 2`（0 に向かって）、`v - 1`（0 に向かって） |
| `Float64` | `0.0`、切り捨て、`v / 2` |
| `Bool` | `false` |
| `String` | `""`、前半、1 文字ずつ取り除いたもの |
| `List<T>` | `[]`、各半分、各要素を取り除いたもの、各要素を縮小したもの |
| `Option<T>` | `None`、中身を縮小したもの |
| レコード | 各フィールドを縮小 |
| バリアント | より前のケース（フィールドは最小の値にする）、フィールドを縮小したもの |
| `distinct`、制約付き型 | 土台を縮小（制約付き型は、条件の内側に保つ） |

結果の反例は、局所的に最小です。どの 1 ステップを試しても、単純になって失敗するものがありません。たとえば `x < 100` が偽になる入力は `x: 100`、`xs.length < 3` が偽になる入力は `xs: [0, 0, 0]` に縮小されます。

```text
passed   property guide.props.double_is_even
failed   property guide.props.double_doubles  counterexample: value: 101
passed   example  guide.props.small
3 tests: 2 passed, 1 failed
```

この `double_doubles` は、`double(value)` が `value > 100` のとき `value` をそのまま返す関数について、`double(value) == value * 2` を主張したものです。最初に見つかった反例は `9223372036854775808`（2^63）でしたが、縮小の結果は境界の `101` になっています。

## 反例からの回帰テスト

失敗したプロパティは、**回帰テスト**の example をソースの形で出力します。

```text
example regression_double_doubles_722e5041 for double {
  double_doubles(value: 101)
}
```

- プロパティは、`example` や `property` と仕様の中で、関数のように**呼べます**（実行できるコードの中では呼べません。`E0331`）。`requires` 節が偽なら `true` を返し（節は含意として働きます）、そうでなければブロックの値を返します。生成された example は、縮小された値を Cernith の式として書いて、プロパティを呼びます。
- 名前は `regression_<プロパティ名>_<8 桁の 16 進数>` です。数字は引数の文字列のハッシュで、同じ反例なら同じ名前になります。`regression_` で始まる名前は予約されていて、プロパティを呼ばないとエラー `E0223` です。
- バグを直す前は失敗し、直した後は成功して、そのケースを守り続けます。

```cernith-error E0223
module guide.testing_names;

pub fn identity(value: Int) -> Int {
  value
}

example regression_something_1234abcd for identity {
  identity(value: 1) == 1
}
```

## 効果のあるコードのテスト

`example` と `property` は純粋です（[11 効果](11-effects.md)）。入出力、時計、乱数を使うコードは、標準ライブラリ `std.test` の `with_world` で、**シミュレート世界**の中で実行して試します。

```text
test.with_world(
  world: test.World(input: ["1", "2", "3"], now_ms: 0, seed: 0),
  run: fn() -> Int uses io {
    main(args: [])
  },
).output == ["total 6"]
```

`with_world` は、入力の行（`io.read_all_lines` が返すもの）、`clock.now_ms` の値、`random` の種を決めた世界で、関数を実行し、関数の値と書き出された行を持つ `Outcome` を返します。世界は決定的で、`with_world` 自体は純粋です。詳しくは [std.test](../stdlib/test.md) にあります。ファイル、環境変数、ネットワーク、ケイパビリティを含む世界は [std.sim](../stdlib/sim.md) です。非同期なコードの実行は、スケジュールを入力として決められます（[18 並行処理](18-concurrency.md)）。

## 予算

すべての example と、プロパティの各ケースは、実行予算（評価器のステップ数、既定は 100 万。`--budget N`）の下で実行されます。使い切ると、`error`（`resource-exhausted`）です。`diverge` のコードを試すための仕組みで、時間の制限ではありません。

## 契約とテスト

テストの実行中は、すべての契約の節が検査されます。呼ばれた関数の中の節も含みます。違反すると、テストが欠陥で止まります（事前条件は責任が呼び出し側、事後条件は呼ばれた関数）。実行できない節（型全体を動く量化子）は、失敗ではなく、飛ばされます。実行されて、すべてのテストで成り立ったが証明されていない節は、`tested-only` の状態です（[10 証明義務と保証レベル](10-verification.md)）。

## 一部のテストの実行と、テストが見逃すもの

`cernith test --only テキスト` は、名前、対象の関数、または `module.name` に `テキスト`（大文字と小文字を区別します）を含むテストだけを実行します。1 つの関数のテストを実行するには、その関数の名前を指定します。一致するテストがなければ、標準エラー出力にそう出力して `2` で終了します。名前の打ち間違いが「すべて通った」と受け取られないようにするためです。

`--coverage` と `--mutate` は、テストが取りこぼしているものを示します。次のモジュールには、どのテストも呼ばない関数と、最後の分岐をどの example も通らない `if` があります。

```cernith
module guide.coverage;

pub fn double(value: Int) -> Int {
  value * 2
}

example double_of_three for double {
  double(value: 3) == 6
}

pub fn sign(value: Int) -> Int {
  if value > 0 {
    1
  } else if value < 0 {
    -1
  } else {
    0
  }
}

example sign_of_negative for sign {
  sign(value: -5) == -1
}

example sign_of_positive for sign {
  sign(value: 5) == 1
}

pub fn unused(value: Int) -> Int {
  value + 1
}
```

`--coverage` は、要約と、どのテストも到達しなかった場所（呼ばれなかった関数、`if` の分岐、`match` の腕、評価されなかった契約の節）を追加します。`coverage.cnth` として保存した場合の出力です。

```text
$ cernith test --coverage coverage.cnth
passed   example  guide.coverage.double_of_three
passed   example  guide.coverage.sign_of_negative
passed   example  guide.coverage.sign_of_positive
3 tests: 3 passed
functions    2/3 (66.7%)
if branches  3/4 (75.0%)
match arms   0/0
clauses      0/0
  coverage.cnth:14:10 if-branch sign: the `else` branch of `if value < 0` never ran
  coverage.cnth:29:1 function unused: no test calls it
```

`--mutate` は、テストを実行し、すべて通れば、テスト対象のコードに小さな変更（ミュータント）を 1 つずつ加えて、変更した場所に到達するテストだけをもう一度実行します。テストが失敗するようになるミュータントは**殺された**（killed）ミュータントです。どのテストも気づかないミュータントは**生き残った**（survived）ミュータントで、example が足りないことを示します（ここでは `sign(value: 0)`）。

```text
$ cernith test --mutate coverage.cnth
passed   example  guide.coverage.double_of_three
passed   example  guide.coverage.sign_of_negative
passed   example  guide.coverage.sign_of_positive
3 tests: 3 passed
mutants 9 of 12 run: 5 killed, 4 survived, 3 unreached, score 55.6%
  SURVIVED coverage.cnth:12:6 sign: `>` becomes `>=` in `value > 0`
  SURVIVED coverage.cnth:12:14 sign: the constant 0 becomes 1
  SURVIVED coverage.cnth:14:13 sign: `<` becomes `<=` in `value < 0`
  SURVIVED coverage.cnth:14:21 sign: the constant 0 becomes 1
```

- 加える変更は、比較の入れ替え（`==` と `!=`、`<` と `<=`、`>` と `>=`）、算術の入れ替え（`+` と `-`、`*` を `+` に）、整数定数に 1 を足す変更、`if` の 2 つの分岐の入れ替え、隣り合う 2 つの `match` の腕の入れ替え、`requires`・`ensures`・ループの `invariant` を 1 つ取り除く変更です。
- 変更した場所をどのテストも実行しないミュータントは**到達しなかった**（unreached）ミュータントです。スコアは `killed / (killed + survived)` です。
- 生き残ったミュータントは、example が足りない証拠であって、証明ではありません。意味を変えない変更（`x + 0` など）は、いつまでも生き残ります。ミュータントは均等に選ばれ、実行は再現できます。`--mutants N` は実行する数の上限（既定 200）、`--mutation-cases N` はプロパティごとのケース数（既定 20）、`--min-score PERCENT` は、スコアがそれを下回ると終了ステータスを `1` にします。
- `--json` を付けると、文書には `coverage`（`cernith-coverage/0.1`）か `mutation`（`cernith-mutation/0.1`）のキーが加わります。

## インターフェースの法則

インターフェースの `law`（法則）は、そのインターフェースの `impl`（実装）のうち、型パラメータを持たないものそれぞれに対して、テストとして実行されます（[22 インターフェース](22-interfaces.md)）。テストランナーは、法則を `impl` の型で具体化し、`property` と同じように、生成した値で実行します。名前は `law_<インターフェース>_<法則>_for_<型>` で、`--only` で選べます。失敗したケースは、法則と `impl` の名前と反例を示します。次のモジュールでは、法則 `not_negative` は `Square`（フィールドが `where` で制限されています）では成り立ち、`Strip`（制限されていません）では成り立ちません。

```cernith-snippet
module guide.testing_laws;

pub interface Area {
  fn area(shape: Self) -> Int;

  law not_negative(shape: Self) {
    area(shape: shape) >= 0
  }
}

pub record Square {
  side: Int,
} where side >= 0;

pub record Strip {
  length: Int,
  width: Int,
}

impl Area for Square {
  fn area(shape: Square) -> Int {
    shape.side * shape.side
  }
}

impl Area for Strip {
  fn area(shape: Strip) -> Int {
    shape.length * shape.width
  }
}
```

テストの出力は次のとおりです（`area.cnth` として保存した場合）。

```text
$ cernith test --only law_ area.cnth
passed   property guide.testing_laws.law_Area_not_negative_for_Square
failed   property guide.testing_laws.law_Area_not_negative_for_Strip  counterexample: shape: Strip(length: -1, width: 1)
2 tests: 1 passed, 1 failed
```

検証器は法則の証明を試みません。ある `impl` について偽になる法則は、その `impl` の欠陥です。

## 結果と終了ステータス

`cernith test` は、テストごとに 1 行を出力します。`--json` を付けると、`cernith-test-results` の JSON になります。

```text
failed   property guide.props.double_doubles  counterexample: value: 101
gave-up  property guide.testing_gave_up.rarely_applies  only 0 of 100 generated inputs satisfied the requires clauses; ...
error    example  guide.c.never_ends  resource-exhausted:
```

| JSON のフィールド | 意味 |
| --- | --- |
| `module`、`kind`、`name`、`target` | どのテストか |
| `status` | `passed`、`failed`、`error`、`gave-up` |
| `cases`、`discarded`、`seed` | プロパティの場合 |
| `counterexample`、`original`、`shrink_steps` | 失敗したプロパティの、最小の反例、最初に見つかった反例、試した候補の数 |
| `defect` | テストが欠陥で止まったときの、欠陥の報告 |
| `regression` | 生成された回帰テストの example |

| オプション | 意味 |
| --- | --- |
| `--seed N` | グローバルな種（既定 0） |
| `--cases N` | プロパティの通るケース数（既定 100） |
| `--budget N` | example とプロパティの各ケースのステップ数 |
| `--only テキスト` | 名前、対象の関数、`module.name` に `テキスト` を含むテストだけを実行する |
| `--coverage` | どのテストも到達しなかった関数、分岐、腕、契約の節も報告する |
| `--mutate` | テスト対象のコードに小さな変更を加えて、どのテストも気づかないものを一覧する |
| `--json` | JSON で出力する |

終了ステータスは、すべて通れば `0`、失敗か `gave-up` があれば `1`、プログラムが型検査を通らないか、`error` のテストがあるか、`--only` がどのテストにも一致しなければ `2` です。テストは、エラーなく検査を通るプログラムに対してだけ実行されます。

## 診断コード

| コード | 名前 | 意味 |
| --- | --- | --- |
| `E0205` | unknown-example-target | `for` で指す関数が、そのモジュールにない |
| `E0222` | test-construct-in-code | `defects(of: ...)` を `example` と `property` の外で使った |
| `E0223` | reserved-test-name | `regression_` で始まる名前が、プロパティを呼んでいない |
| `E0331` | spec-function-in-code | プロパティ（や `spec fn`）を、実行できるコードから呼んだ |

## 関連

- 契約：[09 契約](09-contracts.md)
- 欠陥：[12 エラーと欠陥](12-errors-and-defects.md)
- `std.test`：[stdlib/test.md](../stdlib/test.md)
