# std.floats と std.ints

数値の配列 `Floats`（`Float64` の列）と `Ints`（`Int` の列）を扱う 2 つのモジュールです。使うには、`import std.floats;` や `import std.ints;` と書き、`floats.get(array, at)`、`ints.range(from, to)` のように呼びます。型名 `Floats` と `Ints` はプレリュードにあるので、型を書くだけなら `import` は要りません（[02 字句](../language/02-lexical-structure.md)）。

2 つのモジュールは、ほぼ同じ関数を持ちます。以下では、共通の関数をまとめて説明し、片方にしかない関数は別に示します。

## Floats と Ints

`Floats` と `Ints` は、数値だけを並べて持つ列です。`List<Float64>` や `List<Int>` と同じように値ですが、要素を 1 つずつ別の値として確保しない（未ボックスの）形で持ちます。

- `Floats` の要素は、有限の `Float64` です（`Float64` はもともと有限の値だけです。[03 型](../language/03-types.md)）。
- `Ints` の要素は、64 ビットで表せる `Int`、つまり $-2^{63}$ から $2^{63}-1$ の整数です。この範囲の外の値は、事前条件で拒否されます（欠陥になります）。`Int` 自体は桁数に上限がありませんが、`Ints` に入れられる値には上限があります。
- `==` と `!=` は、長さが等しく、要素が先頭から順にすべて等しいかを比べます。`<` などの大小比較はありません。
- リテラルはありません。`floats.from_list` や `ints.range` などの関数で作ります。

`List` との違いは、次のとおりです。

| 操作 | `List<T>` | `Floats` / `Ints` |
| --- | --- | --- |
| 要素の読み出し | `xs[i]` | `floats.get(array, at)` |
| 長さ | `xs.length` | `floats.length(array)` |
| `for x in ...` で順にたどる | できる | できない。`0..floats.length(array: a)` の範囲を添字に使うか、`to_list` でリストにする |
| 要素の型 | 任意 | 有限の `Float64` / 64 ビットの `Int` |

`Floats` と `Ints` は、どちらも `List` に変換できます（`to_list`）。リストから作るには `from_list` を使います。

### 更新は新しい列を返す

`floats.set(array, at, value)` は、`at` の要素だけを置き換えた**新しい列**を返します。元の列は変わりません。他の言語の配列のように、その場で書き換える関数ではありません。

```text
a = floats.set(array: a, at: i, value: x);
```

のように、`var` に結果を代入し直す書き方をします。この `var` の列を他のどこからも参照していなければ、列はその場で更新されるので、要素を 1 つずつ埋めていくループは要素数に比例する手間で済みます。列が他からも参照されているときは、最初の更新でコピーされます。どちらの場合も、見える結果は同じです。

### ints.fits と 64 ビットの範囲

```text
pub spec fn fits(value: Int) -> Bool
```

`ints.fits(value)` は、`value` が $-2^{63}$ 以上 $2^{63}-1$ 以下のとき `true` です。`spec fn` なので、契約（`requires`、`ensures`、ループ不変条件）と、`example`・`property` の中でだけ呼べます。実行されるコードの中で呼ぶと、エラー `E0331` です（[09 契約](../language/09-contracts.md)）。

`Ints` を作る・更新する関数は、渡す値に `fits` を求めます（下の事前条件の表）。逆に、`ints.get` は結果が `fits` を満たすこと、`ints.to_list` は全要素が `fits` を満たすことを保証します。

`ints.sum` と `ints.dot` は、`Ints` の要素を足し合わせますが、結果の型は桁数に上限のない `Int` です。合計が 64 ビットの範囲を超えても、欠陥にはなりません（`ints.dot` の例を参照）。

`Floats` の演算で結果が有限の範囲を超える場合（`floats.sum` が桁あふれするなど）は、通常の `Float64` の演算と同じく欠陥です。

## 関数一覧

どちらのモジュールにもある関数です。`array` は、`floats` では `Floats`、`ints` では `Ints` です。`T` は、`floats` では `Float64`、`ints` では `Int` です。

| 関数 | 戻り値の型 | 意味 |
| --- | --- | --- |
| `new(count, fill)` | 列 | 要素が `count` 個、すべて `fill` の列 |
| `from_list(items)` | 列 | `List<T>` から作る |
| `to_list(array)` | `List<T>` | `List<T>` にする |
| `length(array)` | `Int` | 要素の数 |
| `get(array, at)` | `T` | 添字 `at`（0 始まり）の要素 |
| `set(array, at, value)` | 列 | `at` の要素を `value` にした新しい列 |
| `slice(array, from, to)` | 列 | 添字 `from` 以上 `to` 未満の部分 |
| `concat(first, second)` | 列 | `first` の後ろに `second` をつないだ列 |
| `fill(array, from, to, value)` | 列 | 添字 `from` 以上 `to` 未満を `value` にした新しい列 |
| `reverse(array)` | 列 | 逆順の列 |
| `sum(array)` | `T`（`ints` では `Int`） | 要素の合計。空なら 0 |
| `dot(first, second)` | `T`（`ints` では `Int`） | 内積（対応する要素の積の合計）。空なら 0 |
| `max(array)` | `Option<T>` | 最大の要素。空なら `None` |
| `min(array)` | `Option<T>` | 最小の要素。空なら `None` |

`std.floats` だけにある関数です。

| 関数 | 戻り値の型 | 意味 |
| --- | --- | --- |
| `floats.map(array, apply)` | `Floats` | 各要素に `apply` を適用した列 |
| `floats.from_function(count, make)` | `Floats` | 添字 `0` から `count - 1` について `make(index: i)` を呼んで作る列 |
| `floats.cumulative_sum(array)` | `Floats` | 累積和。添字 `i` の要素が、先頭から `i` 番目までの合計 |

`std.ints` だけにある関数です。

| 関数 | 戻り値の型 | 意味 |
| --- | --- | --- |
| `ints.range(from, to)` | `Ints` | `from` から `to - 1` まで 1 ずつ増える列（`to` は含まない） |
| `ints.fits(value)` | `Bool` | `value` が 64 ビットの範囲に入るか（`spec fn`） |

`floats.map` と `floats.from_function` に渡す関数は、効果多相です。渡す関数が純粋なら呼び出しも純粋で、`uses io` の関数を渡すなら、呼び出し側も `uses io` と宣言します（[11 効果](../language/11-effects.md)）。

## 事前条件

次の表にない関数（`length`、`to_list`、`concat`、`reverse`、`sum`、`max`、`min`、`map`、`cumulative_sum`）には、事前条件がありません。ここに挙げる事前条件に違反する呼び出しは、欠陥です。

| 関数 | 事前条件 |
| --- | --- |
| `floats.new(count, fill)` | `0 <= count <= 1073741823` |
| `ints.new(count, fill)` | `0 <= count <= 1073741823`、`ints.fits(value: fill)` |
| `ints.from_list(items)` | すべての要素が `ints.fits` を満たす |
| `get(array, at)` | `0 <= at < length(array)` |
| `floats.set(array, at, value)` | `0 <= at < length(array)` |
| `ints.set(array, at, value)` | `0 <= at < length(array)`、`ints.fits(value: value)` |
| `slice(array, from, to)` | `0 <= from <= to <= length(array)` |
| `floats.fill(array, from, to, value)` | `0 <= from <= to <= length(array)` |
| `ints.fill(array, from, to, value)` | `0 <= from <= to <= length(array)`、`ints.fits(value: value)` |
| `dot(first, second)` | `length(array: first) == length(array: second)` |
| `floats.from_function(count, make)` | `0 <= count <= 1073741823` |
| `ints.range(from, to)` | `from <= to`、`to - from <= 1073741823`、`ints.fits(value: from)`、`ints.fits(value: to)` |

要素数の上限 $1073741823$ は $2^{30}-1$ です。

なぜ `get` は `Option` を返さず、事前条件にしているのですか。`List` の添字 `xs[i]` と同じ扱いにしているためです。`xs[i]` も、範囲外の添字は欠陥です（チェッカーは、範囲外になりうる箇所をエラー `E0506` で報告します）。範囲内かどうか分からないときは、先に `floats.length(array: a)` と比べます。

## 関数の詳細

### new

```text
pub fn new(count: Int, fill: Float64) -> Floats {
  requires 0 <= count and count <= 1073741823;
  ensures length(array: result) == count;
  ...
}

pub fn new(count: Int, fill: Int) -> Ints {
  requires 0 <= count and count <= 1073741823 and fits(value: fill);
  ensures length(array: result) == count;
  ...
}
```

`count` 個の要素がすべて `fill` の列を作ります。`count` が `0` なら空の列です。

### from_list と to_list

```text
pub fn from_list(items: List<Float64>) -> Floats {
  ensures length(array: result) == items.length;
  ...
}

pub fn from_list(items: List<Int>) -> Ints {
  requires forall i in 0..items.length: fits(value: items[i]);
  ensures length(array: result) == items.length;
  ...
}

pub fn to_list(array: Floats) -> List<Float64> {
  ensures result.length == length(array: array);
  ...
}

pub fn to_list(array: Ints) -> List<Int> {
  ensures result.length == length(array: array)
    and (forall i in 0..result.length: fits(value: result[i]));
  ...
}
```

`List` と相互に変換します。要素の順序は変わりません。

### length、get、set

```text
pub fn length(array: Floats) -> Int {
  ensures result >= 0;
  ...
}

pub fn get(array: Floats, at: Int) -> Float64 {
  requires 0 <= at and at < length(array: array);
  ...
}

pub fn set(array: Floats, at: Int, value: Float64) -> Floats {
  requires 0 <= at and at < length(array: array);
  ensures length(array: result) == length(array: array);
  ...
}
```

`ints` では、型が `Ints` と `Int` になり、`get` に `ensures fits(value: result);` が、`set` の `requires` に `and fits(value: value)` が付きます。`set` は新しい列を返し、元の列は変わりません。`set` の結果の長さは元の列と同じです。

### slice と concat

```text
pub fn slice(array: Floats, from: Int, to: Int) -> Floats {
  requires 0 <= from and from <= to and to <= length(array: array);
  ensures length(array: result) == to - from;
  ...
}

pub fn concat(first: Floats, second: Floats) -> Floats {
  ensures length(array: result) == length(array: first) + length(array: second);
  ...
}
```

`slice` は添字 `from` 以上 `to` 未満の部分を取り出します。`from == to` なら空の列です。`concat` は 2 つの列をつなぎます。どちらか（または両方）が空でもかまいません。

### fill

```text
pub fn fill(array: Floats, from: Int, to: Int, value: Float64) -> Floats {
  requires 0 <= from and from <= to and to <= length(array: array);
  ensures length(array: result) == length(array: array);
  ...
}
```

添字 `from` 以上 `to` 未満の要素を `value` にした新しい列を返します。`from == to` なら、何も変わりません。

### reverse

```text
pub fn reverse(array: Floats) -> Floats {
  ensures length(array: result) == length(array: array)
    and (
      forall i in 0..length(array: result): get(array: result, at: i)
        == get(array: array, at: length(array: array) - 1 - i)
    );
  ...
}
```

要素を逆順に並べた列です。

### sum、dot、max、min

```text
pub fn sum(array: Floats) -> Float64
pub fn dot(first: Floats, second: Floats) -> Float64 {
  requires length(array: first) == length(array: second);
  ...
}

pub fn max(array: Floats) -> Option<Float64> {
  ensures match result {
    None => length(array: array) == 0,
    Some(value: _) => length(array: array) > 0,
  };
  ...
}
```

`ints` では、`sum` と `dot` の戻り値が `Int` になります。`min` は `max` の最小版で、同じ形の `ensures` を持ちます。

`sum` と `dot` は、先頭から順に足します。`Floats` では足す順序で丸めの結果が変わりうるので、この順序が結果の定義です。空の列の `sum` は `0.0`（`ints` では `0`）です。`dot` も、空の列どうしなら `0.0`（`0`）です。`max` と `min` は、空の列では `None` を返します（事前条件にはしていません）。

### floats.map

```text
pub fn map<effect E>(
  array: Floats,
  apply: fn(value: Float64) -> Float64 uses E,
) -> Floats uses E {
  ensures length(array: result) == length(array: array);
  ...
}
```

各要素に `apply` を適用した列を返します。`apply` の引数のラベルは `value` です。

### floats.from_function

```text
pub fn from_function<effect E>(
  count: Int,
  make: fn(index: Int) -> Float64 uses E,
) -> Floats uses E {
  requires 0 <= count and count <= 1073741823;
  ensures length(array: result) == count;
  ...
}
```

添字 `0` から `count - 1` まで順に `make(index: i)` を呼び、その結果を並べた列を返します。`make` の引数のラベルは `index` です。

### floats.cumulative_sum

```text
pub fn cumulative_sum(array: Floats) -> Floats {
  ensures length(array: result) == length(array: array)
    and (
      forall i in 0..length(array: result): get(array: result, at: i)
        == (
          if i == 0 {
            get(array: array, at: 0)
          } else {
            get(array: result, at: i - 1) + get(array: array, at: i)
          }
        )
    );
  ...
}
```

累積和です。`[1.0, 2.0, 3.5]` なら `[1.0, 3.0, 6.5]` です。空の列は空の列になります。

### ints.range

```text
pub fn range(from: Int, to: Int) -> Ints {
  requires from <= to
    and to - from <= 1073741823
    and fits(value: from)
    and fits(value: to);
  ensures length(array: result) == to - from;
  ...
}
```

`from`、`from + 1`、…、`to - 1` の列です。`from == to` なら空の列です。`for i in from..to` の範囲式とは違い、`Ints` の値として作られます。

## 例: Floats

```cernith
module guide.float_arrays;

import std.float;
import std.floats;

pub fn samples() -> Floats {
  floats.from_list(items: [3.5, 1.0, 2.5])
}

example arrays_with_the_same_elements_are_equal for samples {
  samples() == floats.from_list(items: [3.5, 1.0, 2.5])
    and samples() != floats.from_list(items: [3.5, 1.0])
    and floats.new(count: 0, fill: 1.0) == floats.from_list(items: [])
}

example concat_joins_two_arrays for samples {
  floats.to_list(
    array: floats.concat(
      first: samples(),
      second: floats.from_list(items: [4.0]),
    ),
  )
    == [3.5, 1.0, 2.5, 4.0]
    and floats.length(
      array: floats.concat(
        first: floats.new(count: 0, fill: 0.0),
        second: floats.new(count: 0, fill: 0.0),
      ),
    )
      == 0
}

example cumulative_sum_adds_up for samples {
  floats.to_list(array: floats.cumulative_sum(array: samples()))
    == [3.5, 4.5, 7.0]
    and floats.length(
      array: floats.cumulative_sum(array: floats.from_list(items: [])),
    )
      == 0
}

example dot_multiplies_and_adds for samples {
  floats.dot(first: samples(), second: floats.from_list(items: [2.0, 4.0, 2.0]))
    == 16.0
    and floats.dot(
      first: floats.from_list(items: []),
      second: floats.from_list(items: []),
    )
      == 0.0
}

example dot_needs_equal_lengths for samples {
  defects(
    of: floats.dot(first: samples(), second: floats.from_list(items: [1.0])),
  )
}

example fill_outside_the_array_is_a_defect for samples {
  defects(of: floats.fill(array: samples(), from: 1, to: 4, value: 0.0))
    and defects(of: floats.fill(array: samples(), from: 2, to: 1, value: 0.0))
}

example fill_sets_a_range for samples {
  floats.to_list(
    array: floats.fill(array: samples(), from: 1, to: 3, value: 0.0),
  )
    == [3.5, 0.0, 0.0]
    and floats.fill(array: samples(), from: 2, to: 2, value: 0.0) == samples()
}

example from_function_calls_make_for_each_index for samples {
  floats.to_list(
    array: floats.from_function(
      count: 3,
      make: fn(index: Int) -> Float64 {
        float.from_int(value: index) + 0.5
      },
    ),
  )
    == [0.5, 1.5, 2.5]
}

example from_function_rejects_a_negative_count for samples {
  defects(
    of: floats.from_function(
      count: -1,
      make: fn(index: Int) -> Float64 {
        1.0
      },
    ),
  )
}

example from_list_keeps_the_elements for samples {
  floats.to_list(array: samples()) == [3.5, 1.0, 2.5]
    and floats.length(array: floats.from_list(items: [])) == 0
}

example get_outside_the_array_is_a_defect for samples {
  defects(of: floats.get(array: samples(), at: 3))
    and defects(of: floats.get(array: samples(), at: -1))
}

example get_reads_one_element for samples {
  floats.get(array: samples(), at: 1) == 1.0
}

example length_counts_the_elements for samples {
  floats.length(array: samples()) == 3
}

example map_applies_to_each_element for samples {
  floats.to_list(
    array: floats.map(
      array: samples(),
      apply: fn(value: Float64) -> Float64 {
        value * 2.0
      },
    ),
  )
    == [7.0, 2.0, 5.0]
}

example max_is_the_largest for samples {
  floats.max(array: samples()) == Some(value: 3.5)
    and floats.max(array: floats.new(count: 0, fill: 0.0)) == None
}

example min_is_the_smallest for samples {
  floats.min(array: samples()) == Some(value: 1.0)
    and floats.min(array: floats.new(count: 0, fill: 0.0)) == None
}

example new_makes_equal_elements for samples {
  floats.to_list(array: floats.new(count: 2, fill: 1.5)) == [1.5, 1.5]
}

example new_rejects_a_bad_count for samples {
  defects(of: floats.new(count: -1, fill: 0.0))
    and defects(of: floats.new(count: 1073741824, fill: 0.0))
}

example reverse_turns_the_array_around for samples {
  floats.to_list(array: floats.reverse(array: samples())) == [2.5, 1.0, 3.5]
    and floats.length(array: floats.reverse(array: floats.from_list(items: [])))
      == 0
}

example set_gives_a_new_array for samples {
  floats.to_list(array: floats.set(array: samples(), at: 0, value: 9.0))
    == [9.0, 1.0, 2.5]
    and floats.get(array: samples(), at: 0) == 3.5
}

example set_outside_the_array_is_a_defect for samples {
  defects(of: floats.set(array: samples(), at: 3, value: 0.0))
}

example slice_outside_the_array_is_a_defect for samples {
  defects(of: floats.slice(array: samples(), from: 2, to: 4))
    and defects(of: floats.slice(array: samples(), from: 2, to: 1))
    and defects(of: floats.slice(array: samples(), from: -1, to: 1))
}

example slice_takes_a_range for samples {
  floats.to_list(array: floats.slice(array: samples(), from: 1, to: 3))
    == [1.0, 2.5]
    and floats.length(array: floats.slice(array: samples(), from: 2, to: 2))
      == 0
}

example sum_adds_from_the_left for samples {
  floats.sum(array: samples()) == 7.0
    and floats.sum(array: floats.new(count: 0, fill: 1.0)) == 0.0
}

example sum_beyond_the_range_is_a_defect for samples {
  defects(of: floats.sum(array: floats.from_list(items: [1.0e308, 1.0e308])))
}

example to_list_gives_the_elements for samples {
  floats.to_list(array: floats.new(count: 2, fill: 0.5)) == [0.5, 0.5]
}
```

## 例: Ints

```cernith
module guide.int_arrays;

import std.ints;

pub fn counts() -> Ints {
  ints.from_list(items: [30, 10, 20])
}

example arrays_with_the_same_elements_are_equal for counts {
  counts() == ints.from_list(items: [30, 10, 20])
    and counts() != ints.from_list(items: [30, 10])
    and ints.new(count: 0, fill: 1) == ints.from_list(items: [])
}

example concat_joins_two_arrays for counts {
  ints.to_list(
    array: ints.concat(first: counts(), second: ints.from_list(items: [40])),
  )
    == [30, 10, 20, 40]
    and ints.length(
      array: ints.concat(
        first: ints.new(count: 0, fill: 0),
        second: ints.new(count: 0, fill: 0),
      ),
    )
      == 0
}

example dot_can_exceed_64_bits for counts {
  ints.dot(
    first: ints.from_list(items: [4611686018427387904]),
    second: ints.from_list(items: [4]),
  )
    == 18446744073709551616
}

example dot_multiplies_and_adds for counts {
  ints.dot(first: counts(), second: ints.from_list(items: [1, 2, 3])) == 110
    and ints.dot(
      first: ints.from_list(items: []),
      second: ints.from_list(items: []),
    )
      == 0
}

example dot_needs_equal_lengths for counts {
  defects(of: ints.dot(first: counts(), second: ints.from_list(items: [1])))
}

example fill_outside_the_array_is_a_defect for counts {
  defects(of: ints.fill(array: counts(), from: 1, to: 4, value: 0))
    and defects(of: ints.fill(array: counts(), from: 2, to: 1, value: 0))
    and defects(
      of: ints.fill(
        array: counts(),
        from: 0,
        to: 1,
        value: 9223372036854775808,
      ),
    )
}

example fill_sets_a_range for counts {
  ints.to_list(array: ints.fill(array: counts(), from: 1, to: 3, value: 0))
    == [30, 0, 0]
    and ints.fill(array: counts(), from: 2, to: 2, value: 0) == counts()
}

example from_list_keeps_the_elements for counts {
  ints.to_list(array: counts()) == [30, 10, 20]
    and ints.length(array: ints.from_list(items: [])) == 0
}

example from_list_rejects_a_value_outside_the_range for counts {
  defects(of: ints.from_list(items: [1, 9223372036854775808]))
}

example get_outside_the_array_is_a_defect for counts {
  defects(of: ints.get(array: counts(), at: 3))
    and defects(of: ints.get(array: counts(), at: -1))
}

example get_reads_one_element for counts {
  ints.get(array: counts(), at: 1) == 10
}

example length_counts_the_elements for counts {
  ints.length(array: counts()) == 3
}

example max_is_the_largest for counts {
  ints.max(array: counts()) == Some(value: 30)
    and ints.max(array: ints.new(count: 0, fill: 0)) == None
}

example min_is_the_smallest for counts {
  ints.min(array: counts()) == Some(value: 10)
    and ints.min(array: ints.new(count: 0, fill: 0)) == None
}

example new_makes_equal_elements for counts {
  ints.to_list(array: ints.new(count: 2, fill: 15)) == [15, 15]
}

example new_rejects_a_bad_count_or_fill for counts {
  defects(of: ints.new(count: -1, fill: 0))
    and defects(of: ints.new(count: 1073741824, fill: 0))
    and defects(of: ints.new(count: 1, fill: 9223372036854775808))
}

example range_counts_up_by_one for counts {
  ints.to_list(array: ints.range(from: 2, to: 5)) == [2, 3, 4]
    and ints.to_list(array: ints.range(from: -1, to: 1)) == [-1, 0]
    and ints.length(array: ints.range(from: 3, to: 3)) == 0
}

example range_rejects_a_bad_range for counts {
  defects(of: ints.range(from: 3, to: 2))
    and defects(of: ints.range(from: 0, to: 1073741824))
    and defects(of: ints.range(from: 0, to: 9223372036854775808))
}

example reverse_turns_the_array_around for counts {
  ints.to_list(array: ints.reverse(array: counts())) == [20, 10, 30]
    and ints.length(array: ints.reverse(array: ints.from_list(items: []))) == 0
}

example set_gives_a_new_array for counts {
  ints.to_list(array: ints.set(array: counts(), at: 0, value: 9)) == [9, 10, 20]
    and ints.get(array: counts(), at: 0) == 30
}

example set_rejects_a_bad_index_or_value for counts {
  defects(of: ints.set(array: counts(), at: 3, value: 0))
    and defects(
      of: ints.set(array: counts(), at: 0, value: 9223372036854775808),
    )
}

example slice_outside_the_array_is_a_defect for counts {
  defects(of: ints.slice(array: counts(), from: 2, to: 4))
    and defects(of: ints.slice(array: counts(), from: 2, to: 1))
    and defects(of: ints.slice(array: counts(), from: -1, to: 1))
}

example slice_takes_a_range for counts {
  ints.to_list(array: ints.slice(array: counts(), from: 1, to: 3)) == [10, 20]
    and ints.length(array: ints.slice(array: counts(), from: 2, to: 2)) == 0
}

example sum_adds_from_the_left for counts {
  ints.sum(array: counts()) == 60
    and ints.sum(array: ints.new(count: 0, fill: 1)) == 0
}

example sum_can_exceed_64_bits for counts {
  ints.sum(
    array: ints.from_list(items: [9223372036854775807, 9223372036854775807]),
  )
    == 18446744073709551614
}

example to_list_gives_the_elements for counts {
  ints.to_list(array: ints.new(count: 2, fill: 7)) == [7, 7]
}

pub fn only_if_it_fits(value: Int) -> Int {
  requires ints.fits(value: value);

  value
}

example a_value_outside_the_range_is_a_defect for only_if_it_fits {
  defects(of: only_if_it_fits(value: 9223372036854775808))
}

example fits_is_the_64_bit_range for only_if_it_fits {
  ints.fits(value: 9223372036854775807)
    and ints.fits(value: -9223372036854775807 - 1)
    and not ints.fits(value: 9223372036854775808)
    and not ints.fits(value: -9223372036854775807 - 2)
}
```

## 関連

- [list](list.md)：`List` の関数
- [bytes](bytes.md)：バイトの列 `Bytes`。`Floats`、`Ints` と同じく、値として比較できる未ボックスの列です
- [03 型](../language/03-types.md)：`Floats`、`Ints` と `Float64` の有限性
- [09 契約](../language/09-contracts.md)：`requires`、`ensures`、`spec fn`
