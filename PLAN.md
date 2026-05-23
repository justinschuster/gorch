# Initial Commit Plan

## Product Direction

gorch is a Python-first data pipeline orchestrator with a Go runtime and control-plane foundation.

The target users are:

- Python data engineers defining, testing, and eventually running pipelines.
- Platform engineers operating pipeline state, runs, storage, workers, and schedulers.

The first commit should prove the core contract:

- Python users define pipelines with decorators.
- Python users can run pipelines locally.
- Python users can export a pipeline spec.
- Go validates and persists pipeline metadata/state.
- SQLite is the initial durable state store.
- PostgreSQL can be added later through the same storage abstraction.

## Confirmed Decisions

- Primary user API is Python.
- Python package import is `gorch`.
- Public decorator is `@pipeline`.
- Use decorators for task and pipeline definitions.
- Start with a Python local runner.
- Go runtime does not execute Python tasks in the initial commit.
- CLI focuses on platform/runtime operations.
- SQLite stores metadata/state only.
- Default SQLite path is `.gorch/gorch.db`.
- No config file in the initial commit.
- Defer contribution/governance docs.
- Use `pipeline` terminology consistently.
- Python runtime dependencies should be zero for the initial commit.
- Python tests can use either standard-library `unittest` or a dev-only `pytest` dependency. Prefer `unittest` if keeping the first commit dependency-free is more important than pytest ergonomics.

## Terminology Rules

Use these names consistently:

| Area | Name |
| --- | --- |
| Python decorator | `@pipeline` |
| Python object | `Pipeline` |
| Go domain model | `PipelineSpec` |
| CLI noun | `pipeline` |
| SQLite table | `pipelines` |
| Foreign key | `pipeline_id` |
| JSON spec | `pipeline spec` |
| Execution instance | `run` |
| Task execution instance | `task_run` |
| Graph internals | `node`, `edge`, `dependency` |

Avoid these as primary names:

- `workflow`
- `flow`
- `dag`

`DAG` can appear only in explanatory docs when comparing concepts or describing graph validation.

## Initial Commit Scope

Include only the foundation needed to demonstrate the architecture.

Must include:

- Go module and CLI skeleton.
- Python package skeleton.
- Python `@pipeline` and `@task` decorators.
- Python dependency inference from task composition.
- Python `run_local()` execution.
- Python pipeline spec export.
- Go `PipelineSpec` parser.
- Go pipeline validator.
- SQLite-backed metadata store.
- CLI commands for pipeline registration and run state creation.
- Architecture, storage, Python API, and roadmap docs.
- Basic Go and Python tests.

Explicitly defer:

- Go executing Python tasks.
- Python worker process.
- Scheduler.
- Retries.
- Backfills.
- Distributed execution.
- API server.
- Web UI.
- Artifact/result storage.
- PostgreSQL implementation.
- Config file format.

## Proposed Repository Layout

```text
gorch/
  cmd/
    gorch/
      main.go

  internal/
    cli/
      root.go
      db.go
      pipeline.go
      run.go

    engine/
      spec.go
      validate.go
      graph.go
      state.go

    storage/
      store.go
      models.go

    storage/sqlite/
      store.go
      migrations.go
      migrations/
        001_initial.sql

  pkg/
    gorch/
      spec.go
      validate.go

  python/
    gorch/
      __init__.py
      pipeline.py
      task.py
      context.py
      runner.py
      spec.py

    tests/
      test_pipeline.py
      test_task.py
      test_runner.py
      test_spec.py

    pyproject.toml
    README.md

  examples/
    python/
      hello_pipeline.py

  docs/
    architecture.md
    python-api.md
    storage.md
    roadmap.md

  go.mod
  go.sum
  README.md
  LICENSE
  .gitignore
```

## Python API

Initial user experience:

```python
from gorch import pipeline, task

@task
def extract():
    return [1, 2, 3]

@task
def transform(values):
    return [value * 2 for value in values]

@task
def load(values):
    print(values)

@pipeline
def daily_pipeline():
    load(transform(extract()))

if __name__ == "__main__":
    daily_pipeline.run_local()
    daily_pipeline.export("daily_pipeline.json")
```

Support both forms if the implementation stays small:

```python
@pipeline
def daily_pipeline():
    ...

@pipeline(name="daily")
def daily_pipeline():
    ...
```

## Python Local Runner

The local runner should:

- Execute task functions in dependency order.
- Pass task outputs to dependent task functions.
- Raise the original task error on failure.
- Keep implementation simple and synchronous.
- Avoid multiprocessing, async execution, retries, and persistence for now.

Initial local execution can remain entirely in Python and does not need SQLite.

## Dependency Inference

Use dependency inference from task composition.

Example:

```python
load(transform(extract()))
```

Produces:

```text
extract -> transform -> load
```

Implementation idea:

- Calling a decorated task inside a pipeline context creates a task invocation node.
- Arguments that are outputs of previous task invocations become dependencies.
- The pipeline object records invocation order and dependency edges.
- Outside a pipeline context, task functions should behave like normal Python callables.

If the same task function is invoked more than once, generate stable unique task IDs:

```text
extract
extract_2
extract_3
```

This should be documented in the Python API docs.

## Pipeline Spec

Initial JSON spec:

```json
{
  "spec_version": "v1",
  "name": "daily_pipeline",
  "tasks": [
    {
      "id": "extract",
      "name": "extract",
      "dependencies": []
    },
    {
      "id": "transform",
      "name": "transform",
      "dependencies": ["extract"]
    },
    {
      "id": "load",
      "name": "load",
      "dependencies": ["transform"]
    }
  ]
}
```

## Go Engine

The Go engine should own validation and graph behavior for exported pipeline specs.

Initial responsibilities:

- Parse pipeline spec JSON.
- Validate required fields.
- Reject duplicate task IDs.
- Reject missing dependencies.
- Reject self-dependencies.
- Detect cycles.
- Produce deterministic topological order.
- Return useful validation errors for CLI output.

Suggested Go types:

```go
type PipelineSpec struct {
    SpecVersion string     `json:"spec_version"`
    Name        string     `json:"name"`
    Tasks       []TaskSpec `json:"tasks"`
}

type TaskSpec struct {
    ID           string   `json:"id"`
    Name         string   `json:"name"`
    Dependencies []string `json:"dependencies"`
}
```

## SQLite Storage

SQLite stores only metadata/state. It should not store task outputs, return values, datasets, logs, or artifacts in the initial commit.

Use `database/sql` and isolate SQLite-specific code under `internal/storage/sqlite`.

Recommended driver:

```text
modernc.org/sqlite
```

Reason: pure Go, easier cross-platform setup, no cgo dependency.

## Storage Interface

Keep the interface database-neutral:

```go
type Store interface {
    CreatePipeline(ctx context.Context, pipeline Pipeline) error
    GetPipeline(ctx context.Context, id string) (Pipeline, error)
    ListPipelines(ctx context.Context) ([]Pipeline, error)

    CreateRun(ctx context.Context, run Run) error
    GetRun(ctx context.Context, id string) (Run, error)
    ListRuns(ctx context.Context, pipelineID string) ([]Run, error)
    UpdateRunStatus(ctx context.Context, id string, status RunStatus) error

    CreateTaskRun(ctx context.Context, taskRun TaskRun) error
    ListTaskRuns(ctx context.Context, runID string) ([]TaskRun, error)
    UpdateTaskRunStatus(ctx context.Context, id string, status TaskRunStatus, errText *string) error
}
```

## SQLite Schema

```sql
CREATE TABLE schema_migrations (
    version INTEGER PRIMARY KEY,
    applied_at TIMESTAMP NOT NULL
);

CREATE TABLE pipelines (
    id TEXT PRIMARY KEY,
    name TEXT NOT NULL,
    spec_version TEXT NOT NULL,
    spec_json TEXT NOT NULL,
    created_at TIMESTAMP NOT NULL
);

CREATE TABLE runs (
    id TEXT PRIMARY KEY,
    pipeline_id TEXT NOT NULL,
    status TEXT NOT NULL,
    started_at TIMESTAMP,
    finished_at TIMESTAMP,
    created_at TIMESTAMP NOT NULL,
    FOREIGN KEY (pipeline_id) REFERENCES pipelines(id)
);

CREATE TABLE task_runs (
    id TEXT PRIMARY KEY,
    run_id TEXT NOT NULL,
    task_id TEXT NOT NULL,
    status TEXT NOT NULL,
    started_at TIMESTAMP,
    finished_at TIMESTAMP,
    error TEXT,
    created_at TIMESTAMP NOT NULL,
    FOREIGN KEY (run_id) REFERENCES runs(id)
);

CREATE INDEX idx_runs_pipeline_id ON runs(pipeline_id);
CREATE INDEX idx_task_runs_run_id ON task_runs(run_id);
CREATE INDEX idx_task_runs_status ON task_runs(status);
```

Enable SQLite foreign keys on connection:

```sql
PRAGMA foreign_keys = ON;
```

## State Model

Initial run statuses:

```text
pending
running
success
failed
cancelled
```

Initial task run statuses:

```text
pending
running
success
failed
skipped
```

For the first commit, `run create` should create:

- One `runs` row.
- One `task_runs` row per task in the registered pipeline spec.
- Initial statuses as `pending`.

It should not execute tasks.

## CLI

Platform/runtime-oriented commands:

```bash
gorch db init --database .gorch/gorch.db

gorch pipeline validate pipeline.json

gorch pipeline register pipeline.json \
  --database .gorch/gorch.db

gorch pipeline list \
  --database .gorch/gorch.db

gorch run create <pipeline-id> \
  --database .gorch/gorch.db

gorch run get <run-id> \
  --database .gorch/gorch.db
```

Optional if small:

```bash
gorch run list <pipeline-id> \
  --database .gorch/gorch.db
```

Default behavior:

- If `--database` is omitted, use `.gorch/gorch.db`.
- Commands requiring the DB should create `.gorch/` if needed.
- `db init` should be idempotent.
- `pipeline register` should validate the spec before storing it.

## README

The root README should explain:

- `gorch` is Python-first.
- Go is the runtime/control-plane foundation.
- Pipelines are defined in Python.
- Pipeline metadata and run state are stored in SQLite.
- Python task execution is local only in the initial version.
- Go runtime execution of Python tasks is future work.

Include quickstart examples:

```bash
go test ./...
```

```bash
cd python
python -m unittest discover
```

```python
from gorch import pipeline, task
```

```bash
gorch pipeline validate examples/python/hello_pipeline.json
gorch pipeline register examples/python/hello_pipeline.json
gorch run create <pipeline-id>
```

## Docs

Add these docs:

`docs/architecture.md`

Cover:

- Python SDK.
- Pipeline spec.
- Go validation engine.
- SQLite storage.
- CLI.
- Future Python worker.
- Future scheduler.

`docs/python-api.md`

Cover:

- `@pipeline`.
- `@task`.
- Dependency inference.
- `run_local()`.
- `export()`.
- Duplicate task invocation naming.

`docs/storage.md`

Cover:

- SQLite-first storage.
- Metadata-only state.
- Schema overview.
- Why outputs/artifacts are excluded.
- PostgreSQL expansion path.

`docs/roadmap.md`

Use phases:

1. Python SDK and SQLite metadata foundation.
2. Go runtime-to-Python worker bridge.
3. Durable execution features: retries, timeouts, cancellation.
4. PostgreSQL storage backend.
5. Scheduler and backfills.
6. API server, UI, metrics, distributed workers.

## Testing Plan

Go tests:

- Parse valid `PipelineSpec`.
- Reject missing `spec_version`.
- Reject unsupported `spec_version`.
- Reject missing pipeline name.
- Reject duplicate task IDs.
- Reject missing dependencies.
- Reject self-dependencies.
- Detect cycles.
- Produce deterministic topological order.
- Apply SQLite migrations.
- Create/list/get pipelines.
- Create/get/list runs.
- Create/list/update task runs.
- Enforce foreign keys.

Python tests:

- `@task` preserves function metadata.
- `@pipeline` creates a pipeline object.
- Task composition infers dependencies.
- `run_local()` executes tasks in dependency order.
- `run_local()` passes upstream outputs to downstream tasks.
- `run_local()` surfaces task exceptions.
- `export()` writes valid JSON.
- Repeated task invocation generates unique IDs.
- Pipeline name defaults from function name.
- Explicit pipeline name works if supported.

## Dependency Choices

Go:

```text
github.com/spf13/cobra
github.com/google/uuid
modernc.org/sqlite
```

Python runtime dependencies:

```text
none
```

Python dev/test dependencies:

```text
none initially, if using unittest
```

If the project later prefers pytest ergonomics, add it as dev-only tooling, not as a runtime dependency.

## Implementation Order For The Initial Commit

1. Initialize Go module and CLI skeleton.
2. Define Go `PipelineSpec` and validation logic.
3. Add SQLite storage interface and implementation.
4. Add CLI commands around validation, pipeline registration, and run creation.
5. Add Python package skeleton.
6. Implement `@task`, `@pipeline`, local runner, and export.
7. Add example Python pipeline.
8. Add tests.
9. Update README and docs.
10. Run formatting and tests.

## Recommended Commit Message

```text
Initialize Python-first pipeline orchestration foundation
```
