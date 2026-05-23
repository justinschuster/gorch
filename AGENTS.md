# Agent Notes

## Current State

- This repo is still an initial planning skeleton; `PLAN.md` is the highest-signal source of project direction.
- `README.md` currently contains only the project title, so do not infer commands or architecture from it yet.
- `.sbx/` is a local sandbox/worktree area and is ignored; do not treat files under it as source of truth.

## Product Constraints

- gorch is Python-first: target users are Python data engineers and platform engineers.
- Use `pipeline` terminology consistently in user-facing code, CLI, docs, DB tables, and specs.
- Avoid `workflow`, `flow`, and `dag` as primary names; `DAG` is only acceptable in explanatory graph-validation docs.
- Public Python import should be `gorch`; public decorator should be `@pipeline`.
- Initial Python execution is local-only through the Python SDK; the Go runtime should not execute Python tasks in the first commit.

## Planned Initial Architecture

- Python SDK owns `@pipeline`, `@task`, dependency inference, `run_local()`, and pipeline spec export.
- Go owns `PipelineSpec` parsing/validation, graph validation, CLI, and persisted metadata/state.
- SQLite is the first state store at `.gorch/gorch.db`; design storage through interfaces so PostgreSQL can be added later.
- SQLite stores metadata/state only, not task outputs, artifacts, logs, datasets, or return values.
- Do not add a config file format in the initial commit.

## Expected Commands

- Once Go code exists, use `go test ./...` for Go verification.
- Once Python code exists, prefer standard-library tests with `cd python && python -m unittest discover` unless the repo explicitly adds dev-only pytest tooling.
- Planned runtime CLI shape is platform-oriented: `gorch db init`, `gorch pipeline validate`, `gorch pipeline register`, `gorch pipeline list`, `gorch run create`, and `gorch run get`.

## Dependency Guidance

- Keep Python runtime dependencies at zero for the initial commit.
- If Python test tooling is added, keep it dev-only; `unittest` is preferred while the project is dependency-free.
- Planned Go dependencies are `github.com/spf13/cobra`, `github.com/google/uuid`, and `modernc.org/sqlite`.

## Files To Read First

- Read `PLAN.md` before making implementation choices.
- Read `.gitignore` before adding generated files; it currently ignores Go build/test artifacts, env files, Go workspaces, and `.sbx/`.
