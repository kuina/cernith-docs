# 効果

**効果**は、外部の環境や時刻との、観測可能なやり取りを表します。関数は、起こしうる効果をシグネチャに宣言します。効果を宣言しない関数は**純粋**です。本体を読まなくても、シグネチャから関数と外部環境との関わりを確認できます。

| 効果 | 意味 |
| --- | --- |
| `io` | 外の世界との入出力（コンソール、ファイル、ネットワーク、環境変数） |
| `clock` | 現在時刻の読み出し |
| `random` | 非決定的な選択（乱数） |
| `diverge` | 終わらないかもしれない（[13 停止性](13-termination.md)） |
| `async` | 途中で中断するかもしれない（[18 並行処理](18-concurrency.md)） |

失敗は効果ではありません。起こりうる失敗は `Result` や `Option` の値で表し（[12 エラーと欠陥](12-errors-and-defects.md)）、契約違反は欠陥として扱います。局所変数 `var` の更新も、関数の外から観測できないため効果には含まれません。

## 宣言と検査

効果は、戻り値の型のあとに `uses` で列挙します。

```text
pub fn stamp() -> Int uses clock, io { ... }
```

- 列挙の順序に意味はありません。正準形では、効果の名前をアルファベット順に並べ（効果パラメータはその後ろ）、重複を除きます（[19 正準形](19-canonical-form.md)）。
- **すべての呼び出しで、呼ばれる関数の効果が、呼ぶ関数（またはラムダ）の宣言する効果の部分集合でなければなりません。** そうでないと `E0401`（undeclared-effect）で、足りない効果が示されます。
- 本体が必要とする以上の効果を宣言しても構いません。
- 存在しない効果の名前は `E0405` です。
- 定数の初期化式は純粋です。効果のある関数を呼ぶと `E0401` です。

```cernith-error E0401
module guide.effects;

import std.io;

pub fn greet() -> Int {
  discard io.write_line(text: "hi");
  0
}
```

```cernith
module guide.effects;

import std.clock;
import std.io;

pub fn stamp() -> Int uses clock, io {
  discard io.write_line(text: "stamp");
  clock.now_ms()
}
```

`stamp` は、`example` では呼べません（`E0403`）。効果のあるコードのテストは、シミュレート世界の中で行います（[16 テスト](16-testing.md)）。

## 関数値の効果

関数型は効果を持ちます。`fn(value: Int) -> Unit uses io` は、入出力を行う関数の型です。効果の少ない関数値は、多くの効果を許す位置に置けます（`fn(...) -> T` は `fn(...) -> T uses io` のサブタイプ）。逆は `E0404`（effect-mismatch）です。

関数式（ラムダ）は、外側の関数とは独立に効果を宣言します。ラムダ内で使えるのは、そのラムダが宣言した効果だけです。また、そのラムダを呼び出す関数が必要な効果を宣言していなければ、呼び出しがエラー `E0401` になります。

```cernith-error E0404
module guide.effects;

import std.io;

pub fn run(apply: fn(value: Int) -> Int) -> Int {
  apply(value: 1)
}

pub fn noisy() -> Int {
  run(
    apply: fn(value: Int) -> Int uses io {
      discard io.write_line(text: "x");
      value
    },
  )
}
```

## 効果パラメータ

関数を引数に取る関数は、その引数の効果について**多相**にできます。パラメータ一覧に `effect E` を宣言し、関数型の `uses E` と、関数自身の `uses E` で使います。

```cernith
module guide.polymorphism;

import std.io;

pub fn map_list<A, B, effect E>(
  items: List<A>,
  apply: fn(value: A) -> B uses E,
) -> List<B> uses E {
  var mapped: List<B> = [];
  for item in items {
    mapped = mapped ++ [apply(value: item)];
  }
  mapped
}

pub fn noisy(numbers: List<Int>) -> List<Int> uses io {
  map_list(
    items: numbers,
    apply: fn(value: Int) -> Int uses io {
      discard io.write_line(text: "x");
      value
    },
  )
}

pub fn pure_double(numbers: List<Int>) -> List<Int> {
  map_list(
    items: numbers,
    apply: fn(value: Int) -> Int {
      value * 2
    },
  )
}

example pure_double_doubles for pure_double {
  pure_double(numbers: [1, 2]) == [2, 4]
}
```

呼び出しでは、`E` は、`E` を型に含む関数引数が持つ効果の和集合になります。純粋なラムダを渡せば呼び出しも純粋で、`uses io` のラムダを渡せば呼び出しも `uses io` で、呼ぶ側が `io` を宣言しなければなりません。関数の中では、`E` は宣言した効果のように扱われます。`apply(...)` を呼べるのは、関数が `uses E` を宣言しているからです。

標準ライブラリの `list.map` や `list.filter` は、この形で書かれています（[stdlib/list.md](../stdlib/list.md)）。

効果 `async` は、効果パラメータを通して渡せません（`E1205`）。非同期な関数を `list.map` に渡すことはできず、ループで書きます（[18 並行処理](18-concurrency.md)）。

## 仕様とテストの中

- 契約の節、`where`、ループの節、`spec fn` の本体は純粋です。効果のある関数を呼ぶと `E0402`、`spec fn` が `uses` を宣言しても `E0402` です。
- `example` と `property` は、純粋なコードと `diverge` の関数（実行予算の下で実行されます）を呼べます。`io`、`clock`、`random` のコードを呼ぶと `E0403` です。これらをテストするには、シミュレートされた世界を使います（[16 テスト](16-testing.md)）。

## ケイパビリティと io

ファイルやネットワークなどの外部環境を操作するには、ケイパビリティを引数に渡し、**かつ** `uses io` を宣言する必要があります。ケイパビリティを持っているだけでは足りません。呼び出し側の関数に `uses io` の宣言がなければ、エラー `E0401` になります（[17 リソースとケイパビリティ](17-resources-and-capabilities.md)）。

## 診断コード

| コード | 名前 | 意味 |
| --- | --- | --- |
| `E0401` | undeclared-effect | 呼び出し（またはラムダの本体）が、宣言のない効果を使う |
| `E0402` | effect-in-spec | 仕様の中の効果のある呼び出し、または効果を宣言する `spec fn` |
| `E0403` | effect-in-test | `example` や `property` の中の効果のある呼び出し |
| `E0404` | effect-mismatch | 許される効果より多くの効果を持つ関数値 |
| `E0405` | unknown-effect | 存在しない効果の名前 |
| `E1203` | missing-await | `async` の関数の呼び出しに `await` がない |
| `E1205` | async-through-effect-parameter | 効果 `async` を効果パラメータに通した |

## 関連

- 関数の宣言：[06 関数](06-functions.md)
- 並行処理：[18 並行処理](18-concurrency.md)
