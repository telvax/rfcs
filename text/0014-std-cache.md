# RFC: a cache with a size limit and an expiry

* Status: accepted
* Issues: telvax/telvax#173
* Implementation: telvax/telvax (pull request to be linked)

## Summary

`std.cache.new(max, ttl_seconds)` makes a cache that drops its least recently used entry when it is full and treats
an expired entry as missing.

## Motivation

A `Map` keeps its keys in order, not the order they came in, so a program cannot drop the oldest entry, and it has to
store the time of each entry itself. The Python service has `SentimentCache(maxsize, ttl)`.

## Design

```telvax
let cache = std.cache.new(10000, 300.0)

fn lookup(key: Text) -> Scores?:
    cache.get(key)

fn remember(key: Text, scores: Scores):
    cache.set(key, scores)
```

* `new(max: Int, ttl_seconds: Float) -> Cache[V]`; the value type comes from the annotation of the `let`.
* `get(key: Text) -> V?`, `set(key: Text, value: V)`, `remove(key) -> V?`, `clear() -> Int` (how many it held), `len() -> Int`.
* When the cache is full, `set` drops the least recently used entry (a `get` counts as use). An entry older than the ttl
  is not returned and is dropped when met.
* Shared by all requests like a top-level `List` or `Map`: every call is atomic, two in a row are not.
* `max` must be above 0 and `ttl_seconds` above 0, or the call is an error.

## Alternatives

* **Options on `Map`** (`Map.limited(...)`): changes a type that is simple today.
* **A library in Telvax.** Needs a lock-free map the language cannot write yet.
