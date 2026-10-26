# 停止性

**効果 `diverge` を宣言しない関数は、必ず停止しなければなりません。** チェッカーは、停止することを次の規則で確認し、確認できなければプログラムを拒否します。`uses diverge` を宣言した関数は、停止について何も約束せず、それを呼ぶ関数も `diverge` を宣言しなければなりません（効果の規則。[11 効果](11-effects.md)）。この連鎖は `main` まで続きます。

ループも再帰もない関数は、自明に停止します（呼び出す関数が停止するなら）。停止の根拠が必要なのは、次の 3 つだけです。

## ループ

- **`for`** は、有限で変更できないリストか範囲をたどるので、必ず停止します。根拠は要りません。
- **`while`** には、`decreases` 節が必要です。ループに入るとき、および 1 回繰り返すたびに 0 以上で、繰り返すたびに厳密に小さくなる `Int` の式を書きます。

```text
while low < high {
  invariant 0 <= low and low <= high;
  decreases high - low;
  ...
}
```

根拠がないと `E0601`（termination-unproved）です。減少量が実際には減らないと、`cernith check --verify` が `E0509`（measure-not-decreasing）を報告し、反例の入力を示します。

```cernith-error E0601
module guide.termination;

pub fn countdown(n: Int) -> Int {
  var i: Int = n;
  while i > 0 {
    i = i - 1;
  }
  i
}
```

`E0601` のメッセージは、`decreases` 節の候補を示します（`while a < b` なら `decreases b - a;`）。候補は提案であり、勝手に適用されません。減少量が自動で推論されることはありません。

## 再帰

自分自身を呼ぶ関数、または互いに呼び合う関数のグループは、次のどれかが必要です。

1. **構造再帰**（単独の関数だけ）。パラメータの 1 つを `match` し、再帰呼び出しがそのパラメータの位置に、その `match` のパターンで束縛された変数を渡す。呼び出しの引数は、元の値の真の部分になります。`list.drop` のようなライブラリ関数で作った部分は数えません。パターンで得た部分でなければなりません。
2. **`decreases`**。グループのすべての関数に `decreases M;` を書く。グループ内の関数の呼び出しごとに、呼ばれる側の減少量は 0 以上で、呼ぶ側が入口で持っていた減少量より小さくなければなりません。`decreases` 節を複数書くと、書いた順の**辞書式順序**の減少量になります（グループ内の関数は、同じ個数の成分を持たなければなりません）。
3. **`uses diverge`**。

互いに呼び合う関数には、構造再帰の形がありません。グループのすべての関数が減少量を持つ必要があります。

```cernith
module guide.termination;

pub variant Chain {
  Link(size: Int, rest: Chain),
  End,
}

pub fn countdown(outer: Int, inner: Int) -> Int {
  requires outer >= 0 and inner >= 0;
  decreases outer;
  decreases inner;

  if inner > 0 {
    countdown(outer: outer, inner: inner - 1)
  } else if outer > 0 {
    countdown(outer: outer - 1, inner: 5)
  } else {
    0
  }
}

example countdown_reaches_zero for countdown {
  countdown(outer: 2, inner: 3) == 0
}

pub fn factorial(n: Int) -> Int {
  requires n >= 0;
  decreases n;

  if n == 0 {
    1
  } else {
    n * factorial(n: n - 1)
  }
}

example factorial_of_5 for factorial {
  factorial(n: 5) == 120
}

pub fn is_even(n: Int) -> Bool {
  requires n >= 0;
  decreases n;

  if n == 0 {
    true
  } else {
    is_odd(n: n - 1)
  }
}

example parity for is_even {
  is_even(n: 4) and not is_even(n: 3) and is_odd(n: 3)
}

pub fn is_odd(n: Int) -> Bool {
  requires n >= 0;
  decreases n;

  if n == 0 {
    false
  } else {
    is_even(n: n - 1)
  }
}

pub fn total_size(chain: Chain) -> Int {
  match chain {
    Chain.End => 0,
    Chain.Link(size: size, rest: rest) => size + total_size(chain: rest),
  }
}

example total_size_adds_the_links for total_size {
  total_size(
    chain: Chain.Link(size: 2, rest: Chain.Link(size: 3, rest: Chain.End)),
  )
    == 5
}
```

`total_size` は構造再帰です。`match` で得た `rest` を渡しているので、`decreases` は要りません。`factorial` は、呼び出しのたびに `n` が 1 小さくなります。`is_even` と `is_odd` は、互いに呼び合うので、両方に減少量があります。`countdown` は、`outer` が減るか、同じ `outer` で `inner` が減るかのどちらかなので、辞書式の減少量です。

減少量が減らない再帰は、`E0509` です（`cernith check --verify` で報告されます）。

```cernith-error E0509
module guide.termination;

pub fn grow(n: Int) -> Int {
  requires n >= 0;
  decreases n;

  if n <= 0 {
    0
  } else {
    grow(n: n + 1)
  }
}
```

減少量のない再帰は、`E0601` です。

```cernith-error E0601
module guide.termination;

pub fn count_down(n: Int) -> Int {
  if n <= 0 {
    0
  } else {
    count_down(n: n - 1)
  }
}
```

## 再帰が呼び出しグラフから隠れないこと

チェッカーは、関数本体の呼び出しから呼び出しグラフを作り（ラムダの中の呼び出しは、それを囲む関数のものとして数えます）、相互に呼び合う強連結成分を見つけます。この分析が正しいのは、再帰がグラフから隠れられない場合だけで、次の 2 つの規則がそれを保証します。

- 関数式（ラムダ）は、束縛される名前を自分の本体の中で参照できません。再帰できるのは名前の付いた関数だけで、その呼び出しは呼び出しグラフの辺になります。
- **再帰的なレコードやバリアントは、自分の再帰グループの値を引数に取る関数を、フィールドに持てません**（`E0602`）。持てると、`record W { call: fn(w: W) -> Int }` のように、呼び出しグラフに辺がない再帰（不動点コンビネータ）が書けてしまいます。自分の型を引数に取らない関数は、持てます。

```cernith-error E0602
module guide.termination;

pub record Wrapper {
  call: fn(w: Wrapper) -> Int,
}
```

## 終わらない処理

入力が尽きるまで読み続けるようなループには、減少量がありません。関数に `uses diverge` を宣言します。ただし、有限のリストに読み込んで `for` で回すほうが、停止性を保てます。`main` が `diverge` を宣言することはできます。

停止性は、実行時間については何も言いません。停止する関数でも、実行ステップの上限を超えて止められることはありえます（[12 エラーと欠陥](12-errors-and-defects.md)）。

```cernith
module guide.diverging;

import std.int;

pub fn collatz_steps(n: Int) -> Int uses diverge {
  requires n >= 1;

  if n == 1 {
    0
  } else if int.rem_euclid(dividend: n, divisor: 2) == 0 {
    1 + collatz_steps(n: int.div_floor(dividend: n, divisor: 2))
  } else {
    1 + collatz_steps(n: 3 * n + 1)
  }
}

example collatz_of_6 for collatz_steps {
  collatz_steps(n: 6) == 8
}
```

`collatz_steps` を呼ぶ関数は、`uses diverge` を宣言しなければなりません（`E0401`）。

## 限界

減少量は `Int` の式で、辞書式で組み合わせられるのが最大です。ほかの整礎な順序はありません。停止の根拠にそれが必要な関数（一部のグラフアルゴリズムなど）は、`diverge` を宣言します。

## 診断コード

| コード | 名前 | 意味 |
| --- | --- | --- |
| `E0601` | termination-unproved | 根拠のない `while` ループ、または再帰のグループ |
| `E0602` | non-positive-recursive-type | 再帰的な型が、自分の再帰グループの値を取る関数を持つ |
| `E0509` | measure-not-decreasing | 減少量が小さくならない（反例の入力つき） |
| `W0504` | obligation-unproved | ソルバーが決められなかった `term` の義務 |

## 関連

- 契約の節：[09 契約](09-contracts.md)
- 証明義務 `term`：[10 証明義務と保証レベル](10-verification.md)
