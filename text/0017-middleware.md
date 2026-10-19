# RFC: middleware written in Telvax

* Status: accepted
* Issues: telvax/telvax#182
* Implementation: telvax/telvax (pull request to be linked)

## Summary

A function marked `@middleware` wraps every request of a server. The `serve:` block lists them in the order they run.
The built-in middleware stays in Rust and keeps its options; the ones a program writes run in addition.

## Motivation

The server has built-in middleware (metrics, process time, rate limit, security headers, request id, CORS, error
format) that `serve:` turns on. A program cannot add its own. [sentiment-analysis-api](https://github.com/thentsation/sentiment-analysis-api)
has a `middlewares/` folder in its Python version, and two things in it cannot be written in Telvax: a token check on a
group of routes, and an error reporter.

## Design

```telvax
import std.http.Request
import std.http.Response

@middleware
fn version_header(req: Request, next: Next) -> Response:
    next(req).header("x-api-version", "1")

@middleware
fn admin_only(req: Request, next: Next) -> Response:
    if req.header("x-admin-token") != Some(admin_token):
        return Response.new(Detail(detail: "Invalid admin token")).status(401)
    next(req)

serve:
    middleware: [admin_only, version_header]
    metrics: True
```

* `@middleware` marks a function of type `fn(Request, Next) -> Response`. `Next` is a value that can be called with a
  request and gives the response of the rest of the chain. `check` refuses another signature.
* `serve:` takes `middleware: [names]`. The list is the only place that orders them; a marked function that is not in the
  list does not run, and `check` says so as a warning. This is the one `serve:` value that is not a literal: it is a
  list of names of functions, which `check` resolves.
* Order, outside in: the built-in middleware first (request id, log, metrics, rate limit, CORS, security headers, process
  time), then the listed ones in the order written, then the route. So the CORS answer comes before any check of the
  program, and a listed middleware sees the request id.
* A middleware that does not call `next` ends the chain with its own response. One that fails (an error value) answers
  with the error format of the server, as a route does.
* A first version reads the request and the response as a route does: method, path, headers, and `Response` with its
  status and headers. It does not read or change the body.
* The built-in ones stay native. The ones in the list run in the interpreter on every request, which costs more, and the
  docs say so.
* A built-in name is never taken over: if a later version adds a built-in option called like a middleware someone
  wrote, the program's own function keeps running as it did.
* A middleware can be imported from a package (`from pkg.mod import name`) as any function can, so a library of them
  needs nothing else.

## Alternatives

* **More flags in the server, one per case.** Every case becomes a Rust change and an RFC, and the Sentry case does not
  fit in a flag.
* **A decorator on each route.** Repeats the check on every route, and does not cover the unknown routes.
* **Scope by path in the first version.** Left out; `req.path` can be tested inside the middleware, and a scope can come
  later without changing this.

## Open questions

* Whether a later version reads and changes the body (needed for rewriting an error to another format).
* Scope by path as an argument of `@middleware`.
* Which middleware a library of the organization should start with, and the name of the repository for it.
