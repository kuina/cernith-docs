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

条件は `Bool` の式で書き、`self` で検査対象の値を参照します。この型に属する値は、すべてその条件を満たします。

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

入力が信頼できるかどうかで使い分けます。条件を満たすと分かっている値には `Percent(value: n)` を使います。外部からの入力には `Percent.check(value: n)` を使い、`None` の場合を処理します。

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

`Percent` は、`Int` が求められる場所でそのまま使えます。`percent + 1` や `percent > 0` と書くときに、値を取り出す操作は不要です。`.value` は使えません（`E0317`）。`+ 1` の結果は 0 から 100 の範囲に収まるとは限らないため、結果の型は `Int` になります。

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

distinct 型は、どちらも `Int` で表すユーザー ID と注文 ID のように、表現は同じでも意味が異なる値の混同を防ぎます。

`distinct` 型は、ほかのモジュールからは**抽象型**として扱われます。型名は使えますが、`UserId(value: ...)` や `.value` は定義元のモジュール内でしか使えません（`E0215`）。値の生成や読み出しに必要な関数は、定義元のモジュールで公開します（[14 モジュールとパッケージ](14-modules-and-packages.md)）。一方、制約付き型は条件を満たす限り、どのモジュールからでも構築できます。

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

## 型パラメータとタグ

どちらの形も、レコードと同じように、境界つきの型パラメータを取れます（[06 関数](06-functions.md)）。

```text
type Id<T> = distinct Int;
type Stack<T> = distinct List<T>;
type NonEmpty<T> = List<T> where self.length > 0;
```

- 土台の型には、指定した型引数が代入されます。`Stack<Int>.value` の型は `List<Int>` で、`NonEmpty<Int>` は `List<Int>` を土台とする制約付き型です。
- 土台の型に現れないパラメータは**タグ**です。`Id<Enemy>` と `Id<Bullet>` は、どちらも `Int` を持ちますが、別の型です（`E0303`）。タグの実行時のコストはありません。
- コンストラクタの型引数は、期待される型から推論されます。`Id<Enemy>` が求められる場所では、`Id(value: 7)` は `Id<Enemy>` になります。文脈から決まらない場合は、`Id::<Enemy>(value: 1)` のように `::<>` で型引数を明示します。明示しなければエラー `E0320` になります。
- 制約の条件では、型パラメータを参照できます。必要な境界は `Keyed<K: Ord>` のようにパラメータに書きます。境界を満たさない型引数を渡すと `E0316` になります。
- 制約付きのリストは、土台と同じように添字で読めます（`items[0]`）。`NonEmpty.check(value: ...)` も、ほかの制約付き型と同じように使えます。これまでどおり、`NonEmpty<Int>` は `List<Int>` のサブタイプです。

```cernith
module guide.tags;

pub record Bullet {
  damage: Int,
}

pub record Enemy {
  hp: Int,
}

pub type Id<T> = distinct Int;

pub type NonEmpty<T> = List<T> where self.length > 0;

pub type Stack<T> = distinct List<T>;

pub fn as_list(items: NonEmpty<Int>) -> List<Int> {
  items
}

example as_list_keeps_the_elements for as_list {
  as_list(items: NonEmpty(value: [1, 2])) == [1, 2]
}

pub fn bullet_number() -> Int {
  Id::<Bullet>(value: 1).value
}

example bullet_number_reads_the_number for bullet_number {
  bullet_number() == 1
}

pub fn depth(stack: Stack<Int>) -> Int {
  stack.value.length
}

example depth_reads_the_list for depth {
  depth(stack: Stack(value: [1, 2, 3])) == 3
}

pub fn first_or(items: List<Int>, fallback: Int) -> Int {
  match NonEmpty.check(value: items) {
    Some(value: found) => found[0],
    None => fallback,
  }
}

example first_or_checks_the_length for first_or {
  first_or(items: [4, 5], fallback: 0) == 4
    and first_or(items: [], fallback: 9) == 9
}

pub fn next_enemy(id: Id<Enemy>) -> Id<Enemy> {
  Id(value: id.value + 1)
}

example next_enemy_adds_one for next_enemy {
  next_enemy(id: Id(value: 7)).value == 8
}

pub fn same_enemy(first: Id<Enemy>, second: Id<Enemy>) -> Bool {
  first == second
}

example same_enemy_compares_numbers for same_enemy {
  same_enemy(first: Id(value: 1), second: Id(value: 1))
    and not same_enemy(first: Id(value: 1), second: Id(value: 2))
}
```

```cernith-error E0303
module guide.tags;

pub record Bullet {
  damage: Int,
}

pub record Enemy {
  hp: Int,
}

pub type Id<T> = distinct Int;

pub fn to_bullet(id: Id<Enemy>) -> Id<Bullet> {
  id
}
```

```cernith-error E0320
module guide.tags;

pub type Id<T> = distinct Int;

pub fn number() -> Int {
  Id(value: 1).value
}
```

```cernith-error E0316
module guide.tags;

pub record Box {
  count: Int,
}

pub type Keyed<K: Ord> = distinct List<K>;

pub fn size(keys: Keyed<Box>) -> Int {
  0
}
```

## サブタイプ

Cernith のサブタイプ関係の基本は、**制約付き型は土台の型のサブタイプになる**という規則です。たとえば `Percent` は `Int` のサブタイプです。この関係は推移的で、制約付き型を土台にした制約付き型にも適用されます。

この関係は、複合型や関数型にも次のように適用されます。

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
