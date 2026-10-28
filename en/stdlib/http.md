# std.http

This module sends HTTP requests to other hosts. Import it with `import std.http;` and call functions such as `http.get(access: ..., url: ...)`. `get` and `get_bytes` send GET requests and return the body. `send` accepts a method, headers, and body, and also returns the response status and headers. Each call waits for the response.

Using HTTP requires the capability `Http`. A program asks for `Http` in its `manifest` declaration, together with **a list of host names that may be connected to**. It is passed as the second argument of `main`. It cannot be created anywhere else (error `E1301`). A function that receives an `Http` also declares `uses io` ([17 Resources and capabilities](../language/17-resources-and-capabilities.md)).

```text
pub manifest Caps {
  net: http.Http(hosts: ["api.example.com"]),
}
```

Depending on the execution environment, real HTTP requests may not be available. In that case, `get`, `get_bytes`, and `send` return `Err(error: HttpError.Unsupported)`. Real requests work on the interpreter, on Python, on Wasm under Wasmtime, and in native code on Windows; JavaScript and Wasm on Node answer `Unsupported`.

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
  BadRequest(detail: String),
}

pub record Header {
  name: String,
  value: String,
}

pub variant Method {
  Get,
  Head,
  Post,
  Put,
  Patch,
  Delete,
  Options,
}

pub record Request {
  method: Method,
  url: String,
  headers: List<Header>,
  body: Bytes,
  timeout_ms: Int,
}

pub record Response {
  status: Int,
  headers: List<Header>,
  body: Bytes,
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
| `BadRequest(detail: ...)` | `send` refused the request before sending it. `detail` says what is wrong |

`Header`, `Method`, `Request`, and `Response` are used by `send`: a `Request` has a `method`, a `url`, a list of `headers`, a `body`, and a time limit `timeout_ms` in milliseconds; a `Response` has the `status` code, the `headers`, and the `body` of the answer.

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
| `http.send(access, request)` | `Result<Response, HttpError>` | Sends a request with any method, headers, and body; any status is a `Response` |
| `http.new_request(method, url)` | `Request` | A request with no headers, no body, and a time limit of 30 seconds |
| `http.header_value(response, name)` | `Option<String>` | The value of the first header with the name (the case is ignored) |
| `http.method_name(method)` | `String` | The word of the method, such as `"GET"` |
| `http.allows(access, url)` | `Bool` | Whether the host of `url` is permitted |
| `http.host_of(url)` | `String` | The host name of a URL |
| `http.describe(error)` | `String` | Turns an `HttpError` into a string |
| `http.simulated()` | `Http` | For testing. Permits all hosts |
| `http.simulated_with(hosts)` | `Http` | For testing. Permits only the specified hosts |

`get`, `get_bytes`, and `send` require `uses io`. The others are pure.

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

Non-success status codes become errors so that a response such as a 404 page is not mistaken for the expected data.

### http.get_bytes

```text
pub fn get_bytes(access: Http, url: String) -> Result<Bytes, HttpError> uses io
```

Returns the body as `Bytes`, under the same rules as `get`. Use this for a body that is not text, such as an image or a compressed file.

### http.send

```text
pub fn send(
  access: Http,
  request: Request,
) -> Result<Response, HttpError> uses io
```

Sends `request` and returns the response, **whatever its status is**: a 404 or a 500 is a `Response`, not an error, and the program decides what to do with `response.status`. (`get` turns a status outside 200 to 299 into `Status`.)

1. If the host of `request.url` is not permitted, the result is `Err(error: HttpError.Denied(host: ...))`.
2. If the request is not valid, the result is `Err(error: HttpError.BadRequest(detail: ...))`, and nothing is sent. The following are refused:
   - an address that does not start with `http://` or `https://`, or that has a space or a control character;
   - `timeout_ms` outside the range from 1 to 300000;
   - a body of more than 16 MiB;
   - a body on a `Get` or `Head` request;
   - a header name that is not a token (letters, digits, and the marks `!#$%&'*+-.^_`|~`);
   - the headers `Host`, `Content-Length`, `Transfer-Encoding`, and `Connection`, which the host sets itself (the names are compared without regard to case);
   - a header value that contains a line break (this prevents header injection).
3. Otherwise the request is sent. If no response is obtained, the result is `Err(error: HttpError.Network(detail: ...))`; `detail` is `"timeout"` when the time limit passed and `"certificate"` when the check of the certificate failed (the check cannot be switched off). A body of more than 64 MiB in the answer is also a `Network` failure. Where the target cannot make the request, the result is `Err(error: HttpError.Unsupported)`.

**Redirects are not followed automatically.** A 3xx response is returned with its `Location` header. The program must issue a new request, which checks the destination against the permitted hosts again.

### http.new_request

```text
pub fn new_request(method: Method, url: String) -> Request
```

Returns a request with no headers, an empty body, and `timeout_ms` of 30000. Build the other requests by writing out the `Request` record.

### http.header_value

```text
pub fn header_value(response: Response, name: String) -> Option<String>
```

Returns the value of the first header of the response whose name is `name`, or `None`. Names are compared without regard to case, so `"content-type"` finds `Content-Type`.

### http.method_name

```text
pub fn method_name(method: Method) -> String
```

Returns the word of the method: `"GET"`, `"HEAD"`, `"POST"`, `"PUT"`, `"PATCH"`, `"DELETE"`, or `"OPTIONS"`.

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
| `BadRequest(detail: "d")` | `"bad request: d"` |

### http.simulated and http.simulated_with

```text
pub fn simulated() -> Http
pub fn simulated_with(hosts: List<String>) -> Http
```

Return an `Http` for testing. `simulated()` is the same as `hosts: ["*"]` and permits all hosts. `simulated_with(hosts: ...)` lets a test choose the hosts that are permitted. A test can check that a request to a host that is not permitted results in `Denied`.

Both can be called only inside an `example` or a `property`, and inside a function passed to `std.test.with_world` or `std.sim.with_sim`. Calling them anywhere else is error `E1302`.

In the test world, responses come from the routes written in the `setup` of `std.sim.with_sim` (`sim.route` and `sim.route_bytes`; see [sim](sim.md)). If there is a route whose URL matches exactly, the world responds with its status code and body. A URL with no route results in `Network(detail: "network")`.

For `send`, the response is chosen by `sim.route_for(method, url, ...)`, which names the method; a `GET` that has no such route falls back to the plain routes. The requests that the run sent with `send` are listed by `sim.requests_made` ([sim](sim.md)).

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

### Sending a request

The following module sends a POST request with headers and a body, reads a response header, and tests requests that `send` rejects. In the simulated world, `sim.route_for` supplies responses and `sim.requests_made` records the requests sent.

```cernith
module guide.http_send;

import std.bytes;
import std.http;
import std.option;
import std.sim;

pub fn ask(
  access: http.Http,
  request: http.Request,
) -> Result<Int, http.HttpError> uses io {
  let response = http.send(access: access, request: request)?;
  Ok(value: response.status)
}

example a_get_falls_back_to_the_plain_routes for ask {
  simulate(
    setup: [
      sim.route(url: "https://api.example.com/", status: 200, body: "hello"),
    ],
    body: fn(access: http.Http) -> List<Result<Int, http.HttpError>> uses io {
      [
        ask(
          access: access,
          request: http.new_request(
            method: http.Method.Get,
            url: "https://api.example.com/",
          ),
        ),
        ask(
          access: access,
          request: http.new_request(
            method: http.Method.Delete,
            url: "https://api.example.com/",
          ),
        ),
      ]
    },
  ).value
    == [Ok(value: 200), Err(error: http.HttpError.Network(detail: "network"))]
}

example a_request_is_checked_before_it_is_sent for ask {
  match simulate(
    setup: [],
    body: fn(access: http.Http) -> List<Result<Int, http.HttpError>> uses io {
      let base = http.new_request(
        method: http.Method.Get,
        url: "https://api.example.com/",
      );
      [
        ask(access: http.simulated_with(hosts: ["other.org"]), request: base),
        ask(
          access: access,
          request: http.Request(
            method: http.Method.Get,
            url: base.url,
            headers: [],
            body: bytes.from_text(text: "x"),
            timeout_ms: 1000,
          ),
        ),
        ask(
          access: access,
          request: http.Request(
            method: http.Method.Get,
            url: base.url,
            headers: [http.Header(name: "Host", value: "evil.example")],
            body: bytes.empty(),
            timeout_ms: 1000,
          ),
        ),
        ask(
          access: access,
          request: http.Request(
            method: http.Method.Get,
            url: "ftp://api.example.com/",
            headers: [],
            body: bytes.empty(),
            timeout_ms: 1000,
          ),
        ),
        ask(
          access: access,
          request: http.Request(
            method: http.Method.Get,
            url: base.url,
            headers: [],
            body: bytes.empty(),
            timeout_ms: 0,
          ),
        ),
      ]
    },
  ) {
    outcome => outcome.value
      == [
        Err(error: http.HttpError.Denied(host: "api.example.com")),
        Err(
          error: http.HttpError.BadRequest(detail: "GET and HEAD have no body"),
        ),
        Err(
          error: http.HttpError.BadRequest(
            detail: "the host sets the header Host",
          ),
        ),
        Err(
          error: http.HttpError.BadRequest(
            detail: "the address is not http or https",
          ),
        ),
        Err(
          error: http.HttpError.BadRequest(
            detail: "timeout_ms is from 1 to 300000",
          ),
        ),
      ]
      and sim.requests_made(outcome: outcome) == [],
  }
}

pub fn create_item(
  access: http.Http,
  name: String,
) -> Result<String, http.HttpError> uses io {
  let request = http.Request(
    method: http.Method.Post,
    url: "https://api.example.com/items",
    headers: [
      http.Header(name: "Content-Type", value: "application/json"),
      http.Header(name: "X-Key", value: "k1"),
    ],
    body: bytes.from_text(text: "{\"name\": \"" ++ name ++ "\"}"),
    timeout_ms: 5000,
  );
  let response = http.send(access: access, request: request)?;
  if response.status == 201 {
    Ok(
      value: option.unwrap_or(
        option: http.header_value(response: response, name: "location"),
        default: "",
      ),
    )
  } else {
    Err(error: http.HttpError.Status(code: response.status))
  }
}

example create_item_reads_the_location_header for create_item {
  match simulate(
    setup: [
      sim.route_for(
        method: "POST",
        url: "https://api.example.com/items",
        status: 201,
        headers: ["Location: /items/7"],
        body: bytes.empty(),
      ),
    ],
    body: fn(access: http.Http) -> Result<String, http.HttpError> uses io {
      create_item(access: access, name: "pen")
    },
  ) {
    outcome => outcome.value == Ok(value: "/items/7")
      and sim.requests_made(outcome: outcome)
        == [
          sim.SentRequest(
            method: "POST",
            url: "https://api.example.com/items",
            headers: ["Content-Type: application/json", "X-Key: k1"],
            body: bytes.from_text(text: "{\"name\": \"pen\"}"),
          ),
        ],
  }
}

example create_item_turns_other_statuses_into_errors for create_item {
  simulate(
    setup: [
      sim.route_for(
        method: "POST",
        url: "https://api.example.com/items",
        status: 409,
        headers: [],
        body: bytes.from_text(text: "exists"),
      ),
    ],
    body: fn(access: http.Http) -> Result<String, http.HttpError> uses io {
      create_item(access: access, name: "pen")
    },
  ).value
    == Err(error: http.HttpError.Status(code: 409))
    and simulate(
      setup: [],
      body: fn(access: http.Http) -> Result<String, http.HttpError> uses io {
        create_item(access: access, name: "pen")
      },
    ).value
      == Err(error: http.HttpError.Network(detail: "network"))
}

fn simulate<T>(
  setup: List<String>,
  body: fn(access: http.Http) -> T uses io,
) -> sim.SimOutcome<T> {
  sim.with_sim(
    world: sim.quiet_world(),
    setup: setup,
    run: fn() -> T uses clock, io, random {
      body(access: http.simulated())
    },
  )
}

pub fn verb(method: http.Method) -> String {
  http.method_name(method: method)
}

example verb_is_the_word_of_http for verb {
  verb(method: http.Method.Get) == "GET"
    and verb(method: http.Method.Head) == "HEAD"
    and verb(method: http.Method.Post) == "POST"
    and verb(method: http.Method.Put) == "PUT"
    and verb(method: http.Method.Patch) == "PATCH"
    and verb(method: http.Method.Delete) == "DELETE"
    and verb(method: http.Method.Options) == "OPTIONS"
}
```

## Related

- [console](console.md): the capability `Console`
- [sim](sim.md): builds a world that has responses, using `route` and `route_bytes`
- [fs](fs.md), [env](env.md): other capabilities
- [bytes](bytes.md): the functions of `Bytes`
- [17 Resources and capabilities](../language/17-resources-and-capabilities.md)
