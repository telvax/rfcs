# RFC: a `serve` block for the options of the server

* Status: accepted
* Issues: telvax/telvax#180
* Implementation: telvax/telvax (pull request to be linked)

## Summary

A program can say how it is served in a `serve:` block of its entry file, in place of (or besides) the flags
of `telvax serve`. The block holds fixed values; a flag or a `TELVAX_*` variable still wins.

## Motivation

Everything that shapes a service at run time is a flag: `--metrics`, `--log-format`, `--cors`, `--rate-limit`,
`--security-headers`, `--process-time-header`, `--trust-proxy`, `--shutdown-wait`, `--error-format`. The program says
nothing about them, so they are copied into the Makefile, the Dockerfile, the compose file, the deploy workflow and
the README, and drift apart. In the FastAPI service that
[sentiment-analysis-api](https://github.com/thentsation/sentiment-analysis-api) was ported from, `main.py` states the
middlewares, the origins, the rate limit and the log format beside the routes.

## Design

```telvax
serve:
    metrics: True
    log_format: "json"
    cors: "*"
    rate_limit: "100/min"
    security_headers: True
    process_time_header: True
    error_format: "detail"
    shutdown_wait: 30
```

* `serve` is a word with a meaning only at the start of a top-level line followed by `:`; it stays a valid name
  elsewhere.
* One block per program, in the entry file. A block in an imported file is an error.
* Each line is `name: value`, and the value is a literal (`True`, `False`, a whole number or a text). The names and
  their types are the ones of the flags, with `-` written `_`:

  | name | type | flag |
  |---|---|---|
  | `metrics` | Bool | `--metrics` |
  | `log_format` | Text, `"json"` or `"text"` | `--log-format` |
  | `cors` | Text, origins separated by commas | `--cors` |
  | `rate_limit` | Text, `"100/min"` | `--rate-limit` |
  | `security_headers` | Bool | `--security-headers` |
  | `process_time_header` | Bool | `--process-time-header` |
  | `trust_proxy` | Bool | `--trust-proxy` |
  | `shutdown_wait` | Int, seconds | `--shutdown-wait` |
  | `error_format` | Text, `"telvax"` or `"detail"` | `--error-format` |

* `telvax check` reports an unknown name, a value of the wrong type, a text that the flag would refuse, and a name
  written twice, in the words of the flag's own error.
* Order of precedence, last wins: the defaults, the `serve` block, the `TELVAX_*` variables, the flags.
* `--port`, `--host`, `--live`, `--allow`, `--seed`, `--quantize` and `--device` are not in the block: they depend on
  where the program runs, not on what it is.
* `telvax build --serve` keeps the block in the binary, so the binary behaves the same as `telvax serve` on the source.

## Alternatives

* **Values from the environment** (`cors: setting("CORS")`). The `TELVAX_*` variables already do this for every
  option, without evaluating code before the server starts; a block of literals can be read by `check`, the editor
  and the OpenAPI document.
* **A table in `telvax.toml`.** Not code, and a single file program has no `telvax.toml`.
* **A decorator on `main`.** A service has no `main`.

## Open questions

* Whether `port` and `host` belong in the block as defaults that a flag overrides.
