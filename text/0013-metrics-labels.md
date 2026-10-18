# RFC: labelled counters and the time of a request

* Status: accepted
* Issues: telvax/telvax#172
* Implementation: telvax/telvax (pull request to be linked)

## Summary

`std.metrics.count(name, labels)` takes a `Map[Text, Text]` of labels, listed as `<name>{label="value"}`, and
`--process-time-header` adds `x-process-time-ms` to every answer.

## Motivation

The Python service exports `sentiment_analysis_total{language,sentiment}` and `batch_jobs_total{status}`, and sends
`X-Process-Time-Ms`. Dashboards and alerts depend on those names.

## Design

```telvax
std.metrics.count("sentiment_analysis_total", {"language": "en", "sentiment": "positive"})
```

* The counter is listed in `/metrics` as `sentiment_analysis_total{language="en",sentiment="positive"} 1` with its own
  `# TYPE`. Without labels the counter is still `telvax_counter_total{name="..."}`.
* A name is `[a-zA-Z_:][a-zA-Z0-9_:]*` and a label name `[a-zA-Z_][a-zA-Z0-9_]*`; anything else is an error. Values are escaped.
* At most 1,000 label sets per name; one more is an error, so a label made from user input cannot fill the memory.
* `--process-time-header` (`TELVAX_PROCESS_TIME_HEADER=1`): `x-process-time-ms` with two decimals, as `observe` measures it.

## Alternatives

* **A `Counter` type with `.inc(labels)`.** Nicer, bigger; labels on the existing function are the step under it.
* **Histograms and gauges.** Needed later (the cache size is a gauge); not here.
