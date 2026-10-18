# Telvax RFCs

An RFC is a written proposal for a change to the language, the standard library or the tools.
Small fixes and bug reports do not need one: open an issue or a pull request in
[telvax/telvax](https://github.com/telvax/telvax).

## When to write one

* new syntax or a change to existing syntax;
* a new standard library module, or a breaking change to one;
* a change to the type system, tensor shapes or the `@endpoint` model;
* anything that would be a `feat!:` in the main repository.

## Process

1. Copy `0000-template.md` to `text/0000-short-name.md`.
2. Open a pull request with it. The pull request number becomes the RFC number: rename the file to `text/NNNN-short-name.md`.
3. Discuss in the pull request. Change the text as the discussion goes.
4. A maintainer merges the pull request when there is agreement, and the RFC is `accepted`. If not, it is closed with the reason.
5. The implementation is a separate pull request in `telvax/telvax` that links the RFC. When it ships, set the status to `implemented`.

An accepted RFC is a decision, not a promise of a date.

## License

Dual-licensed under MIT (`LICENSE-MIT`) or Apache 2.0 (`LICENSE-APACHE`), at your option.
