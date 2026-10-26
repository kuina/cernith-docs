# エラーと欠陥

失敗には 3 種類あり、扱いが違います。

| 種類 | 何か | 型に現れるか | プログラムが扱えるか |
| --- | --- | --- | --- |
| **エラー** | 呼び出し側が判断すべき、起こりうる結果（解析の失敗、キーがない、入出力の失敗） | 現れる。戻り値の型の `Result<T, E>` か `Option<T>` | 扱える。`match`、`?` |
| **欠陥** | 破られた主張（事前条件、事後条件、制約、`assert`、不変条件、添字の範囲、ゼロによる割り算、有限でない `Float64`、`hole`） | 現れない。ただしチェッカーが、欠陥が起こりえないことを証明できたかを報告する | 扱えない。プログラムは停止する |
| **発散** | 終わらない計算 | 現れる。効果 `diverge` | 扱えない（結果がないこと自体） |

これとは別に、メモリ・スタックの深さ・実行ステップの上限を超えた**資源の枯渇**があります。これは独自のステータスで報告され、捕まえられず、Cernith のエラーでも欠陥でもありません。

例外と `catch` はありません。起こりうる失敗を持つ関数は `Result` を返し、呼び出し側が処理するか、`?` で渡します。

## 値としてのエラー

エラーの型は、普通のバリアントです。モジュールや操作ごとに 1 つ定義します。すべてのエラーに共通の型や継承はありません。理由なく「値がない」ことを表すときは `Option` を使います。

```text
variant ParseError {
  EmptyInput,
  NotANumber(piece: String, position: Int),
}
```

`e?` は、`e` が成功なら値を取り出し、エラーなら、そのエラーで**囲んでいる関数から戻ります**。

- `Result<T, E>` の値に使うとき、囲む関数は `Result<_, E2>` を返し、`E` が `E2` と同じ（またはサブタイプ）でなければなりません。エラーの型が違うと `E0309` です。違うエラーへの変換は、`match` や `result.map_error` で、明示的に書きます。
- `Option<T>` の値に使うときは、囲む関数も `Option<_>` を返さなければなりません。

エラーの型を自動で変換しないので、エラーが通る経路が、すべてコードに現れます。

`Result` や `Option` の値を、使わずに捨てることはできません（`E0306`）。意図して捨てるときは `discard` と書きます（[05 文と束縛](05-statements.md)）。

```cernith
module guide.ports;

import std.int;
import std.result;

pub variant ConfigError {
  BadPort(error: PortError),
}

pub type Port = Int where self >= 1 and self <= 65535;

pub variant PortError {
  NotANumber,
  OutOfRange(value: Int),
}

pub fn config_port(text: String) -> Result<Port, ConfigError> {
  result.map_error(
    outcome: parse_port(text: text),
    apply: fn(error: PortError) -> ConfigError {
      ConfigError.BadPort(error: error)
    },
  )
}

example config_wraps_the_error for config_port {
  config_port(text: "x")
    == Err(error: ConfigError.BadPort(error: PortError.NotANumber))
}

pub fn head_plus_one(items: List<Int>) -> Option<Int> {
  let first = list_head(items: items)?;
  Some(value: first + 1)
}

example head_plus_one_propagates_none for head_plus_one {
  head_plus_one(items: []) == None
    and head_plus_one(items: [4]) == Some(value: 5)
}

pub fn list_head(items: List<Int>) -> Option<Int> {
  if items.length == 0 {
    None
  } else {
    Some(value: items[0])
  }
}

pub fn next_port(text: String) -> Result<Int, PortError> {
  let port = parse_port(text: text)?;
  Ok(value: port + 1)
}

example next_of_80 for next_port {
  next_port(text: "80") == Ok(value: 81)
}

example next_of_a_word for next_port {
  next_port(text: "x") == Err(error: PortError.NotANumber)
}

example next_of_zero for next_port {
  next_port(text: "0") == Err(error: PortError.OutOfRange(value: 0))
}

pub fn parse_port(text: String) -> Result<Port, PortError> {
  match int.parse(text: text) {
    None => Err(error: PortError.NotANumber),
    Some(value: number) => match Port.check(value: number) {
      Some(value: port) => Ok(value: port),
      None => Err(error: PortError.OutOfRange(value: number)),
    },
  }
}
```

```cernith-error E0309
module guide.errors;

pub variant First {
  One,
}

pub variant Second {
  Two,
}

pub fn from_first() -> Result<Int, First> {
  Err(error: First.One)
}

pub fn convert() -> Result<Int, Second> {
  let n = from_first()?;
  Ok(value: n)
}
```

### エラーにするか、事前条件にするか

呼び出し側が正しく書かれていても起こりうる結果（利用者の入力が不正、ファイルがない）は、エラーとして `Result` で返します。**正しい呼び出し側が起こせない状況**だけを、事前条件にします。事前条件に違反する呼び出しは欠陥になり、プログラムが止まります。たとえば、不正な入力に対してエラーコードを返したいなら、関数の中で入力を検査して `Err(error: ...)` を返します。`requires` で入力を禁じてはいけません。

## 欠陥

欠陥が起きると、プログラムは停止します（`cernith test` の中では、そのテストが失敗します）。捕まえることも、値にすることも、無視することもできません。

欠陥の原因は、次のとおりです。

| 分類（`class`） | 原因 |
| --- | --- |
| `precondition` | `requires` に違反する呼び出し |
| `postcondition` | `ensures` に違反する戻り値 |
| `restriction` | 制約付き型・不変条件付きレコードの条件に違反する生成 |
| `assertion` | `assert` の失敗 |
| `invariant` | ループ不変条件の違反 |
| `measure` | `decreases` の減少量が減らない |
| `index` | 範囲外の添字 |
| `float-div` | `Float64` のゼロによる割り算 |
| `float-range` | 結果が有限でない `Float64` の演算 |
| `hole` | `hole(reason: ...)` への到達 |
| `foreign-failure` | foreign 関数の失敗（[15 foreign 関数](15-foreign-functions.md)） |

どの欠陥が起こりうるかは、静的に決まる問いです。[10 証明義務と保証レベル](10-verification.md) の証明義務が、まさに、プログラムが持ちうる欠陥です。チェッカーは、まだ起こりえないと証明できていないものを報告します。

### 欠陥の報告

欠陥は、標準エラー出力に 1 つの JSON オブジェクトとして報告され、プログラムは終了ステータス **70** で終わります。

| フィールド | 意味 |
| --- | --- |
| `schema` | 形式の名前（`cernith-defect/0.1`） |
| `class` | 上の分類 |
| `blame` | 責任：事前条件は `caller`、事後条件は `callee`、それ以外は `site`（欠陥が起きた箇所を含む関数） |
| `clause` | 違反した節の文字列、または操作 |
| `location` | モジュールと行 |
| `function` | 実行されていた関数 |
| `values` | 節が参照する変数の値 |
| `message` | 説明 |

次のプログラムを実行した場合です。

```text
module guide.defect;

pub fn halve(value: Int) -> Int {
  requires value >= 0;

  value
}

pub fn main(args: List<String>) -> Int {
  halve(value: 0 - args.length - 1)
}
```

```text
{
 "schema": "cernith-defect/0.1",
 "class": "precondition",
 "blame": "caller",
 "clause": "value >= 0",
 "location": {
  "line": 10,
  "module": "guide.defect"
 },
 "function": "halve",
 "values": {
  "value": "-1"
 },
 "message": "the call to halve violates its precondition"
}
```

プログラムが選ぶ終了ステータス（`main` の戻り値）は、70 と 71 以外の任意の値です。この 2 つは処理系が予約しています。

### 資源の枯渇

メモリ、呼び出しの深さ、実行ステップの上限を超えると、終了ステータス **71** で止まり、`class` が `resource-exhausted` の報告が出ます。報告の `resource` は `memory`、`call-depth`、`steps` のいずれかです。停止性の保証があっても、終了する関数が上限を超えることはありえます。

```text
{"schema": "cernith-defect/0.1", "class": "resource-exhausted", "resource": "steps"}
```

### テストで欠陥を観察する

欠陥はコードの中では捕まえられませんが、`example` と `property` の中で、「この呼び出しは欠陥になる」と述べられます。

```cernith
module guide.non_negative;

pub fn keep(value: Int) -> Int {
  requires value >= 0;

  value
}

example keep_rejects_a_negative_number for keep {
  defects(of: keep(value: -1)) and not defects(of: keep(value: 4))
}
```

`defects(of: 式)` は、式が欠陥で止まれば `true`、普通に返れば `false` です。テストの中でだけ書け、ほかでは `E0222` です。欠陥の分類や責任は取り出せません。資源の枯渇は捕まえません。詳しくは [16 テスト](16-testing.md) にあります。

## 発散

止まらない可能性のある関数は、`uses diverge` を宣言します。宣言していない関数は必ず停止すると、チェッカーが保証します（[13 停止性](13-termination.md)）。

## 関連

- 契約：[09 契約](09-contracts.md)
- 効果：[11 効果](11-effects.md)
- `Option` と `Result` の関数：[stdlib/option.md](../stdlib/option.md)、[stdlib/result.md](../stdlib/result.md)
