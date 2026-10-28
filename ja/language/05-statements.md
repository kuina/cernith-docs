# 文と束縛

関数やループの本体はブロックです。ブロックには 0 個以上の文を書き、必要に応じて最後に式を置きます。末尾に `;` を付けない最後の式が、ブロックの値になります。最後の式を省略すると、ブロックの値は `()`（型は `Unit`）になります。

## 文の種類

| 文 | 形 | 意味 |
| --- | --- | --- |
| `let` | `let 名前 = 式;` | 変更できない名前を束縛する。型は式の型になり、書けない |
| `var` | `var 名前: 型 = 式;` | 変更できる局所変数を宣言する。型が必須 |
| 代入 | `名前 = 式;` | `var` の値を置き換える |
| 式文 | `f(...);` | 呼び出しを実行する。結果は `Unit` でなければならない |
| `discard` | `discard 式;` | 値を捨てることを明示する |
| `return` | `return 式;` | 関数をすぐに終わらせ、値を返す |
| `assert` | `assert 式;` | 条件が成り立つことを主張する |
| `if` / `match` / `for` / `while` | | 制御構造。文として書くときは `;` を付けない |

文は、`;` で終わります。`if`、`match`、`for`、`while` は例外で、ブロックを持つ文には `;` を付けません。

## let と var

`let` は変更できない束縛を作ります。型は右辺から推論されるため、型注釈を付けるとエラー `E0323` になります。`var` は変更できる局所変数を作り、型注釈を省略するとエラー `E0311` になります。値を変更するには `名前 = 式;` と書きます。`let` や引数への代入はエラー `E0321` です。

```cernith
module guide.bindings;

pub fn sum(items: List<Int>) -> Int {
  var total: Int = 0;
  for item in items {
    total = total + item;
  }
  total
}

example sum_adds_up for sum {
  sum(items: [1, 2, 3]) == 6 and sum(items: []) == 0
}
```

`var` は、宣言した関数内でのみ使える局所変数です。変数自体をほかの関数に渡すことはできず、渡されるのはその値です。そのため、変数の書き換えが関数の外から観測されることはありません。関数式（ラムダ）からも、外側の関数の `var` を読み書きすることはできません（`E0322`）。

```cernith-error E0323
module guide.bindings;

pub fn f() -> Int {
  let a: Int = 1;
  a
}
```

```cernith-error E0311
module guide.bindings;

pub fn f() -> Int {
  var a = 1;
  a
}
```

```cernith-error E0321
module guide.bindings;

pub fn f() -> Int {
  let a = 1;
  a = 2;
  a
}
```

```cernith-error E0322
module guide.bindings;

pub fn f() -> Int {
  var n: Int = 0;
  let get = fn() -> Int {
    n + 1
  };
  get()
}
```

> 補足：`let` に型を書けないのは、型の情報が 2 か所に分かれないようにするためです。型を明示したいときは、型を書く `var` で宣言するか、右辺を型が明らかな式にします。

## 名前の規則

関数内の名前をあいまいにしないために、次の 3 つの規則があります。

**1 つの関数の中で、すでにスコープにある名前を、もう一度束縛することはできません。** `let`、`var`、パラメータ、パターンで導入する名前が、すでに束縛されている名前と同じなら、`E0203`（shadowing）です。外側の名前を隠す（シャドーイング）ことはできません。これにより、同じ名前が関数の途中で別のものを指すことがありません。スコープが重ならない名前、たとえば続けて書いた 2 つの `for` のループ変数や、`if` の 2 つの分岐の中の名前は、同じでも構いません。

**束縛した名前は、1 回以上読まなければなりません。** 使わない `let`、`var`、パターンの名前は `E0204`（unused-binding）です。直し方は、束縛を消す、`discard` で結果だけを捨てる、パターンやループ変数の名前を `_` にする、のいずれかです。パラメータだけは、使わなくても構いません。

**パラメータや変数には、読み込んだモジュールの別名と同じ名前を付けられません。** `import std.int;` があるとき、`int` という変数は宣言できません（`E0202`）。この場合は、`import std.int as int_lib;` のように、モジュールの別名を変えてください。

```cernith-error E0203
module guide.names;

pub fn f(a: Int) -> Int {
  let a = 2;
  a
}
```

```cernith-error E0204
module guide.names;

pub fn f() -> Int {
  let unused = 1;
  2
}
```

```cernith-error E0202
module guide.names;

import std.int;

pub fn f(int: Int) -> Int {
  int
}
```

## 式文と discard

式文は `f(...);` の形で関数を呼び出します。結果が `Unit` でない場合、その値を暗黙に捨てることはできず、エラー `E0306`（unused-result）になります。意図的に結果を使わない場合は、`discard` を付けます。

```cernith
module guide.discards;

import std.io;

pub fn log_and_double(value: Int) -> Int uses io {
  discard io.write_line(text: "doubling");
  value * 2
}
```

`io.write_line` は `Result` を返します。`discard` を付けずに文として書くと、失敗を見落としたまま進むことになるので、拒否されます。

```cernith-error E0306
module guide.discards;

import std.io;

pub fn log_and_double(value: Int) -> Int uses io {
  io.write_line(text: "doubling");
  value * 2
}
```

`let` の名前に `_` は使えません。値を使わずに結果を受けるときは、`discard` を使います。

## return

`return 式;` は、関数をすぐに終了して、その値を返します。ブロックの途中、特にループの中から抜けるときに使います。**関数本体の最後の文には書けません**（`E0312`）。その場合は、値を末尾の式にします。

```cernith
module guide.returns;

pub fn first_negative(items: List<Int>) -> Option<Int> {
  for item in items {
    if item < 0 {
      return Some(value: item);
    }
  }
  None
}

example finds_the_first for first_negative {
  first_negative(items: [3, -1, -2]) == Some(value: -1)
    and first_negative(items: [3, 4]) == None
}
```

```cernith-error E0312
module guide.returns;

pub fn one() -> Int {
  return 1;
}
```

## assert

`assert 式;` は、その地点で式が成り立つことを主張します。成り立たなければ欠陥です。検証では、`assert` の後ろの文は、式が成り立つものとして調べられます（[10 証明義務と保証レベル](10-verification.md)）。

## ループ

### for

```text
for 名前 in リスト { ... }
for i in a..b { ... }       // a 以上 b 未満
for i in a..=b { ... }      // a 以上 b 以下
```

`for` は、リストの要素や `Int` の範囲を順にたどり、各要素を名前に束縛します。`1..0` のように終端が始端より後にない範囲は空です。名前には `_` も使えます。レコードパターン `Point(x: a, y: b)` のように必ず一致するパターンも使えますが、一致しない可能性のあるパターンはエラー `E0305` になります。リストと範囲は有限なので、それらをたどる `for` に停止性の根拠を明示する必要はありません。`Map` をたどるには、`std.map` の `entries` が返すリストを使います。`String` は直接たどれないため、`text.chars` で `List<String>` に変換します。

使わない要素は、名前を `_` にします。

`std.seq.Seq` もたどれます（[std.seq](../stdlib/seq.md)）。`Seq` は遅延評価される列で、長さの上限 `bound` を持ちます。ループは各回で列を 1 ステップ進め、最大で `bound` 回繰り返します。この上限によって停止性が保証されるので、明示的な尺度は不要です。途中で `return` すると、残りの要素は計算されません。

```text
for 名前 in seq { ... }     // 最大で seq.bound 回
```

`Seq` に対する `for` は、`invariant` 節も `decreases` 節も取れません（`E1406`）。本体からは隠れたカーソルの名前を書けず、`bound` が尺度になるためです。節が必要なループは、リストか範囲に対して書きます。

```cernith
module guide.seq_loops;

import std.seq;

pub fn first_above(limit: Int) -> Option<Int> {
  for number in seq.iterate(
    seed: 1,
    next: fn(value: Int) -> Int {
      value * 2
    },
    bound: 1000,
  ) {
    if number > limit {
      return Some(value: number);
    }
  }
  None
}

example first_above_stops_early for first_above {
  first_above(limit: 100) == Some(value: 128)
}

pub fn sum_of_squares(count: Int) -> Int {
  var total: Int = 0;
  for number in seq.range(from: 1, to: count + 1) {
    total = total + number * number;
  }
  total
}

example sum_of_squares_adds_up for sum_of_squares {
  sum_of_squares(count: 3) == 14 and sum_of_squares(count: 0) == 0
}
```

```cernith-error E1406
module guide.seq_loops;

import std.seq;

pub fn total() -> Int {
  var total: Int = 0;
  for number in seq.range(from: 0, to: 3) {
    invariant total >= 0;

    total = total + number;
  }
  total
}
```

### while

```text
while 条件 {
  invariant 条件;     // 省略可
  decreases 式;       // 必須
  ...
}
```

`while` には、`decreases` 節が必要です。ループを繰り返すたびに小さくなる、0 以上の `Int` の式を書き、ループが止まる根拠を示します。`invariant` は、各回の前後で成り立つ条件で、省略できます（[13 停止性](13-termination.md)、[09 契約](09-contracts.md)）。節は、ブロックの先頭に書きます。

```cernith
module guide.loops;

pub fn count_items(items: List<Int>) -> Int {
  var count: Int = 0;
  for _ in items {
    count = count + 1;
  }
  count
}

example count_items_counts for count_items {
  count_items(items: [5, 6, 7]) == 3
}

pub fn sum_below(n: Int) -> Int {
  requires n >= 0;
  ensures result >= 0;

  var total: Int = 0;
  var i: Int = 0;
  while i < n {
    invariant total >= 0 and i >= 0;
    decreases n - i;

    total = total + i;
    i = i + 1;
  }
  total
}

example sum_below_adds_the_numbers_below for sum_below {
  sum_below(n: 4) == 6
}
```

```cernith-error E0601
module guide.loops;

pub fn countdown(n: Int) -> Int {
  var i: Int = n;
  while i > 0 {
    i = i - 1;
  }
  i
}
```

ループに `break` や `continue` はありません。途中で抜けるには、関数の中で `return` を使います。

## if と match を文として使う

`if` は、`else` なしで文として使えます。このとき型は `Unit` で、本体の値は使えません。

```cernith
module guide.conditionals;

pub fn bump_if(flag: Bool, start: Int) -> Int {
  var n: Int = start;
  if flag {
    n = n + 1;
  }
  n
}

example bump_only_when_flagged for bump_if {
  bump_if(flag: true, start: 1) == 2 and bump_if(flag: false, start: 1) == 1
}
```

`if` や `match` を式として使う規則は、[04 式](04-expressions.md) にあります。

## 関連

- 関数の定義：[06 関数](06-functions.md)
- ループの停止性：[13 停止性](13-termination.md)
