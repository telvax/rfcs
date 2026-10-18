# RFC: random values with std.random

* Status: accepted
* Issues: telvax/telvax#152
* Implementation: telvax/telvax (pull request to be linked)

## Summary

`std.random.uuid()` is a random identifier as `Text`, and `std.random.int(low, high)` a random whole number.
Both come from the operating system's random source and need no capability.

## Motivation

[sentiment-analysis-api](https://github.com/thentsation/sentiment-analysis-api) names each batch job with
`uuid.uuid4().hex`. The id is the only thing that protects `GET /v1/results/{job_id}`, so it must not be
guessable. Telvax has no function that makes a random value, and an id built from the clock can be guessed.

## Design

```telvax
let id = std.random.uuid()        # "3f2b8c1e9a4d4b7f8e6a0c5d1b2a3f4e"
let roll = std.random.int(1, 6)   # 1 to 6, both included
```

* `uuid() -> Text` is a version 4 UUID written as 32 lower case hexadecimal digits, with no dashes.
* `int(low: Int, high: Int) -> Int` is uniform in `low..=high`. `low > high` is an error.
* The source is the operating system (`getrandom`), so the values are fit for ids and tokens, not for
  cryptographic keys of long life: a key has its own function when one is needed.
* No capability: reading the random source gives a program nothing it could not do without it.
* There is no seed. A test that needs a known value does not call `random`; it passes the value in.

## Alternatives

* **`std.id.new()`.** Names the use, not the source; `random` also covers `int`.
* **A seeded generator type.** Needed for simulations; a bigger design, and it can come later as
  `std.random.Rng` without changing these two functions.

## Open questions

* Whether `std.random.bytes(n)` and `std.random.choice(list)` belong here.
