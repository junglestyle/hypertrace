<!-- junglestyle/.github code-standards/README.md at 326fdbfb5fe925b46d2313b3fa1aa61a640d4802. Change it there, then run make sync-standards. -->
# Standards

Each principle from [PRINCIPLES.md](https://github.com/junglestyle/.github/blob/main/PRINCIPLES.md) turned into rules you can check in review, arranged by where a reviewer looks. When they pull against each other, pick what works and is easiest to undo.

Each repo keeps this file unchanged as `docs/org-standards.md`. It lists the topics it also follows, such as `python`, in `.org-standards` at its root, and keeps each topic's file here unchanged as `docs/org-standards-<topic>.md`. Every copy is headed by the commit it came from. The repo puts its own rules and examples in `docs/standards.md`, under the heading here they belong to, or under one of its own when none fits; where that differs from a copy, the repo's file wins and names the rule it overrides. Where the repo doesn't follow a copy yet, its `docs/standards.md` says so under Known deviations, as debt to pay down. The repo's checks fail if a copy was edited by hand, this directory has moved on, or a copy's topic isn't listed; resyncing is one command.

## Dependencies

- Use the stdlib first, then what's already in the stack, then the OS, the database, and the init system. A new dependency comes last and gets a one-line reason in the commit.
- Lean on the database's transactions, the OS's process model, and the supervisor's restarts before writing your own.
- A new language, datastore, framework, or runtime gets an ADR saying what it does that the current stack can't.

## Structure

- No plugin systems, factories, registries, base classes, config frameworks, or extension points without two real cases that need them today.
- One code path that the CLI, API, and anything else share.
- A rule has one executable implementation.
- Two blocks that look alike but would change for different reasons stay separate. Duplication is cheaper than the wrong abstraction.
- Extract on the third real use, not the second similar-looking one. A critical invariant is the exception: two implementations of it are already one too many.
- Layers talk through explicit interfaces (an API, a schema, a file format) and don't know each other's internals. The repo's standards name its layers and what each one mustn't know.
- Flat over nested: early returns, shallow control flow, one module per concept.
- No import-time side effects and no hidden globals.
- Delete code that nothing calls. Git keeps it.
- Before removing something you didn't write, find out why it's there. The commit says what it was for.
- Leave each file you touch a little cleaner, within the change's scope.

## Errors and recovery

- Catch only what you can handle, and say what you did with it. Errors never pass silently.
- An impossible state raises. It isn't logged and continued.
- Prefer state the system converges to (expired leases requeue, stale locks clear) over steps someone must remember.

## Config

- No option, flag, or parameter that nothing sets.
- No magic default that differs by environment.

## Comments and names

- Names say what. Comments say why.
- Keep a comment only if a maintainer would lose a fact that the code, types, and tests don't already state. No comments that restate the code, and no commented-out code.
- The facts worth a comment are constraints the code can't show: concurrency and lifecycle rules, security boundaries, protocol quirks, and the tradeoff behind a non-obvious choice.
- A known limit goes in the docs or one comment. A workaround is fine when it's small and says what it works around.
- Project names are dumb and Linuxy: short, lowercase, nothing producty.

## Testing

Few, focused tests of consequential behavior, at boundaries that stay put.

- Test what a user would notice: the behaviors the spec promises, recovery after a crash, and the security boundaries.
- Don't test plumbing, getters, or internal call order. Don't mock half the system to prove a call happened.
- Don't pin prose: check what a message means, not its words. Output another program reads is interface, and a test may pin it.
- Before adding a test, ask what realistic regression it catches and whether it survives a refactor. If the answer is weak, skip it.

## Running it

- `make check` runs every check CI runs, offline. The one exception is `make standards`, which checks the org copies against this directory and needs the network; CI runs it as its own job. `make sync-standards` resyncs the copies, and following another topic means adding it to `.org-standards` and running it.
- Installing takes as few maintained scripts as the job needs. Setup steps don't get scattered across docs.

## Privacy

- Never commit hostnames, tokens, personal context, or deployment config. In docs, name roles, not machines: "the NAS" or "a GPU host".
- Commit the shape of config, never its values. `.env.example` holds it unless the repo's standards name another place.

## Docs

- The README says what the project is and how to run it, in that order.
- Docs describe the behavior a user can observe (a default, a state name, an env var), so a reader never has to open the code to learn what the project does. They don't carry a second copy of the logic.
- A deferred requirement gets a line in the spec or `docs/ROADMAP.md`, not a stub in code.
- A decision that's hard to undo gets an ADR in `docs/stack.md`.

## Prose

For docs, commit messages, and PR descriptions. Follow the rules in
[agent-style](https://github.com/yzhao062/agent-style/blob/7da99e0b0610aca4e4d340351bd802bfd448c894/RULES.md)
but skip its House Style section, which is for academic papers. These rules take precedence:

- Describe the project as it is meant to be. No dates, approval trails, or "replacing X"; history lives in git.
- Link instead of restating. Each fact has one home, the same as in code.
- Arrange sections in the order a reader needs them.
- Error messages, log lines, and CLI output are interface. Changing their wording is an interface change, not a style edit.
- Contractions are fine, and headings use sentence case.
- "X, not Y" is fine where it prevents a real confusion.
- When writing up a problem, separate evidence, inference, and unknowns, and label which is which.
