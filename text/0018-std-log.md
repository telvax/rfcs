# RFC: std.log

* Status: accepted
* Issues: telvax/telvax#184, telvax/telvax#183
* Implementation: telvax/telvax (pull request to be linked)

## Summary

`std.log` lets a program write log lines in the format and at the level of the server. The request log of the server gets
a `level` and `--log-level` filters both.

## Motivation

A program can only `print`. The server's request log has a format (`--log-format json|text`), so the lines of the
program and the lines of the server cannot be read, filtered or shipped together. The Python service that
[sentiment-analysis-api](https://github.com/thentsation/sentiment-analysis-api) was ported from used `structlog` for both,
with a level (`LOG_LEVEL`).

## Design

```telvax
std.log.info("cache cleared", {"entries": "12"})
std.log.warn("postgres: connection refused")
```

* `std.log.debug`, `info`, `warn` and `error` take a `Text` message and, optionally, a `Map[Text, Text]` of fields.
* A line goes to standard error, as the request line does, in the log format of the server:
  `{"ts": "...", "level": "info", "message": "cache cleared", "entries": "12"}` in `json`, and
  `info cache cleared entries=12` in `text`. A field called `ts`, `level` or `message` is dropped, not allowed to replace
  the others.
* When the server has no `--log-format`, the lines of the program are written in `text`; the request line stays off, as it
  is today.
* The request line gets `level`: `info` for a status under 400, `warn` for 4xx, `error` for 5xx.
* `--log-level debug|info|warn|error`, `TELVAX_LOG_LEVEL` and `log_level` in the `serve:` block (same precedence as the
  other options). A line below the level is not written. The default is `info`, so the output of today only gains the
  `level` field of the request line.
* Outside `telvax serve` (`telvax run`, tests) the level is `info` and the format `text`.
* It needs no capability: it writes to standard error as `print` writes to standard output.
* A message that is not text is an error in `telvax check`, in the words of any call with a wrong argument.

## Alternatives

* **Only `print`, with the format chosen by the program.** Each program invents a format, and the level of the server does
  not reach it.
* **Fields of any type.** `Map[Text, Text]` is enough for a first version and keeps the output the same in both formats;
  a later version can take `Map[Text, Json]`.
* **A logger object per module** (`let log = std.log.get("db")`). Adds a name to every line; left for later.

## Open questions

* Whether the request id of the request being served is added to the program's lines automatically (the Python service did,
  with a context variable).
