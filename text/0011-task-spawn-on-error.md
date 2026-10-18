# RFC: a background task tells a function when it fails

* Status: accepted
* Issues: telvax/telvax#170
* Implementation: telvax/telvax (pull request to be linked)

## Summary

`std.task.spawn(task, on_error)` takes an optional second argument, a function of one `Text`, that is called with the
message when the task fails.

## Motivation

[sentiment-analysis-api](https://github.com/thentsation/sentiment-analysis-api) marks a batch job `failed` when its
task raises. With `spawn` the error only goes to the standard error, so the job stays `processing` for ever.

## Design

```telvax
fn failed(message: Text):
    jobs.insert(id, "failed")

std.task.spawn(work(id, texts), failed)
```

* `on_error: (Text) -> ()`. It runs on the thread of the task, after the failure, and its own error goes to the standard error.
* Without it nothing changes: the message goes to the standard error.
* In `telvax run` and `telvax test`, where `spawn` waits, `on_error` is called before `spawn` returns.

## Alternatives

* **A result handle** (`let t = spawn(...)`, `await t`): larger, and needs cancellation. This does not close that door.
* **`try`/`catch` around `spawn`:** the task has already left the caller.
