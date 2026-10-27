# インターフェース

**インターフェース**は、プログラムが自分で宣言する境界です。型が持つと約束する関数を並べ、約束が守るべき法則を述べることもできます。型は、明示的な `impl` で約束します。ジェネリック関数は、`<T: Priced>` のように境界を書いて、約束を求めます。

何かを探したり、推測したりすることはありません。呼び出しで使われる `impl` は、引数の型で決まります。`impl` がなければ、呼び出しの位置でエラーです（`E0316`）。

## 最初の例

```cernith
module guide.priced;

import std.int;
import std.show;

pub record Book {
  title: String,
  pages: Int,
} where pages >= 0;

pub record Pen {
  color: String,
}

pub interface Priced {
  fn price(item: Self) -> Int;

  law never_negative(item: Self) {
    price(item: item) >= 0
  }
}

impl Priced for Book {
  fn price(item: Book) -> Int {
    item.pages * 2
  }
}

impl<T: Priced> Priced for List<T> {
  fn price(item: List<T>) -> Int {
    var total: Int = 0;
    for element in item {
      total = total + price(item: element);
    }
    total
  }
}

impl Priced for Pen {
  fn price(item: Pen) -> Int {
    150
  }
}

impl show.Show for Book derive;

impl show.Show for Pen derive;

pub fn bill<T: Priced>(items: List<T>) -> Int {
  price(item: items)
}

example bill_adds_the_prices for bill {
  bill(items: [Pen(color: "red"), Pen(color: "blue")]) == 300
}

pub fn label<T: Priced + show.Show>(item: T) -> String {
  show.text(value: item) ++ " costs " ++ int.to_text(value: price(item: item))
}

example label_names_the_item_and_its_price for label {
  label(item: Pen(color: "red")) == "Pen(color: \"red\") costs 150"
    and label(item: Book(title: "Cernith", pages: 10))
      == "Book(title: \"Cernith\", pages: 10) costs 20"
}
```

`Priced` がインターフェースです。`Book` と `Pen` は、それぞれ `impl` で `Priced` を約束します。`bill` と `label` は、約束した型すべてに使えるジェネリック関数です。`cernith test` は example を実行し、さらに法則 `never_negative` を、生成した `Book` と `Pen` の値で実行します。

この章の以下では、各部分を順に見ます。

## インターフェースを宣言する

```text
[pub] interface 名前 {
  fn メソッド(ラベル: 型, ...) -> 型 [uses 効果];

  law 名前(ラベル: 型, ...) {
    [requires 条件;]
    本体
  }
}
```

- `Self` は、インターフェースを実装する型を表します。インターフェースの中でだけ書けます。
- **メソッド署名**は、名前、ラベル付きのパラメータ、戻り値の型、そして省略できる `uses` から成ります。本体はなく、`requires` も `ensures` も書けません。メソッドに何を期待するかは、法則が述べます。
- メソッドは、パラメータか戻り値に `Self` を含まなければなりません。`Self` がないメソッドは、どの型の `impl` を使うか決められないからです（`E1401`）。
- メソッドは自分の型パラメータを持てず、インターフェースも型パラメータを持てません（`E1401`）。
- **法則**は、`Self` の値についての property です。本体は `Bool` の式で、メソッドを裸の名前で呼びます。`requires` の行は、それが退ける場合を捨てます。`property` と同じです（[16 テスト](16-testing.md)）。
- インターフェースは境界の名前であって、型ではありません。`fn cost(item: Priced) -> Int` はエラーです（`E1405`）。型パラメータを書きます：`fn cost<T: Priced>(item: T) -> Int`。

```cernith-error E1401
module guide.maker;

pub interface Maker {
  fn make() -> Int;
}
```

```cernith-error E1405
module guide.priced;

pub interface Priced {
  fn price(item: Self) -> Int;
}

pub fn cost(item: Priced) -> Int {
  1
}
```

`interface`、`impl`、`law`、`derive` は予約語ではありません。これらを名前に使っているプログラムも、そのまま動きます。文法がキーワードを期待する位置でだけ、キーワードとして読まれます。

## インターフェースを実装する

```text
impl パス.インターフェース for 型 {
  fn メソッド(ラベル: 型, ...) -> 型 { ... }
}

impl<T: 境界> パス.インターフェース for 型<T> { ... }
impl パス.インターフェース for 型 derive;
```

- `型` は名前のある型です。レコード、バリアント、`distinct` 型や制約付き型、プレリュードの型（`Int`、`String`、`List<T>`、`Option<T>` など）です。型パラメータや関数型は書けません。
- メソッドには `pub` を付けません。`impl` にも `pub` を付けません。`impl` は、そのインターフェースと型が見える場所ならどこでも見えます。
- `impl` の型パラメータは、すべて `型` の中に現れなければなりません。パラメータの境界は、メソッドの本体で使えます。`impl<T: Priced> Priced for List<T>` は、要素が `Priced` であるすべてのリストの `impl` です。
- インターフェースは、自分のモジュールの中では `名前`、それ以外では `エイリアス.名前` と書きます。ほかのモジュールの名前と同じです。
- `derive;` は、メソッドをコンパイラに書かせます。使えるのは `Show`、`Encode`、`Decode` で、[std.show](../stdlib/show.md) と [std.codec](../stdlib/codec.md) で説明します。

`impl` は、そのインターフェースに照らして検査されます（`E1403`）。メソッドは、インターフェースのものと過不足なく同じです。各メソッドは、同じラベルを持ち、`Self` を `impl` の型に置き換えたときに、パラメータと戻り値の型が同じです。使える効果は、インターフェースがそのメソッドに宣言したものまでです。メソッドの本体は、ほかの関数と同じように検査される、普通の関数本体です。

```cernith-error E1403
module guide.priced;

pub record Pen {
  color: String,
}

pub interface Priced {
  fn price(item: Self) -> Int;
}

impl Priced for Pen {
  fn price(pen: Pen) -> Int {
    150
  }
}
```

## メソッドを呼ぶ

メソッドは、インターフェースを宣言したモジュールのジェネリック関数です。`guide.priced` の中では `price`、そのモジュールを `priced` として `import` した別のモジュールからは `priced.price(item: x)` です。ほかの関数と同じく、ラベルを付けて呼びます。レシーバはなく、値のフィールドでもありません。

`impl` は、引数の型で決まります。`price(item: Pen(color: "red"))` は `Pen` の `impl`、`price(item: [Pen(color: "red")])` は `List<T>` の `impl` を使います。メソッドは、`apply: price` のように、値として渡すこともできます。

`Self` が戻り値にしか現れないときは、どの `impl` を使うかを教える引数がありません。`fresh::<Int>(seed: 1)` のように、`::<T>` で型を書きます。標準ライブラリにも、このようなメソッドがあります。`codec.decode::<Point>(value: v)` です。文脈で型がすでに決まっている場合（たとえば、戻り値の型が `Int` の関数の結果として呼ぶ場合）は、`::<T>` がなくても受け付けられます。何も型を決めなければ、`E0320` です（[06 関数](06-functions.md)）。

```cernith
module guide.fresh;

pub interface Fresh {
  fn fresh(seed: Int) -> Self;
}

impl Fresh for Int {
  fn fresh(seed: Int) -> Int {
    seed * 2
  }
}

impl Fresh for String {
  fn fresh(seed: Int) -> String {
    "item"
  }
}

pub fn number() -> Int {
  fresh::<Int>(seed: 4)
}

example number_uses_the_int_impl for number {
  number() == 8
}

pub fn text() -> String {
  fresh::<String>(seed: 4)
}

example text_uses_the_string_impl for text {
  text() == "item"
}
```

```cernith-error E0320
module guide.fresh;

pub interface Fresh {
  fn fresh(seed: Int) -> Self;
}

impl Fresh for Int {
  fn fresh(seed: Int) -> Int {
    seed * 2
  }
}

pub fn drop_it() -> Int {
  discard fresh(seed: 4);
  0
}
```

## 境界

境界は、`Eq`、`Ord`、またはインターフェースの名前です。型パラメータには、`+` でつないで複数の境界を付けられます：`<K: Ord + show.Show>`。これらのどれでもない境界は `E0318` です。ほかのモジュールの非公開のインターフェースは、未知のものとして扱われます。

```cernith-error E0318
module guide.keep;

pub fn keep<T: Priced>(item: T) -> T {
  item
}
```

型が境界を満たすのは、次の 3 つのどれかの場合です。

1. 型が型パラメータで、その境界の並びに、その境界がある（`Ord` は `Eq` も与える）。
2. その型の構成子に対するインターフェースの `impl` があり、その `impl` 自身のパラメータの境界が、型の引数で成り立つ。`Pen` が `Priced` なので、`List<Pen>` も `Priced` です。`String` が `Priced` でなければ、`List<String>` もそうではありません。
3. 型が制約付き型で、自分の `impl` を持たず、土台の型がその境界を満たす。`distinct` 型は抽象なので、自分の `impl` が必要です。

境界を満たさないと `E0316` です。型推論のあとで検査されるので、メッセージには具体的な型が現れ、`impl` がないことが示されます。

```cernith-error E0316
module guide.priced;

pub record Pen {
  color: String,
}

pub interface Priced {
  fn price(item: Self) -> Int;
}

impl Priced for Pen {
  fn price(item: Pen) -> Int {
    150
  }
}

pub fn cost<T: Priced>(item: T) -> Int {
  price(item: item)
}

pub fn cost_of_text() -> Int {
  cost(item: "pen")
}
```

境界 `T: Priced` を持つ関数の中では、型 `T` の値に `Priced` のメソッドを呼べます。`T` について、ほかには何も分かりません。境界のない `T` にメソッドを呼ぶと `E0316` で、`Eq` でない `T` に `==` を使ったときも同じです。

```cernith-error E0316
module guide.priced;

pub interface Priced {
  fn price(item: Self) -> Int;
}

pub fn twice<T>(item: T) -> Int {
  price(item: item) * 2
}
```

## 型ごとに `impl` は 1 つ

インターフェースと型構成子の組ごとに、プログラム全体で `impl` は高々 1 つです。それは、インターフェースを宣言したモジュールか、型を宣言したモジュールに書きます。`Int` や `List<T>` のようなプレリュードの型にはモジュールがないので、書けるのはインターフェースのモジュールだけです。この規則のおかげで、2 つのライブラリがインスタンスについて食い違うことはなく、`import` をたどって探す必要もありません。規則を破ると `E1402` です。

```text
shapes.cnth   module shapes;   pub interface Area { ... }   pub record Square { ... }
report.cnth   module report;   pub record Tile { ... }
```

`impl shapes.Area for report.Tile` は、`shapes` にも `report` にも書けます。両方を `import` する 3 つ目のモジュールに書くと、`E1402` です。

`List<Int>` と `List<String>` は同じ型構成子なので、1 つのインターフェースに別々の `impl` は持てません。2 通りの振る舞いが欲しいプログラムは、関数を 2 つ書くか、ラッパー型を使います。

```cernith-error E1402
module guide.priced;

pub interface Priced {
  fn price(item: Self) -> Int;
}

impl Priced for List<Int> {
  fn price(item: List<Int>) -> Int {
    1
  }
}

impl Priced for List<String> {
  fn price(item: List<String>) -> Int {
    2
  }
}
```

型パラメータそのものへの `impl`（`impl<T> Priced for T`）や、型パラメータが型の中に現れない `impl`（`impl<T> Priced for Int`）も `E1402` です。

## 導出される `impl`

`impl インターフェース for 型 derive;` は、メソッドをコンパイラに書かせます。使えるのは、標準ライブラリに規則があるインターフェース（`std.show`、`std.codec`）と、フィールドがすべてそのインターフェースを満たすレコードまたはバリアントです。それ以外は `E1404` で、メッセージが、満たさない最初のフィールドを示します。`Priced` のように規則のないインターフェースに `derive` を書いても `E1404` です。

```cernith-error E1404
module guide.pen;

import std.show;

pub record Color {
  name: String,
}

pub record Pen {
  color: Color,
}

impl show.Show for Pen derive;
```

先に `Color` に `impl show.Show for Color derive;` を書けば直ります。ジェネリックな型には、パラメータに境界が必要です：`impl<T: show.Show> show.Show for Box<T> derive;`。

## 法則

法則は、インターフェースの `impl` のうち、型パラメータを持たないものそれぞれについて実行されます。`cernith test` は、法則を `impl` の型で具体化し、生成した値で実行します。`property` と同じです（[16 テスト](16-testing.md)）。テストの名前は、`law_` + インターフェース + 法則 + `for` + 型で、`law_Priced_never_negative_for_Pen` のようになります。ほかのモジュールのインターフェースの法則も実行されます。`Show` の法則 `stable` は、あなたのモジュールにある `Show` の `impl` について実行されます。

`impl` に対して偽になる法則は、その `impl` の欠陥です。テストは失敗し、反例が示されます。

```cernith-snippet
module guide.refunds;

pub interface Priced {
  fn price(item: Self) -> Int;

  law never_negative(item: Self) {
    price(item: item) >= 0
  }
}

pub record Refund {
  amount: Int,
}

impl Priced for Refund {
  fn price(item: Refund) -> Int {
    0 - item.amount
  }
}
```

```text
$ cernith test refunds.cnth
failed   property guide.refunds.law_Priced_never_negative_for_Refund  counterexample: item: Refund(amount: 1)
1 tests: 1 failed
```

`cernith test --only never_negative` は、名前にそのテキストを含むテストだけを実行します。このバージョンでは、検証器は法則を証明しません。

## Eq と Ord

`Eq` と `Ord` は、そのままです。`==` と `<` に `impl` は要らず、利用者が `impl Eq` や `impl Ord` を書くと `E1402` です。

```cernith-error E1402
module guide.order;

pub record Version {
  major: Int,
}

impl Ord for Version derive;
```

そのため、レコードを `Map` のキーや `Set` の要素にすることは、今もできません。型にとって順序が何を意味するかをインターフェース（`compare` メソッドを持つもの）で表し、自分の関数で使うことはできますが、それで `Ord` 自体が変わることはありません。

## このバージョンにないもの

- デフォルトメソッド、ほかのインターフェースを拡張するインターフェース
- 関連型
- インターフェースやメソッドの型パラメータ、境界の引数
- `Eq` と `Ord` の利用者による `impl`
- 関数型への `impl`
- 検証器による法則の証明（法則はテストされます）

## 診断コード

| コード | 名前 | 意味 |
| --- | --- | --- |
| `E0316` | bound-not-satisfied | `Eq`、`Ord`、またはインターフェースを満たすべき型に、その `impl` がない |
| `E0318` | unknown-bound | `Eq`、`Ord`、スコープにあるインターフェースのどれでもない境界 |
| `E1401` | invalid-interface | `Self` のないメソッド、型パラメータを持つメソッド、インターフェースの型パラメータ |
| `E1402` | incoherent-impl | インターフェースと型構成子に対する 2 つ目の `impl`、2 つのモジュールの外にある `impl`、名前のある型でないものへの `impl`、型の中に現れない型パラメータ、`Eq` や `Ord` の `impl` |
| `E1403` | impl-does-not-match | メソッドの不足や余分、ラベルや型の違い、インターフェースの宣言より多い効果 |
| `E1404` | cannot-derive | 規則のないインターフェース、レコードでもバリアントでもない型、インターフェースを満たさないフィールドへの `derive` |
| `E1405` | interface-is-not-a-type | 型の位置に書かれたインターフェース |

`pub` を付けた `impl` は構文エラーです（`E0001`）。

## 関連

- ジェネリック関数と、境界 `Eq`、`Ord`：[06 関数](06-functions.md)
- テストとしての法則：[16 テスト](16-testing.md)
- `Show`：[std.show](../stdlib/show.md)、`Encode` と `Decode`：[std.codec](../stdlib/codec.md)
- 可視性：[14 モジュールとパッケージ](14-modules-and-packages.md)
