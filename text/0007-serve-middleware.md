# RFC: metrics, logs and middleware for telvax serve

* Status: accepted
* Pull request: #7
* Issues: telvax/telvax#129, telvax/telvax#130
* Implementation: (none yet)

## Summary

`telvax serve` and a binary made by `telvax build --serve` get options for what a service usually puts around
its handlers: a request id, a metrics endpoint, one log line per request, CORS, a rate limit and security
headers. They are options of the server, not syntax of the language: the program does not change.

## Motivation

[sentiment-analysis-api](https://github.com/thentsation/sentiment-analysis-api) needs `/metrics` (requests and
latency per route, cache hits), JSON logs with a request id, CORS, a per-client rate limit and security headers.
Today the answer is a reverse proxy in front of the service.

## Design

Flags, all off by default, so a program that runs today answers the same:

```sh
telvax serve app.tvx --metrics --log-format json \
    --cors "*" --rate-limit 100/min --security-headers
```

Each flag also has an environment variable, for a binary that is started by a container
(`TELVAX_METRICS=1`, `TELVAX_LOG_FORMAT=json`, `TELVAX_CORS=*`, `TELVAX_RATE_LIMIT=100/min`,
`TELVAX_SECURITY_HEADERS=1`). A flag wins over the variable.

### Request id (with `--log-format` or `--metrics`)

Every answer has an `x-request-id` header. The value of the same header in the request is kept (up to 128
visible characters); otherwise the server makes one.

### `--metrics`

`GET /metrics` in the Prometheus text format:

* `telvax_http_requests_total{method,route,status}`: a counter. `route` is the path as written in the
  program (`/v1/results/{job_id}`), not the URL, so the number of series stays small. A URL that matches no
  route is `route="unmatched"`.
* `telvax_http_request_duration_seconds{method,route}`: a histogram.
* `telvax_counter_total{name}`: counters the program makes with `std.metrics.count("cache_hit")`
  (`count(name, by)` adds more than one). This is how a program reports what only it knows, such as cache hits.

An endpoint of the program at `/metrics` takes the place of the built-in one.

### `--log-format json|text`

One line on standard error per request, after the answer is built. `json`:
`{"ts":"2026-10-18T12:00:00.123Z","request_id":"...","method":"GET","path":"/v1/x","status":200,"duration_ms":3.2}`.
`text`: `GET /v1/x 200 3.2ms request_id=...`. The path is the URL without the query string, so a secret in the
query is not logged. Without the flag nothing is logged, as today.

### `--cors ORIGINS`

`*` or a comma-separated list of origins. A request with an `Origin` that is allowed gets
`access-control-allow-origin`, and `OPTIONS` (preflight) is answered `204` with the allowed methods and
headers. Credentials are not supported: `*` with credentials is not allowed by browsers, and this keeps it
simple.

### `--rate-limit N/second|minute|hour`

A fixed window per client. The client is the address of the connection, or the first address of
`x-forwarded-for` when `--trust-proxy` is also given. Over the limit the answer is `429` with
`{"error": "rate limit exceeded"}` and `retry-after` in seconds. `/healthz` is not limited.

### `--security-headers`

`x-content-type-options: nosniff`, `x-frame-options: DENY`, `referrer-policy: no-referrer` and
`cache-control: no-store` on answers that do not set it themselves.

### Where it is built

`telvax_server` gets `router_with(routes, handlers, &Options)`; `router` is `router_with` with the defaults.
The options are layers of the axum router, in this order from the outside: request id and log, metrics,
security headers, CORS, rate limit. `std.metrics.count` is a function of the runtime and the CLI passes the
counters to the server.

## Alternatives

* **A reverse proxy.** Works, and is still the right answer for TLS, but it is one more thing to run for what
  a flag can do.
* **Middleware written in Telvax** (a function that wraps every endpoint). The most general, but it needs a
  design of its own (order, short-circuit, types of the request and the answer). These options do not rule
  it out.
* **Decorators** (`@cors("*")` on an endpoint). Per-route CORS is rare, and flags work for a binary without
  changing the source.

## Open questions

* Rate limits per route, or by an API key header.
* A sliding window instead of a fixed one.
