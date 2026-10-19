# RFC: functions that run before the server starts

* Status: implemented
* Issues: telvax/telvax#185
* Implementation: telvax/telvax#187

## Summary

`on_start: [names]` in the `serve:` block lists functions that run once, in order, before the server listens. If one
fails, the server does not start.

## Motivation

A program has no code that runs at start. [sentiment-analysis-api](https://github.com/thentsation/sentiment-analysis-api)
makes its PostgreSQL table in the first request that needs it, behind a `Map` used as a flag, and a comment says why.
Any service with a database, a cache to warm or a setting to check has the same need, and today a wrong setting shows
up as a failed request instead of a server that does not start.

## Design

```telvax
fn init_db():
    db.execute("CREATE TABLE IF NOT EXISTS jobs (...)", [])

fn nothing():
    pass

fn check_config() -> Result[(), Text]:
    if admin_token == "" and database_url != "":
        return Err("ADMIN_TOKEN is required when DATABASE_URL is set")
    Ok(nothing())

serve:
    on_start: [check_config, init_db]
    metrics: True
```

* The value is a list of names of top-level functions with no parameters, returning nothing or `Result[(), Text]`.
  Like `middleware` in RFC 0017, it is the one kind of `serve:` value that is not a literal: `check` resolves the names.
  The function may be imported (`from db import init_db`).
* They run in the order written, one after the other, in the interpreter, with the capabilities of the program (`--live`,
  `--allow`), before the port is opened. The server does not answer, not even `/health`, until they finish.
* An `Err(message)` or a runtime error stops the start: the message goes to standard error, as
  `on_start: check_config failed: ...`, and the exit code is 1.
* It runs in `telvax serve` and in a binary from `telvax build --serve`; not in `telvax run`, tests or `check`.
* `check` refuses a name that is not a function, a function with parameters, another return type and a name written
  twice, in the words of the call it would be.

## Alternatives

* **A decorator, `@startup`.** Needs a second place to say the order, as RFC 0017 found for middleware; the block is the
  one place that shows how the service starts.
* **Top-level statements.** A module's top-level code runs at import, also in tests and in `check`-like tools, and
  has no order across modules.
* **Run in the first request.** What programs do today.

## Open questions

* A program cannot write `Ok(())` yet (there is no `()` in an expression), so the example ends with a call to a function that gives nothing.
* A matching `on_stop`, to run after the requests and tasks of a stopping server (RFC 0015) finish.
* A time limit for a function that never returns.
