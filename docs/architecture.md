# Architecture

`framecontrol` exists for long-horizon video generation by
autoregressive chaining: each run starts from the previous run's output.
Each run of a chained job selects its own backend: Draw Things through
`draw-things-cli`, or H3 image-to-audio-video through the project-owned h3.c
installation, and one job may mix them. One executor runs the chain; front ends (CLI, TUI,
API, MCP) are thin layers over it. The run lock permits one supervised backend
command at a time on the machine.

## Overview

```
CLI (Typer) ─┐   TUI (Textual) ─┐   HTTP API ─┐            MCP server ─▶ HTTP API
             ▼                  ▼             ▼
                 services: Toolkit, JobRunSession, JobCatalog, HistoryReader
             │
             ▼
   state (SQLite: Store, repositories, recorder) ── run lock (flock)
             │
             ▼
   jobs: JobExecutor ◀── events, cancel() ── JobPlanner, RunLauncher, JobRecords, JobLogWriter
             │                             │
             └──────── backend registry ───┘ (Draw Things and H3 adapters)
             │
             ▼
   core: supervised process runner ──▶ backend executable
```

## Source layout

Everything is one package, `src/framecontrol/`; nothing lives in the
project root but configuration, data, docs, and tests. This directory layout
is the responsibility guide for the source tree; changing it requires owner
approval. Dependencies point one way, from front ends down to `core`, and
`tests/test_architecture.py` enforces it from the imports.

```
src/framecontrol/
├── core/        # paths, errors, exit codes, YAML, configuration, arguments, the runner and its process package
├── jobs/        # job definition and parsing, the executor and its parts, events, records, inputs, media
├── state/       # SQLite database, repositories, typed rows, recorder, history import
├── services/    # the use cases every front end shares, and the wiring of the real tools
├── cli/         # Typer app: the `fc` command
├── tui/         # Textual app: screens, panes/, text/, the command controller
├── server/      # HTTP API and gRPC monitoring service; its queue worker is in services/
└── mcp_server/  # MCP server: typed tools over the HTTP API, its SSE watch included, on stdio or Streamable HTTP
tests/           # mirrors the package: tests/core, tests/jobs, tests/state, tests/services, ...
```

Import direction: `cli`, `tui`, `server` -> `services` -> `state` -> `jobs` ->
`core`; `mcp_server` imports nothing else from the package and talks to `server`
over HTTP only, with no gRPC. Front ends never import each other, except that three commands in
`cli/app.py` start one: `fc tui` the TUI app, `fc serve` the HTTP API and gRPC
service, and `fc mcp` the MCP server (none of those imports `cli`). Nothing below the front ends imports Typer, Textual, a
web, MCP, or gRPC framework.
Launch with `fc` or `python -m framecontrol`.

Services receive their runner and executable lookup as dependencies, so tests
never start a process.

**Process supervision.** The runner starts the CLI in its own process group and
reads stdout and stderr concurrently, as bytes, stamping each line when its last
byte arrives (not when the supervisor next looks at the queue). `SIGHUP`, `SIGINT`, and `SIGTERM` send
`SIGTERM` to the group, wait `--shutdown-grace` (10 s), then `SIGKILL`.
`--timeout` uses the same sequence. Without `--output`, the child inherits the
terminal for inline preview. Every project-owned subprocess starts through a
clean helper interpreter reached with `posix_spawn`; that helper closes inherited
descriptors, creates the requested session, changes directory, and execs the
real command. The threaded gRPC server and clients therefore never fork their
live C-core state into a child.

**Exit codes.** 0 success; 1 run wrote no output, last-frame extraction
failed, or the state database or lock cannot be used; 2 invalid input; 75 run
lock held by another run; 124 timeout; 128+N stopped by signal N (130 for
Ctrl-C); otherwise the CLI's own code.

## Runtime ownership

`fc serve` owns the run lock, state store, queue host, and the only worker
allowed to start `draw-things-cli`. It executes one entry at a time. Shutdown
stops the current child and releases the lock only after the worker has joined;
startup recovers entries left running by a previous server.

A queued job stores a versioned snapshot of the exact job text, each Draw Things
profile's base-configuration text, backend identity, resolved directories, and
cooldown policy. Version-2 snapshots (versions 7 and 8; version 8 also enforces
that the job name matches its filename) store the compiled plan, which is
compared with what the frozen text recompiles to on every read, so a
resume reads the snapshot and never a live file, profile, or reference. H3 queue snapshots also store private copies, digests, and media facts for every reference pair. Retention removes those copies when their queue snapshots are gone. H3 snapshots deliberately exclude the mutable local model snapshot, which is rechecked before execution and resume. Older
rows without a backend read as Draw Things. Backend IDs in older queued snapshots are not translated; a snapshot naming `draw-things` or `h3` must be resubmitted with `draw-things-cli` or `h3-metal-cli`. A queued one-off
generation stores its resolved request and also identifies Draw Things as its
backend. Later file edits therefore do not change queued work. Resuming copies
the original snapshot and backend into a new entry at the next unfinished run
with the original seed; an interrupted run always restarts from its boundary.

The SQLite store keeps execution, run, queue, job-ID, audit, and preference
state. Public IDs such as `E0012`, `Q0007`, and `J0001` are stable and
monotonic. Retention prunes database history, not generated output; resumable
parked chains are retained while they are still needed.

**Stage timing.** Every run of either backend leaves a measured timeline: the
child's stages as its output named them, then Framecontrol's own finishing, on
one run clock whose gaps are named `unaccounted`, so the stages add up to the
run. It is kept in `run_stages` and `run_timing`, which belong to the
execution: `history_retention_days` pruning and deleting an execution remove
them with it, so a forecast fitted on them sees the retained window and nothing
older. The timeline is the executor's, built from events a front end can also
read (`StageStarted`, `StageFinished`, `TimingStarted`, `TimingFinished`); the
recorder keeps each span when its stage finishes. A one-off `generate` is not a
job run and is not timed.

**Forecast.** The server computes one completion-time forecast of the running job
and every front end shows it. `services/timing_history.py` loads the retained
`complete` runs and fits `FittedModel` once per completed run (timing rows
recorded under the pre-rename backend ids join the current backends),
`jobs/timing/forecast.py` turns the queue worker's job events into a `Forecast`
(the active stage and its units, `run` and `job` spans with a range, and a
basis), and `services/forecast_publisher.py` applies the publishing rule (at a
change, and within a stage at most once every 2 s), holds the span's `floor` so
a bar never moves back, and sends `forecast_updated` through
`QueueEventPublisher`, not as a `JobEvent`, so the recorder and the job log never
see it. The HTTP queue entry, the SSE watch, gRPC's `QueueEntrySnapshot` and
MCP carry it; `services/forecast_view.py` is the one place that turns a span
into seconds left, a share done and end text, and
`services/timing_replay.py` scores an estimator on the retained runs
(`fc timing replay`). `tests/test_architecture.py` fails if `tui/`, `cli/` or
`mcp_server/` imports the model, its knob types or factory or fit, or its
loader and updater, so no front end can grow a second forecast; `fc timing knobs`
(`cli/timing_app.py`) calls only the public `services/timing_knobs.py`.

**Timing knobs.** The forecast's editable numbers are two plain values handed to
`FittedModel`: `Tunables` (estimator decisions, from `tuning.yaml`) and
`Calibration` (what this host measured, from one file per backend with a section
per pipeline). The factory pair is code in `jobs/timing/factory.py`: the two
measured points (E0053 and E0051/E0054, still priced narrowly where a run matches
them exactly) and broad power-law priors for everything else. `TimingHistory`
reads the files through `KnobReader`, which parses them again only when a file's
size or time changes, so a hand edit or an update reaches the next forecast
without a restart. The updater is on only under `fc serve`
(`QueueHost(learn_timing=True)`), so a test or a library user never writes the
project's data directory; the data directory comes from `ProjectPaths.timing_data`
and is passed down like every other path.

## Interfaces

The CLI and TUI use the HTTP API for queue control. The TUI and
`fc queue add --wait` observe live work over authenticated gRPC; MCP uses the
API's authenticated SSE watch. `fc mcp` is an HTTP-only client of
`fc serve` and is available over stdio or Streamable HTTP.

The TUI reads display data through services and renders typed events; it never
parses logs. The API and MCP layers translate requests and responses but keep
business rules in `services/`. The MCP package imports no other project
package and reaches the server only over HTTP.

Job-file creation, replacement, and recoverable deletion require
`fc serve --allow-write`. Replacements use SHA-256 optimistic concurrency,
and backups and trashed files remain under `data/jobs/`. Execution deletion is
separate and permanent.

## Safety boundaries

- Paths originate in `ProjectPaths` and are passed down. Job input, output,
  job-file, and named-configuration paths are confined and bounded before use.
- The API is bearer-authenticated except for health. Non-loopback binding
  requires an explicit override; request bodies, job sizes, run counts, and
  worst-case durations are limited.
- Queue mutation is serialized with submission and worker claims. Agents cannot
  control entries or holds created by people.
- Audits record actions and typed outcomes without prompt text, YAML, or
  credentials. Credentials never enter events, storage, logs, or responses.
- `data/params/*.json`, `data/params/*.yaml`, and
  `config/global-config.yaml` are never edited by an interface.

## Design history

The architecture above describes the current system. Rationale, superseded
choices, and implementation history stay in the phase documents rather than
being repeated here:

- [Phase 1](archive/phase-1/README.md): jobs and chained runs
- [Phase 2](archive/phase-2/README.md): TUI, shared state, and the run lock
- [Phase 3](archive/phase-3/README.md): API, queue, and MCP
- [Phase 4](phase-4/README.md): multiple backends, H3 video, the Framecontrol rename, and version-2 jobs
