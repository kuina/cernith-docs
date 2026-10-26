# foreign 関数

関数の実装を、Cernith ではなく、別の言語で書くことができます。このような関数を **foreign 関数**と呼びます。Cernith で書けない処理（既存のライブラリの呼び出しなど）に使います。

## 宣言

```text
foreign python "math.isqrt";
foreign javascript "cernith_runtime.isqrt";
fn isqrt(value: Int) -> Int {
  requires value >= 0;
  ensures result >= 0;
}
```

- `foreign` の行は、実装する言語（バックエンド）の名前と、`モジュール.シンボル` の形のシンボル名の文字列を書きます。バックエンドは `python`、`javascript`、`wasm`、`native` です。ほかの名前や、ドットのないシンボル、同じバックエンドの重複は `E0212` です。
- `foreign` の行は、対応する `fn` の直前に置きます。複数書くと、バックエンドごとの実装を宣言できます。
- 宣言には契約の節だけを書き、本体の文は書きません。
- `foreign native` は、C の関数を指します（`foreign native "c.name"` の形）。
- 引数と戻り値には、境界を正確に越えられる型だけを使えます。

| 使える型 |
| --- |
| `Int`、`Float64`、`Bool`、`String`、`Unit` |
| それらの `List` |
| それらの制約付き型 |

レコード、バリアント、`Option`、`Result`、`distinct` 型、関数は越えられません（`E0213`）。Cernith の側で包んで使います。この制限は `foreign python`、`foreign javascript`、`foreign native` のものです。`foreign wasm` の関数はホスト関数で、レコード、バリアント、`Option`、`Result` も使えます（関数とリソースは越えられません）。結果のデコードが失敗しうる場合（たとえば結果に `String` を含むとき）は、結果の型を `Result<T, host.BoundaryError>` にします（`E1304`）。`native` の C 関数が受け渡せる型は、さらに限られます（`Int`、`Float64`、`Bool`、`String`、戻り値の `Unit`）。

```cernith
module guide.root;

pub fn is_perfect_square(value: Int) -> Bool {
  requires value >= 0;

  let root = isqrt(value: value);
  root * root == value
}

example squares_and_non_squares for is_perfect_square {
  is_perfect_square(value: 9) and not is_perfect_square(value: 10)
}

foreign javascript "cernith_runtime.isqrt";
foreign python "math.isqrt";
fn isqrt(value: Int) -> Int {
  requires value >= 0;
  ensures result >= 0;
  ensures result * result <= value;
  ensures (result + 1) * (result + 1) > value;
}
```

```cernith-error E0213
module guide.root;

foreign python "math.isqrt";
fn isqrt(value: Option<Int>) -> Int {
  requires true;
}
```

## 契約は信頼される

**foreign 関数の本体は検査されません。** 契約はそのまま信じられます。呼び出し側は、通常の関数と同じく `requires` の証明義務を負い、`ensures` は成り立つものとして使われます。つまり、foreign 関数の契約は、プログラムの信頼の基礎です。処理系の `audit` コマンドは、すべての foreign 関数とその契約を、信頼の境界として一覧にします。

- 署名に `uses` がなければ、その関数は純粋であると主張しています。
- 停止することも主張しています。止まらないかもしれない foreign 関数は `uses diverge` を宣言します。
- foreign のコードが失敗した場合（Python の例外など）や、宣言と違う型の値を返した場合は、`foreign-failure` の欠陥です。責任は foreign 関数にあります（契約が守られなかったため）。

## どの実装が使われるか

`cernith run` と `cernith test` の標準の実行（リファレンスインタプリタ）は、`foreign python` の行の実装を使います。その行がない foreign 関数を実行すると、`foreign-failure` の欠陥になります。

特定のバックエンド向けの実行ファイルを作るとき（`cernith build --target ...`）は、そのバックエンド向けの `foreign` 行がない foreign 関数があると `E0220` です。

## 診断コード

| コード | 名前 | 意味 |
| --- | --- | --- |
| `E0212` | bad-foreign-declaration | 未知のバックエンド、バックエンドの重複、`モジュール.シンボル` でないシンボル |
| `E0213` | foreign-type-not-supported | 境界を越えられない型 |
| `E0220` | missing-foreign-target | ビルドするバックエンド向けの実装がない |

## 関連

- 効果：[11 効果](11-effects.md)
- 欠陥：[12 エラーと欠陥](12-errors-and-defects.md)
- モジュール：[14 モジュールとパッケージ](14-modules-and-packages.md)
