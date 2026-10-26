# std.http

A module that fetches content from other hosts over HTTP. Write `import std.http;` and call functions as in `http.get(access: ..., url: ...)`. Only GET requests are supported, and a call waits until the response returns.

Using HTTP requires the capability `Http`. A program asks for `Http` in its `manifest` declaration, together with **a list of host names that may be connected to**. It is passed as the second argument of `main`. It cannot be created anywhere else (error `E1301`). A function that receives an `Http` also declares `uses io` ([17 Resources and capabilities](../language/17-resources-and-capabilities.md)).

```text
pub manifest Caps {
  net: http.Http(hosts: ["api.example.com"]),
}
```

Depending on the execution environment, real HTTP requests may not be available. In that case, `get` and `get_bytes` return `Err(error: HttpError.Unsupported)`.

## Types

```text
pub capability Http {
  hosts: List<String>,
}

pub variant HttpError {
  Network(detail: String),
  Status(code: Int),
  Unsupported,
  Denied(host: String),
}
```

- `Http` is a capability that holds the permitted host names in `hosts`. The element `"*"` in `hosts` permits all hosts (this is for testing). Values of `Http` cannot be compared with `==` (error `E0316`). `hosts` cannot be read directly either (error `E0215`). Use `http.allows` to check a permission.
- `HttpError` is the reason a request did not succeed.

| Case | Meaning |
| --- | --- |
| `Network(detail: ...)` | The connection or the communication failed. `detail` is a string giving the reason |
| `Status(code: ...)` | A response was obtained, but its status code is not in the range from 200 to 299 (`code` is that code) |
| `Unsupported` | Real requests are not available in this execution environment |
| `Denied(host: ...)` | The host of the URL is not among the permitted hosts. The request is not sent |

```cernith-error E0215
module guide.http_reads_the_grant;

import std.http;

pub fn granted(access: http.Http) -> List<String> {
  access.hosts
}
```

```cernith-error E1301
module guide.http_made_by_hand;

import std.http;

pub fn open_access() -> http.Http {
  http.Http(hosts: ["*"])
}
```

## Function list

| Function | Return type | Meaning |
| --- | --- | --- |
| `http.get(access, url)` | `Result<String, HttpError>` | Returns the body of the response from `url` as text |
| `http.get_bytes(access, url)` | `Result<Bytes, HttpError>` | Returns the body of the response as bytes |
| `http.allows(access, url)` | `Bool` | Whether the host of `url` is permitted |
| `http.host_of(url)` | `String` | The host name of a URL |
| `http.describe(error)` | `String` | Turns an `HttpError` into a string |
| `http.simulated()` | `Http` | For testing. Permits all hosts |
| `http.simulated_with(hosts)` | `Http` | For testing. Permits only the specified hosts |

`get` and `get_bytes` require `uses io`. The others are pure.

## Function details

### http.get

```text
pub fn get(access: Http, url: String) -> Result<String, HttpError> uses io
```

Sends a GET request to `url` and returns the body.

1. If the host of `url` (`http.host_of(url)`) is not permitted, it returns `Err(error: HttpError.Denied(host: ...))` without sending the request.
2. If the status code of the response is from 200 to 299 inclusive, the result is `Ok(value: body)`.
3. For any other status code, the result is `Err(error: HttpError.Status(code: ...))`. The body is not returned.
4. If no response is obtained, for example because the connection cannot be made, the result is `Err(error: HttpError.Network(detail: ...))`; if the execution environment does not support requests, it is `Err(error: HttpError.Unsupported)`.

The reason a status code outside the success range is turned into an error is to prevent the body from being mistaken for a success and used. The body of a 404 page is not processed as the data that was expected.

### http.get_bytes

```text
pub fn get_bytes(access: Http, url: String) -> Result<Bytes, HttpError> uses io
```

Returns the body as `Bytes`, under the same rules as `get`. Use this for a body that is not text, such as an image or a compressed file.

### http.allows

```text
pub fn allows(access: Http, url: String) -> Bool
```

Returns whether the host of `url` is permitted. The result is `true` if `hosts` contains `"*"` or if any element of `hosts` is equal to `http.host_of(url)`. The comparison is an exact match, is case-sensitive, and does not include subdomains (`hosts: ["a.org"]` does not permit `sub.a.org`). `get` and `get_bytes` check with this before sending the request.

### http.host_of

```text
pub fn host_of(url: String) -> String
```

Returns the host name of a URL. It is the part after `://` up to just before the first `/`, `:`, `?`, or `#` that appears (or to the end if there is none of them). It does not include the port number. For a string that does not contain `://`, the result is `""`.

### http.describe

```text
pub fn describe(error: HttpError) -> String
```

| Error | String |
| --- | --- |
| `Network(detail: "x")` | `"network: x"` |
| `Status(code: 404)` | `"status 404"` |
| `Unsupported` | `"unsupported here"` |
| `Denied(host: "h")` | `"denied: h"` |

### http.simulated and http.simulated_with

```text
pub fn simulated() -> Http
pub fn simulated_with(hosts: List<String>) -> Http
```

Return an `Http` for testing. `simulated()` is the same as `hosts: ["*"]` and permits all hosts. `simulated_with(hosts: ...)` lets a test choose the hosts that are permitted. A test can check that a request to a host that is not permitted results in `Denied`.

Both can be called only inside an `example` or a `property`, and inside a function passed to `std.test.with_world` or `std.sim.with_sim`. Calling them anywhere else is error `E1302`.

In the test world, responses come from the routes written in the `setup` of `std.sim.with_sim` (`sim.route` and `sim.route_bytes`; see [sim](sim.md)). If there is a route whose URL matches exactly, the world responds with its status code and body. A URL with no route results in `Network(detail: "network")`.

## Example

```cernith
module guide.fetch;

import std.bytes;
import std.console;
import std.http;
import std.int;
import std.sim;

pub manifest Caps {
  console: console.Console,
  net: http.Http(hosts: ["api.example.com"]),
}

fn bytes_of(count: Int) -> Bytes {
  requires count >= 0 and count <= 1000;

  bytes.new(count: count, fill: 7)
}

pub fn explain(error: http.HttpError) -> String {
  http.describe(error: error)
}

example explain_names_each_failure for explain {
  explain(error: http.HttpError.Network(detail: "refused"))
    == "network: refused"
    and explain(error: http.HttpError.Status(code: 404)) == "status 404"
    and explain(error: http.HttpError.Unsupported) == "unsupported here"
    and explain(error: http.HttpError.Denied(host: "other.org"))
      == "denied: other.org"
}

pub fn host(url: String) -> String {
  http.host_of(url: url)
}

example host_is_between_the_scheme_and_the_next_mark for host {
  host(url: "https://api.example.com/v1?x=1") == "api.example.com"
    and host(url: "http://localhost:8080/") == "localhost"
    and host(url: "https://a.org?x=1") == "a.org"
    and host(url: "https://a.org#top") == "a.org"
    and host(url: "https://a.org") == "a.org"
    and host(url: "no-scheme") == ""
    and host(url: "") == ""
}

fn item_length_in(
  routes: List<String>,
  hosts: List<String>,
) -> Result<Int, http.HttpError> {
  sim.with_sim(
    world: sim.quiet_world(),
    setup: routes,
    run: fn() -> Result<Int, http.HttpError> uses clock, io, random {
      item_text_length(access: http.simulated_with(hosts: hosts))
    },
  ).value
}

pub fn item_text_length(
  access: http.Http,
) -> Result<Int, http.HttpError> uses io {
  let body = http.get(access: access, url: "https://api.example.com/v1/items")?;
  Ok(value: body.length)
}

example item_text_length_reads_the_body for item_text_length {
  item_length_in(
    routes: [
      sim.route(
        url: "https://api.example.com/v1/items",
        status: 200,
        body: "[1,2]",
      ),
    ],
    hosts: ["api.example.com"],
  )
    == Ok(value: 5)
    and item_length_in(
      routes: [
        sim.route(
          url: "https://api.example.com/v1/items",
          status: 204,
          body: "",
        ),
      ],
      hosts: ["*"],
    )
      == Ok(value: 0)
}

example item_text_length_reports_what_went_wrong for item_text_length {
  item_length_in(
    routes: [
      sim.route(
        url: "https://api.example.com/v1/items",
        status: 404,
        body: "missing",
      ),
    ],
    hosts: ["api.example.com"],
  )
    == Err(error: http.HttpError.Status(code: 404))
    and item_length_in(routes: [], hosts: ["api.example.com"])
      == Err(error: http.HttpError.Network(detail: "network"))
    and item_length_in(
      routes: [
        sim.route(
          url: "https://api.example.com/v1/items",
          status: 200,
          body: "[]",
        ),
      ],
      hosts: ["other.org"],
    )
      == Err(error: http.HttpError.Denied(host: "api.example.com"))
    and item_length_in(
      routes: [
        sim.route(
          url: "https://api.example.com/v1/items",
          status: 301,
          body: "",
        ),
      ],
      hosts: [],
    )
      == Err(error: http.HttpError.Denied(host: "api.example.com"))
}

pub fn main(args: List<String>, caps: Caps) -> Int uses io {
  match item_text_length(access: caps.net) {
    Ok(value: length) => {
      discard console.write_line(
        access: caps.console,
        text: int.to_text(value: length),
      );
      0
    },
    Err(error: error) => {
      discard console.write_error_line(
        access: caps.console,
        text: explain(error: error),
      );
      1
    },
  }
}

example main_writes_the_length for main {
  sim.with_sim(
    world: sim.quiet_world(),
    setup: [
      sim.route(
        url: "https://api.example.com/v1/items",
        status: 200,
        body: "[1,2]",
      ),
    ],
    run: fn() -> Int uses clock, io, random {
      main(
        args: [],
        caps: Caps(console: console.simulated(), net: http.simulated()),
      )
    },
  ).output
    == ["5"]
}

pub fn picture_size(access: http.Http) -> Result<Int, http.HttpError> uses io {
  let picture = http.get_bytes(
    access: access,
    url: "https://api.example.com/picture",
  )?;
  Ok(value: bytes.length(bytes: picture))
}

example picture_size_counts_the_bytes for picture_size {
  picture_size_in(
    routes: [
      sim.route_bytes(
        url: "https://api.example.com/picture",
        status: 200,
        body: bytes_of(count: 3),
      ),
    ],
  )
    == Ok(value: 3)
    and picture_size_in(
      routes: [
        sim.route_bytes(
          url: "https://api.example.com/picture",
          status: 200,
          body: bytes_of(count: 0),
        ),
      ],
    )
      == Ok(value: 0)
    and picture_size_in(
      routes: [
        sim.route_bytes(
          url: "https://api.example.com/picture",
          status: 500,
          body: bytes_of(count: 3),
        ),
      ],
    )
      == Err(error: http.HttpError.Status(code: 500))
    and picture_size_in(routes: [])
      == Err(error: http.HttpError.Network(detail: "network"))
}

fn picture_size_in(routes: List<String>) -> Result<Int, http.HttpError> {
  sim.with_sim(
    world: sim.quiet_world(),
    setup: routes,
    run: fn() -> Result<Int, http.HttpError> uses clock, io, random {
      picture_size(access: http.simulated())
    },
  ).value
}

pub fn reachable(access: http.Http, url: String) -> Bool {
  http.allows(access: access, url: url)
}

example reachable_follows_the_grant for reachable {
  reachable(
    access: http.simulated_with(hosts: ["a.org"]),
    url: "https://a.org/x",
  )
    and not reachable(
      access: http.simulated_with(hosts: ["a.org"]),
      url: "https://b.org/x",
    )
    and not reachable(
      access: http.simulated_with(hosts: ["a.org"]),
      url: "https://sub.a.org/x",
    )
    and not reachable(
      access: http.simulated_with(hosts: ["A.org"]),
      url: "https://a.org/x",
    )
    and reachable(access: http.simulated(), url: "https://b.org/x")
    and not reachable(
      access: http.simulated_with(hosts: []),
      url: "https://a.org",
    )
}
```

## Related

- [console](console.md): the capability `Console`
- [sim](sim.md): builds a world that has responses, using `route` and `route_bytes`
- [fs](fs.md), [env](env.md): other capabilities
- [bytes](bytes.md): the functions of `Bytes`
- [17 Resources and capabilities](../language/17-resources-and-capabilities.md)
