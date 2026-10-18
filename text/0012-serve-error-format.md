# RFC: errors in the form FastAPI clients read

* Status: accepted
* Issues: telvax/telvax#171
* Implementation: telvax/telvax (pull request to be linked)

## Summary

`telvax serve --error-format detail` (and `TELVAX_ERROR_FORMAT=detail`) answers errors as `{"detail": "..."}`, and a body
that does not fit its type with `422` instead of `400`.

## Motivation

A service that moves from FastAPI has clients that read `detail` and expect `422` for a body that does not validate.
Telvax answers `400` and `{"error": ...}`.

## Design

* `telvax` (default): today's answers.
* `detail`: the `Err(Status...)` of an endpoint, a path parameter that does not convert, and the checks of the body are
  `{"detail": "<message>"}`; a body or query that does not fit the types is `422` with `{"detail": [<messages>]}`.
  `404` and `405` for an unknown route are `{"detail": "Not Found"}`.
* The OpenAPI document describes the same status and body.
* A program that returns its own `Response` is not changed.

## Alternatives

* **A type for the error body.** More general, and a language feature; this covers what a migration needs now.
* **Always `422`.** Breaks every client of today.
