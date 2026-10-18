# RFC: request and response of an @endpoint

* Status: draft
* Pull request: #
* Issues: telvax/telvax#125, telvax/telvax#126, telvax/telvax#127
* Implementation: (link, when there is one)

## Summary

An `@endpoint` function can take values from the URL path and the request headers, and can choose the
status and the headers of a successful answer. Today it takes one JSON payload and answers `200` with JSON.

## Motivation

[sentiment-analysis-api](https://github.com/thentsation/sentiment-analysis-api) has three things that
Telvax cannot write:

* `GET /v1/results/{job_id}`: an id in the path. Today GET reads only the query string.
* `DELETE /v1/cache` checks an `X-Admin-Token` header. `std.http.Request` has no members.
* `POST /v1/analyze_batch` answers `202 Accepted`. Only error statuses can be chosen.

## Design

### Path parameters

A segment `{name}` in the path is read from the URL into the parameter called `name`.

```telvax
@endpoint(GET, "/v1/results/{job_id}")
fn result(job_id: Text) -> Result[Job, Status]:
    match jobs.get(job_id):
        Some(job): Ok(job)
        None: Err(Status.NotFound("job not found"))
```

* The type of the parameter is `Text`, `Int`, `Float` or `Bool`. A value that does not convert is an HTTP 400:
  `invalid path parameter job_id: expected Int`.
* The parameters that are not named in the path work as today: at most one, a type, read from the JSON body
  (POST, PUT, PATCH) or from the query string (GET, DELETE).
* Compile errors, in the words of the language: `the path has {id} but the function has no parameter named id`;
  `the parameter id is in the path, so its type must be Text, Int, Float or Bool`; `the path names id twice`.
* Two routes that differ only in the name of a parameter (`/a/{x}` and `/a/{y}`) are the same route and a
  duplicate, as two equal paths are today.

### Request headers

A parameter of type `Request` receives the request. It may be in any position and is not part of the payload.

```telvax
import std.http.Request

@endpoint(DELETE, "/v1/cache")
fn clear(req: Request) -> Result[Cleared, Status]:
    if req.header("x-admin-token") != Some(token):
        return Err(Status.Unauthorized("invalid token"))
    Ok(Cleared(cleared: cache.clear()))
```

* `Request.header(name: Text) -> Text?`: the first value of the header; the name is not case sensitive.
* `Request.method` and `Request.path` are `Text`. Nothing else for now.

### Status and headers of the answer

A function may return `Response[T]` instead of `T`.

```telvax
@endpoint(POST, "/v1/analyze_batch")
fn submit(input: Batch) -> Response[Accepted]:
    let job = start(input)
    Response(status: 202, headers: {"location": "/v1/results/" + job.id}, body: Accepted(job_id: job.id))
```

* Fields: `status: Int` (default 200), `headers: Map[Text, Text]` (default empty), `body: T`.
* The body is written as JSON, as today. `content-type: application/json` is added unless the headers set it.
* `Result[Response[T], Status]` is allowed. `T` and `Result[T, Status]` keep working and answer `200`.
* A status outside 100 to 599 is a runtime error and the answer is 500.

### Where it is built

`telvax_server`: the handler takes a `RequestParts { path, params, query, headers, body }` and returns
a `Reply { status, headers, body }` instead of a JSON `Value` in and out. `telvax_hir` checks the path and the
parameters; the interpreter builds `Request` and `Response`. `std.http.Request` and `std.http.Response` get the
members above and are documented in `std.md`.

## Alternatives

* **A typed parameter for a header** (`token: Header["x-admin-token"]`). It is checked at compile time, but it
  adds a syntax for a thing a method does. `Request` can be added to later without it.
* **A `status` argument in the decorator** (`@endpoint(POST, "/x", status: 202)`). It cannot change by case and
  cannot set headers.
* **Path parameters in the payload type** (the field `job_id` of the struct is filled from the path). It hides
  where a value comes from and makes the same struct mean two things.

## Open questions

* Should a path parameter that does not convert be 400 or 404?
* Is a `Request.query(name)` needed, or is the payload type enough?
* Do repeated headers need `Request.headers(name) -> List[Text]`?
