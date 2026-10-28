# Working on hypertrace

A research harness where an LLM proposes searches and reads sources, but code decides what counts as evidence.

Read before changing behavior:

- [README](README.md): what hypertrace does and how to run it
- [Installing and operating](docs/install.md): configuration, review and failover settings, evidence rules
- [Coding standards](docs/standards.md): how the code should read, and the research invariants, on top of the org's [principles](https://github.com/junglestyle/.github/blob/main/PRINCIPLES.md) and standards (`docs/org-standards*.md`)

Docs live in `docs/` and change with behavior. Never commit API keys, tokens, or secret-bearing config (see
[Privacy](docs/standards.md#privacy)).

Run `make check` before committing. It runs everything CI runs, except the org standards check: `make standards` (needs the network).
