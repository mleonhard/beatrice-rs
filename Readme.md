Servlin
========
[![crates.io version](https://img.shields.io/crates/v/servlin.svg)](https://crates.io/crates/servlin)
[![license: Apache 2.0](https://raw.githubusercontent.com/mleonhard/servlin/main/license-apache-2.0.svg)](http://www.apache.org/licenses/LICENSE-2.0)
[![unsafe forbidden](https://raw.githubusercontent.com/mleonhard/servlin/main/unsafe-forbidden-success.svg)](https://github.com/rust-secure-code/safety-dance/)
[![pipeline status](https://github.com/mleonhard/servlin/workflows/CI/badge.svg)](https://github.com/mleonhard/servlin/actions)

A modular HTTP server library in Rust.

# Features
- `forbid(unsafe_code)`
- Threaded request handlers:<br>
  `FnOnce(Request) -> Response + 'static + Clone + Send + Sync`
- Uses async code internally for excellent performance under load
- JSON
- Server-Sent Events (SSE)
- Saves large request bodies to temp files
- Sends 100-Continue
- Limits number of threads and connections
- Modular: roll your own logging, write custom versions of internal methods, etc.
- No macros or complicated type params
- Good test coverage (63%)

# Limitations
- New, not proven in production.
- To do:
  - Request timeouts
  - `chunked` transfer-encoding for request bodies
  - gzip
  - brotli
  - TLS
  - automatically getting TLS certs via ACME
  - Drop idle connections when approaching connection limit.
  - Denial-of-Service mitigation: source throttling, minimum throughput
  - Complete functional test suite
  - Missing load tests
  - Disk space usage limits

# Examples
Complete examples: [`examples/`](https://github.com/mleonhard/servlin/tree/main/examples).

Simple example:
```rust
use serde::Deserialize;
use serde_json::json;
use servlin::{
    print_log_response,
    socket_addr_127_0_0_1,
    HttpServerBuilder,
    Request,
    Response
};
use servlin::reexport::{safina_executor, safina_timer};
use std::sync::Arc;
use temp_dir::TempDir;

struct State {}

fn hello(_state: Arc<State>, req: &Request) -> Result<Response, Response> {
    #[derive(Deserialize)]
    struct Input {
        name: String,
    }
    let input: Input = req.json()?;
    Ok(Response::json(200, json!({"message": format!("Hello, {}!", input.name)}))
    .unwrap())
}

fn handle_req(state: Arc<State>, req: &Request) -> Result<Response, Response> {
    match (req.method(), req.url().path()) {
        ("GET", "/ping") => Ok(Response::text(200, "ok")),
        ("POST", "/hello") => hello(state, req),
        _ => Ok(Response::text(404, "Not found")),
    }
}

let state = Arc::new(State {});
let request_handler = move |req: Request| {
    print_log_response(&req, handle_req(state, &req))
};
let cache_dir = TempDir::new().unwrap();
safina_timer::start_timer_thread();
let executor = safina_executor::Executor::new(1, 9).unwrap();
executor.block_on(
    HttpServerBuilder::new()
        .listen_addr(socket_addr_127_0_0_1(8009))
        .max_conns(1000)
        .small_body_len(64 * 1024)
        .receive_large_bodies(cache_dir.path())
        .spawn_and_join(request_handler)
).unwrap();
```
# Cargo Geiger Safety Report
```

Metric output format: x/y
    x = unsafe code used by the build
    y = total unsafe code found in the crate

Symbols: 
    🔒  = No `unsafe` usage found, declares #![forbid(unsafe_code)]
    ❓  = No `unsafe` usage found, missing #![forbid(unsafe_code)]
    ☢️  = `unsafe` usage found

Functions  Expressions  Impls  Traits  Methods  Dependency

0/0        0/0          0/0    0/0     0/0      🔒  servlin 0.1.2
0/0        4/4          0/0    0/0     2/2      ☢️  ├── async-fs 1.6.0
4/4        91/91        16/16  0/0     1/1      ☢️  │   ├── async-lock 2.7.0
0/0        106/116      4/8    0/0     0/0      ☢️  │   │   └── event-listener 2.5.3
0/0        28/28        4/4    0/0     0/0      ☢️  │   ├── blocking 1.3.1
0/0        0/0          0/0    0/0     0/0      🔒  │   │   ├── async-channel 1.8.0
0/0        168/168      2/2    0/0     1/1      ☢️  │   │   │   ├── concurrent-queue 2.2.0
4/4        32/94        4/16   0/0     0/3      ☢️  │   │   │   │   └── crossbeam-utils 0.8.15
0/0        0/0          0/0    0/0     0/0      ❓  │   │   │   │       └── cfg-if 1.0.0
0/0        106/116      4/8    0/0     0/0      ☢️  │   │   │   ├── event-listener 2.5.3
0/0        37/37        2/2    0/0     0/0      ☢️  │   │   │   └── futures-core 0.3.28
4/4        91/91        16/16  0/0     1/1      ☢️  │   │   ├── async-lock 2.7.0
1/1        858/858      4/4    0/0     12/12    ☢️  │   │   ├── async-task 4.4.0
0/0        33/33        2/2    0/0     0/0      ☢️  │   │   ├── atomic-waker 1.1.1
0/0        0/0          0/0    0/0     0/0      🔒  │   │   ├── fastrand 1.9.0
0/0        0/0          0/0    0/0     0/0      ❓  │   │   ├── futures-lite 1.13.0
0/0        0/0          0/0    0/0     0/0      🔒  │   │   │   ├── fastrand 1.9.0
0/0        37/37        2/2    0/0     0/0      ☢️  │   │   │   ├── futures-core 0.3.28
0/0        0/0          0/0    0/0     0/0      ❓  │   │   │   ├── futures-io 0.3.28
36/37      2067/2144    0/0    0/0     21/21    ☢️  │   │   │   ├── memchr 2.5.0
1/24       10/449       0/2    0/0     5/50     ☢️  │   │   │   │   └── libc 0.2.141
0/0        0/0          0/0    0/0     0/0      🔒  │   │   │   ├── parking 2.1.0
0/0        11/165       0/0    0/0     2/2      ☢️  │   │   │   ├── pin-project-lite 0.2.9
0/0        21/21        0/0    0/0     4/4      ☢️  │   │   │   └── waker-fn 1.1.0
1/1        16/18        1/1    0/0     0/0      ☢️  │   │   └── log 0.4.17
0/0        0/0          0/0    0/0     0/0      ❓  │   │       ├── cfg-if 1.0.0
0/0        5/5          0/0    0/0     0/0      ☢️  │   │       └── serde 1.0.160
0/0        0/0          0/0    0/0     0/0      ❓  │   │           └── serde_derive 1.0.160
0/0        15/15        0/0    0/0     3/3      ☢️  │   │               ├── proc-macro2 1.0.56
0/0        4/4          0/0    0/0     0/0      ☢️  │   │               │   └── unicode-ident 1.0.8
0/0        0/0          0/0    0/0     0/0      ❓  │   │               ├── quote 1.0.26
0/0        15/15        0/0    0/0     3/3      ☢️  │   │               │   └── proc-macro2 1.0.56
0/0        79/79        3/3    0/0     2/2      ☢️  │   │               └── syn 2.0.15
0/0        15/15        0/0    0/0     3/3      ☢️  │   │                   ├── proc-macro2 1.0.56
0/0        0/0          0/0    0/0     0/0      ❓  │   │                   ├── quote 1.0.26
0/0        4/4          0/0    0/0     0/0      ☢️  │   │                   └── unicode-ident 1.0.8
0/0        0/0          0/0    0/0     0/0      ❓  │   └── futures-lite 1.13.0
                                                       │   [build-dependencies]
0/0        0/0          0/0    0/0     0/0      ❓  │   └── autocfg 1.1.0
0/0        0/0          0/0    0/0     0/0      🔒  ├── async-net 1.7.0
                                                       │   [build-dependencies]
0/0        0/0          0/0    0/0     0/0      ❓  │   └── autocfg 1.1.0
0/0        2/4          0/0    0/0     0/0      ☢️  │   ├── async-io 1.13.0
4/4        91/91        16/16  0/0     1/1      ☢️  │   │   ├── async-lock 2.7.0
0/0        0/0          0/0    0/0     0/0      ❓  │   │   ├── cfg-if 1.0.0
0/0        168/168      2/2    0/0     1/1      ☢️  │   │   ├── concurrent-queue 2.2.0
0/0        0/0          0/0    0/0     0/0      ❓  │   │   ├── futures-lite 1.13.0
1/1        16/18        1/1    0/0     0/0      ☢️  │   │   ├── log 0.4.17
0/0        0/0          0/0    0/0     0/0      🔒  │   │   ├── parking 2.1.0
0/1        11/250       5/16   1/4     0/5      ☢️  │   │   ├── polling 2.7.0
0/0        0/0          0/0    0/0     0/0      ❓  │   │   │   ├── cfg-if 1.0.0
1/24       10/449       0/2    0/0     5/50     ☢️  │   │   │   ├── libc 0.2.141
1/1        16/18        1/1    0/0     0/0      ☢️  │   │   │   └── log 0.4.17
                                                       │   │   │   [build-dependencies]
0/0        0/0          0/0    0/0     0/0      ❓  │   │   │   └── autocfg 1.1.0
44/360     1571/6038    1/2    0/0     6/21     ☢️  │   │   ├── rustix 0.37.11
                                                       │   │   │   [build-dependencies]
0/1        0/201        0/2    0/0     0/4      ❓  │   │   │   └── cc 1.0.79
0/0        0/0          0/0    0/0     0/0      ❓  │   │   │   ├── bitflags 1.3.2
0/0        32/100       0/0    0/0     0/0      ☢️  │   │   │   ├── errno 0.3.1
1/24       10/449       0/2    0/0     5/50     ☢️  │   │   │   │   └── libc 0.2.141
0/0        24/666       27/36  2/2     6/14     ☢️  │   │   │   ├── io-lifetimes 1.0.10
1/24       10/449       0/2    0/0     5/50     ☢️  │   │   │   │   ├── libc 0.2.141
3/6        540/673      2/4    0/0     3/4      ☢️  │   │   │   │   └── socket2 0.4.9
1/24       10/449       0/2    0/0     5/50     ☢️  │   │   │   │       └── libc 0.2.141
0/0        7/7          0/0    0/0     0/0      ☢️  │   │   │   ├── itoa 1.0.6
1/24       10/449       0/2    0/0     5/50     ☢️  │   │   │   └── libc 0.2.141
0/0        24/24        0/0    0/0     3/3      ☢️  │   │   ├── slab 0.4.8
0/0        5/5          0/0    0/0     0/0      ☢️  │   │   │   └── serde 1.0.160
                                                       │   │   │   [build-dependencies]
0/0        0/0          0/0    0/0     0/0      ❓  │   │   │   └── autocfg 1.1.0
3/6        540/673      2/4    0/0     3/4      ☢️  │   │   ├── socket2 0.4.9
0/0        21/21        0/0    0/0     4/4      ☢️  │   │   └── waker-fn 1.1.0
                                                       │   │   [build-dependencies]
0/0        0/0          0/0    0/0     0/0      ❓  │   │   └── autocfg 1.1.0
0/0        28/28        4/4    0/0     0/0      ☢️  │   ├── blocking 1.3.1
0/0        0/0          0/0    0/0     0/0      ❓  │   └── futures-lite 1.13.0
0/0        0/0          0/0    0/0     0/0      🔒  ├── fixed-buffer 0.5.0
0/0        0/0          0/0    0/0     0/0      ❓  │   └── futures-io 0.3.28
0/0        0/0          0/0    0/0     0/0      ❓  ├── futures-io 0.3.28
0/0        0/0          0/0    0/0     0/0      ❓  ├── futures-lite 1.13.0
0/0        0/0          0/0    0/0     0/0      ❓  ├── include_dir 0.7.3
0/0        0/0          0/0    0/0     0/0      ❓  │   └── include_dir_macros 0.7.3
0/0        15/15        0/0    0/0     3/3      ☢️  │       ├── proc-macro2 1.0.56
0/0        0/0          0/0    0/0     0/0      ❓  │       └── quote 1.0.26
0/0        0/0          0/0    0/0     0/0      🔒  ├── permit 0.2.0
0/0        0/0          0/0    0/0     0/0      🔒  ├── safe-regex 0.2.5
0/0        0/0          0/0    0/0     0/0      🔒  │   └── safe-regex-macro 0.2.5
0/0        0/0          0/0    0/0     0/0      🔒  │       ├── safe-proc-macro2 1.0.36
0/0        0/0          0/0    0/0     0/0      🔒  │       │   └── unicode-xid 0.2.4
0/0        0/0          0/0    0/0     0/0      🔒  │       └── safe-regex-compiler 0.2.5
0/0        0/0          0/0    0/0     0/0      🔒  │           ├── safe-proc-macro2 1.0.36
0/0        0/0          0/0    0/0     0/0      🔒  │           └── safe-quote 1.0.15
0/0        0/0          0/0    0/0     0/0      🔒  │               └── safe-proc-macro2 1.0.36
0/0        0/0          0/0    0/0     0/0      🔒  ├── safina-executor 0.3.3
0/0        0/0          0/0    0/0     0/0      🔒  │   ├── safina-sync 0.2.4
0/0        0/0          0/0    0/0     0/0      🔒  │   └── safina-threadpool 0.2.4
0/0        0/0          0/0    0/0     0/0      🔒  ├── safina-sync 0.2.4
0/0        0/0          0/0    0/0     0/0      🔒  ├── safina-timer 0.1.11
1/1        79/125       5/9    0/0     2/4      ☢️  │   └── once_cell 1.17.1
0/0        5/5          0/0    0/0     0/0      ☢️  ├── serde 1.0.160
0/0        4/7          0/0    0/0     0/0      ☢️  ├── serde_json 1.0.96
0/0        7/7          0/0    0/0     0/0      ☢️  │   ├── itoa 1.0.6
7/9        579/715      0/0    0/0     2/2      ☢️  │   ├── ryu 1.0.13
0/0        5/5          0/0    0/0     0/0      ☢️  │   └── serde 1.0.160
0/0        0/0          0/0    0/0     0/0      🔒  ├── serde_urlencoded 0.7.1
0/0        2/2          0/0    0/0     0/0      ☢️  │   ├── form_urlencoded 1.1.0
0/0        3/3          0/0    0/0     0/0      ☢️  │   │   └── percent-encoding 2.2.0
0/0        7/7          0/0    0/0     0/0      ☢️  │   ├── itoa 1.0.6
7/9        579/715      0/0    0/0     2/2      ☢️  │   ├── ryu 1.0.13
0/0        5/5          0/0    0/0     0/0      ☢️  │   └── serde 1.0.160
0/0        0/0          0/0    0/0     0/0      🔒  ├── temp-dir 0.1.11
0/0        0/0          0/0    0/0     0/0      🔒  ├── temp-file 0.1.7
0/0        0/0          0/0    0/0     0/0      ❓  └── url 2.3.1
0/0        2/2          0/0    0/0     0/0      ☢️      ├── form_urlencoded 1.1.0
0/0        0/0          0/0    0/0     0/0      ❓      ├── idna 0.3.0
0/0        5/5          0/0    0/0     0/0      ☢️      │   ├── unicode-bidi 0.3.13
0/0        5/5          0/0    0/0     0/0      ☢️      │   │   └── serde 1.0.160
0/0        20/20        0/0    0/0     0/0      ☢️      │   └── unicode-normalization 0.1.22
0/0        0/0          0/0    0/0     0/0      🔒      │       └── tinyvec 1.6.0
0/0        5/5          0/0    0/0     0/0      ☢️      │           ├── serde 1.0.160
0/0        0/0          0/0    0/0     0/0      🔒      │           └── tinyvec_macros 0.1.1
0/0        3/3          0/0    0/0     0/0      ☢️      ├── percent-encoding 2.2.0
0/0        5/5          0/0    0/0     0/0      ☢️      └── serde 1.0.160

102/449    6488/13169   82/129 3/6     75/158 

```
# Alternatives
See [rust-webserver-comparison.md](https://github.com/mleonhard/servlin/blob/main/rust-webserver-comparison.md).

# Changelog
- v0.1.2 - Add:
   - `Response::ok_200()`
   - `Response::unauthorized_401()`
   - `Response::forbidden_403()`
   - `Response::internal_server_errror_500()`
   - `Response::not_implemented_501()`
   - `Response::service_unavailable_503()`
   - `EventSender::is_connected()`
   - `PORT_env()`
- v0.1.1 - Add `EventSender::unconnected`.
- v0.1.0 - Rename library to Servlin.

# TO DO
- Fix limitations above
- Support [HEAD](https://developer.mozilla.org/en-US/docs/Web/HTTP/Methods/HEAD)
  responses that have Content-Length set and no body.
- Update `rust-webserver-comparison.md`
  - Add missing data
  - Add other servers from <https://www.arewewebyet.org/topics/frameworks/>
  - Rearrange
  - Generate geiger reports for each web server

License: MIT OR Apache-2.0
