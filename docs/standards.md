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

- Dependencies: there's no `docs/stack.md`, so the hard-to-undo choices (SQLite, Brave, an OpenAI-compatible API) have
  no ADR.
- Versions and pins: `pyproject.toml` has no `[tool.uv] required-version`, so CI's `setup-uv`
  (`.github/workflows/test.yml:12`) installs the newest uv.
- Structure: `Database.add_lead`, `has_pending_archive_retry` and `has_pending_search_retry` have no caller.
- Privacy: `HYPERTRACE_MAX_CONSECUTIVE_TARGET_SEARCHES` isn't in [install.md](install.md).
- Docs: the `target-sources` and `archive-target` commands (`cli.py:24-34`) aren't in the README or
  [install.md](install.md), whose "other flags and commands" stop at `run` and `activate-query`.
- Docs: there's no `docs/SPEC.md` or `docs/ROADMAP.md`. What Hypertrace is meant to do is spread across the
  [README](../README.md) and [install.md](install.md), and install.md's "v1 backend" limits (paywalls, JavaScript-only
  pages, book scans, archive-specific dating) have no slice saying whether a later version lifts them.
- Tooling: there's no `.python-version`; `requires-python` is `>=3.13`, and the tests pass on 3.15. The dev tools are an
  extra, not a `dev` dependency group, and CI installs with `uv sync --extra dev`, not `uv sync --frozen`.
- Tooling: ruff's line length is 100, not 110, and the rules are ruff's defaults, not the org `ruff.toml`'s (75 findings
  under it).
- Tooling: `make check` runs ruff and pytest. mypy isn't configured or run; `mypy --strict` reports 79 errors.
- Data modeling: internal records (`ResearchQuestion`, `Hypothesis`, `Source`, `Evidence`, `ResearchRun` and others in
  `models.py`) are Pydantic models, not frozen dataclasses. Two dataclasses are mutable (`llm/base.py`, `research.py`).
- Data modeling: closed sets the code compares or parses from model output have no `StrEnum`. `target_purpose`,
  `queries.information_value` and `queries.admission_basis` are `Literal`s (`models.py:123,211-217`), and
  `research_notes.kind`, `candidates.assessment_mode`, `target_search_progress.outcome`, `source_targets.status` and
  `.dating_status`, `queries.status`, `runs.status` and `provider_attempts.provider_role` are plain strings the code
  compares (`db.py:725,891,995-997,1621,2766`, `research.py:1208`, `reports.py:104`). No test asserts a `StrEnum`'s
  values are its column's `CHECK` list.
- Data modeling: the schema is an inline `SCHEMA` string and `Database._migrate`'s `ALTER TABLE`s, not numbered
  `NNN_name.sql` files.
- Errors: an operator error raises `ValueError` with a capitalized message ("Set LLM_API_KEY for research runs"), so it
  ends in a traceback, not `SystemExit`.
- Output and logging: `llm/openai_compatible.py:440` logs each completion attempt with `logger.info`, but nothing
  configures a handler, so the line never reaches stderr.
- Concurrency: the code is async (`httpx.AsyncClient`, one `asyncio.run`) without fanning out.
- Testing: `test_core.py` isn't named for the behavior it covers.
- Dependencies: `httpx` and `pydantic` are capped (`<1`, `<3`) without a comment naming the break.
- SQL queries: `Database._migrate`'s `ALTER TABLE`s (`db.py:473`) run inside `with self.conn:` but before its first
  `UPDATE` opens the transaction, so each commits on its own.
- SQL queries: `hypertrace research` (`cli.py:125-133`) and `init` (`planner.py:142-162`) commit the question and each
  hypothesis and query in its own transaction. After a crash, `init` finds the question and never adds the rest.
- SQL schema: closed sets with no `CHECK`: the `StrEnum` columns `hypotheses.status`,
  `hypothesis_suggestions.proposed_status`, `evidence.evidence_type`, `.quote_region`, `.term_sense` and `.confidence`,
  `relationships.kind` and `leads.kind`; `target_purpose` on `queries`, `query_target_associations` and
  `plan_query_rejections` (`db.py:97,148,157`), though its other three columns have one; `queries.information_value`,
  `.admission_basis` and `.status`; and sets that live only as strings in code, such as `runs.status`,
  `archive_lookups.status`, `source_targets.evidence_status` and `.dating_status`, `review_cadence_events.decision` and
  `provider_attempts.provider_role`. A database made before `_migrate` added `candidates.assessment_mode` (`db.py:413`)
  has no `CHECK` on it.
- SQLite: runs for different questions can open the database at once, but it isn't in WAL, and writes don't start with
  `BEGIN IMMEDIATE`.
