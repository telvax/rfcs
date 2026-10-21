# RFC: `std.metrics.set`, a gauge for a value that goes up and down

* Status: implemented
* Issues: telvax/telvax#203
* Implementation: telvax/telvax#220

## Summary

`std.metrics.set(name, value, labels?)` sets a gauge: a number that the program replaces each time it knows a new one.
`/metrics` lists it with `# TYPE name gauge`.

## Motivation

`std.metrics` has only `count`, and a counter only grows. The Python sentiment service exports `cache_size` (the number
of entries in its cache) as a gauge, and the Telvax port cannot: a program that wants the number of jobs running, the
size of a queue or the entries of a cache has no way to say it. RFC 0013 left gauges for later, naming the cache size as the
case.

```telvax
fn remember(text: Text, scores: Scores):
    cache.set(text, scores)
    std.metrics.set("cache_size", cache.len())
```

## Design

```telvax
std.metrics.set("cache_size", cache.len())
std.metrics.set("jobs_running", 3, {"queue": "high"})
```

* `name` is `[a-zA-Z_:][a-zA-Z0-9_:]*`, the rule of `std.metrics.count`; `value` is an `Int` or a `Float`, read as a
  `Float`; `labels` is a `Map[Text, Text]` with the label names and the limit of 1,000 label sets per name of
  RFC 0013, so a label made from user input cannot fill the memory.
* The last call wins. A gauge is not summed over threads as a counter is: the value is held in one table, so two
  threads that set it at the same time leave one of the two values, whichever came last.
* `/metrics` lists it under its own name, with its own `# TYPE`, as RFC 0013 does for labelled counters:

  ```text
  # TYPE cache_size gauge
  cache_size 1042
  # TYPE jobs_running gauge
  jobs_running{queue="high"} 3
  ```

  A value that is not finite is written as Prometheus writes it: `NaN`, `+Inf`, `-Inf`.
* A name is a counter or a gauge, not both. `std.metrics.set("hits", 1)` after `std.metrics.count("hits")` stops the
  program with `` std.metrics.set: `hits` is a counter: use `std.metrics.count` ``, and the other way round with the
  words swapped. The same name with and without labels is one metric, so it follows the same rule.
* The gauge exists from its first `set`; there is no value before it. With no `--metrics` the call does nothing
  visible, as `count` does.
* Needs no capability, and has no effect in `telvax run` beyond holding the value.
* `telvax check` takes the same arguments as `count`: a `name` that is not `Text`, a `value` that is not a number
  and `labels` that are not a `Map[Text, Text]` are errors that say what to write.

## Alternatives

* **A `Gauge` value, `let size = std.metrics.gauge("cache_size")` and `size.set(n)`.** Nicer when the same gauge is set from
  several places, and the place for `inc` and `dec`. It needs a type of the standard library, and `count` is a function; the
  function is the step under it, as RFC 0013 chose for labels.
* **`std.metrics.add(name, delta)` to move a gauge up and down.** Two calls that read and then set are not atomic, which
  is the case for `add`; it can be a later RFC and does not need `set` to change.
* **A callback that the server calls when `/metrics` is read** (`std.metrics.gauge("cache_size", () => cache.len())`).
  The value is always current and nothing is set on each request, but it runs program code in the interpreter on every
  scrape. It can be added beside `set`.
* **Make the gauge a counter that can go down.** Prometheus reads a counter that decreases as a reset, so the rates are wrong.

## Open questions

* Whether a gauge should also be listed when the program never reached its first `set`, with `0`. This RFC says no, so a
  dashboard does not show a number that nobody set.
* Whether `inc` and `dec` are worth having once a `Gauge` value exists.
