# std.result

`Result<T, Fail>` を扱う関数のモジュールです。`Result<T, Fail>` は、成功（`Ok(value: x)`）か、失敗（`Err(error: e)`）かを表す型で、プレリュードにあります。このモジュールの関数を使うには、`import std.result;` と書き、`result.map(...)` のように呼びます。`Option<T>` の関数は [option](option.md) にあり、このページはその対になります。

`Result` を取り出す基本の方法は `match` です（[07 レコードとバリアント](../language/07-records-and-variants.md)）。また、`Err` をそのまま呼び出し元へ返す `?` があります（[12 エラーと欠陥](../language/12-errors-and-defects.md)）。このモジュールの関数は、よくある `match` の定型を短く書くためのものです。

`Result` を受け取る引数のラベルは、すべて `outcome` です（`option` モジュールでは `option`）。

## 関数一覧

| 関数 | 戻り値の型 | 意味 |
| --- | --- | --- |
| `result.is_ok(outcome)` | `Bool` | `Ok` なら `true` |
| `result.is_err(outcome)` | `Bool` | `Err` なら `true` |
| `result.unwrap_or(outcome, default)` | `T` | `Ok(value: x)` なら `x`、`Err` なら `default` |
| `result.map(outcome, apply)` | `Result<U, Fail>` | `Ok(value: x)` なら `Ok(value: apply(value: x))`、`Err` ならそのまま |
| `result.map_error(outcome, apply)` | `Result<T, Other>` | `Err(error: e)` なら `Err(error: apply(error: e))`、`Ok` ならそのまま |
| `result.and_then(outcome, apply)` | `Result<U, Fail>` | `Ok(value: x)` なら `apply(value: x)`、`Err` ならそのまま |
| `result.ok(outcome)` | `Option<T>` | `Ok(value: x)` なら `Some(value: x)`、`Err` なら `None` |

`apply` を受け取る関数（`map`、`map_error`、`and_then`）は、効果多相です。`apply` が純粋なら呼び出しも純粋で、`apply` が `uses io` なら、呼び出し側も `uses io` と宣言します（[11 効果](../language/11-effects.md)）。

このモジュールの関数には、事前条件がありません。どの `Result` を渡しても、関数自身は欠陥になりません（`apply` や `default` の評価が欠陥になる場合を除きます）。`Err` のときに欠陥にして、中身を取り出す関数もありません。`Err` の場合の扱いは、`unwrap_or` で既定値を与えるか、`match` で書きます。

## 関数の詳細

### result.map

```text
pub fn map<T, U, Fail, effect E>(
  outcome: Result<T, Fail>,
  apply: fn(value: T) -> U uses E,
) -> Result<U, Fail> uses E
```

`Ok` の中身に `apply` を適用します。`Err` に対しては `apply` を呼ばず、エラーの値をそのまま返します。`apply` の引数のラベルは `value` です。

### result.map_error

```text
pub fn map_error<T, Fail, Other, effect E>(
  outcome: Result<T, Fail>,
  apply: fn(error: Fail) -> Other uses E,
) -> Result<T, Other> uses E
```

`Err` の中身に `apply` を適用して、エラーの型を `Fail` から `Other` に変えます。`Ok` に対しては `apply` を呼びません。`apply` の引数のラベルは `error` です。`option.map` に対応するものがない、`Result` だけの関数です。

### result.and_then

```text
pub fn and_then<T, U, Fail, effect E>(
  outcome: Result<T, Fail>,
  apply: fn(value: T) -> Result<U, Fail> uses E,
) -> Result<U, Fail> uses E
```

`apply` 自身が `Result` を返すときに使います。`map` なら `Result<Result<U, Fail>, Fail>` になる場面で、入れ子にならず `Result<U, Fail>` が得られます。失敗しうる処理を続けて書くと、最初の `Err` で止まり、その `Err` が結果になります。後ろの処理は呼ばれません。

`outcome` と `apply` の結果は、エラーの型が同じ `Fail` でなければなりません。違う型なら、型エラー `E0301` です。

```cernith-error E0301
module guide.result_types;

import std.result;

pub fn lookup(code: Result<Int, Int>) -> Result<Int, String> {
  result.and_then(
    outcome: code,
    apply: fn(value: Int) -> Result<Int, String> {
      Ok(value: value)
    },
  )
}
```

エラーの型をそろえるのは `result.map_error` です。`outcome` を `result.map_error` に通して、`Result<Int, String>` にしてから渡します（`guide.results` の `lookup`）。`?` も、エラーの型が等しいときだけ使えます。

> 補足：`let half = halve(value: value)?; halve(value: half)` は、`result.and_then(outcome: halve(value: value), apply: ...)` と同じ結果になります（`guide.results` の `quarter`）。

### result.unwrap_or

```text
pub fn unwrap_or<T, Fail>(outcome: Result<T, Fail>, default: T) -> T
```

`Err` の場合の値を与えて、`Result` を外します。エラーの値は捨てられます。`default` は、`Ok` の場合でも評価されます（Cernith は、引数を先に評価します）。`default` の評価が欠陥になる式なら、`Ok` を渡しても欠陥です。

### result.ok

```text
pub fn ok<T, Fail>(outcome: Result<T, Fail>) -> Option<T>
```

`Result` を `Option` に変えます。`Ok(value: x)` を `Some(value: x)` に、`Err` を `None` にし、エラーの値は捨てます。`Option` を `Result` に変えるのは、`option.ok_or` です（[option](option.md)）。

### result.is_ok と result.is_err

```text
pub fn is_ok<T, Fail>(outcome: Result<T, Fail>) -> Bool
pub fn is_err<T, Fail>(outcome: Result<T, Fail>) -> Bool
```

中身が必要でなく、成功か失敗かだけを知りたいときに使います。同じ `outcome` に対して、2 つの結果は必ず反対です。`==` による比較（`outcome == Ok(value: 1)`）は、`T` と `Fail` が等価比較できるときにしか書けませんが、この 2 つは `T` と `Fail` によらず使えます。

## 例

```cernith
module guide.results;

import std.int;
import std.result;

pub fn halve(value: Int) -> Result<Int, String> {
  if int.rem_euclid(dividend: value, divisor: 2) == 0 {
    Ok(value: int.div_floor(dividend: value, divisor: 2))
  } else {
    Err(error: "odd")
  }
}

example and_then_stops_at_the_first_error for halve {
  result.and_then(
    outcome: halve(value: 8),
    apply: fn(value: Int) -> Result<Int, String> {
      halve(value: value)
    },
  )
    == Ok(value: 2)
    and result.and_then(
      outcome: halve(value: 3),
      apply: fn(value: Int) -> Result<Int, String> {
        halve(value: value)
      },
    )
      == Err(error: "odd")
    and result.and_then(
      outcome: halve(value: 6),
      apply: fn(value: Int) -> Result<Int, String> {
        halve(value: value)
      },
    )
      == Err(error: "odd")
}

example is_ok_and_is_err for halve {
  result.is_ok(outcome: halve(value: 4))
    and not result.is_err(outcome: halve(value: 4))
    and result.is_err(outcome: halve(value: 3))
    and not result.is_ok(outcome: halve(value: 3))
}

example map_changes_only_the_value for halve {
  result.map(
    outcome: halve(value: 4),
    apply: fn(value: Int) -> Int {
      value * 3
    },
  )
    == Ok(value: 6)
    and result.map(
      outcome: halve(value: 3),
      apply: fn(value: Int) -> Int {
        value * 3
      },
    )
      == Err(error: "odd")
}

example map_error_changes_only_the_error for halve {
  result.map_error(
    outcome: halve(value: 3),
    apply: fn(error: String) -> Int {
      error.length
    },
  )
    == Err(error: 3)
    and result.map_error(
      outcome: halve(value: 4),
      apply: fn(error: String) -> Int {
        error.length
      },
    )
      == Ok(value: 2)
}

example ok_drops_the_error for halve {
  result.ok(outcome: halve(value: 4)) == Some(value: 2)
    and result.ok(outcome: halve(value: 3)) == None
}

example unwrap_or_evaluates_the_default_even_for_an_ok for halve {
  defects(
    of: result.unwrap_or(
      outcome: halve(value: 4),
      default: int.div_floor(dividend: 1, divisor: 0),
    ),
  )
}

example unwrap_or_gives_the_default_for_an_error for halve {
  result.unwrap_or(outcome: halve(value: 4), default: 0) == 2
    and result.unwrap_or(outcome: halve(value: 3), default: 0) == 0
}

pub fn lookup(code: Result<Int, Int>) -> Result<Int, String> {
  result.and_then(
    outcome: result.map_error(
      outcome: code,
      apply: fn(error: Int) -> String {
        "code"
      },
    ),
    apply: fn(value: Int) -> Result<Int, String> {
      halve(value: value)
    },
  )
}

example map_error_aligns_the_error_types for lookup {
  lookup(code: Ok(value: 4)) == Ok(value: 2)
    and lookup(code: Ok(value: 3)) == Err(error: "odd")
    and lookup(code: Err(error: 7)) == Err(error: "code")
}

pub fn quarter(value: Int) -> Result<Int, String> {
  let half = halve(value: value)?;
  halve(value: half)
}

example question_mark_is_and_then for quarter {
  quarter(value: 8) == Ok(value: 2)
    and quarter(value: 6) == Err(error: "odd")
    and quarter(value: 3) == Err(error: "odd")
    and quarter(value: 8)
      == result.and_then(
        outcome: halve(value: 8),
        apply: fn(value: Int) -> Result<Int, String> {
          halve(value: value)
        },
      )
    and quarter(value: 6)
      == result.and_then(
        outcome: halve(value: 6),
        apply: fn(value: Int) -> Result<Int, String> {
          halve(value: value)
        },
      )
}
```

## 関連

- [option](option.md)：`Option<T>` の関数。`option.ok_or` は `Option` を `Result` に変える
- [07 レコードとバリアント](../language/07-records-and-variants.md)：`match` による取り出し
- [11 効果](../language/11-effects.md)：効果多相
- [12 エラーと欠陥](../language/12-errors-and-defects.md)：エラーと欠陥の使い分け、`?`
