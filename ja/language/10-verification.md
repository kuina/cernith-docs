# 証明義務と保証レベル

チェッカーは、契約（[09 契約](09-contracts.md)）や、範囲外の添字・ゼロ除算など欠陥を起こしうる操作から、**証明義務**を生成します。証明を試みるには `cernith check --verify` を使います。`--verify` なしの `cernith check` は、構文・名前・型・効果・網羅性・停止性を検査しますが、証明義務は調べません。

## 証明義務

証明義務とは、「この箇所で、この条件が成り立つ」とチェッカーが示すべき事柄です。各義務には、種類・発生箇所・責任を負う側が定められています。

| 種類 | 生まれる場所 | 示すこと | 責任 | 診断コード（反証されたとき） |
| --- | --- | --- | --- | --- |
| `pre` | 呼び出しごと | 呼ばれる関数の `requires` | 呼び出し側 | `E0501` |
| `post` | 関数の終わりと各 `return` | 各 `ensures` | 関数 | `E0502` |
| `restrict` | 制約付き型・不変条件付きレコードの生成と `with` | 条件が成り立つ | 生成した箇所 | `E0503` |
| `assert` | `assert` ごと | 主張した式 | 関数 | `E0504` |
| `loop-invariant` | ループごと | 入るときと、1 回繰り返した後で不変条件が成り立つ | 関数 | `E0505` |
| `index` | `xs[i]` ごと | `0 <= i < xs.length` | 関数 | `E0506` |
| `float-div` | `Float64` の `/` ごと | 除数が 0 でない | 関数 | `E0507` |
| `float-range` | `Float64` の `+ - * /` ごと | 結果が有限 | 関数 | `E0508` |
| `term` | 再帰呼び出しと `decreases` 付きのループ | 減少量が 0 以上で、小さくなる | 関数 | `E0509` |

整数の割り算・`float.sqrt` のように欠陥になりうる標準ライブラリの関数は、その関数の `requires`（`pre` の義務）として扱われます。型の正しさ、効果の規則、`match` の網羅性は、証明義務ではなく、静的な検査です。

### 義務の作り方

証明義務は、関数本体の記号実行によって生成されます。検証は**関数ごとに独立して**行われ、**その関数自身の契約**と**呼び出す関数の契約**だけを使って本体を解析します。呼び出し先の本体は調べません。

1. **入口。** 各パラメータは、その型の任意の値です。成り立つ事実は、制約付き型のパラメータの条件と、関数の `requires` です。
2. **分岐のないコード。** その時点で成り立つ事実と分岐条件をすべて使って、証明義務を調べます。証明が済んだあとや、欠陥を起こしうる箇所を通過したあとでは、その条件が成り立つと仮定して解析を続けます。
3. **分岐。** `if c { a } else { b }` では、`a` の義務は `c` を仮定して、`b` の義務は `not c` を仮定して調べます。`match` も、パターンの条件で同様です。
4. **短絡。** `a and b` の `b` の義務は `a` を、`a or b` の `b` の義務は `not a` を、`a implies b` の `b` の義務は `a` を仮定して調べます。
5. **呼び出し。** 呼ばれる関数の `requires` が `pre` の義務になります。戻り値は未知の値ですが、戻り値の型の条件と、呼ばれる関数の `ensures` が成り立つものとします。`ensures` のない関数の戻り値には、何の仮定もありません。
6. **再帰。** 自分自身への呼び出しは普通の呼び出しです。`decreases M` があれば、`0 <= M(引数) < M(パラメータ)` の `term` の義務も生まれます。
7. **ループ。** `invariant I` と `decreases M` を持つループでは、入るときに `I` が成り立ち、`I` とループ条件を仮定して 1 回繰り返すと `I` が再び成り立ち `M` が減る、ことを示します。ループのあとでは、`I` とループ条件の否定が成り立つとし、ループで代入された `var` はそれ以外には何も分からないものとします。不変条件のないループも、代入された `var` を未知の値にするので、それに依存する義務は決まらなくなります。
8. **途中の脱出。** `return e;` では、その地点で `post` の義務を調べます。

`Float64` の値は、有限であると仮定されます。

## 状態

義務は、次のいずれかの状態になります。

| 状態 | 意味 |
| --- | --- |
| 証明済み（`proved`） | 成り立つことが示せた。実行時の検査は要らない |
| 反証済み（`refuted`） | 成り立たない入力が実在する。反例の入力を添えて報告される |
| 不明（`unknown`） | ソルバーが決められなかった（時間切れなど）。実行時の検査が残る |
| テストのみ（`tested-only`） | 実行できる条件で、すべてのテストで成り立ったが、証明はない |
| 未試行（`not-attempted`） | 証明専用の条件が証明されていない、またはチェッカーが扱えない構文を含む |

**`refuted` は、反例が実在するときだけ報告されます。** すべての経路の表現が正確で近似を含まないか、または関数を実際に反例の入力で実行して失敗を確認できたときです。呼び出す関数の契約を近似として使った経路では、反例が実在しない可能性があるので、`unknown` で、反例は「候補」と記されます。`unknown` は、使った予算（`rlimit`）を示します。

| 状態 | 重大度 |
| --- | --- |
| 反証済み | エラー（`E05xx`）。反例の入力と責任が添えられる。ただし `float-range`（`E0508`）は、最低レベルが `safe` の定義（既定では `pub` の定義）でだけエラー扱いで、それ以外の定義では報告されても検査は失敗しない |
| 不明 | 警告 `W0504` |
| テストのみ、未試行 | 注記（エラーでも警告でもない） |

反例の入力は、Cernith のコードの形で書かれます。次は、`value` が負のときに事後条件が破れる関数です。

```cernith-error E0502
module guide.refuted;

pub fn absolute(value: Int) -> Int {
  ensures result >= 0;

  value
}
```

```text
t.cnth:6:3 E0502 postcondition-violated: The body can return a value that violates its postcondition. [refuted, blame callee; value=-1]
```

次の例は、`xs[i]`、`assert`、ループ不変条件、`Float64` の割り算の義務が、反証される例です。

```cernith-error E0506
module guide.refuted;

pub fn at(xs: List<Int>, i: Int) -> Int {
  xs[i]
}
```

```cernith-error E0504
module guide.refuted;

pub fn positive(n: Int) -> Int {
  requires n >= 0;

  assert n > 0;
  n
}
```

```cernith-error E0505
module guide.refuted;

pub fn count_up(n: Int) -> Int {
  requires n >= 0;

  var i: Int = 0;
  while i < n {
    invariant i < 0;
    decreases n - i;

    i = i + 1;
  }
  i
}
```

```cernith-error E0507
module guide.refuted;

pub fn ratio(a: Float64, b: Float64) -> Float64 {
  requires a >= -1.0e100 and a <= 1.0e100;

  a / b
}
```

次の例では、`else` の分岐で `result <= a` を示すために、文字列の順序について推論する必要があります。ソルバーは予算内で判断できないため、事後条件は `unknown`（`W0504`）になります。この場合も実行時の検査は残り、契約に違反すると欠陥として停止します。

```cernith-error W0504
module guide.unknown;

pub fn smaller(a: String, b: String) -> String {
  ensures result <= a and result <= b;

  if a < b {
    a
  } else {
    b
  }
}
```

> 補足：チェッカーが証明に使えない構文や型があります。関数式（クロージャ）、再帰的な `spec fn`、`Map`、そして自分自身のリストを含む型（たとえば `Branch(children: List<Node>)` というケースを持つ `Node`）です。これらを含む義務は、証明も反証もされず、実行時の検査になります。これらを含む定義（や、`int.pow` のように契約でこれらを使う関数を呼ぶ定義）は、保証レベルが `typed` のままで、診断も出ません。「診断が出ない」ことは「証明できた」ことと同じではないので、何が証明されたかを知りたいときは `summary.levels` を見ます。

## 保証レベル

チェッカーは定義ごとに、到達した最も高い**保証レベル**を報告します。

| レベル | 要求 |
| --- | --- |
| `typed` | 型検査を通り、効果の規則が守られ、`match` が網羅的 |
| `safe` | `pre`、`restrict`、`assert`、`index`、`float-div`、`float-range` の義務がすべて証明済み |
| `specified` | `safe` に加え、少なくとも 1 つの `example` と、空虚（後述）でない `ensures` か property が 1 つ以上ある |
| `proved` | `specified` に加え、すべての `ensures`、`loop-invariant`、`term` の義務が証明済み |

レベルは、`cernith check --verify` の JSON 出力の `summary.levels` に、定義ごとに出ます。

```cernith
module guide.levels;

pub fn add(first: Int, second: Int) -> Int {
  first + second
}

example add_adds for add {
  add(first: 1, second: 2) == 3
}

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

example clamp_above for clamp {
  clamp(value: 12, low: 0, high: 9) == 9
}

pub fn first_of(items: List<Int>) -> Int {
  requires items.length > 0;

  items[0]
}

example first_of_reads_the_head for first_of {
  first_of(items: [7, 8]) == 7
}
```

上のモジュールで `cernith check --verify` を実行すると、`summary.levels` に次が出ます。`add` は契約を持たず、証明すべき義務も残らないので `safe`、`first_of` も事前条件で添字が守られているので `safe`、`clamp` はすべての義務が証明されて `proved` です。

```text
"levels": {
  "guide.levels::add": "safe",
  "guide.levels::clamp": "proved",
  "guide.levels::first_of": "safe"
}
```

## 空虚な契約

常に成り立つだけの契約では、実装の正しさを保証できません。チェッカーは、次のような事後条件に `W0801`（vacuous-contract）を出し、その契約を根拠に保証レベルを `specified` に上げることはしません。

- **型だけから導かれる**事後条件（どんな本体でも成り立つもの）。
- **本体を繰り返しているだけ**の事後条件（`result == value + value` で、本体も `value + value` のとき）。
- 本体を変異させても（演算子の入れ替え、定数の変更）成り立ち続ける事後条件。

```cernith-error W0801
module guide.vacuous;

pub fn twice(value: Int) -> Int {
  ensures result == value + value;

  value + value
}
```

```cernith-error W0801
module guide.vacuous;

pub fn identity(value: Int) -> Int {
  ensures result >= value or result < value;

  value
}
```

`--no-vacuity` を付けると、この検出を省略します。

## 実行時との関係

証明済みの条件は実行時に検査されません。証明できず、実行できる条件は、実行時に検査されます（[09 契約](09-contracts.md)）。つまり、証明が決まらないことは、プログラムの安全性が失われることではなく、検査が実行時に回ることを意味します。

## 関連

- 契約の書き方：[09 契約](09-contracts.md)
- 欠陥：[12 エラーと欠陥](12-errors-and-defects.md)
- 停止性（`term` の義務）：[13 停止性](13-termination.md)
