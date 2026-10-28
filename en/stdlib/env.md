# std.env

This module reads process environment variables through the `Env` capability. Import it with `import std.env;` and call functions such as `env.get(access: ..., name: ...)`. It provides read access only; there is no function for setting variables.

`Env` is passed as the second argument of `main` by listing it in the program’s `manifest` declaration. It cannot be created anywhere else (error `E1301`). A function that receives an `Env` also declares `uses io` ([17 Resources and capabilities](../language/17-resources-and-capabilities.md)).

## Types

```text
pub capability Env;
```

`Env` is a capability with no fields or grant contents. Possessing the value grants access. It does not support `==`.

## Function list

| Function | Return type | Meaning |
| --- | --- | --- |
| `env.get(access, name)` | `Option<String>` | The value of the environment variable `name`, or `None` if it is not set |
| `env.simulated()` | `Env` | A capability for testing |

## Function details

### env.get

```text
pub fn get(access: Env, name: String) -> Option<String> uses io
```

Returns `Some(value: value)` if the environment variable `name` is set, and `None` if it is not. When the variable is set to an empty string, the result is `Some(value: "")`, which is distinct from `None`.

It returns an `Option` rather than a `Result` because a missing environment variable is an ordinary situation and no reason needs to be conveyed.

There is no precondition. Any string can be passed as `name`.

### env.simulated

```text
pub fn simulated() -> Env
```

Returns an `Env` for tests. It can be called only inside an `example` or a `property`, and inside a function passed to `std.test.with_world` or `std.sim.with_sim`. Calling it anywhere else is error `E1302`.

The environment variables of a test world are given by writing `sim.env_var(name: ..., value: ...)` in the `setup` of `std.sim.with_sim` ([sim](sim.md)). The world of `std.test.with_world` has no environment variables, and `env.get` always returns `None` in it.

```cernith-error E1302
module guide.env_simulated_outside_a_test;

import std.env;

pub fn home() -> Option<String> uses io {
  env.get(access: env.simulated(), name: "HOME")
}
```

## Example

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

## Related

- [console](console.md): the capability `Console`
- [sim](sim.md): builds a world that has environment variables, using `env_var`
- [fs](fs.md), [http](http.md): other capabilities
- [17 Resources and capabilities](../language/17-resources-and-capabilities.md)
