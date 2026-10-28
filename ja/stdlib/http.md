# std.http

ほかのホストに HTTP リクエストを送るモジュールです。`import std.http;` で読み込み、`http.get(access: ..., url: ...)` のように呼び出します。`get` と `get_bytes` は GET リクエストの応答本文を返します。`send` ではメソッド・ヘッダー・本文を指定でき、応答のステータスとヘッダーも取得できます。いずれも応答が返るまで待機します。

HTTP を使うには、ケイパビリティ `Http` が必要です。`Http` は、プログラムの `manifest` 宣言で、**接続してよいホスト名のリスト**を付けて求めます。`main` の第 2 引数として渡されます。ほかの場所では作れません（エラー `E1301`）。`Http` を受け取る関数は、`uses io` も宣言します（[17 リソースとケイパビリティ](../language/17-resources-and-capabilities.md)）。

```text
pub manifest Caps {
  net: http.Http(hosts: ["api.example.com"]),
}
```

実行環境によっては、実際の HTTP リクエストが使えません。その場合の `get`、`get_bytes`、`send` は、`Err(error: HttpError.Unsupported)` を返します。実際のリクエストは、インタープリタ、Python、Wasmtime 上の Wasm、Windows のネイティブコードで動きます。JavaScript と Node 上の Wasm は `Unsupported` を返します。

## 型

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

- `Http` は、`hosts` に許可するホスト名を持つケイパビリティです。`hosts` の要素 `"*"` は、すべてのホストを許可します（テスト用です）。`Http` の値を `==` で比べることはできません（エラー `E0316`）。`hosts` を直接読むこともできません（エラー `E0215`）。許可の確認には `http.allows` を使います。
- `HttpError` は、リクエストが成功しなかった理由です。

| ケース | 意味 |
| --- | --- |
| `Network(detail: ...)` | 接続や通信に失敗した。`detail` は理由の文字列 |
| `Status(code: ...)` | 応答は得られたが、ステータスコードが 200 から 299 の範囲にない（`code` がそのコード） |
| `Unsupported` | この実行環境では、実際のリクエストが使えない |
| `Denied(host: ...)` | URL のホストが、許可されたホストに含まれない。リクエストは送られない |
| `BadRequest(detail: ...)` | `send` が、送る前にリクエストを拒否した。`detail` は何が悪いかを表す |

`Header`、`Method`、`Request`、`Response` は `send` が使います。`Request` は `method`、`url`、`headers` のリスト、`body`、ミリ秒の制限時間 `timeout_ms` を持ち、`Response` は応答の `status` コード、`headers`、`body` を持ちます。

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

## 関数一覧

| 関数 | 戻り値の型 | 意味 |
| --- | --- | --- |
| `http.get(access, url)` | `Result<String, HttpError>` | `url` の応答の本文をテキストで返す |
| `http.get_bytes(access, url)` | `Result<Bytes, HttpError>` | 応答の本文をバイト列で返す |
| `http.send(access, request)` | `Result<Response, HttpError>` | 任意のメソッド、ヘッダー、本文でリクエストを送る。どのステータスでも `Response` になる |
| `http.new_request(method, url)` | `Request` | ヘッダーも本文もなく、制限時間が 30 秒のリクエスト |
| `http.header_value(response, name)` | `Option<String>` | その名前の最初のヘッダーの値（大文字と小文字は区別しない） |
| `http.method_name(method)` | `String` | `"GET"` のような、メソッドを表す語 |
| `http.allows(access, url)` | `Bool` | `url` のホストが許可されているか |
| `http.host_of(url)` | `String` | URL のホスト名 |
| `http.describe(error)` | `String` | `HttpError` を文字列にする |
| `http.simulated()` | `Http` | テスト用。すべてのホストを許可する |
| `http.simulated_with(hosts)` | `Http` | テスト用。指定したホストだけを許可する |

`get`、`get_bytes`、`send` は、`uses io` が必要です。ほかは純粋です。

## 関数の詳細

### http.get

```text
pub fn get(access: Http, url: String) -> Result<String, HttpError> uses io
```

`url` に GET リクエストを送り、本文を返します。

1. `url` のホスト（`http.host_of(url)`）が許可されていなければ、リクエストを送らずに `Err(error: HttpError.Denied(host: ...))` を返します。
2. 応答のステータスコードが 200 以上 299 以下なら、`Ok(value: 本文)` です。
3. ほかのステータスコードなら、`Err(error: HttpError.Status(code: ...))` です。本文は返りません。
4. 接続できないなど、応答が得られなければ `Err(error: HttpError.Network(detail: ...))`、実行環境が対応していなければ `Err(error: HttpError.Unsupported)` です。

成功以外のステータスコードをエラーにすることで、404 ページなどの本文を、期待したデータと取り違えて処理するのを防ぎます。

### http.get_bytes

```text
pub fn get_bytes(access: Http, url: String) -> Result<Bytes, HttpError> uses io
```

`get` と同じ規則で、本文を `Bytes` で返します。画像や圧縮ファイルのように、テキストでない本文には、こちらを使います。

### http.send

```text
pub fn send(
  access: Http,
  request: Request,
) -> Result<Response, HttpError> uses io
```

`request` を送り、**ステータスにかかわらず**応答を返します。404 や 500 もエラーではなく `Response` になり、`response.status` をどう扱うかはプログラムが決めます。（`get` は、200 から 299 の範囲にないステータスを `Status` にします。）

1. `request.url` のホストが許可されていなければ、結果は `Err(error: HttpError.Denied(host: ...))` です。
2. リクエストが正しくなければ、結果は `Err(error: HttpError.BadRequest(detail: ...))` で、何も送られません。次のものは拒否されます。
   - `http://` か `https://` で始まらないアドレス、または空白や制御文字を含むアドレス
   - 1 から 300000 の範囲にない `timeout_ms`
   - 16 MiB を超える本文
   - `Get` と `Head` のリクエストにある本文
   - トークンではないヘッダー名（使えるのは英数字と記号 `!#$%&'*+-.^_`|~`）
   - ホストが自分で設定するヘッダー `Host`、`Content-Length`、`Transfer-Encoding`、`Connection`（名前は大文字と小文字を区別せずに比べます）
   - 改行を含むヘッダーの値（ヘッダーインジェクションを防ぎます）
3. それ以外は送られます。応答が得られなければ、結果は `Err(error: HttpError.Network(detail: ...))` です。制限時間を過ぎたときの `detail` は `"timeout"`、証明書の検査に失敗したときは `"certificate"` です（この検査は切れません）。応答の本文が 64 MiB を超えるときも `Network` の失敗です。対象がリクエストを出せないときは、`Err(error: HttpError.Unsupported)` です。

**リダイレクトは自動ではたどりません。** 3xx の応答は、`Location` ヘッダーとともにそのまま返されます。移動先に接続するには、プログラムから改めてリクエストを送ります。その際、移動先も許可されたホストかどうか検査されます。

### http.new_request

```text
pub fn new_request(method: Method, url: String) -> Request
```

ヘッダーなし、本文が空、`timeout_ms` が 30000 のリクエストを返します。それ以外のリクエストは、`Request` の各フィールドを指定して作成します。

### http.header_value

```text
pub fn header_value(response: Response, name: String) -> Option<String>
```

応答のヘッダーのうち、名前が `name` である最初のものの値を返します。なければ `None` です。名前は大文字と小文字を区別せずに比べるので、`"content-type"` で `Content-Type` が見つかります。

### http.method_name

```text
pub fn method_name(method: Method) -> String
```

メソッドを表す語を返します。`"GET"`、`"HEAD"`、`"POST"`、`"PUT"`、`"PATCH"`、`"DELETE"`、`"OPTIONS"` のいずれかです。

### http.allows

```text
pub fn allows(access: Http, url: String) -> Bool
```

`url` のホストが、許可されているかどうかを返します。`hosts` に `"*"` があるか、`hosts` のどれかが `http.host_of(url)` と等しければ `true` です。比較は完全一致で、大文字小文字は区別され、サブドメインは含まれません（`hosts: ["a.org"]` は `sub.a.org` を許可しません）。`get` と `get_bytes` は、これで確かめてからリクエストを送ります。

### http.host_of

```text
pub fn host_of(url: String) -> String
```

URL のホスト名を返します。`://` のあとから、最初に現れる `/`、`:`、`?`、`#` の手前まで（どれもなければ最後まで）の部分です。ポート番号は含みません。`://` を含まない文字列では `""` です。

### http.describe

```text
pub fn describe(error: HttpError) -> String
```

| エラー | 文字列 |
| --- | --- |
| `Network(detail: "x")` | `"network: x"` |
| `Status(code: 404)` | `"status 404"` |
| `Unsupported` | `"unsupported here"` |
| `Denied(host: "h")` | `"denied: h"` |
| `BadRequest(detail: "d")` | `"bad request: d"` |

### http.simulated と http.simulated_with

```text
pub fn simulated() -> Http
pub fn simulated_with(hosts: List<String>) -> Http
```

テストのための `Http` を返します。`simulated()` は `hosts: ["*"]` と同じで、すべてのホストを許可します。`simulated_with(hosts: ...)` は、テストが許可するホストを選びます。許可されないホストへのリクエストが `Denied` になることを、テストで確かめられます。

どちらも、`example` と `property` の中、および `std.test.with_world` や `std.sim.with_sim` に渡す関数の中でだけ呼べます。ほかの場所で呼ぶとエラー `E1302` です。

テストの世界での応答は、`std.sim.with_sim` の `setup` に書いたルート（`sim.route`、`sim.route_bytes`）から返ります（[sim](sim.md)）。URL が完全に一致するルートがあれば、そのステータスコードと本文で応答します。ルートがない URL は、`Network(detail: "network")` になります。

`send` の応答は、メソッドを指定する `sim.route_for(method, url, ...)` で設定します。対応するルートがない `GET` は、通常のルートにフォールバックします。実行中に `send` で送信したリクエストは、`sim.requests_made` で一覧できます（[sim](sim.md)）。

## 例

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

### リクエストを送る

次のモジュールでは、ヘッダーと本文を含む POST リクエストを送り、応答ヘッダーを読み取ります。また、`send` が不正なリクエストを拒否することも検査します。シミュレート世界では、`sim.route_for` で応答を設定し、`sim.requests_made` で送信内容を確認します。

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

## 関連

- [console](console.md)：ケイパビリティ `Console`
- [sim](sim.md)：`route` と `route_bytes` で応答を持つ世界を作る
- [fs](fs.md)、[env](env.md)：ほかのケイパビリティ
- [bytes](bytes.md)：`Bytes` の関数
- [17 リソースとケイパビリティ](../language/17-resources-and-capabilities.md)
