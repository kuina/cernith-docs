# 契約

**契約**は、関数やループや型が満たすべき条件を、コードと同じ言語で書いたものです。契約を破ると、プログラムは欠陥で停止します（[12 エラーと欠陥](12-errors-and-defects.md)）。チェッカーは、契約が必ず守られることの証明を試みます（[10 証明義務と保証レベル](10-verification.md)）。

## 節の種類

| 節 | 書く場所 | 意味 | 破ったときの責任 |
| --- | --- | --- | --- |
| `requires P;` | 関数、property | 呼び出されるときに `P` が成り立つ | 呼び出し側 |
| `ensures Q;` | 関数 | 関数が返るときに `Q` が成り立つ。`result` が戻り値 | 関数自身 |
| `decreases M;` | 関数、ループ | 再帰やループのたびに小さくなる、0 以上の `Int`（[13 停止性](13-termination.md)） | 関数 |
| `invariant I;` | ループ | ループに入るとき、および各回の終わりに `I` が成り立つ | 関数 |
| `where P` | 制約付き型、レコード | 型のすべての値が `P` を満たす（`self` が値、レコードならフィールド名が見える） | 値を作った箇所 |
| `assert P;` | 文 | その地点で `P` が成り立つ。以降は `P` が成り立つとして調べられる | 関数 |

「責任」は、診断と欠陥の報告に現れます。`requires` に違反する呼び出しは、**呼び出し側**の誤りです。`ensures` に違反する戻り値は、**関数**の誤りです。

## 位置と順序

節は、関数やループの本体の先頭に、`requires`、`ensures`、`decreases`（ループでは `invariant`、`decreases`）の順で書きます。同じ種類の節を複数書くと、すべてを満たす（論理積）ことになります。節と、そのあとの文の間には、空行を 1 つ置きます。節の順序を変えたプログラムは、正準形（[19 正準形](19-canonical-form.md)）では順序が直されます。

`old(...)` はありません。パラメータは変更されず、局所変数 `var` は呼び出し側から観察できないので、関数の入口の値と出口の値を区別する必要がないためです。ループ不変条件の中で、ループに入る前の `var` の値を参照したいときは、ループの前に `let` で束縛します。

```cernith
module guide.contracts;

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

example clamp_inside for clamp {
  clamp(value: 5, low: 0, high: 9) == 5
}

example clamp_rejects_a_reversed_range for clamp {
  defects(of: clamp(value: 5, low: 9, high: 0))
}
```

`defects(of: 式)` は、式の評価が欠陥で止まるときに `true` になる、テストの中だけで使える式です（[16 テスト](16-testing.md)）。

事前条件に違反する呼び出しがあるとき、`cernith check --verify` は `E0501`（precondition-violated）を報告し、違反する入力の例を示します。事後条件に違反しうる本体は `E0502` です。

```cernith-error E0501
module guide.contracts;

pub fn square_root_floor(value: Int) -> Int {
  requires value >= 0;

  value
}

pub fn caller(x: Int) -> Int {
  square_root_floor(value: x)
}
```

```cernith-error E0502
module guide.contracts;

pub fn absolute(value: Int) -> Int {
  ensures result >= 0;

  value
}
```

## 仕様式

契約の条件は、**仕様式**です。`Bool`（`decreases` では `Int`）の普通の式ですが、通常の式には書けない次のものを使えます。

| 要素 | 意味 |
| --- | --- |
| `result` | `ensures` の中で、関数の戻り値 |
| `self` | 型の `where` の中で、検査される値 |
| `implies` | 含意。`a implies b` は、`a` が偽なら真 |
| `forall`、`exists` | 量化子 |
| `spec fn` の呼び出し | 仕様のためだけの関数 |

仕様式は**純粋**でなければなりません。効果のある関数は呼べません（`E0402`）。ラムダも書けません。

式が書かれる場所は 3 種類あり、使える要素が違います。

| 文脈 | 場所 | 使えるもの |
| --- | --- | --- |
| コード | 関数本体、ループ本体、定数 | 通常の式 |
| 仕様 | `requires`、`ensures`、`decreases`、`invariant`、`where`、`spec fn` の本体 | コードに加えて、量化子、`spec fn` の呼び出し、`result`、`self` |
| テスト | `example`、`property` の本体 | コードと仕様の両方 |

コードの中に量化子や `spec fn` の呼び出しを書くとエラー（`E0332`、`E0331`）です。`result` を `ensures` の外に、`self` を `where` の外に書くと `E0313` です。仕様のための構文はコードではありませんが、チェッカーが証明していない条件は、量化子や `spec fn` の呼び出しを含めて、プログラムの実行時にも評価され、違反は欠陥です。

### 量化子

```text
forall i in 0..n: p        // 0 <= i < n のすべての i について p
forall x in items: p       // リストのすべての要素について p
exists i in a..=b: p       // a <= i <= b を満たす i が存在して p
forall v: Int: p           // すべての整数 v について p（証明専用）
```

整数の範囲かリストの要素を動く量化子は、**実行できます**。範囲もリストも有限なので、全部の場合を実際に調べられます。型全体を動く量化子（`forall v: Int: ...`）は**証明専用**です。実行時には評価されず、証明できなければ「未検証」と報告されます。黙って成功したことにはなりません。

量化子の本体は、できるだけ右に伸びます。演算子の中に量化子を置くときは、括弧で囲みます。

### spec fn

`spec fn` は、仕様の中だけで呼べる関数です。純粋で、停止し、通常のコードには現れません。繰り返し使う条件に名前を付けるために使います。

```cernith
module guide.specs;

pub spec fn all_positive(items: List<Int>) -> Bool {
  forall i in 0..items.length: items[i] > 0
}

pub fn sum_positive(items: List<Int>) -> Int {
  requires all_positive(items: items);
  ensures result >= 0;

  var total: Int = 0;
  for index in 0..items.length {
    invariant total >= 0;

    total = total + items[index];
  }
  total
}

example sum_positive_adds for sum_positive {
  sum_positive(items: [1, 2, 3]) == 6
}
```

```cernith-error E0331
module guide.specs;

pub spec fn is_positive(value: Int) -> Bool {
  value > 0
}

pub fn check(value: Int) -> Bool {
  is_positive(value: value)
}
```

```cernith-error E0332
module guide.specs;

pub fn all_positive(items: List<Int>) -> Bool {
  forall i in 0..items.length: items[i] > 0
}
```

```cernith-error E0402
module guide.specs;

import std.clock;

pub fn after_epoch(value: Int) -> Int {
  requires clock.now_ms() > 0;

  value
}
```

## 契約が表す条件の書き方

事後条件に、`match` や `implies` を使えます。

```cernith
module guide.postconditions;

pub fn first_or_none(items: List<Int>) -> Option<Int> {
  ensures match result {
    None => items.length == 0,
    Some(value: _) => items.length > 0,
  };

  if items.length == 0 {
    None
  } else {
    Some(value: items[0])
  }
}

example first_or_none_reports_emptiness for first_or_none {
  first_or_none(items: []) == None
    and first_or_none(items: [4]) == Some(value: 4)
}

pub fn pass_through(value: Int) -> Int {
  ensures value > 0 implies result > 0;

  value
}

example pass_through_keeps_the_value for pass_through {
  pass_through(value: 3) == 3
}
```

`first_or_none` の事後条件は、`None` が返るのは入力が空のときだけ、という関数の性質を、型が表せる範囲を超えて述べています。

## 実行時の契約

契約は、処理系が実行時に検査することがあります。

- チェッカーが**証明した**条件は、実行時に検査しません。
- 証明できず、実行できる条件は、実行時に検査されます。違反すると欠陥になり、プログラムはその報告とともに停止します。
- **証明専用**の条件（型全体を動く量化子）は、実行時に検査されません。
- `cernith test` の実行では、すべての実行可能な条件が検査されます。

契約の検査は、プログラムの結果を変えません。検査は、通るか、停止するかのどちらかです。

## 関連

- 証明義務の種類と保証レベル：[10 証明義務と保証レベル](10-verification.md)
- 欠陥：[12 エラーと欠陥](12-errors-and-defects.md)
- 型に付ける条件（`where`）：[08 制約付き型と distinct 型](08-refined-and-distinct-types.md)
