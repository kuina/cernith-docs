# 制約付き型と distinct 型

既存の型を土台に、新しい型を作る方法が 2 つあります。

| 種類 | 宣言 | 目的 |
| --- | --- | --- |
| 制約付き型 | `type 名前 = 土台の型 where 条件;` | 値の範囲を制限する。土台の型の演算がそのまま使える |
| distinct 型 | `type 名前 = distinct 土台の型;` | 意味の違う値を取り違えないようにする。土台の型の演算は使えない |

型の別名（`type UserId = Int;`）は書けません（`E0314`）。この 2 つが、型を作る唯一の方法です。

## 制約付き型

```text
[pub] type 名前 = 土台の型 where 条件;
```

条件は `Bool` の式で、`self` が検査される値を表します。型の値は、すべて条件を満たします。

```cernith
module guide.percent;

pub type Percent = Int where self >= 0 and self <= 100;

pub fn bump(percent: Percent) -> Int {
  percent + 1
}

example bump_adds_one for bump {
  bump(percent: Percent(value: 7)) == 8
}

pub fn full() -> Percent {
  Percent(value: 100)
}

example full_is_100 for full {
  full() == 100
}

pub fn parse_percent(number: Int) -> Option<Percent> {
  Percent.check(value: number)
}

example check_accepts_and_rejects for parse_percent {
  parse_percent(number: 50) == Some(value: Percent(value: 50))
    and parse_percent(number: 101) == None
}
```

### 作る

| 式 | 意味 |
| --- | --- |
| `Percent(value: n)` | `n` から `Percent` を作る。条件を満たさなければ欠陥 |
| `Percent.check(value: n)` | 条件を満たせば `Some(value: ...)`、満たさなければ `None`。欠陥にはならない |

どちらも、入力が信頼できるかどうかで使い分けます。条件を満たすことが分かっている値には `Percent(value: n)` を使い、入力として外から来た値には `Percent.check(value: n)` を使って、`None` の場合を処理します。

整数や浮動小数点数のリテラルは、そのままでは制約付き型の値になりません。`var p: Percent = 50` や `fn f() -> Percent { 50 }` はエラー `E0301` で、`Percent(value: 50)` と書きます。診断は、この書き換えを提案します。

```cernith-error E0301
module guide.percent;

pub type Percent = Int where self >= 0 and self <= 100;

pub fn half() -> Percent {
  50
}
```

`Percent(value: n)` は、**証明義務**を生みます。`n` が条件を満たすことをチェッカーが証明できれば、実行時の検査は省かれます。`cernith check --verify` で反例が見つかれば `E0503`（restriction-violated）で、決められなければ実行時に検査されます（[10 証明義務と保証レベル](10-verification.md)）。

```cernith-error E0503
module guide.percent;

pub type Percent = Int where self >= 0 and self <= 100;

pub fn make(number: Int) -> Percent {
  Percent(value: number)
}
```

入力を事前条件で絞れば、証明されます。

```cernith
module guide.percent;

pub type Percent = Int where self >= 0 and self <= 100;

pub fn make(number: Int) -> Percent {
  requires number >= 0 and number <= 100;

  Percent(value: number)
}

example make_keeps_the_number for make {
  make(number: 5) == 5
}
```

### 使う

`Percent` は、`Int` が期待される場所にそのまま使えます。`percent + 1` や `percent > 0` に、取り出す操作は要りません。`.value` は書けません（`E0317`）。結果は `Int` です。`+ 1` の結果は 0 から 100 の範囲とは限らないからです。

## distinct 型

```text
[pub] type 名前 = distinct 土台の型;
```

`distinct` 型は、土台の型と同じ表現を持つ、別の型です。次のように扱います。

| 式 | 意味 |
| --- | --- |
| `UserId(value: 7)` | 土台の値から作る |
| `id.value` | 土台の値を取り出す |
| `==`、`!=` | 同じ `distinct` 型どうしで使える |
| `<` など | 土台の型に順序があれば、同じ `distinct` 型どうしで使える |

それ以外の演算（算術など）はできず、土台の型や別の `distinct` 型と混ぜることもできません（`E0303`、incompatible-identities）。値を計算に使うときは、`.value` で取り出し、結果をまたコンストラクタで包みます。

```cernith
module guide.ids;

pub type OrderId = distinct Int;

pub type UserId = distinct Int;

pub fn earlier(first: UserId, second: UserId) -> Bool {
  first < second
}

example earlier_orders for earlier {
  earlier(first: UserId(value: 1), second: UserId(value: 2))
}

pub fn next_id(id: UserId) -> UserId {
  UserId(value: id.value + 1)
}

example next_id_adds_one for next_id {
  next_id(id: UserId(value: 7)).value == 8
}

pub fn same_user(first: UserId, second: UserId) -> Bool {
  first == second
}

example same_user_compares for same_user {
  same_user(first: UserId(value: 1), second: UserId(value: 1))
}
```

```cernith-error E0303
module guide.ids;

pub type OrderId = distinct Int;

pub type UserId = distinct Int;

pub fn to_order(id: UserId) -> OrderId {
  id
}
```

```cernith-error E0303
module guide.ids;

pub type UserId = distinct Int;

pub fn next_id(id: UserId) -> UserId {
  id + 1
}
```

どちらも `Int` のユーザー ID と注文 ID のように、同じ表現で意味の違う値を、取り違えないようにするためのものです。

`distinct` 型は、他のモジュールから見ると**抽象**です。型の名前は使えますが、`UserId(value: ...)` も `.value` も、定義したモジュールの外では使えません（`E0215`）。値を作る関数と読む関数を、定義したモジュールが公開します（[14 モジュールとパッケージ](14-modules-and-packages.md)）。制約付き型は、逆に誰でも構築できます（条件が番人になります）。

`distinct` 型の土台には、制約付き型も使えます。

```cernith
module guide.counts;

pub type Cents = Int where self >= 0;

pub type Count = distinct Cents;

pub fn three() -> Count {
  Count(value: Cents(value: 3))
}

example three_is_three for three {
  three().value == 3
}
```

## サブタイプ

Cernith のサブタイプ関係は、1 つの規則だけでできています。**制約付き型は、その土台の型のサブタイプです**（`Percent` は `Int` のサブタイプ、推移的に、制約付き型を土台にした制約付き型も）。

この関係は、次のように持ち上がります。

- 不変なコンストラクタを通して、共変に：`List<Percent>` は `List<Int>` のサブタイプ。`Option`、`Result`、レコード、型パラメータを持つバリアントも同様です。
- 関数型を通して：パラメータは反変、戻り値は共変、効果は「少ない効果」のほうがサブタイプです。パラメータの名前は比較しません。

ほかの関係はありません。`Int` は `Float64` のサブタイプではなく、`distinct` 型は土台の型のサブタイプではなく、土台の型は制約付き型のサブタイプではありません（逆向きには、コンストラクタか `check` が必要です）。

サブタイプの値は、土台の型の値が期待される場所にそのまま置けます。これは変換ではなく、値の表現も意味も変わりません。`if` の 2 つの分岐、`match` のケース、リストの要素、比較のオペランドのように、2 つの型が出会う場所の型は、一方がもう一方の上位であればその上位の型です。どちらでもなければ、拒否されます。

```cernith
module guide.subtypes;

pub type Percent = Int where self >= 0 and self <= 100;

pub fn ints_of(percents: List<Percent>) -> List<Int> {
  percents
}

example ints_of_keeps_elements for ints_of {
  ints_of(percents: [Percent(value: 1)]) == [1]
}

pub fn pick(percent: Percent, use_percent: Bool) -> Int {
  if use_percent {
    percent
  } else {
    1000
  }
}

example pick_joins_to_int for pick {
  pick(percent: Percent(value: 5), use_percent: true) == 5
    and pick(percent: Percent(value: 5), use_percent: false) == 1000
}
```

## 関連

- レコードの不変条件（`where`）：[07 レコードとバリアント](07-records-and-variants.md)
- 契約と仕様式：[09 契約](09-contracts.md)
- 証明義務：[10 証明義務と保証レベル](10-verification.md)
