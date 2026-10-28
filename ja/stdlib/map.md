# std.map

キーと値の対応を保持する `Map<K, V>` を提供します。キーの型 `K` は `Ord` を満たす必要があります。`import std.map;` で読み込み、`map.insert(...)` のように呼び出します。

`Map` は値です。`map.insert` や `map.remove` は元のマップを変更せず、新しいマップを返します。

## 型

```text
pub record Entry<K, V> {
  key: K,
  value: V,
}

pub record Map<K: Ord, V> {
  root: Tree<K, V>,
}

pub variant Tree<K, V> {
  Leaf,
  Node(
    left: Tree<K, V>,
    key: K,
    value: V,
    priority: Int,
    size: Int,
    right: Tree<K, V>,
  ),
}
```

- `Entry<K, V>` は、キーと値の組です。`map.entries`、`map.first`、`map.last` が返します。
- `Map<K, V>` は、マップ本体です。`K` に `Ord` の境界が付いているので、キーに使えるのは `Int`、`Float64`、`String`、`Bool` と、それらを土台にした `distinct` 型・制約付き型だけです（[03 型](../language/03-types.md)）。リストやレコードをキーにすると、エラー `E0316` です。
- `Tree<K, V>` は、`Map` が内部で使う二分探索木です。`Map` のフィールド `root` の型であるため公開されていますが、通常は直接扱う必要はありません。

`Map` の作成や読み出しには、このモジュールの関数（`map.empty`、`map.insert` など）を使います。`Tree` を自分で組み立てて `Map(root: ...)` に渡すことも型の上では可能ですが、各関数は、キーが左から右へ昇順に並ぶなどの規則を前提としています。その規則を満たさない木での結果は保証されません。

同じ要素（キーと値の組）を持つ 2 つの `Map` は、追加した順序によらず `==` で等しくなります。木の形が、キーだけで決まるからです。`==` を使うには、値の型 `V` も等価比較できる（`Eq`）必要があります。

> 補足：`map.priority(key)` という関数も公開されていますが、マップの内部で木の形を決めるための関数です。利用者は使わなくてかまいません。

## 関数一覧

| 関数 | 戻り値の型 | 意味 |
| --- | --- | --- |
| `map.empty()` | `Map<K, V>` | 要素のないマップ |
| `map.insert(map, key, value)` | `Map<K, V>` | `key` に `value` を対応づけたマップ。`key` が既にあれば値を置き換える |
| `map.remove(map, key)` | `Map<K, V>` | `key` を取り除いたマップ。`key` が無ければそのまま |
| `map.get(map, key)` | `Option<V>` | `key` に対応する値。無ければ `None` |
| `map.has(map, key)` | `Bool` | `key` があれば `true` |
| `map.size(map)` | `Int` | 要素の数 |
| `map.entries(map)` | `List<Entry<K, V>>` | すべての要素。**キーの昇順** |
| `map.first(map)` | `Option<Entry<K, V>>` | キーが最小の要素。空なら `None` |
| `map.last(map)` | `Option<Entry<K, V>>` | キーが最大の要素。空なら `None` |

どの関数にも事前条件はありません。キーが無い場合や空のマップは、`Option` の `None` や `false` で表します。

`get`、`has`、`insert`、`remove`、`size` は、他の要素を調べません。要素数 n に対して、おおよそ log n 回の手順で済みます。`first` と `last` も同様で、要素の一覧は作りません。`entries` だけが、全要素のリストを作ります。

## 関数の詳細

### map.empty

```text
pub fn empty<K: Ord, V>() -> Map<K, V> {
  ensures size(map: result) == 0;
  ...
}
```

要素のないマップです。型引数が文脈から決まらないときは、`map.empty::<String, Int>()` のように書きます（[06 関数](../language/06-functions.md)）。

### map.insert

```text
pub fn insert<K: Ord, V>(map: Map<K, V>, key: K, value: V) -> Map<K, V> {
  ensures size(map: result) >= size(map: map)
    and size(map: result) <= size(map: map) + 1;
  ...
}
```

`key` に `value` を対応づけます。`key` が既にあるときは、その値を `value` に置き換えるので、要素数は変わりません。`key` が無かったときだけ、要素数が 1 増えます。元の `map` は変わりません。

### map.remove

```text
pub fn remove<K: Ord, V>(map: Map<K, V>, key: K) -> Map<K, V> {
  ensures size(map: result) <= size(map: map)
    and size(map: result) + 1 >= size(map: map);
  ...
}
```

`key` の要素を取り除きます。`key` が無ければ、`map` と等しいマップを返します（欠陥にはなりません）。結果の要素数は、`map` と同じか 1 少なくなります。

### map.get と map.has

```text
pub fn get<K: Ord, V>(map: Map<K, V>, key: K) -> Option<V>
pub fn has<K: Ord, V>(map: Map<K, V>, key: K) -> Bool
```

`get` は `key` に対応する値を `Some(value: ...)` で返し、無ければ `None` を返します。値が要らず、あるかないかだけを知りたいときは `has` を使います。

### map.size

```text
pub fn size<K: Ord, V>(map: Map<K, V>) -> Int {
  ensures result >= 0;
  ...
}
```

要素数を返します。`entries` の結果の長さを調べる場合と違い、リストを作る必要がありません。

### map.entries

```text
pub fn entries<K: Ord, V>(map: Map<K, V>) -> List<Entry<K, V>>
```

すべての要素を、キーの昇順に並べたリストにします。順序は `Ord` の順序です。`Int` なら数の大小、`String` ならコードポイントの辞書式順序です（[03 型](../language/03-types.md)）。追加した順序は関係ありません。各キーは 1 回だけ現れます。

### map.first と map.last

```text
pub fn first<K: Ord, V>(map: Map<K, V>) -> Option<Entry<K, V>>
pub fn last<K: Ord, V>(map: Map<K, V>) -> Option<Entry<K, V>>
```

`first` はキーが最小の要素、`last` はキーが最大の要素を、`Some(value: Entry(key: ..., value: ...))` で返します。マップが空なら、どちらも `None` です。`entries` の先頭・末尾と同じ要素を返しますが、一覧を作らずに済みます。

## 例

```cernith
module guide.maps;

import std.map;

pub fn by_number() -> map.Map<Int, String> {
  map.insert(
    map: map.insert(
      map: map.insert(map: map.empty(), key: 10, value: "ten"),
      key: 9,
      value: "nine",
    ),
    key: 2,
    value: "two",
  )
}

example entries_are_in_key_order for by_number {
  map.entries(map: by_number())
    == [
      map.Entry(key: 2, value: "two"),
      map.Entry(key: 9, value: "nine"),
      map.Entry(key: 10, value: "ten"),
    ]
}

pub fn prices() -> map.Map<String, Int> {
  map.insert(
    map: map.insert(
      map: map.insert(map: map.empty(), key: "pear", value: 120),
      key: "apple",
      value: 80,
    ),
    key: "melon",
    value: 400,
  )
}

example empty_has_no_entries for prices {
  map.size(map: map.empty::<String, Int>()) == 0
    and map.entries(map: map.empty::<String, Int>()) == []
    and map.first(map: map.empty::<String, Int>()) == None
    and map.last(map: map.empty::<String, Int>()) == None
}

example first_and_last_are_the_ends_of_the_keys for prices {
  map.first(map: prices()) == Some(value: map.Entry(key: "apple", value: 80))
    and map.last(map: prices())
      == Some(value: map.Entry(key: "pear", value: 120))
}

example get_present_and_absent for prices {
  map.get(map: prices(), key: "pear") == Some(value: 120)
    and map.get(map: prices(), key: "fig") == None
}

example has_present_and_absent for prices {
  map.has(map: prices(), key: "melon")
    and not map.has(map: prices(), key: "fig")
}

example insert_adds_a_new_key for prices {
  map.size(map: map.insert(map: prices(), key: "fig", value: 300)) == 4
    and map.size(map: prices()) == 3
}

example insert_replaces_an_existing_key for prices {
  map.size(map: map.insert(map: prices(), key: "pear", value: 150)) == 3
    and map.get(
      map: map.insert(map: prices(), key: "pear", value: 150),
      key: "pear",
    )
      == Some(value: 150)
}

example maps_with_the_same_entries_are_equal for prices {
  prices()
    == map.insert(
      map: map.insert(
        map: map.insert(map: map.empty(), key: "melon", value: 400),
        key: "apple",
        value: 80,
      ),
      key: "pear",
      value: 120,
    )
    and prices() != map.insert(map: prices(), key: "pear", value: 150)
}

example remove_present_and_absent for prices {
  map.size(map: map.remove(map: prices(), key: "pear")) == 2
    and not map.has(map: map.remove(map: prices(), key: "pear"), key: "pear")
    and map.remove(map: prices(), key: "fig") == prices()
}

example size_counts_the_entries for prices {
  map.size(map: prices()) == 3
}
```

キーに使えない型を指定した例です。

```cernith-error E0316
module guide.map_keys;

import std.map;

pub fn by_list() -> map.Map<List<Int>, Int> {
  map.empty()
}
```

## 関連

- [set](set.md)：キーだけを持つ集合 `Set<T>`。`std.map` の上に作られています
- [list](list.md)：`entries` が返す `List`
- [option](option.md)：`get`、`first`、`last` が返す `Option`
- [03 型](../language/03-types.md)：`Eq` と `Ord`
