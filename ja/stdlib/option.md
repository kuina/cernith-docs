# std.option

`Option<T>` を扱う関数のモジュールです。`Option<T>` は、値があること（`Some(value: x)`）か、無いこと（`None`）を表す型で、プレリュードにあります。このモジュールの関数を使うには、`import std.option;` と書き、`option.map(...)` のように呼びます。

値を取り出す基本の方法は `match` です（[07 レコードとバリアント](../language/07-records-and-variants.md)）。このモジュールの関数は、よくある `match` の定型を短く書くためのものです。

## 関数一覧

| 関数 | 戻り値の型 | 意味 |
| --- | --- | --- |
| `option.is_some(option)` | `Bool` | `Some` なら `true` |
| `option.is_none(option)` | `Bool` | `None` なら `true` |
| `option.unwrap_or(option, default)` | `T` | `Some(value: x)` なら `x`、`None` なら `default` |
| `option.map(option, apply)` | `Option<U>` | `Some(value: x)` なら `Some(value: apply(value: x))`、`None` なら `None` |
| `option.and_then(option, apply)` | `Option<U>` | `Some(value: x)` なら `apply(value: x)`、`None` なら `None` |
| `option.ok_or(option, error)` | `Result<T, Fail>` | `Some(value: x)` なら `Ok(value: x)`、`None` なら `Err(error: error)` |

`apply` を受け取る関数（`map`、`and_then`）は、効果多相です。`apply` が純粋なら呼び出しも純粋で、`apply` が `uses io` なら、呼び出し側も `uses io` と宣言します（[11 効果](../language/11-effects.md)）。

## 関数の詳細

### option.map

```text
pub fn map<T, U, effect E>(
  option: Option<T>,
  apply: fn(value: T) -> U uses E,
) -> Option<U> uses E
```

`Some` の中身に `apply` を適用します。`None` に対しては `apply` を呼びません。`apply` の引数のラベルは `value` です。

### option.and_then

```text
pub fn and_then<T, U, effect E>(
  option: Option<T>,
  apply: fn(value: T) -> Option<U> uses E,
) -> Option<U> uses E
```

`apply` 自身が `Option` を返すときに使います。`map` なら `Option<Option<U>>` になる場面で、入れ子にならず `Option<U>` が得られます。`apply` が `None` を返せば結果は `None` です。

### option.unwrap_or

```text
pub fn unwrap_or<T>(option: Option<T>, default: T) -> T
```

`None` の場合の値を与えて、`Option` を外します。`default` は、`Some` の場合でも評価されます（Cernith は、引数を先に評価します）。

### option.ok_or

```text
pub fn ok_or<T, Fail>(option: Option<T>, error: Fail) -> Result<T, Fail>
```

`Option` を `Result` に変えます。`None` を、指定したエラーの値に対応づけます。`Result` を `Option` に変えるのは、`result.ok` です（[result](result.md)）。

### option.is_some と option.is_none

```text
pub fn is_some<T>(option: Option<T>) -> Bool
pub fn is_none<T>(option: Option<T>) -> Bool
```

中身が必要でなく、あるかないかだけを知りたいときに使います。`==` による比較（`option == None`）は、`T` が等価比較できるときにしか書けませんが、この 2 つは `T` によらず使えます。

## 例

```cernith
module guide.options;

import std.int;
import std.option;

pub fn even_or_none(value: Int) -> Option<Int> {
  if int.rem_euclid(dividend: value, divisor: 2) == 0 {
    Some(value: value)
  } else {
    None
  }
}

example and_then_chains for even_or_none {
  option.and_then(
    option: Some(value: 4),
    apply: fn(value: Int) -> Option<Int> {
      even_or_none(value: value + 2)
    },
  )
    == Some(value: 6)
    and option.and_then(
      option: Some(value: 3),
      apply: fn(value: Int) -> Option<Int> {
        even_or_none(value: value + 2)
      },
    )
      == None
}

example is_some_and_is_none for even_or_none {
  option.is_some(option: even_or_none(value: 4))
    and option.is_none(option: even_or_none(value: 3))
}

example map_applies_to_some for even_or_none {
  option.map(
    option: even_or_none(value: 4),
    apply: fn(value: Int) -> Int {
      value + 1
    },
  )
    == Some(value: 5)
    and option.map(
      option: even_or_none(value: 3),
      apply: fn(value: Int) -> Int {
        value + 1
      },
    )
      == None
}

example ok_or_gives_a_result for even_or_none {
  option.ok_or(option: even_or_none(value: 4), error: "odd") == Ok(value: 4)
    and option.ok_or(option: even_or_none(value: 3), error: "odd")
      == Err(error: "odd")
}

example unwrap_or_gives_the_default for even_or_none {
  option.unwrap_or(option: even_or_none(value: 4), default: 0) == 4
    and option.unwrap_or(option: even_or_none(value: 3), default: 0) == 0
}
```

## 関連

- [result](result.md)：`Result<T, E>` の関数
- [06 関数](../language/06-functions.md)：効果多相
