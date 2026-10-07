# Contributing to TCI-X

Thank you for taking an interest. TCI-X only works if servers and clients agree, so changes go through a short, open process.

## Before you write anything

**Open an issue first** for anything that adds, changes or removes a command. Say:

- **What the radio does:** the radio, its CAT command, and what it returns. A link to the manufacturer's reference, or a capture from the radio, helps most.
- **What you propose on the wire:** the command, its arguments and its `cap` line.
- **How a client would use it**, and what a base TCI client would see (normally nothing).

Typos, clearer wording and examples can go straight to a pull request.

## The rules a proposal has to meet

1. **Base TCI keeps working.** Nothing may change what a client that hasn't sent `tcix:` sees.
2. **TCI syntax:** `name:arg,arg;`, lower case, `<trx>` (and `<ch>` where it applies) first.
3. **Engineering units on the wire.** Raw radio values (0–255, BCD, menu step numbers) are converted by the server. Where a radio only has unitless steps, say so in the `cap` line (`step`).
4. **In the manifest.** Every new command has a `cap` line, so clients can tell whether it's there.
5. **Not tied to one radio.** Name a second radio that would use it, or explain why the shape would fit one.
6. **No vendor prefixes.**

## Versions

- New commands go into the next **minor** version (1.1, 1.2, …). Changing an existing command needs a **major** version.
- While the document is at Draft 0.x, wire version 1.0 can still be corrected. Every such change is listed in [CHANGELOG.md](CHANGELOG.md).
- Larger features start as a file in [drafts/](drafts/) and move into the specification once they've been implemented in at least one server and one client.

## Editor

The specification is edited by Mike G0KAD. Decisions are made in the open, on the issue or pull request.
