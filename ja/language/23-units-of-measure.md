# 単位

数には単位を付けられます。チェッカーは単位を区別して扱います。長さに時間を足すとエラーになり、距離を時間で割ると速さになります。単位はチェッカーの中だけにあります。検証器も各ターゲットも、普通の `Float64` や `Int` として扱うので、単位に実行時のコストはありません。

## 単位を宣言する

```text
[pub] unit 名前;
```

`unit m;` は、モジュールの中に基本単位を宣言します。同じものを測る単位でも、別の型です。`m` と `ft` は、変換なしには混ぜられません。単位に触れる `pub` の関数や型では、その単位も `pub` でなければなりません（そうでなければ `E0216`）。ほかのモジュールの単位は、`import si.units;` のあとの `units.m` のように、インポートの別名を付けて書きます。

```cernith-error E0216
module guide.errors;

unit m;

pub fn read(length: Float64<m>) -> Float64 {
  length.value
}
```

## 量の型

`Float64<UNIT>` と `Int<UNIT>` は、単位を持つ数の型です。単位のない型は、普通の `Float64` や `Int` です。`UNIT` は、次のいずれかの形で書きます。

| 形 | 例 | 意味 |
| --- | --- | --- |
| 1 つの単位 | `m` | 基本単位 |
| 積 | `m * s` | 単位の積 |
| 商 | `m / s` | 単位を単位で割ったもの |
| べき | `m / s^2` | べきは `^` と 2 以上の整数で書く |
| 分母に複数の因子 | `kg / (m * s)` | 分母は括弧で囲む |
| 逆数 | `1 / s` | 分母だけの単位 |

### 正準形

単位の書き方は 1 通りだけです。別の書き方は `E0362` で、メッセージが正準形を示します。

- 各辺の因子は、名前の順に並べます（`s * m` ではなく `m * s`）。
- べきは 1 回で書きます（`m * m` ではなく `m^2`）。
- 横棒は 1 本だけで、分母が複数の因子なら括弧で囲みます。
- 打ち消し合う単位（`m / m`）は、単位のない普通の型です。

```cernith-error E0362
module guide.errors;

pub unit m;

pub fn area(width: Float64<m * m>) -> Float64<m^2> {
  width
}
```

## 量を作る・読む

コンストラクタは、単位と同じ名前です。`m(value: 5.0)` の型は `Float64<m>`、`m(value: 5)` の型は `Int<m>` です。`length.value` は、量の普通の数です。どちらも、実行時には恒等です。

| 式 | 意味 |
| --- | --- |
| `m(value: 5.0)` | 普通の数 `5.0` から `Float64<m>` を作る |
| `m(value: 5)` | 普通の数 `5` から `Int<m>` を作る |
| `length.value` | 量の普通の数 |

コンストラクタの引数は `Int` か `Float64` でなければならず（`E0361`）、量から量をもう一度作ることはできません（`E0360`）。先に `.value` で数を読みます。

```cernith-error E0361
module guide.errors;

pub unit m;

pub fn name() -> Float64<m> {
  m(value: "far")
}
```

```cernith-error E0360
module guide.errors;

pub unit m;

pub fn again(length: Float64<m>) -> Float64<m> {
  m(value: length)
}
```

## 算術

| 式 | 規則 |
| --- | --- |
| `a + b`、`a - b` | 両辺の単位が等しい。結果もその単位 |
| `a == b`、`a < b` などの比較 | 両辺の単位が等しい。結果は `Bool` |
| `-a` | 単位はそのまま |
| `a * b` | 単位を掛ける（`m * m` は `m^2`） |
| `a / b` | 単位を割る（`m / s`）。同じ単位どうしなら、結果は普通の数 |
| 普通の数との積・商 | 量を拡大・縮小し、単位はそのまま（`2.0 * length`、`length / 2.0`） |
| 普通の数を量で割る | 逆数の単位になる（`1.0 / time` の単位は `1 / s`） |

`Int` と `Float64` は、単位があってもなくても混ぜられません（`E0302`）。`Int / Int` は、普通の整数と同じく `E0307` なので、分母のある単位の量は `Float64` で作ります。次の `Int` の例では、積と同じ単位だけを使います。

```cernith
module guide.areas;

pub unit frames;

pub unit m;

pub fn area(width: Int<m>, height: Int<m>) -> Int<m^2> {
  width * height
}

example area_multiplies_the_units for area {
  area(width: m(value: 3), height: m(value: 4)).value == 12
}

pub fn doubled(length: Int<m>) -> Int<m> {
  2 * length
}

example doubled_scales_the_length for doubled {
  doubled(length: m(value: 5)).value == 10
}

pub fn later(first: Int<frames>, second: Int<frames>) -> Bool {
  first > second
}

example later_compares_the_same_unit for later {
  later(first: frames(value: 30), second: frames(value: 20))
}

pub fn total(first: Int<m>, second: Int<m>) -> Int<m> {
  first + second
}

example total_adds_the_same_unit for total {
  total(first: m(value: 2), second: m(value: 3)) == m(value: 5)
}
```

```cernith-error E0302
module guide.errors;

pub unit m;

pub fn mixed(whole: Int<m>, part: Float64<m>) -> Float64<m> {
  part + whole
}
```

単位が違う数、または単位のない数が、単位が期待される場所にあると `E0360`（unit-mismatch）です。メッセージには両方の型が書かれます。単位の間の暗黙の変換はなく、メートルとフィートの間でもありません。修正案は 2 つあり、どちらも確認が必要です。普通の数を `m(value: ...)` で包むか、普通の数が期待される場所の量に `.value` を付けます。

```cernith-error E0360
module guide.errors;

pub unit ft;

pub unit m;

pub fn total(first: Float64<m>, second: Float64<ft>) -> Float64<m> {
  first + second
}
```

```cernith-error E0360
module guide.errors;

pub unit m;

pub fn longer(length: Float64<m>) -> Float64<m> {
  length + 1.0
}
```

## 変換とライブラリ関数

単位の間の変換は、プログラムが書く関数です。`.value` で数を読み、コンストラクタで新しい量を作ります。係数は普通の数です。

ライブラリ関数は、普通の数を取ります。引数には `.value` を書き、結果をもう一度量にします。

```cernith-error E0360
module guide.errors;

import std.float;

pub unit m;

pub fn root(area: Float64<m^2>) -> Float64<m> {
  m(value: float.sqrt(value: area))
}
```

量は、ホストの境界を越えません。`foreign` 関数は、量を取ることも返すこともできません（`E0213`）。`.value` とコンストラクタで変換する Cernith の関数で、呼び出しを包みます。

```cernith-error E0213
module guide.errors;

pub unit m;

foreign python "math.sqrt";
pub fn root(x: Float64<m>) -> Float64<m> {
  requires x.value >= 0.0;
}
```

## 検証

ソルバーは普通の数を見るので、契約は `.value` について書きます（`ensures result.value == ...`）。`Float64` の算術は、一般には判定できません。結果が無限になりうる `Float64` の演算は、`cernith check --verify` が `E0508` と報告し、判定できないものは `W0504` です（[03 型](03-types.md)、[10 証明義務と保証レベル](10-verification.md)）。次の例のように、`requires` で入力を狭めます。

## 例

```cernith
module guide.motion;

import std.float;

pub unit ft;

pub unit m;

pub unit s;

pub fn is_faster(first: Float64<m / s>, second: Float64<m / s>) -> Bool {
  first > second
}

example is_faster_compares_speeds for is_faster {
  is_faster(
    first: m(value: 5.0) / s(value: 1.0),
    second: m(value: 3.0) / s(value: 1.0),
  )
}

pub fn side_of(area: Float64<m^2>) -> Float64<m> {
  requires area.value >= 0.0 and area.value <= 1000000.0;

  m(value: float.sqrt(value: area.value))
}

example side_of_takes_the_root for side_of {
  side_of(area: m(value: 3.0) * m(value: 3.0)).value == 3.0
}

pub fn speed(distance: Float64<m>, time: Float64<s>) -> Float64<m / s> {
  requires distance.value >= 0.0 and distance.value <= 1000000.0;
  requires time.value >= 1.0 and time.value <= 1000000.0;

  distance / time
}

example speed_divides_the_units for speed {
  speed(distance: m(value: 100.0), time: s(value: 4.0)).value == 25.0
}

pub fn to_metres(length: Float64<ft>) -> Float64<m> {
  requires length.value >= 0.0 and length.value <= 1000000.0;

  m(value: length.value * 0.3048)
}

example to_metres_converts for to_metres {
  to_metres(length: ft(value: 100.0)).value == 30.48
}

pub fn travelled(speed: Float64<m / s>, time: Float64<s>) -> Float64<m> {
  requires speed.value >= 0.0 and speed.value <= 1000.0;
  requires time.value >= 0.0 and time.value <= 1000000.0;

  speed * time
}

example travelled_multiplies_the_units for travelled {
  travelled(speed: m(value: 25.0) / s(value: 1.0), time: s(value: 4.0)).value
    == 100.0
}
```

## 関連

- 数：[03 型](03-types.md)
- 値を区別して保つ型：[08 制約付き型と distinct 型](08-refined-and-distinct-types.md)
- モジュールの公開宣言：[14 モジュールとパッケージ](14-modules-and-packages.md)
- foreign 関数：[15 foreign 関数](15-foreign-functions.md)
- `Float64` の関数：[std.float](../stdlib/float.md)
