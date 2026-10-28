# Coding standards

Hypertrace follows the [org standards](org-standards.md) and the org's standards for [Python](org-standards-python.md)
and [SQL](org-standards-sql.md), unchanged copies of
[junglestyle's](https://github.com/junglestyle/.github/tree/main/code-standards). `.org-standards` lists the topics it
copies beyond the general one. This file adds Hypertrace's own rules and examples. Where it differs from an org rule, it
says so, and it wins.

## Invariants

Hypertrace is an evidence-preserving research harness. Correctness of the research
record is more important than convenience or model fluency.

- The model is not an authority. Persisted source evidence is the system of record.
- Do not promote inference, similarity, chronology, or model confidence into causal
  or genealogical claims without the required evidence.
- Preserve provenance across retrieval, extraction, interpretation, review, and
  reporting.
- Never treat search snippets, page furniture, retroactive metadata, or inferred
  dates as equivalent to verified contemporary source text.
- Keep page publication dates separate from dates verified for the quoted text.
- Keep observed historical usage separate from claims about origin, lineage, or
  transmission.
- Strong epistemic states must be enforced by code-level rules, not merely requested
  in prompts.
- Provider failures, malformed output, retries, partial writes, and process crashes
  are expected operating conditions. They must not silently corrupt or advance
  research state.
- Durable multi-record state transitions should be atomic when partial completion
  would make resume behavior ambiguous.
- Preserve failed retrievals and unresolved sources as explicit gaps rather than
  silently dropping them.
- Do not weaken provenance or review requirements merely to accommodate a model or
  provider.
- Treat provider-reported usage as authoritative when available. Clearly distinguish
  reported, locally estimated, and unknown usage.
- Keep application-side limits independent from provider-side spending limits.

## Structure

- When several solutions satisfy the requirements, prefer, in order: fewer concepts, fewer moving parts, fewer
  dependencies, less hidden behavior, easier failure recovery.
- Keep functions cohesive and reasonably small, but don't split code just to meet a line count.
- `db.py` alone talks to SQLite. The adapters in `retrieval/` (search, fetch, Wayback) and `llm/` (OpenAI-compatible
  chat) don't import `db` or `research`; `research.py` drives them.

## Testing

- The behaviors worth a test: durable state transitions and recovery, protocol parsing and validation, deduplication and
  retry, persistence and crash consistency, provenance and epistemic promotion boundaries, authority and permission
  boundaries, artifact identity and integrity, provider failure handling, and externally observable workflow behavior.
- Prefer an integration test when the property crosses a process, persistence, protocol, provider or filesystem
  boundary.
- Don't test every enum value or branch on its own, or what types, compilation or the framework already guarantee.
- Where compiling, type checking, linting or a manual run gives more confidence than a unit test, use that instead.
- Skip a test for behavior that a slice's end-to-end acceptance check already covers better.
- When fixing a live failure, first identify the concrete failure mode and preserve a regression test for that boundary.
  Do not use the incident as justification for unrelated refactoring or architecture expansion.

## Running it

- `make check` runs ruff and the tests; none of them needs an API key or the network.

## Privacy

- Config shape lives in [install.md](install.md): every setting is an environment variable it names.
- Read secrets from the environment or explicitly designated external secret files.
- Do not log secrets or full request headers.
- Diagnostic logging for model/provider failures should preserve useful response
  metadata without leaking credentials or unnecessarily storing full prompts.

## Known deviations

Where Hypertrace doesn't follow the org standards yet. These are debt to pay down, not overrides: each one should go.

- Tooling: there's no `.python-version`; `requires-python` is `>=3.13`, and the tests pass on 3.15. The dev tools are an
  extra, not a `dev` dependency group, and CI installs with `uv sync --extra dev`, not `uv sync --frozen`.
- Ruff: the line length is 100, not 110, and the rules are ruff's defaults, not the org `ruff.toml`'s (75 findings under
  it).
- `make check` runs ruff and pytest. mypy isn't configured or run; `mypy --strict` reports 79 errors.
- Data modeling: internal records (`ResearchQuestion`, `Hypothesis`, `Source`, `Evidence`, `ResearchRun` and others in
  `models.py`) are Pydantic models, not frozen dataclasses. Two dataclasses are mutable (`llm/base.py`, `research.py`).
- Schema: an inline `SCHEMA` string and `Database._migrate`'s `ALTER TABLE`s, not numbered `NNN_name.sql` files.
- Errors: an operator error raises `ValueError` with a capitalized message ("Set LLM_API_KEY for research runs"), so it
  ends in a traceback, not `SystemExit`.
- Concurrency: the code is async (`httpx.AsyncClient`, one `asyncio.run`) without fanning out.
- Testing: `test_core.py` isn't named for the behavior it covers.
- Dependencies: `httpx` and `pydantic` are capped (`<1`, `<3`) without a comment naming the break.
- Dependencies: there's no `docs/stack.md`, so the hard-to-undo choices (SQLite, Brave, an OpenAI-compatible API) have
  no ADR.
- Structure: `Database.add_lead`, `has_pending_archive_retry` and `has_pending_search_retry` have no caller.
- Privacy: `HYPERTRACE_MAX_CONSECUTIVE_TARGET_SEARCHES` isn't in [install.md](install.md).
- SQL: seven `StrEnum` columns (`hypotheses.status`, `evidence.evidence_type` and others) have no `CHECK`, and five
  `CHECK`ed sets (`research_notes.kind` and others) have no `StrEnum`.
- SQL: `Database._migrate`'s `ALTER TABLE`s run before its transaction starts, so each commits on its own.
- SQL: runs for different questions can open the database at once, but it isn't in WAL, and writes don't start with
  `BEGIN IMMEDIATE`.
- Docs: there's no `docs/SPEC.md` or `docs/ROADMAP.md`. What Hypertrace is meant to do is spread across the
  [README](../README.md) and [install.md](install.md), and install.md's "v1 backend" limits (paywalls, JavaScript-only
  pages, book scans, archive-specific dating) have no slice saying whether a later version lifts them.
