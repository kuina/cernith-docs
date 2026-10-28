# std.list

不変の列 `List<T>` を扱うモジュールです。各関数は、元のリストを変更せず、新しいリストや計算結果を返します。`import std.list;` で読み込み、`list.map(items: ..., apply: ...)` のように呼び出します。引数はすべてラベル付きです。リストを受け取るラベルは基本的に `items` ですが、`flatten` は `lists`、`zip` は `first` と `second` を使います（[06 関数](../language/06-functions.md)）。

リストの長さと添字（`items.length`、`items[i]`）は言語の組み込み機能です（[03 型](../language/03-types.md)）。このページでは、それ以外の操作を扱います。また、このモジュールで定義されるレコード型 `Pair<A, B>` は、[list.zip](#listzip-と-listpair) で使います。

## 関数一覧

| 関数 | 戻り値の型 | 意味 | 事前条件 |
| --- | --- | --- | --- |
| `list.all(items, test)` | `Bool` | すべての要素で `test` が `true`（空なら `true`） | なし |
| `list.any(items, test)` | `Bool` | `test` が `true` の要素がある（空なら `false`） | なし |
| `list.contains(items, item)` | `Bool` | `item` と等しい要素がある | なし |
| `list.count(items, test)` | `Int` | `test` が `true` の要素の個数 | なし |
| `list.drop(items, count)` | `List<T>` | 先頭の `count` 個を除いた残り | `count >= 0 and count <= items.length` |
| `list.drop_while(items, test)` | `List<T>` | 先頭から `test` が `true` の間の要素を除いた残り | なし |
| `list.filter(items, keep)` | `List<T>` | `keep` が `true` の要素だけ（順序は保つ） | なし |
| `list.find(items, test)` | `Option<T>` | `test` が `true` の最初の要素 | なし |
| `list.first(items)` | `Option<T>` | 先頭の要素 | なし |
| `list.flat_map(items, apply)` | `List<B>` | 各要素に、リストを返す `apply` を適用して連結 | なし |
| `list.flatten(lists)` | `List<T>` | リストのリストを連結 | なし |
| `list.fold(items, initial, combine)` | `A` | 先頭から順に畳み込む | なし |
| `list.index_of(items, item)` | `Option<Int>` | `item` と等しい最初の要素の位置 | なし |
| `list.last(items)` | `Option<T>` | 最後の要素 | なし |
| `list.map(items, apply)` | `List<B>` | 各要素に `apply` を適用 | なし |
| `list.max(items)` | `Option<T>` | 最大の要素 | なし |
| `list.min(items)` | `Option<T>` | 最小の要素 | なし |
| `list.range(from, to)` | `List<Int>` | `from` 以上 `to` 未満の整数 | `from <= to` |
| `list.repeat(item, count)` | `List<T>` | `item` を `count` 個並べたリスト | `count >= 0` |
| `list.reverse(items)` | `List<T>` | 逆順 | なし |
| `list.slice(items, from, to)` | `List<T>` | 位置 `from` 以上 `to` 未満の部分 | `0 <= from and from <= to and to <= items.length` |
| `list.sort(items)` | `List<T>` | 昇順に並べ替え | なし |
| `list.sort_by(items, less_than)` | `List<T>` | `less_than` の順に並べ替え（安定） | なし |
| `list.sum(items)` | `Int` | 合計（空なら `0`） | なし |
| `list.take(items, count)` | `List<T>` | 先頭の `count` 個 | `count >= 0 and count <= items.length` |
| `list.take_while(items, test)` | `List<T>` | 先頭から `test` が `true` の間の要素 | なし |
| `list.unique(items)` | `List<T>` | 重複を除く（最初に現れたものを残す） | なし |
| `list.windows(items, size)` | `List<List<T>>` | 長さ `size` の連続する部分の列 | `size >= 1` |
| `list.zip(first, second)` | `List<Pair<A, B>>` | 同じ位置の要素を組にする | なし |

## 共通の規則

### 関数を受け取る引数と効果

`test`、`keep`、`apply`、`combine` などのコールバックを受け取る関数は、効果多相です。純粋な関数を渡せば呼び出しも純粋になり、`uses io` の関数を渡せば呼び出し側にも `uses io` が必要です（[11 効果](../language/11-effects.md)）。シグネチャでは `effect E` と `uses E` で表します。

例外は `list.sort_by` の `less_than` で、効果を持てません。効果のある関数を渡すと、エラー `E0404` になります。

関数を呼ぶ順序は、リストの先頭から末尾です。`list.all`、`list.any`、`list.find`、`list.take_while`、`list.drop_while` は、結果が決まった要素で止まり、残りの要素には関数を呼びません。

### 等価比較と順序の境界

`list.contains`、`list.index_of`、`list.unique` は、要素の型に境界 `Eq` を要求します（`==` で比べるため）。`list.max`、`list.min`、`list.sort` は境界 `Ord` を要求します（`<` で比べるため）。どの型が満たすかは [03 型](../language/03-types.md) にあります。

### 事前条件を満たさない呼び出しは欠陥

`list.take`、`list.drop`、`list.slice`、`list.range`、`list.repeat`、`list.windows` には事前条件（`requires`）があります。事前条件を満たさない呼び出しは**欠陥**で、プログラムはそこで止まります。欠陥はプログラム内では捕捉できません（[12 エラーと欠陥](../language/12-errors-and-defects.md)）。

テストでは、`defects(of: ...)` で、式が欠陥になることを確かめられます（[16 テスト](../language/16-testing.md)）。このページの例は、事前条件のある関数のすべてで、これを使っています。

事前条件に違反する入力をチェッカーが見つけた呼び出しは、`cernith check --verify` でエラー `E0501`（precondition-violated）として報告されます。

```cernith-error E0501
module guide.list_precondition;

import std.list;

pub fn preview(items: List<Int>) -> List<Int> {
  list.take(items: items, count: 3)
}
```

`items` が 3 個より短いと、`list.take` の事前条件に違反します。診断には、そうなる入力（反例の入力）として `items=[]` が示されます。呼び出し側の `requires` で同じ条件を要求すれば、受理されます（`guide.list_slicing` の `preview`）。事前条件の書き方は [09 契約](../language/09-contracts.md)、証明の扱いは [10 検証](../language/10-verification.md) にあります。

> 補足：`list.take(items, count)` は、範囲外の `count` を自動で切り詰めません。事後条件で `result.length == count` を保証するためです。指定した個数を返せない場合は、短いリストを返す代わりに欠陥になります。

## 関数の詳細

### 作る

#### list.range

```text
pub fn range(from: Int, to: Int) -> List<Int>
```

`from` 以上 `to` 未満の整数を、昇順に並べたリストです。`to` は含みません。`list.range(from: 2, to: 5)` は `[2, 3, 4]`、`list.range(from: -1, to: 1)` は `[-1, 0]` です。`from` と `to` が等しければ、`list.range(from: 3, to: 3)` は `[]` です。

- 事前条件：`from <= to`。`from` が `to` より大きい呼び出し（`list.range(from: 3, to: 2)`）は欠陥です。
- 事後条件：`result.length == to - from`。

#### list.repeat

```text
pub fn repeat<T>(item: T, count: Int) -> List<T>
```

`item` を `count` 個並べたリストです。`list.repeat(item: "a", count: 3)` は `["a", "a", "a"]`、`count` が `0` なら `[]` です。

- 事前条件：`count >= 0`。負の `count`（`list.repeat(item: "a", count: -1)`）は欠陥です。
- 事後条件：`result.length == count`。

### 取り出す・切り出す

#### list.first と list.last

```text
pub fn first<T>(items: List<T>) -> Option<T>
pub fn last<T>(items: List<T>) -> Option<T>
```

先頭の要素、最後の要素を返します。`list.first(items: [7, 8])` は `Some(value: 7)`、`list.last(items: [7, 8])` は `Some(value: 8)` です。空のリストには要素がないので、どちらも `None` を返します。

- 事前条件：なし。
- 事後条件：結果が `None` であることと、`items.length == 0` であることは同値です。

空のリストのリテラル `[]` だけでは `T` が決まらないので、`list.first::<Int>(items: [])` のように型引数を書きます（[06 関数](../language/06-functions.md)）。

#### list.take と list.drop

```text
pub fn take<T>(items: List<T>, count: Int) -> List<T>
pub fn drop<T>(items: List<T>, count: Int) -> List<T>
```

`list.take` は先頭の `count` 個、`list.drop` はそれらを除いた残りを返します。同じ `count` で `take` と `drop` を `++` でつなぐと、元のリストに戻ります。

- `list.take(items: [1, 2, 3], count: 2)` は `[1, 2]`、`count` が `0` なら `[]`、`count` が長さに等しければ全体です。
- `list.drop(items: [1, 2, 3], count: 1)` は `[2, 3]`、`count` が長さに等しければ `[]`、`count` が `0` なら全体です。

事前条件：`count >= 0 and count <= items.length`。長さを超える `count` や負の `count` は欠陥です。たとえば `list.take(items: [1, 2], count: 3)` や `list.drop(items: [1, 2], count: 3)` は欠陥になります。

事後条件：`take` は `result.length == count`、`drop` は `result.length == items.length - count`。

#### list.slice

```text
pub fn slice<T>(items: List<T>, from: Int, to: Int) -> List<T>
```

位置 `from` 以上 `to` 未満の部分を返します。位置は 0 から数えます。`list.slice(items: [1, 2, 3, 4], from: 1, to: 3)` は `[2, 3]` で、`from` と `to` が等しければ、`list.slice(items: [1, 2], from: 2, to: 2)` のように `[]` です。

- 事前条件：`0 <= from and from <= to and to <= items.length`。`list.slice(items: [1, 2, 3], from: 2, to: 1)`（`from > to`）、`list.slice(items: [1, 2, 3], from: 1, to: 4)`（`to` が長さ超過）、`list.slice(items: [1, 2, 3], from: -1, to: 2)`（`from` が負）は、どれも欠陥です。
- 事後条件：`result.length == to - from`。

#### list.take_while と list.drop_while

```text
pub fn take_while<T, effect E>(
  items: List<T>,
  test: fn(item: T) -> Bool uses E,
) -> List<T> uses E

pub fn drop_while<T, effect E>(
  items: List<T>,
  test: fn(item: T) -> Bool uses E,
) -> List<T> uses E
```

先頭から、`test` が `true` を返す間だけ進みます。`list.take_while` はその部分を、`list.drop_while` は残りを返します。`test` が一度 `false` を返したら、後ろの要素が `test` を満たしていても、そこで止まります。`item < 3` を `test` にすると、`[1, 2, 5, 1]` に対して `list.take_while` は `[1, 2]`、`list.drop_while` は `[5, 1]` です。すべての要素が満たせば `list.take_while` は全体、`list.drop_while` は `[]` になります。

- 事前条件：なし。
- 事後条件：`result.length <= items.length`。

#### list.windows

```text
pub fn windows<T>(items: List<T>, size: Int) -> List<List<T>>
```

長さ `size` の連続する部分を、先頭から 1 つずつずらして並べます。`list.windows(items: [1, 2, 3, 4], size: 2)` は `[[1, 2], [2, 3], [3, 4]]`、`size` が `1` なら `list.windows(items: [1, 2], size: 1)` は `[[1], [2]]` です。`size` がリストの長さに等しければ、全体 1 つだけの列です。`size` がリストより長いと、`list.windows(items: [1, 2], size: 3)` は `[]` を返します。

- 事前条件：`size >= 1`。`size` が `0` や負の呼び出しは欠陥です。
- 事後条件：なし。結果の長さは、`items.length - size + 1` です（これが負になるときは `0`）。

### 探す・調べる

#### list.all と list.any

```text
pub fn all<T, effect E>(
  items: List<T>,
  test: fn(item: T) -> Bool uses E,
) -> Bool uses E

pub fn any<T, effect E>(
  items: List<T>,
  test: fn(item: T) -> Bool uses E,
) -> Bool uses E
```

`list.all` は、すべての要素で `test` が `true` のとき `true` です。`list.any` は、`test` が `true` の要素が 1 つでもあれば `true` です。空のリストでは、`test` が何であっても、`list.all` は `true`、`list.any` は `false` です。

`list.all` は最初に `false` になった要素で、`list.any` は最初に `true` になった要素で止まり、残りの要素に `test` を呼びません。後ろの要素で欠陥になる `test`（たとえば `0` による割り算）でも、その前に結果が決まれば、欠陥にはなりません。

- 事前条件：なし。

#### list.contains と list.index_of

```text
pub fn contains<T: Eq>(items: List<T>, item: T) -> Bool
pub fn index_of<T: Eq>(items: List<T>, item: T) -> Option<Int>
```

`item` と等しい要素があるか（`list.contains`）、あれば最初の位置はどこか（`list.index_of`）を返します。位置は 0 から数えます。`list.index_of(items: [4, 5, 4], item: 4)` は `Some(value: 0)` で、見つからなければ `None` です。

- 事前条件：なし。
- 事後条件（`index_of`）：`Some(value: position)` の `position` は `0` 以上、`items.length` 未満。

#### list.find

```text
pub fn find<T, effect E>(
  items: List<T>,
  test: fn(item: T) -> Bool uses E,
) -> Option<T> uses E
```

`test` が `true` を返す最初の要素を返します。`item > 3` を `test` にすると、`[1, 5, 7]` からは `Some(value: 5)` が、`[1]` からは `None` が返ります。

- 事前条件：なし。

#### list.count

```text
pub fn count<T, effect E>(
  items: List<T>,
  test: fn(item: T) -> Bool uses E,
) -> Int uses E
```

`test` が `true` を返す要素の個数です。すべての要素に `test` を呼びます。`item > 3` を `test` にすると、`[1, 5, 7, 2]` では `2` です。

- 事前条件：なし。
- 事後条件：`result >= 0 and result <= items.length`。

#### list.max と list.min

```text
pub fn max<T: Ord>(items: List<T>) -> Option<T>
pub fn min<T: Ord>(items: List<T>) -> Option<T>
```

最大、最小の要素を返します。`list.max(items: [3, 9, 4])` は `Some(value: 9)`、`list.min(items: [3, 9, 4])` は `Some(value: 3)` です。`String` は、コードポイントの辞書式順序で比べます。空のリストには最大も最小もないので、`None` を返します。

- 事前条件：なし。
- 事後条件：結果が `None` であることと、`items.length == 0` であることは同値です。

### 変換する・畳み込む

#### list.map

```text
pub fn map<A, B, effect E>(
  items: List<A>,
  apply: fn(item: A) -> B uses E,
) -> List<B> uses E
```

各要素に `apply` を適用した結果を、同じ順序で並べたリストです。`apply` の引数のラベルは `item` で、`B` は `A` と違う型でもかまいません。

- 事前条件：なし。
- 事後条件：`result.length == items.length`。

#### list.filter

```text
pub fn filter<T, effect E>(
  items: List<T>,
  keep: fn(item: T) -> Bool uses E,
) -> List<T> uses E
```

`keep` が `true` を返す要素だけを、元の順序のまま残します。

- 事前条件：なし。
- 事後条件：`result.length <= items.length`。

#### list.flat_map と list.flatten

```text
pub fn flat_map<A, B, effect E>(
  items: List<A>,
  apply: fn(item: A) -> List<B> uses E,
) -> List<B> uses E

pub fn flatten<T>(lists: List<List<T>>) -> List<T>
```

`list.flatten` は、リストのリストを、順序を保ってつなげます。`list.flatten(lists: [[1, 2], [], [3]])` は `[1, 2, 3]` です。`list.flat_map` は、各要素に（リストを返す）`apply` を適用して、結果を `list.flatten` と同じようにつなげます。`list.map` を使うと `List<List<B>>` になる場面で、入れ子にならない `List<B>` が得られます。`apply` が `[]` を返した要素は、結果から消えます。

- 事前条件：なし。

#### list.fold

```text
pub fn fold<T, A, effect E>(
  items: List<T>,
  initial: A,
  combine: fn(total: A, item: T) -> A uses E,
) -> A uses E
```

`initial` から始めて、先頭の要素から順に `combine(total: ..., item: ...)` を適用し、最後の `total` を返します。左から右へ畳み込むので、`combine` が `total - item` のとき、`list.fold(items: [1, 2, 3], initial: 10, combine: ...)` は `((10 - 1) - 2) - 3` で `4` です。空のリストなら `initial` がそのまま返ります。`total` の型 `A` は、要素の型 `T` と違ってもかまいません。

- 事前条件：なし。

`list.fold` は逐次に畳み込みます。結合的な `combine` を並列に畳み込む関数は、`parallel.fold` です（[parallel](parallel.md)）。

#### list.sum

```text
pub fn sum(items: List<Int>) -> Int
```

`Int` のリストの合計です。空のリストでは `0` です。`Int` は桁あふれしないので、合計も常に正確な値です。

- 事前条件：なし。

#### list.unique

```text
pub fn unique<T: Eq>(items: List<T>) -> List<T>
```

重複を取り除きます。同じ値のうち、最初に現れたものを残し、順序は元のままです。`list.unique(items: [3, 1, 3, 2, 1])` は `[3, 1, 2]` です。

- 事前条件：なし。
- 事後条件：`result.length <= items.length`。

### 並べ替える・組み合わせる

#### list.reverse

```text
pub fn reverse<T>(items: List<T>) -> List<T>
```

逆順のリストです。

- 事前条件：なし。
- 事後条件：`result.length == items.length`。

#### list.sort と list.sort_by

```text
pub fn sort<T: Ord>(items: List<T>) -> List<T>

pub fn sort_by<T>(
  items: List<T>,
  less_than: fn(first: T, second: T) -> Bool,
) -> List<T>
```

`list.sort` は昇順に並べ替えます。`list.sort_by` は、`less_than(first: a, second: b)` が `true` なら `a` を `b` より前に置きます。どちらも安定ソートで、`less_than` が両方向とも `false` になる要素どうしは、元の順序を保ちます。`list.sort` は、比較に `<` を使う `list.sort_by` と同じです。

- 事前条件：なし。
- 事後条件：`result.length == items.length`。`list.sort_by` は加えて、結果の隣り合う要素の組 `result[i]`、`result[i + 1]` のどれも `less_than(first: result[i + 1], second: result[i])` が `false` であること（逆順の組がないこと）を保証します。

`less_than` には、`<` のように、厳密な（等しい要素の間では `false` になる）順序を与えます。`<=` のように等しい要素の間で `true` になる関数や、常に `true` の関数を渡すと、結果が事後条件を満たせず、欠陥になることがあります。たとえば `<=` を `less_than` にして `[3, 1, 3, 2]` を並べ替えると、欠陥です（重複のない `[3, 1, 2]` では欠陥になりません）。

`less_than` に効果のある関数を渡すと、エラー `E0404` です。

```cernith-error E0404
module guide.list_effects;

import std.list;

pub fn logged_sort(items: List<Int>) -> List<Int> uses io {
  list.sort_by(
    items: items,
    less_than: fn(first: Int, second: Int) -> Bool uses io {
      first < second
    },
  )
}
```

#### list.zip と list.Pair

```text
pub record Pair<A, B> {
  first: A,
  second: B,
}

pub fn zip<A, B>(first: List<A>, second: List<B>) -> List<Pair<A, B>>
```

2 つのリストの同じ位置の要素を `Pair` にして並べます。長さが違うときは、短いほうに合わせ、長いほうの余りは捨てます。`list.zip(first: [1, 2, 3], second: ["a", "b"])` は `[list.Pair(first: 1, second: "a"), list.Pair(first: 2, second: "b")]` です。Cernith にはタプルがないので、組は `Pair` のフィールド `first` と `second` で読み出します（[03 型](../language/03-types.md)）。

- 事前条件：なし。
- 事後条件：`result.length <= first.length and result.length <= second.length`。

## 例

### 作る

```cernith
module guide.list_build;

import std.list;

pub fn blank_row(width: Int) -> List<String> {
  requires width >= 0;

  list.repeat(item: " ", count: width)
}

example repeat_copies_the_item for blank_row {
  list.repeat(item: "a", count: 3) == ["a", "a", "a"]
    and blank_row(width: 2) == [" ", " "]
}

example repeat_negative_count_is_a_defect for blank_row {
  defects(of: list.repeat(item: "a", count: -1))
}

example repeat_zero_is_empty for blank_row {
  list.repeat(item: 1, count: 0) == [] and blank_row(width: 0) == []
}

pub fn squares(count: Int) -> List<Int> {
  requires count >= 0;

  list.map(
    items: list.range(from: 0, to: count),
    apply: fn(item: Int) -> Int {
      item * item
    },
  )
}

example range_backwards_is_a_defect for squares {
  defects(of: list.range(from: 3, to: 2))
}

example range_is_half_open for squares {
  list.range(from: 2, to: 5) == [2, 3, 4]
    and list.range(from: -1, to: 1) == [-1, 0]
    and squares(count: 4) == [0, 1, 4, 9]
}

example range_with_equal_ends_is_empty for squares {
  list.range(from: 3, to: 3) == [] and squares(count: 0) == []
}
```

### 取り出す・切り出す

```cernith
module guide.list_slicing;

import std.int;
import std.list;

pub fn latest(items: List<Int>, count: Int) -> List<Int> {
  requires count >= 0 and count <= items.length;

  list.drop(items: items, count: items.length - count)
}

example drop_beyond_the_length_is_a_defect for latest {
  defects(of: list.drop(items: [1, 2], count: 3))
    and defects(of: list.drop(items: [1, 2], count: -1))
}

example drop_removes_the_prefix for latest {
  list.drop(items: [1, 2, 3], count: 1) == [2, 3]
    and list.drop(items: [1, 2, 3], count: 3) == []
    and list.drop(items: [1, 2, 3], count: 0) == [1, 2, 3]
    and latest(items: [1, 2, 3, 4], count: 2) == [3, 4]
}

pub fn leading_small(items: List<Int>) -> List<Int> {
  list.take_while(
    items: items,
    test: fn(item: Int) -> Bool {
      item < 3
    },
  )
}

example take_while_and_drop_while_stop_at_the_first_miss for leading_small {
  list.take_while(
    items: [20, 0],
    test: fn(item: Int) -> Bool {
      int.div_floor(dividend: 10, divisor: item) > 1
    },
  )
    == []
    and list.drop_while(
      items: [20, 0],
      test: fn(item: Int) -> Bool {
        int.div_floor(dividend: 10, divisor: item) > 1
      },
    )
      == [20, 0]
}

example take_while_stops_at_the_first_miss for leading_small {
  leading_small(items: [1, 2, 5, 1]) == [1, 2]
    and leading_small(items: [1, 2]) == [1, 2]
    and leading_small(items: [5, 1]) == []
    and leading_small(items: []) == []
}

pub fn middle(items: List<Int>) -> List<Int> {
  requires items.length >= 2;

  list.slice(items: items, from: 1, to: items.length - 1)
}

example slice_out_of_range_is_a_defect for middle {
  defects(of: list.slice(items: [1, 2, 3], from: 2, to: 1))
    and defects(of: list.slice(items: [1, 2, 3], from: 1, to: 4))
    and defects(of: list.slice(items: [1, 2, 3], from: -1, to: 2))
}

example slice_takes_a_range for middle {
  list.slice(items: [1, 2, 3, 4], from: 1, to: 3) == [2, 3]
    and list.slice(items: [1, 2], from: 2, to: 2) == []
    and middle(items: [1, 2, 3, 4]) == [2, 3]
}

pub fn moving_sums(items: List<Int>, size: Int) -> List<Int> {
  requires size >= 1;

  list.map(
    items: list.windows(items: items, size: size),
    apply: fn(item: List<Int>) -> Int {
      list.sum(items: item)
    },
  )
}

example windows_longer_than_the_list_is_empty for moving_sums {
  list.windows(items: [1, 2], size: 3) == []
    and list.windows::<Int>(items: [], size: 1) == []
}

example windows_of_size_zero_is_a_defect for moving_sums {
  defects(of: list.windows(items: [1, 2], size: 0))
    and defects(of: list.windows(items: [1, 2], size: -1))
}

example windows_slide_over_the_list for moving_sums {
  list.windows(items: [1, 2, 3, 4], size: 2) == [[1, 2], [2, 3], [3, 4]]
    and list.windows(items: [1, 2], size: 1) == [[1], [2]]
    and list.windows(items: [1, 2], size: 2) == [[1, 2]]
    and moving_sums(items: [1, 2, 3, 4], size: 2) == [3, 5, 7]
}

pub fn preview(items: List<Int>, count: Int) -> List<Int> {
  requires count >= 0 and count <= items.length;

  list.take(items: items, count: count)
}

example take_beyond_the_length_is_a_defect for preview {
  defects(of: list.take(items: [1, 2], count: 3))
    and defects(of: list.take(items: [1, 2], count: -1))
}

example take_keeps_the_prefix for preview {
  list.take(items: [1, 2, 3], count: 2) == [1, 2]
    and list.take(items: [1, 2, 3], count: 0) == []
    and list.take(items: [1], count: 1) == [1]
    and preview(items: [1, 2, 3], count: 2) == [1, 2]
    and list.take(items: [1, 2, 3], count: 2)
      ++ list.drop(items: [1, 2, 3], count: 2)
      == [1, 2, 3]
}

pub fn spread(items: List<Int>) -> Int {
  match list.first(items: items) {
    None => 0,
    Some(value: low) => match list.last(items: items) {
      None => 0,
      Some(value: high) => high - low,
    },
  }
}

example first_and_last_of_an_empty_list for spread {
  list.first::<Int>(items: []) == None
    and list.last::<Int>(items: []) == None
    and spread(items: []) == 0
}

example first_and_last_of_lists for spread {
  list.first(items: [7, 8]) == Some(value: 7)
    and list.last(items: [7, 8]) == Some(value: 8)
    and list.first(items: [7]) == Some(value: 7)
    and list.last(items: [7]) == Some(value: 7)
    and spread(items: [1, 5, 9]) == 8
}

pub fn without_leading_zeros(items: List<Int>) -> List<Int> {
  list.drop_while(
    items: items,
    test: fn(item: Int) -> Bool {
      item == 0
    },
  )
}

example drop_while_skips_the_prefix for without_leading_zeros {
  without_leading_zeros(items: [0, 0, 4, 0]) == [4, 0]
    and without_leading_zeros(items: [0, 0]) == []
    and without_leading_zeros(items: []) == []
    and list.drop_while(
      items: [1, 2, 5, 1],
      test: fn(item: Int) -> Bool {
        item < 3
      },
    )
      == [5, 1]
}
```

### 探す・調べる

```cernith
module guide.list_search;

import std.int;
import std.list;

pub fn all_in_range(scores: List<Int>) -> Bool {
  list.all(
    items: scores,
    test: fn(item: Int) -> Bool {
      item >= 0 and item <= 100
    },
  )
}

example all_checks_every_item for all_in_range {
  all_in_range(scores: [2, 4])
    and not all_in_range(scores: [2, 400])
    and all_in_range(scores: [])
}

example all_stops_at_the_first_false for all_in_range {
  list.all(
    items: [20, 0],
    test: fn(item: Int) -> Bool {
      int.div_floor(dividend: 10, divisor: item) > 1
    },
  )
    == false
}

pub fn any_failing(scores: List<Int>) -> Bool {
  list.any(
    items: scores,
    test: fn(item: Int) -> Bool {
      item < 60
    },
  )
}

example any_looks_for_one_item for any_failing {
  any_failing(scores: [80, 50]) and not any_failing(scores: [80, 90])
}

example any_of_an_empty_list_is_false for any_failing {
  not any_failing(scores: [])
}

example any_stops_at_the_first_true for any_failing {
  list.any(
    items: [5, 0],
    test: fn(item: Int) -> Bool {
      int.div_floor(dividend: 10, divisor: item) > 1
    },
  )
}

pub fn best(scores: List<Int>) -> Option<Int> {
  list.max(items: scores)
}

example max_of_an_empty_list_is_none for best {
  best(scores: []) == None
}

example max_of_lists for best {
  best(scores: [3, 9, 4]) == Some(value: 9)
    and list.max(items: ["b", "a"]) == Some(value: "b")
    and best(scores: [1]) == Some(value: 1)
}

pub fn first_failing(scores: List<Int>) -> Option<Int> {
  list.find(
    items: scores,
    test: fn(item: Int) -> Bool {
      item < 60
    },
  )
}

example find_returns_the_first_match for first_failing {
  first_failing(scores: [80, 50, 40]) == Some(value: 50)
    and first_failing(scores: [80]) == None
    and first_failing(scores: []) == None
}

example find_stops_at_the_first_match for first_failing {
  list.find(
    items: [5, 0],
    test: fn(item: Int) -> Bool {
      int.div_floor(dividend: 10, divisor: item) > 1
    },
  )
    == Some(value: 5)
}

pub fn is_enrolled(names: List<String>, name: String) -> Bool {
  list.contains(items: names, item: name)
}

example contains_present_and_absent for is_enrolled {
  is_enrolled(names: ["ann", "bob"], name: "bob")
    and not is_enrolled(names: ["ann", "bob"], name: "cy")
    and not is_enrolled(names: [], name: "ann")
}

pub fn passing_count(scores: List<Int>) -> Int {
  list.count(
    items: scores,
    test: fn(item: Int) -> Bool {
      item >= 60
    },
  )
}

example count_counts_the_matches for passing_count {
  passing_count(scores: [80, 50, 60, 10]) == 2
    and passing_count(scores: [10]) == 0
    and passing_count(scores: []) == 0
}

pub fn seat_of(names: List<String>, name: String) -> Option<Int> {
  list.index_of(items: names, item: name)
}

example index_of_first_occurrence for seat_of {
  seat_of(names: ["ann", "bob", "ann"], name: "ann") == Some(value: 0)
    and seat_of(names: ["ann", "bob"], name: "bob") == Some(value: 1)
    and seat_of(names: ["ann"], name: "cy") == None
    and seat_of(names: [], name: "ann") == None
}

pub fn worst(scores: List<Int>) -> Option<Int> {
  list.min(items: scores)
}

example min_of_an_empty_list_is_none for worst {
  worst(scores: []) == None
}

example min_of_lists for worst {
  worst(scores: [3, 9, 4]) == Some(value: 3)
    and list.min(items: ["b", "a"]) == Some(value: "a")
    and worst(scores: [1]) == Some(value: 1)
}
```

### 変換する・畳み込む

```cernith
module guide.list_transform;

import std.int;
import std.list;

pub fn distinct_tags(tags: List<String>) -> List<String> {
  list.unique(items: tags)
}

example unique_keeps_the_first_of_each for distinct_tags {
  list.unique(items: [3, 1, 3, 2, 1]) == [3, 1, 2]
    and list.unique(items: [1, 1, 1]) == [1]
    and distinct_tags(tags: ["a", "b", "a"]) == ["a", "b"]
}

example unique_of_an_empty_list for distinct_tags {
  distinct_tags(tags: []) == []
}

pub fn evens(items: List<Int>) -> List<Int> {
  list.filter(
    items: items,
    keep: fn(item: Int) -> Bool {
      int.rem_euclid(dividend: item, divisor: 2) == 0
    },
  )
}

example filter_can_drop_everything for evens {
  evens(items: [1, 3]) == [] and evens(items: []) == []
}

example filter_keeps_order for evens {
  list.filter(
    items: [1, 2, 3, 4, 5],
    keep: fn(item: Int) -> Bool {
      item > 2
    },
  )
    == [3, 4, 5]
    and evens(items: [5, 4, 3, 2]) == [4, 2]
}

pub fn join_rows(rows: List<List<Int>>) -> List<Int> {
  list.flatten(lists: rows)
}

example flatten_in_order for join_rows {
  join_rows(rows: [[1, 2], [], [3]]) == [1, 2, 3]
}

example flatten_of_no_lists for join_rows {
  join_rows(rows: []) == [] and join_rows(rows: [[], []]) == []
}

pub fn mean_floor(items: List<Int>) -> Int {
  requires items.length > 0;

  int.div_floor(dividend: list.sum(items: items), divisor: items.length)
}

example sum_of_an_empty_list_is_zero for mean_floor {
  list.sum(items: []) == 0
}

example sum_of_lists for mean_floor {
  list.sum(items: [1, 2, 3]) == 6
    and list.sum(items: [5, -5]) == 0
    and mean_floor(items: [1, 2, 4]) == 2
}

pub fn product(items: List<Int>) -> Int {
  list.fold(
    items: items,
    initial: 1,
    combine: fn(total: Int, item: Int) -> Int {
      total * item
    },
  )
}

example fold_can_change_the_type for product {
  list.fold(
    items: ["ab", "c"],
    initial: 0,
    combine: fn(total: Int, item: String) -> Int {
      total + item.length
    },
  )
    == 3
}

example fold_from_the_left for product {
  list.fold(
    items: [1, 2, 3],
    initial: 10,
    combine: fn(total: Int, item: Int) -> Int {
      total - item
    },
  )
    == 4
    and product(items: [2, 3, 4]) == 24
}

example fold_of_an_empty_list_is_the_initial_value for product {
  list.fold(
    items: [],
    initial: 7,
    combine: fn(total: Int, item: Int) -> Int {
      total + item
    },
  )
    == 7
    and product(items: []) == 1
}

pub fn spread_out(items: List<Int>) -> List<Int> {
  list.flat_map(
    items: items,
    apply: fn(item: Int) -> List<Int> {
      [item, item * 10]
    },
  )
}

example flat_map_can_drop_items for spread_out {
  list.flat_map(
    items: [1, 2, 3],
    apply: fn(item: Int) -> List<Int> {
      if item == 2 {
        []
      } else {
        [item]
      }
    },
  )
    == [1, 3]
    and spread_out(items: []) == []
}

example flat_map_joins_the_results for spread_out {
  spread_out(items: [1, 2, 3]) == [1, 10, 2, 20, 3, 30]
}

pub fn squared(items: List<Int>) -> List<Int> {
  list.map(
    items: items,
    apply: fn(item: Int) -> Int {
      item * item
    },
  )
}

example map_keeps_length_and_order for squared {
  squared(items: [1, 2, 3]) == [1, 4, 9]
}

example map_of_an_empty_list for squared {
  squared(items: []) == []
}
```

### 並べ替える・組み合わせる

```cernith
module guide.list_order;

import std.list;

pub fn ascending(items: List<Int>) -> List<Int> {
  list.sort(items: items)
}

example sort_ascending for ascending {
  ascending(items: [3, 1, 2]) == [1, 2, 3]
    and list.sort(items: ["b", "a"]) == ["a", "b"]
    and ascending(items: [2, 1, 2]) == [1, 2, 2]
}

example sort_of_an_empty_list for ascending {
  ascending(items: []) == []
}

pub fn by_length(words: List<String>) -> List<String> {
  list.sort_by(
    items: words,
    less_than: fn(first: String, second: String) -> Bool {
      first.length < second.length
    },
  )
}

example sort_by_is_stable for by_length {
  by_length(words: ["ccc", "a", "bb", "d"]) == ["a", "d", "bb", "ccc"]
    and list.sort_by(
      items: [[2, 0], [1, 1], [2, 2], [1, 3]],
      less_than: fn(first: List<Int>, second: List<Int>) -> Bool {
        first[0] < second[0]
      },
    )
      == [[1, 1], [1, 3], [2, 0], [2, 2]]
}

example sort_by_takes_any_strict_order for by_length {
  list.sort_by(
    items: [1, 3, 2],
    less_than: fn(first: Int, second: Int) -> Bool {
      first > second
    },
  )
    == [3, 2, 1]
    and by_length(words: []) == []
}

example sort_by_with_a_non_strict_order_can_be_a_defect for by_length {
  defects(
    of: list.sort_by(
      items: [3, 1, 3, 2],
      less_than: fn(first: Int, second: Int) -> Bool {
        first <= second
      },
    ),
  )
    and not defects(
      of: list.sort_by(
        items: [3, 1, 2],
        less_than: fn(first: Int, second: Int) -> Bool {
          first <= second
        },
      ),
    )
    and defects(
      of: list.sort_by(
        items: [3, 1, 2],
        less_than: fn(first: Int, second: Int) -> Bool {
          true
        },
      ),
    )
}

pub fn newest_first(items: List<Int>) -> List<Int> {
  list.reverse(items: items)
}

example reverse_of_an_empty_list for newest_first {
  newest_first(items: []) == []
}

example reverse_order for newest_first {
  newest_first(items: [1, 2, 3]) == [3, 2, 1]
}

pub fn pair_up(
  names: List<String>,
  scores: List<Int>,
) -> List<list.Pair<String, Int>> {
  list.zip(first: names, second: scores)
}

example zip_fields_are_first_and_second for pair_up {
  pair_up(names: ["ann"], scores: [90])[0].first == "ann"
    and pair_up(names: ["ann"], scores: [90])[0].second == 90
}

example zip_pairs_up_to_the_shorter for pair_up {
  list.zip(first: [1, 2, 3], second: ["a", "b"])
    == [list.Pair(first: 1, second: "a"), list.Pair(first: 2, second: "b")]
    and pair_up(names: ["ann"], scores: [90, 80])
      == [list.Pair(first: "ann", second: 90)]
}

example zip_with_an_empty_list for pair_up {
  pair_up(names: [], scores: [1]) == []
    and pair_up(names: ["ann"], scores: []) == []
}
```

## 関連

- [option](option.md)：`Option<T>` の関数。`list.find` や `list.first` が返す値を扱う
- [parallel](parallel.md)：`list.map`、`list.filter`、`list.fold`、`list.any`、`list.all` の並列版
- [map](map.md)、[set](set.md)：キーで引くコレクション
- [03 型](../language/03-types.md)：`List<T>`、境界 `Eq` と `Ord`
- [09 契約](../language/09-contracts.md)：`requires` と `ensures`
- [11 効果](../language/11-effects.md)：効果多相
- [12 エラーと欠陥](../language/12-errors-and-defects.md)：事前条件を満たさない呼び出しと欠陥
