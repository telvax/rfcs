# RFC: `env()` in the `serve:` block

* Status: implemented
* Issues: telvax/telvax#186
* Implementation: telvax/telvax#187

## Summary

A value of the `serve:` block may be `env("NAME")` or `env("NAME", "default")`, which reads a variable when the server
starts. The `TELVAX_*` variables and the flags still win.

## Motivation

RFC 0016 refused values from the environment in the block, because the `TELVAX_*` variables already do it. That holds when
the deploy is written for Telvax. A service ported from another stack has deploy files (compose, Kubernetes, CI) that
already set `CORS_ORIGINS` or `LOG_FORMAT`; with the block as it is, each must be renamed in every environment, or the
program reads the variable and cannot hand it to the server.

## Design

```telvax
serve:
    cors: env("CORS_ORIGINS", "*")
    log_format: env("LOG_FORMAT", "json")
    metrics: env("METRICS_ENABLED")
    shutdown_wait: env("SHUTDOWN_WAIT", "30")
```

* `env(name)` and `env(name, default)` are allowed as the value of any option of the block that the block accepts today; both
  arguments are text literals. Nothing else is evaluated: no concatenation, no calls.
* The value is the text of the variable, read once at start, and means what the same text means in the flag or in the
  `TELVAX_*` variable of that option: `"json"`, `"100/min"`, origins separated by commas; a `Bool` option is on for
  `1` or `true` and off otherwise; an `Int` option takes a whole number.
* A variable that is unset or empty gives the default; with no default the option is as if it were not in the block.
* Order of precedence, last wins: the defaults, the block (literal or `env`), the `TELVAX_*` variables, the flags.
* `check` refuses another call, a first argument that is not a text literal and a default that the option would refuse
  (a `log_format` of `"xml"`). It cannot check the variable: a value of the variable the option refuses stops the start
  with `serve: CORS_ORIGINS ...` naming the variable, the option and the value.
* `telvax build --serve` keeps the `env` in the binary and reads the variable when the binary starts, not when it was
  built.
* The text of the variable is not converted: `RATE_LIMIT_PER_MINUTE=100` is not `100/min`. A program whose variables
  have another shape keeps reading them with `std.env`.

## Alternatives

* **Aliases in the compiler for the names of one service.** Ties the server to a framework's names.
* **Any expression** (`cors: setting("CORS", "*")`). Runs program code before the server starts and the editor and
  `check` can no longer read the block; `env` keeps the block static.
* **Leave it as RFC 0016 decided.** Costs a rename in every deploy of a ported service.

## Open questions

* Whether a bare number for `rate_limit` should mean per minute, to match the common `..._PER_MINUTE` variables.
* Whether `port` and `host` can take `env` once they are in the block (open in RFC 0016).
