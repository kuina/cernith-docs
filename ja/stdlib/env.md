# std.env

プロセスの環境変数を、ケイパビリティ `Env` を通して読むモジュールです。`import std.env;` と書き、`env.get(access: ..., name: ...)` のように呼びます。読む操作だけで、環境変数を設定する関数はありません。

`Env` は、プログラムの `manifest` 宣言に書くことで、`main` の第 2 引数として渡されます。ほかの場所では作れません（エラー `E1301`）。`Env` を受け取る関数は、`uses io` も宣言します（[17 リソースとケイパビリティ](../language/17-resources-and-capabilities.md)）。

## 型

```text
pub capability Env;
```

`Env` は、フィールドを持たないケイパビリティです。許可の中身（grant）はなく、持っていることだけが意味を持ちます。`==` では比較できません。

## 関数一覧

| 関数 | 戻り値の型 | 意味 |
| --- | --- | --- |
| `env.get(access, name)` | `Option<String>` | 環境変数 `name` の値。設定されていなければ `None` |
| `env.simulated()` | `Env` | テスト用のケイパビリティ |

## 関数の詳細

### env.get

```text
pub fn get(access: Env, name: String) -> Option<String> uses io
```

環境変数 `name` が設定されていれば `Some(value: 値)`、設定されていなければ `None` を返します。値が空文字列として設定されているときは `Some(value: "")` で、`None` とは区別されます。

`Result` ではなく `Option` を返すのは、環境変数が「無い」ことは普通の状況で、理由を伝える必要がないからです。

事前条件はありません。`name` には任意の文字列を渡せます。

### env.simulated

```text
pub fn simulated() -> Env
```

テストのための `Env` を返します。`example` と `property` の中、および `std.test.with_world` や `std.sim.with_sim` に渡す関数の中でだけ呼べます。ほかの場所で呼ぶとエラー `E1302` です。

テストの世界の環境変数は、`std.sim.with_sim` の `setup` に `sim.env_var(name: ..., value: ...)` を書いて与えます（[sim](sim.md)）。`std.test.with_world` の世界には環境変数がなく、`env.get` は常に `None` を返します。

```cernith-error E1302
module guide.env_simulated_outside_a_test;

import std.env;

pub fn home() -> Option<String> uses io {
  env.get(access: env.simulated(), name: "HOME")
}
```

## 例

```cernith
module guide.env_tools;

import std.console;
import std.env;
import std.option;
import std.sim;
import std.test;

pub manifest Caps {
  console: console.Console,
  env: env.Env,
}

pub fn greeting(access: env.Env) -> String uses io {
  "hello "
    ++ option.unwrap_or(
      option: env.get(access: access, name: "GREETING_NAME"),
      default: "world",
    )
}

example greeting_falls_back_when_it_is_not_set for greeting {
  greeting_with(setup: []) == "hello world"
    and greeting_with(setup: [sim.env_var(name: "OTHER", value: "x")])
      == "hello world"
    and test.with_world(
      world: sim.quiet_world(),
      run: fn() -> String uses clock, io, random {
        greeting(access: env.simulated())
      },
    ).value
      == "hello world"
}

example greeting_uses_the_variable_when_it_is_set for greeting {
  greeting_with(setup: [sim.env_var(name: "GREETING_NAME", value: "Ada")])
    == "hello Ada"
    and greeting_with(setup: [sim.env_var(name: "GREETING_NAME", value: "")])
      == "hello "
}

fn greeting_with(setup: List<String>) -> String {
  sim.with_sim(
    world: sim.quiet_world(),
    setup: setup,
    run: fn() -> String uses clock, io, random {
      greeting(access: env.simulated())
    },
  ).value
}

pub fn main(args: List<String>, caps: Caps) -> Int uses io {
  match console.write_line(
    access: caps.console,
    text: greeting(access: caps.env),
  ) {
    Ok(value: _) => 0,
    Err(error: _) => 1,
  }
}

example main_writes_the_greeting for main {
  sim.with_sim(
    world: sim.quiet_world(),
    setup: [sim.env_var(name: "GREETING_NAME", value: "Ada")],
    run: fn() -> Int uses clock, io, random {
      main(
        args: [],
        caps: Caps(console: console.simulated(), env: env.simulated()),
      )
    },
  ).output
    == ["hello Ada"]
}
```

## 関連

- [console](console.md)：ケイパビリティ `Console`
- [sim](sim.md)：`env_var` で環境変数を持つ世界を作る
- [fs](fs.md)、[http](http.md)：ほかのケイパビリティ
- [17 リソースとケイパビリティ](../language/17-resources-and-capabilities.md)
