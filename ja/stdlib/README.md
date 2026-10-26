# 標準ライブラリ std

標準ライブラリは、`std.` で始まる名前のモジュールの集まりで、Cernith のソースで書かれています。使うには、`import std.list;` のように `import` し、**最後の部分を別名**にして `list.take(items: xs, count: 2)` のように呼びます。`std.list.take(...)` のようには書けません（[14 モジュールとパッケージ](../language/14-modules-and-packages.md)）。

プレリュードの型と名前（`Int`、`Float64`、`Bool`、`String`、`Unit`、`List`、`Option`、`Result`、`Bytes`、`Floats`、`Ints`、`Some`、`None`、`Ok`、`Err`、`hole`）は、`import` なしで使えます。それ以外の型、たとえば `Map` や `Set` は、モジュールから、モジュール名を付けて使います（`map.Map<String, Int>`）。

## モジュール一覧

### データ構造と値

| モジュール | 内容 |
| --- | --- |
| [std.list](list.md) | `List<T>` の関数：探す、切り出す、変換する、畳み込む、並べ替える |
| [std.text](text.md) | `String` の関数：分割、検索、置換、整形 |
| [std.int](int.md) | `Int` の関数：丸め方を選ぶ割り算、冪、ビット演算、文字列との変換 |
| [std.float](float.md) | `Float64` の関数：変換、最大・最小、`sqrt` |
| [std.math](math.md) | 数学関数と定数：三角関数、指数・対数、丸め |
| [std.word32 と std.word64](word32-word64.md) | 32 ビット・64 ビットの符号なしの語（あふれると折り返す） |
| [std.map](map.md) | `Map<K, V>`：順序のあるキーの辞書 |
| [std.set](set.md) | `Set<T>`：順序のある要素の集合 |
| [std.option](option.md) | `Option<T>` の関数 |
| [std.result](result.md) | `Result<T, E>` の関数 |
| [std.bytes](bytes.md) | `Bytes`：バイト列、数値の読み書き、16 進・Base64 |
| [std.floats と std.ints](floats-ints.md) | `Floats`、`Ints`：数値の配列 |

### 並行・並列

| モジュール | 内容 |
| --- | --- |
| [std.parallel](parallel.md) | 純粋な関数をリストの各要素に並列に適用する |
| [std.async](async.md) | タスク、スコープ、チャネル、時間の待ち |

### 世界との関わり

| モジュール | 内容 |
| --- | --- |
| [std.io、std.clock、std.random](io-clock-random.md) | 標準入出力、時刻、乱数（ケイパビリティを使わない効果） |
| [std.console](console.md) | 標準入出力（ケイパビリティを使う） |
| [std.env](env.md) | 環境変数 |
| [std.fs](fs.md) | ファイルとフォルダ |
| [std.path](path.md) | パスの文字列操作 |
| [std.http](http.md) | HTTP の取得 |

### テスト

| モジュール | 内容 |
| --- | --- |
| [std.test](test.md) | シミュレート世界で効果のあるコードを試す |
| [std.sim](sim.md) | ファイル・環境変数・ネットワークのシミュレート世界 |

## 規約

標準ライブラリの関数は、次の規約でそろえてあります。

- **ラベルは役割に従います。** 操作の対象は先頭にあり、`items`、`of`、`option`、`outcome`、`map` などの名前です。関数を渡すパラメータは `apply`、`keep`、`test`、`combine`、`less_than` で、その関数のパラメータの名前は `item`、`total`、`first`、`second`、`value` です。
- **関数を引数に取る関数の多くは、効果について多相**です（`<effect E>`）。純粋な関数を渡せば呼び出しも純粋です（[11 効果](../language/11-effects.md)）。例外は、`list.sort_by`（比較は純粋でなければなりません）、`std.parallel`（関数に効果を許しません）、`test.with_world` と `sim.with_sim`（関数が持てる効果は `clock`、`io`、`random` だけです）です。
- **全域な操作は `Option` を返し、部分的な操作は事前条件を持ちます。** `first`、`last`、`max`、`min`、`get`、`index_of`、`parse` は、結果がないことがあるので `Option` を返します。`take`、`drop`、`slice`、`range` のように範囲を取る関数は、範囲が正しいことを事前条件にし、違反する呼び出しは欠陥です（[12 エラーと欠陥](../language/12-errors-and-defects.md)）。
- **位置と長さは、Unicode スカラー値で数えます。** バイトや UTF-16 の単位ではありません。
- **並べ替えは安定**です。`sort_by` は「より小さい」を表す厳密な比較を取ります。
- **ケースの変換は ASCII の英字だけ**です。`to_lower` と `to_upper` は、ロケールも Unicode のケースマッピングも持ちません。
- 結果が `Float64` の関数は、結果が有限でなければなりません。無限大や NaN になる演算は欠陥です。
- **契約を持つ関数のシグネチャは、`{ ... }` の形で示します。** `requires` と `ensures` の節だけを書き、本体は `...` で省略しています。

各ページのコード例は、実行して確かめたものです。
