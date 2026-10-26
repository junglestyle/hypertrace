<!-- junglestyle/.github code-standards/python.md at 326fdbfb5fe925b46d2313b3fa1aa61a640d4802. Change it there, then run make sync-standards. -->
# Python standards

Decisions for Python code, beneath the org [standards](https://github.com/junglestyle/.github/blob/main/code-standards/README.md). `ruff.toml` holds the mechanical rules; this file holds what a linter can't check.

## Tooling

- uv manages the interpreter, the venv and the lock. Commit `uv.lock`; CI and images run `uv sync --frozen`. No pip installs, Poetry, or requirements files.
- Pin one Python in `.python-version`: the newest release every dependency supports. `requires-python` matches it.
- Dev tools go in `[dependency-groups] dev`. Extras are only for optional runtime features, like `models`.
- Build with hatchling.
- Ruff lints and formats with each repo's `[tool.ruff]` in `pyproject.toml`. The org's [`ruff.toml`](ruff.toml) is the reference that bootstrap copies in once. mypy runs with `strict = true` over the package. Tests aren't annotated or type-checked.
- `make check` runs `ruff check`, `ruff format --check`, `mypy` and `pytest`, and CI runs `make check`.

## Layout

- `src/<package>/`, `tests/`, `docs/` at the root. The import name may be shorter than the project's (`ideamachine` imports as `im`).
- Each command is a `[project.scripts]` entry that calls a `main()`. CLIs use argparse, not click or typer.
- `__init__.py` stays empty: no re-exports, no `__all__`, no import-time work.
- Files the code reads at runtime (SQL, HTML, prompts) live inside the package and load through `importlib.resources`.
- Name a module for the one concept it holds (`queue.py`, `egress.py`). No `utils.py`, `helpers.py` or `common.py`.

## Typing

- Annotate every function in the package, private ones included.
- A seam is a `typing.Protocol` defined beside the code that consumes it. Implementations don't inherit from it, and there are no ABCs.
- `Any` appears only where untyped data enters (decoded JSON, a driver row). Parse it into a typed record before it travels further.
- No `cast()`. A `# type: ignore[code]` gets a comment saying why it's safe.

## Data modeling

- Internal records are `@dataclass(frozen=True)`. Make a changed copy with `dataclasses.replace`.
- Pydantic only where outside data needs validating: request bodies and model output. Past that edge, pass dataclasses or plain values.
- A closed set of names (states, kinds) is a `StrEnum` whose values are the strings stored in the database.
- SQL is plain and parameterized, through the stdlib's `sqlite3` or psycopg 3. Raw SQL by default. An ORM or query builder needs an ADR in `docs/stack.md`; the ADR covers its migration tool too (e.g. alembic), and the numbered `NNN_name.sql` rule doesn't apply to that repo.
- Schema changes are numbered `NNN_name.sql` files, each applied once in its own transaction and never edited after it ships.

## Errors

- Raise built-in exceptions (`ValueError`, `LookupError`) unless a caller catches by type. Then define one `<Thing>Error(Exception)` for that boundary, not a hierarchy.
- Messages are lowercase, quote the bad value with `!r`, and say what fixes it when something does: `f"{name} is not set (see .env.example)"`.
- For a failure the operator fixes (a missing setting, an unknown name, a bad argument), raise `SystemExit("message")`: a clear line, exit status 1, no traceback.
- `except Exception` belongs only at a run or loop boundary, where it records the failure and then re-raises or moves on to the next item. Everywhere else, catch the exception you actually handle.
- Docstrings are prose: what the function does and anything the signature can't say. No `Args:`/`Returns:` sections that repeat the signature.

## Output and logging

- A CLI prints its result to stdout, as JSON when another program reads it. Progress and diagnostics go to stderr.
- A service writes plain lines to stdout or stderr and leaves timestamps and retention to journald or Docker. No logging config files or handlers of its own.

## Concurrency

- Synchronous by default. Use async only inside an async framework (FastAPI) or to fan out many I/O calls at once.
- A background thread gets a stop `Event` and is joined on shutdown. Sleep with `stop.wait(n)`, not `time.sleep(n)`.

## Testing

- pytest functions with plain `assert`. No `unittest.TestCase` and no `unittest.mock`.
- Replace a dependency with a small fake that satisfies its Protocol. Use `monkeypatch` for env vars and attributes, and `tmp_path` for files.
- Tests use a real database, never a mocked one. Each run creates its own scratch database, and the code refuses an admin URL that isn't localhost.
- A test that needs an absent service skips with a reason, and `pytest -rs` shows the skips. Security-boundary tests never depend on such a service.
- Fixtures are synthetic and committed. Real data never enters the repo.
- Name test files for the behavior they cover (`test_corrections.py`), not one per module.
- Use `parametrize` when cases differ only in data. Write separate tests when they differ in setup or in what they check.

## Dependencies

- Declare lower bounds only (`httpx>=0.27`); `uv.lock` holds the exact versions. Pin exactly or cap a version only for a known break, with a comment naming it.
- torch comes from the PyTorch CPU index through `[tool.uv.sources]`, unless the code runs on a GPU.

---

Drawn from the [Google Python Style Guide](https://google.github.io/styleguide/pyguide.html) (CC BY 3.0) and the [Scientific Python Development Guide](https://learn.scientific-python.org/development/) (BSD-3-Clause), adapted.
