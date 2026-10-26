# レコードとバリアント

ユーザーが定義できる複合型は、**レコード**（名前の付いたフィールドの組）と**バリアント**（いくつかのケースのうちの 1 つ）の 2 つです。タプルはありません。どちらも不変で、名前的な型です。同じ構造でも、名前が違えば別の型です。

## レコード

```text
[pub] record 名前<型パラメータ> {
  フィールド名: 型,
  ...
} [where 条件;]
```

フィールドは、`名前: 型,` の形で 1 行に 1 つ書き、最後のフィールドにもカンマを付けます。フィールドの順序には意味があります。

| 式 | 意味 |
| --- | --- |
| `Rect(width: 3, height: 4)` | 生成。すべてのフィールドを、宣言順に、ラベル付きで渡す |
| `r.width` | フィールドの読み出し |
| `Rect.with(base: r, width: 5)` | `r` のコピーで、指定したフィールドだけ置き換えたもの。ラベルは宣言順に並べる |

レコードは変更できません。`r.width = 5;` のような代入はなく、フィールドを置き換えた新しい値を `with` で作ります。生成のラベルが足りない・余る・順序違いのときは `E0310`、存在しないフィールドを読むと `E0317` です。

```cernith
module guide.rectangles;

pub record Rect {
  width: Int,
  height: Int,
}

pub fn widen(rect: Rect) -> Rect {
  Rect.with(base: rect, width: rect.width + 1)
}

example widen_keeps_height for widen {
  widen(rect: Rect(width: 1, height: 2)) == Rect(width: 2, height: 2)
}
```

```cernith-error E0310
module guide.rectangles;

pub record Rect {
  width: Int,
  height: Int,
}

pub fn square(side: Int) -> Rect {
  Rect(height: side, width: side)
}
```

### レコードの不変条件

レコードには、`where` で不変条件を付けられます。条件の中では、フィールドの名前をそのまま使います。

```cernith
module guide.grids;

pub record Grid {
  width: Int,
  height: Int,
  cells: List<Int>,
} where width >= 0 and cells.length == width * height;

pub fn empty_grid() -> Grid {
  Grid(width: 0, height: 0, cells: [])
}

example empty_grid_has_no_cells for empty_grid {
  empty_grid().cells.length == 0
}
```

`Grid(...)` の生成と `Grid.with(...)` のたびに、「不変条件が成り立つ」という証明義務が生まれます。証明できれば検査が不要になり、反例があれば `cernith check --verify` がエラー `E0503` を報告し、どちらとも決まらなければ実行時の検査になります（[10 証明義務と保証レベル](10-verification.md)）。次の `blank` は、`cells` が常に空なので、`width * height` が 0 でない入力で不変条件に違反します。`cernith check --verify` はこの違反を見つけます。

```cernith-error E0503
module guide.grids;

pub record Grid {
  width: Int,
  height: Int,
  cells: List<Int>,
} where width >= 0 and cells.length == width * height;

pub fn blank(width: Int, height: Int) -> Grid {
  requires width >= 0 and height >= 0;

  Grid(
    width: width,
    height: height,
    cells: [],
  )
}
```

不変条件は仕様式です。効果のある関数は呼べません（[09 契約](09-contracts.md)）。整数や浮動小数点数の比較、`.length`、`and`・`or`・`not` だけで書いた条件は、チェッカーが前提として扱いやすい、単純な形です。

## バリアント

```text
[pub] variant 名前<型パラメータ> {
  ケース,
  ケース(フィールド名: 型, ...),
  ...
}
```

バリアントの値は、ケースのうちのちょうど 1 つです。ケースはフィールドを持てます。再帰的でも、ジェネリックでも構いません。

ケースは、`バリアント名.ケース名` の形で書きます。フィールドのあるケースは呼び出しの形で生成し、フィールドのないケースは括弧なしで書きます。他のモジュールの型は、`モジュール別名.型名.ケース名` です。`Some`、`None`、`Ok`、`Err` だけは、プレリュードのケースなので、型名なしで書きます。

```cernith
module guide.expressions;

pub variant Expr {
  Add(left: Expr, right: Expr),
  Number(value: Int),
  Zero,
}

pub fn evaluate(expression: Expr) -> Int {
  match expression {
    Expr.Add(left: left, right: right) => evaluate(expression: left)
      + evaluate(expression: right),
    Expr.Number(value: value) => value,
    Expr.Zero => 0,
  }
}

example evaluates_a_sum for evaluate {
  evaluate(
    expression: Expr.Add(
      left: Expr.Number(value: 2),
      right: Expr.Add(left: Expr.Number(value: 3), right: Expr.Zero),
    ),
  )
    == 5
}
```

フィールドのあるケースの名前を、呼び出さずに書くと、そのフィールドのラベルを持つ**関数値**になります。たとえば `Msg.Typed` は `fn(text: String) -> Msg` です。`list.map(items: texts, apply: Msg.Typed)` のように、そのまま関数として渡せます（`Some`、`Ok`、`Err` も同様です）。フィールドのあるケースを、そのバリアントの値が期待される場所で呼び出さずに使ったり（たとえば `Msg` が必要な場所に `Msg.Typed` と書く）、フィールドのないケースに `()` を付けたりすると、`E0301` です。

> 補足：再帰的なバリアントやレコードは、自分自身の再帰グループの値を引数に取る関数を、フィールドに持てません（`E0602`）。持てると、関数を介した再帰が、停止性の検査から見えなくなるためです（[13 停止性](13-termination.md)）。

## パターン

`match` のケースのパターンには、次の形が書けます。

| パターン | 意味 |
| --- | --- |
| `_` | 何にでも合い、値を束縛しない |
| 名前 | 何にでも合い、値をその名前に束縛する |
| リテラル | `0`、`"x"`、`true` など。値が等しいときに合う |
| `Type.Case(field: pattern, ...)` | バリアントのケース。フィールドを**宣言順にすべて**並べ、使わないものは `_` にする |
| `Type.Case` | フィールドのないケース |
| `Type(field: pattern, ...)` | レコード。同様にすべてのフィールドを並べる |
| `Some(value: p)`、`None`、`Ok(value: p)`、`Err(error: p)` | `Option` と `Result` のケース |

パターンは入れ子にできます。リストやタプルのパターン（`[]`、`[x, ...]`）はありません。リストの場合分けは、`if xs.length == 0 { ... }` と `xs[0]` で書きます。ケース名を型名なしで書くとエラー `E0201` で、メッセージが正しい形（`Light.Red`）を示します。

```cernith
module guide.patterns;

pub record Point {
  x: Int,
  y: Int,
}

pub fn describe(point: Point) -> String {
  match point {
    Point(x: 0, y: 0) => "origin",
    Point(x: _, y: 0) => "on the x axis",
    Point(x: 0, y: _) => "on the y axis",
    Point(x: _, y: _) => "elsewhere",
  }
}

example describes_points for describe {
  describe(point: Point(x: 0, y: 0)) == "origin"
    and describe(point: Point(x: 3, y: 0)) == "on the x axis"
    and describe(point: Point(x: 0, y: 2)) == "on the y axis"
    and describe(point: Point(x: 1, y: 1)) == "elsewhere"
}
```

```cernith-error E0201
module guide.patterns;

pub variant Light {
  Green,
  Red,
}

pub fn go(light: Light) -> Bool {
  match light {
    Green => true,
    Red => false,
  }
}
```

## 網羅性

`match` は、被判別値のすべての値を覆わなければなりません。覆っていないと `E0305`（non-exhaustive-match）で、診断が、覆われていない値の例を 1 つ、ソースの書き方で示します（`Light.Yellow`、`Some(value: false)`）。

- バリアント、レコード、`Bool`、`Option`、`Result` は、入れ子の深さにかかわらず、完全に検査されます。
- `Int` と `String` のリテラルは、型全体を覆えません。リテラルのパターンだけの `match` には、`_` か名前のケースが必要です。

覆われていない場合に実行時に失敗する、という扱いはありません。コンパイル時（チェック時）に拒否されます。これによって、バリアントにケースを足したとき、そのバリアントを `match` しているすべての箇所が、検査で見つかります。

```cernith-error E0305
module guide.exhaustive;

pub variant Light {
  Green,
  Red,
  Yellow,
}

pub fn go(light: Light) -> Bool {
  match light {
    Light.Green => true,
    Light.Red => false,
  }
}
```

```cernith
module guide.exhaustive;

pub fn name_of(number: Int) -> String {
  match number {
    0 => "zero",
    1 => "one",
    _ => "many",
  }
}

example name_of_names_small_numbers for name_of {
  name_of(number: 0) == "zero" and name_of(number: 9) == "many"
}
```

## 関連

- 制約付き型と `distinct` 型：[08 制約付き型と distinct 型](08-refined-and-distinct-types.md)
- `Option` と `Result` の扱い：[12 エラーと欠陥](12-errors-and-defects.md)
- ジェネリクス：[06 関数](06-functions.md)
