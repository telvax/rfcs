# RFC: background tasks with std.task.spawn

* Status: accepted
* Pull request: #8
* Issues: telvax/telvax#136
* Implementation: (none yet)

## Summary

`std.task.spawn(task)` starts a task and returns at once. The task goes on after the function that started it
has returned, so an `@endpoint` can answer `202` and finish the work in the background.

## Motivation

[sentiment-analysis-api](https://github.com/thentsation/sentiment-analysis-api) answers `POST /v1/analyze_batch`
with `202 Accepted` and a job id, then does the work while the client polls `GET /v1/results/{job_id}`.

`std.task.run(task)` is `await`: it waits for the value. There is no way to start work and not wait for it.

## Design

```telvax
let jobs = {}

async fn work(id: Text, texts: List[Text]):
    jobs.insert(id, "running")
    # ... the slow part ...
    jobs.insert(id, "done")

@endpoint(POST, "/v1/jobs")
fn start(input: Batch) -> Result[Response, Status]:
    let id = new_id()
    jobs.insert(id, "queued")
    std.task.spawn(work(id, input.texts))
    Ok(Response.new(Accepted(job_id: id)).status(202))
```

* `std.task.spawn(task: Task[T])` returns `()` immediately. The task runs on its own thread.
* The value of the task is dropped. An error in it is written to the standard error as
  `background task failed: <message>`; it cannot reach the caller, which has already returned. A task that
  must report must write its result somewhere the program reads, as `jobs` above.
* At most 256 background tasks run at once; one more is an error in the caller:
  `too many background tasks (256)`. This keeps a flood of requests from making a flood of threads.
* A task shares the program's `let` values, like any call: the `List` and `Map` are shared and atomic per
  operation (see the tour).
* In `telvax run` and `telvax test` there is no long-lived server, so `spawn` waits for the task, as
  `std.task.run` does. A program that depends on the task going on after the call is a server program.
* A task does not stop when the server stops: the process ends, and so does the task.

## Alternatives

* **An `@background` function.** Less general: a task started by a request, a timer or another task would
  need different forms.
* **A queue with workers (`std.queue`).** What a service that must not lose a job needs, and a bigger design;
  `spawn` is the step under it.
* **Returning a handle** (`let t = spawn(...)`, `await t`). Useful, but it brings cancellation and detached
  handles; it can be added without changing this.

## Open questions

* Whether the limit should be a `serve` option.
* Cancelling a task when the server shuts down gracefully.
