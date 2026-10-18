# RFC: stopping a server without cutting its work

* Status: accepted
* Issues: telvax/telvax#174
* Implementation: telvax/telvax (pull request to be linked)

## Summary

On `SIGTERM` or `SIGINT`, `telvax serve` (and a bundled binary) stops accepting connections, waits for the requests and the
background tasks in progress, and exits.

## Motivation

A container stop or a deploy sends `SIGTERM`. Today the process ends at once: requests are cut and background tasks
(a batch job) are lost half done.

## Design

* After the signal, new connections are refused and the requests in progress are finished.
* Background tasks started by `std.task.spawn` are waited for.
* The wait is at most `--shutdown-wait SECONDS` (`TELVAX_SHUTDOWN_WAIT`, 30 by default); after it the process exits with status 1 and says how many requests and tasks were cut.
* A second signal exits at once.
* `/healthz` answers `503` while it waits, so a balancer stops sending requests.

## Alternatives

* **Leave it to the platform** (`terminationGracePeriod`): does not wait for the tasks, which the platform cannot see.
* **A hook in the program** (`on_shutdown`): wanted later, to close connections.
