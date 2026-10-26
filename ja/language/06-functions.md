# 関数

関数は、トップレベルにだけ定義できます（関数の中に関数を定義する構文はありません）。局所的な関数が必要なときは、関数式（ラムダ）を `let` に束縛します。

## 宣言

```text
[pub] fn 名前<型パラメータ>(パラメータ, ...) -> 戻り値の型 uses 効果, ... {
  節...

  文...
  最後の式
}
```

- `pub` を付けると、ほかのモジュールから呼べます。付けなければ、そのモジュールの中だけです（[14 モジュールとパッケージ](14-modules-and-packages.md)）。
- パラメータは `名前: 型` の形で、末尾のカンマは付けても付けなくても構いません。
- 戻り値の型は必須です。値を返さない関数は `Unit` を返します。
- `uses` は、関数が持つ効果の宣言です。効果がなければ書きません（[11 効果](11-effects.md)）。
- 本体の先頭には、`requires`、`ensures`、`decreases` の節を、この順で書けます（[09 契約](09-contracts.md)、[13 停止性](13-termination.md)）。節のあとに空行を 1 つ置き、そのあとに文と、値となる最後の式が続きます。

```cernith
module guide.functions;

pub fn rectangle_area(width: Int, height: Int) -> Int {
  requires width >= 0 and height >= 0;
  ensures result >= 0;

  width * height
}

example area_of_3_by_4 for rectangle_area {
  rectangle_area(width: 3, height: 4) == 12
}
```

## パラメータとラベル

パラメータの名前が、そのまま呼び出しのラベルになります。呼び出し側は、すべての引数を、宣言と同じ順序でラベル付きで渡さなければなりません（`E0304`、`E0310`。[04 式](04-expressions.md)）。したがって、**パラメータの名前と順序は、関数の公開された仕様の一部**です。名前を変えると、すべての呼び出しが壊れます。

パラメータの値は変更できません。パラメータに代入すると `E0321` です。

## 関数値

関数の名前を、呼び出さずに式の位置に書くと、**関数値**になります。関数値は、関数式（ラムダ）でも作れます。

```text
fn(ラベル: 型, ...) -> 戻り値の型 uses 効果 { 本体 }
```

関数式のパラメータの型と戻り値の型は、省略できません。本体は関数と同じ形で、`return` も節も書けます。

関数型は `fn(ラベル: 型, ...) -> 戻り値の型 uses 効果` と書きます。**関数型のラベルは、型の一部です。** 関数値を呼ぶときは、その値を見ている側の型のラベルを使います。渡した関数が別のパラメータ名を持っていても構いません。

```cernith
module guide.function_values;

import std.list;

pub fn add_one(value: Int) -> Int {
  value + 1
}

pub fn apply_twice(apply: fn(item: Int) -> Int, start: Int) -> Int {
  apply(item: apply(item: start))
}

pub fn demo() -> Int {
  apply_twice(apply: add_one, start: 5)
}

example demo_is_7 for demo {
  demo() == 7
}

pub fn double_all(numbers: List<Int>) -> List<Int> {
  list.map(
    items: numbers,
    apply: fn(item: Int) -> Int {
      item * 2
    },
  )
}

example double_all_doubles for double_all {
  double_all(numbers: [1, 2, 3]) == [2, 4, 6]
}
```

`apply_twice` の中で `apply` を呼ぶときのラベルは、`apply` の型に書かれた `item` です。渡した `add_one` のパラメータ名 `value` ではありません。型のラベルと違うラベルで呼ぶと、`E0310` です。

```cernith-error E0310
module guide.function_values;

pub fn call(apply: fn(item: Int) -> Int) -> Int {
  apply(value: 1)
}
```

関数型どうしの互換性は、パラメータについて反変、戻り値について共変、効果について「少ない効果の関数は、多くの効果を許す場所に置ける」です。パラメータ名は比較しません。

関数式は、周りの関数の `var` を使えません（`E0322`）。`let` で束縛された値や、パラメータの値は、取り込めます。

## ジェネリクス

関数、レコード、バリアントは、型パラメータを持てます。型パラメータは大文字で始まる名前で、`<` `>` の中にカンマ区切りで書きます。

```text
pub fn first<T>(items: List<T>) -> Option<T>
pub record Box<T> { item: T, }
```

型パラメータが取れる型を制限するために、**境界**を付けられます。境界は `Eq` と `Ord` の 2 つだけで、利用者が定義することはできません。

| 境界 | 使える演算 | 満たす型 |
| --- | --- | --- |
| `Eq` | `==`、`!=` | [03 型](03-types.md) の「等価と順序」を参照 |
| `Ord` | `Eq` に加えて `<`、`<=`、`>`、`>=`。`Map` のキー、`Set` の要素にもなれる | 同上 |

境界のない型パラメータの値には、`==` も `<` も使えません。使うと `E0316` です。境界を満たさない型を渡しても `E0316` です（型引数が決まったあとで検査されるので、メッセージには具体的な型が現れます）。`Eq` と `Ord` 以外の名前は `E0318` です。

```cernith
module guide.generics;

pub record Box<T> {
  item: T,
}

pub fn same<T: Eq>(first: T, second: T) -> Bool {
  first == second
}

example same_compares_equal_values for same {
  same(first: 1, second: 1) and not same(first: "a", second: "b")
}

pub fn unbox<T>(box: Box<T>) -> T {
  box.item
}

example unbox_returns_the_item for unbox {
  unbox(box: Box(item: 3)) == 3
}
```

```cernith-error E0316
module guide.generics;

pub fn same<T>(first: T, second: T) -> Bool {
  first == second
}
```

型引数は、普通は推論されます。明示するときは `f::<Int>(...)` と書きます。個数は、効果パラメータを除いた型パラメータの数と同じでなければなりません（`E0315`）。`::<` という書き方なのは、式の中の `<` を、比較演算子として読めるようにするためです。

効果の多相性を表す `effect E` というパラメータは、型パラメータとは別に扱います（[11 効果](11-effects.md)）。

## 型推論

型は、次の場所には必ず書きます。

- 関数のパラメータと戻り値
- レコードのフィールド
- `var` の宣言
- 関数式のパラメータと戻り値

そのほかの型は、局所的に推論されます。推論に使う情報は、この順です。

1. 引数の型。ジェネリックな関数の呼び出しでは、パラメータの型を引数の型と合わせて型変数を決めます。
2. **期待される型**。関数の戻り値、コンストラクタ、リストのリテラル、`None`、`Ok`、`Err` の型は、その式が置かれた位置から決まります。たとえば `fn f() -> Option<Int> { None }` に型注釈は要りません。

関数の終わりまでに決まらない型変数があると、`E0320`（cannot-infer）です。`f::<T>(...)` で型引数を書く、`let` の右辺に文脈を与える、`var xs: List<Int> = [];` のように型を書く、のいずれかで直します。数値リテラルの既定の型などはありません。整数リテラルは `Int`、小数のリテラルは `Float64`、`[]` は文脈が与える型の空リストです。

```cernith-error E0320
module guide.inference;

pub fn nothing<T>() -> List<T> {
  []
}

pub fn use_nothing() -> Int {
  let items = nothing();
  items.length
}
```

```cernith
module guide.inference;

pub fn empty_ints() -> List<Int> {
  []
}

example empty_ints_is_empty for empty_ints {
  empty_ints() == []
}

pub fn nothing<T>() -> List<T> {
  []
}

pub fn use_nothing() -> Int {
  nothing::<Int>().length
}

example use_nothing_is_zero for use_nothing {
  use_nothing() == 0
}
```

## 定数

```text
[pub] const 名前: 型 = 式;
```

定数は、モジュールのトップレベルに書く不変の値で、型を必ず書きます。初期化式は純粋で（効果のある関数を呼べません。`E0401`）、プログラムの開始前に、依存の順に一度だけ評価されます。定数が自分自身に依存する循環は `E0211` です。

```cernith
module guide.constants;

pub const limit: Int = 10;

pub fn over_limit(value: Int) -> Bool {
  value > limit
}

example over_limit_compares for over_limit {
  over_limit(value: 11) and not over_limit(value: 10)
}
```

## プログラムの入口

実行できるプログラムは、`main` という関数を持つモジュールです。

```text
pub fn main(args: List<String>) -> Int uses io { ... }
```

`args` はコマンドライン引数、戻り値は終了ステータスです。入出力を行うなら、`uses io` を宣言します。権限（ケイパビリティ）を受け取る形の `main` は、[17 リソースとケイパビリティ](17-resources-and-capabilities.md) にあります。`main` に特別な規則はそれだけで、ほかは普通の関数です。

## 関連

- 関数の呼び出し規則：[04 式](04-expressions.md)
- 契約：[09 契約](09-contracts.md)
- 再帰と停止性：[13 停止性](13-termination.md)
- 他の言語で実装された関数：[15 foreign 関数](15-foreign-functions.md)
