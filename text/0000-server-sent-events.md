# RFC: server-sent events from an @endpoint

* Status: draft
* Pull request: #
* Issues: telvax/telvax#128
* Implementation: (none yet)

## Summary

An `@endpoint` can answer with a stream of events (`text/event-stream`) instead of one JSON body. The first
version needs no new syntax: the function builds the stream with a callback that sends events.

## Motivation

[sentiment-analysis-api](https://github.com/thentsation/sentiment-analysis-api) has `POST /v1/analyze_stream`:
one event per text as each is analysed, then a `done` event. Telvax only answers a complete body, so the client
waits for the whole batch.

The code the issue wishes for:

```telvax
@endpoint(POST, "/v1/analyze_stream")
fn stream(input: Batch) -> Stream[Event]:
    for text in input.texts:
        yield Event(data: score(text))
```

It needs generators (`yield` makes a function that pauses), which Telvax does not have. That is a language
feature of its own, with effects on the type checker, the interpreter and the native backend.

## Design

`Response` (RFC 0002) gets one more constructor, and a new type `std.http.Sink` receives the events.

```telvax
import std.http.Response
import std.http.Sink

@endpoint(POST, "/v1/analyze_stream")
fn stream(input: Batch) -> Result[Response, Status]:
    Ok(Response.events(sink =>
        for text in input.texts:
            sink.send(score(text))
        sink.event("done", "{}")))
```

* `Response.events(producer: fn(Sink) -> Unit) -> Response`. The server answers `200` with
  `content-type: text/event-stream`, `cache-control: no-cache`, and runs `producer` while the client reads.
  `.status` and `.header` work as in RFC 0002.
* `Sink.send(data)` writes one `data:` event. `data` is `Text` or any value that has JSON form (written as JSON).
  `Sink.event(name, data)` writes a named event (`event:` line). `Sink.comment(text)` writes a `:` line, for
  keep-alive.
* Multi-line text is split into one `data:` line per line, as the format requires.
* When the client disconnects, the next `send` is an error that ends `producer`. A `producer` that never sends
  is not interrupted until it does.
* An error inside `producer` ends the stream with an `event: error` whose data is the message. The status was
  already sent, so it cannot change.
* The stream is flushed after each event.
* `telvax serve` and `build --serve` both support it; `telvax run` of an endpoint outside a server has no `Sink`
  to write to and rejects `Response.events`.

### Where it is built

`telvax_server`: `Reply` gets a body variant `Events(receiver)`; the handler returns the response head at once and
axum streams the body from a bounded channel. The interpreter runs `producer` on a blocking task and `Sink`
writes to the channel. A bounded channel means a slow client slows the producer instead of growing memory.

## Alternatives

* **`Stream[Event]` with `yield`.** The nicest to read. Left for later: it needs generators in the language, and
  this design can be the runtime under them (`yield e` becomes `sink.send(e)`).
* **A `List[Event]` return.** Easy, but it is not streaming: nothing is sent until the list is complete.
* **WebSocket.** Two-way, but heavier, and the port only needs server to client.

## Open questions

* Should `Sink.send` take a typed `Event` struct (`id`, `event`, `data`, `retry`) instead of the three methods?
* A limit for how long a stream may stay open, and whether it is set in the decorator.
