# etc: design

`etc` is an experiment tracker for machine learning, with a focus on reinforcement learning. It is
written in Zig and vendors SQLite. It consists of a single binary that acts as both the tracking
client (run in the environments where experiments execute) and the tracking server (an
authenticated web service that ingests experiment data and serves a website with visualizations).

This document describes the design. Inspirations and references are cited inline as [n] and listed
at the end.

## 1. Goals and non-goals

Goals:
1. Track ML experiments: hyperparameters, metrics over time, artifacts (checkpoints, videos,
   figures), provenance (git commit, command, host), and human/AI annotations.
2. Serve RL workloads well: multiple time axes (environment steps, episodes, gradient updates,
   wall clock), seeds as first-class data, evaluation interleaved with training, rollout videos,
   long-running and preemptible jobs.
3. Local-first: experiments log to a local SQLite database and never lose data because a server or
   network is down. Syncing to a server is an incremental, idempotent merge.
4. One self-contained binary: vendored SQLite, embedded web assets, no runtime dependencies.
   Cross-compiles statically for cluster deployment.
5. Databases are legible to humans and AI agents: self-describing schema, discoverable CLI,
   shareable and mergeable database files.

Non-goals (v1):
1. Hyperparameter sweep orchestration (W&B Sweeps [2]). Sweep results can be tracked; scheduling
   them is out of scope.
2. A Python SDK. The client contract is the `etc` CLI (including a streaming stdin mode); a thin
   Python wrapper can come later.
3. Multi-process aggregation of a single logical run. Each process is its own run; grouping
   happens at the experiment level.
4. Horizontally scaled serving. One server, one filesystem, SQLite. Litestream-style replication
   [13] covers backup.
5. Model registry, dataset versioning beyond artifact storage.

## 2. Inspirations

ken [1]. `etc` inherits ken's philosophy wholesale, though not its schema: SQLite files as the
unit of ownership and sharing; a schema whose kind/definition tables carry `description` columns
that act as specifications for both humans and AI agents; UUID identity so databases merge; the
guarantee that schema migrations never fail; a CLI discoverable via `-h` at every level of the
command hierarchy; a `skill` subcommand that generates skills for agentic tools. `etc` is the same
worldview applied to a different domain, with a new schema designed for high-rate time-series
ingestion.

Weights & Biases [2]. The product benchmark. We take: the project/run hierarchy, config and
summary as distinct from history, the insight that time axes are themselves logged metrics
(`define_metric` with a custom `step_metric` [3]), run states with heartbeats, offline mode with
later sync, readable auto-generated run names, and the run-table + chart UI. We reject: data held
hostage in a proprietary cloud, a heavyweight client, and pricing by data volume. `etc`'s answer
to "export" is "your project is a SQLite file; download it."

MLflow [4]. Validates the params/metrics/tags/artifacts decomposition and the local-files-or-server
duality. Its metric model (a single step axis) is too weak for RL; we generalize.

TensorBoard [5]. The original append-only event-log model. Good instincts (logging is cheap,
visualization is derived), but file-per-run event logs make querying across runs painful. A
relational store fixes that.

Aim [6] and Sacred [7]. Aim demonstrates that a local, open-source tracker with a strong query
story is viable. Sacred pioneered config capture and reproducibility metadata; its
observer/MongoDB split is a cautionary tale about heavy dependencies.

rliable / "Deep RL at the Edge of the Statistical Precipice" [8]. RL results are distributions
over seeds, not single curves. The schema and UI must make multi-seed aggregation (mean, IQM,
confidence bands) natural rather than an afterthought.

## 3. Architecture

Three pieces, one binary:

```
 experiment host                          server host
+---------------------------+           +-----------------------------+
| training process          |           | etc serve                   |
|   | JSONL over stdin      |   HTTPS   |   web UI (embedded assets)  |
|   v                       |  ------>  |   JSON API (authenticated)  |
| etc log --stdin           |   sync    |   server.db  (control plane)|
|   v                       |           |   projects/<id>.db (data)   |
| local etc db (SQLite)     |           |   objects/   (artifact CAS) |
+---------------------------+           +-----------------------------+
```

1. Core module (`src/`): schema, migrations, ingestion, query, and merge logic over vendored
   SQLite. Shared by client and server; also exposed as a Zig module for embedding.
2. Client: `etc` subcommands that create runs, log metrics, attach artifacts, and sync. All writes
   land in a local SQLite database first.
3. Server: `etc serve`. Ingests via the sync protocol, stores one SQLite database per project plus
   a control-plane database (users, keys, sessions), and serves the website.

Local-first is the load-bearing decision. RL runs last days on preempted cluster nodes with flaky
egress; a tracker that drops data when the network blips is worthless. Logging is a local
transaction; syncing is a background or explicit step; the server can be absent entirely (scp the
database home and `etc merge` it, ken-style [1]).

## 4. Data model

### 4.1 Principles

Inherited from ken [1]:
1. Entity rows get UUID (v4) text primary keys and ISO 8601 UTC `created_at`/`updated_at`
   timestamps. Exception: the hot `samples` table, which is optimized for volume (4.3).
2. Definition tables are keyed by name and carry a `description` column. Descriptions are
   specifications: they tell humans and AI agents how to interpret keys, semantics, and units. The
   more formal the description, the better.
3. Every table has deterministic merge semantics (section 6).
4. Migrations never fail.

New for etc:
5. Hot-path tables use natural composite keys, not UUIDs, so that re-ingestion is idempotent by
   construction.
6. Metric names, not surrogate ids, are the identity of a time series. Charts render from a raw
   database with no joins; names are shorter than UUID text anyway.

### 4.2 Containers: projects, experiments, runs

`projects` — top-level namespace, e.g. one per paper or agent.
1. `id`: UUID text primary key.
2. `name`: unique, human identity of the project (used for cross-database matching, section 6).
3. `description`.
4. `created_at`, `updated_at`.

`experiments` — an optional grouping of runs that test one hypothesis, e.g. "PPO with entropy
bonus 0.01 on Atari-10, 5 seeds". This is the unit of multi-seed aggregation.
1. `id`: UUID text primary key.
2. `project_id`: FK → projects, delete restricted.
3. `name`: unique within project.
4. `description`: the hypothesis. Write it down; your future self and your agents will thank you.
5. `created_at`, `updated_at`.

`runs` — one tracked process execution.
1. `id`: UUID text primary key.
2. `project_id`: FK → projects, required.
3. `experiment_id`: FK → experiments, nullable.
4. `name`: human-readable; auto-generated (adjective-noun-counter, W&B-style [2]) when absent.
5. `seed`: integer, nullable. First-class because RL conclusions are seed distributions [8].
6. `state`: `running` | `finished` | `failed` | `killed`.
7. `exit_status`: integer, nullable (populated by `etc run exec`).
8. `started_at`, `finished_at`, `heartbeat_at`: ISO 8601 UTC. A `running` run with a stale
   heartbeat is displayed as unresponsive; that judgment is computed, never stored.
9. Provenance: `git_commit`, `git_dirty` (0/1), `command`, `hostname`. Captured by default at run
   start; the dirty diff and environment listings (e.g. `pip freeze`) are attached as artifacts.
10. `created_at`, `updated_at`.

### 4.3 Metrics and samples

`metrics` — definition table, one row per metric name per project. Auto-registered on first log;
enriched by hand or by agents afterward.
1. `project_id`: FK → projects.
2. `name`: e.g. `train/loss`, `eval/return`, `env_step`. `/` namespacing groups charts in the UI.
3. `description`: what the metric means, its units, how it is computed. The ken lesson [1]: this
   column is what makes a database legible to an agent three months later.
4. `goal`: `min` | `max` | `none` — lets the UI and queries answer "best".
5. `default_axis`: metric name to plot this metric against by default (nullable → global step).
   This is W&B `define_metric` [3] as a schema column.
6. `created_at`, `updated_at`.
Primary key: `(project_id, name)`.

`samples` — the hot table; everything else is decoration around it.

```sql
CREATE TABLE samples (
  run_id    TEXT    NOT NULL,          -- runs.id
  metric    TEXT    NOT NULL,          -- metrics.name
  step      INTEGER NOT NULL,          -- run-global log step
  value     REAL    NOT NULL,
  logged_at INTEGER NOT NULL,          -- unix epoch milliseconds
  PRIMARY KEY (run_id, metric, step)
) WITHOUT ROWID;
```

Design notes:
1. The primary key makes ingestion idempotent: re-syncing or re-logging a batch is `INSERT OR
   IGNORE` and cannot duplicate data. This single property powers sync (section 6) and
   crash-restart semantics (section 4.6).
2. `WITHOUT ROWID` clusters rows by `(run_id, metric, step)`, so the dominant query — one metric
   of one run in step order — is a contiguous range scan.
3. Time axes are metrics. An RL run logs `env_step`, `episode`, and `grad_update` as ordinary
   metrics alongside `train/loss` at the same `step`. Plotting loss against environment steps is
   a join of the two series on `step`. This is the W&B history model [2][3] made relational, and
   it is what makes the schema RL-complete without RL-specific tables.
4. `logged_at` is integer milliseconds, deviating from ISO 8601 text: it is a wall-clock axis that
   must be cheap to store and compare at millions of rows.
5. Scalars only. Histograms, images, and videos are artifacts, optionally step-indexed (4.5).

`summaries` — one row per (run, metric), maintained transactionally on ingest: `last_value`,
`last_step`, `min_value`, `max_value`, `count`, `updated_at`. Run tables sort and filter on
summaries without ever scanning `samples`. Equivalent to W&B's run summary [2].

### 4.4 Config and tags

`run_config` — flattened key-value hyperparameters, upserted at run start (and until finish).
1. `run_id`, `key` (dotted path, e.g. `optimizer.lr`), primary key `(run_id, key)`.
2. `value_json`: the value as JSON text (source of truth).
3. `value_num`: REAL, populated when the value is numeric — enables "runs where optimizer.lr <
   1e-3" as an indexed comparison instead of JSON parsing.

The raw config file (YAML/JSON) can additionally be attached as an artifact of kind `config`.

`run_tags` — `(run_id, tag)`, primary key both columns. Cheap labels for filtering.

### 4.5 Artifacts, notes, logs

`artifact_kinds` — ken-style definition table [1]: `name` primary key, `description`. Ships with
`checkpoint`, `video`, `figure`, `config`, `git-diff`, `environment`, `dataset`; users add more.

`artifacts` — metadata in the database, bytes in a content-addressed store.
1. `id`: UUID text primary key.
2. `run_id`: FK → runs.
3. `kind`: FK → artifact_kinds.
4. `name`: e.g. `policy.pt`, `rollout.mp4`.
5. `step`: nullable integer — ties an artifact to a point on the run's step axis (checkpoint at
   step 1M; rollout video at step 2M). Step-indexed videos are the RL payoff.
6. `size_bytes`, `sha256`, `media_type`.
7. `created_at`.

Blobs live outside SQLite at `objects/<aa>/<sha256>` (first byte as fan-out directory), keyed by
content hash, git-style. Identical checkpoints dedupe; the database stays small and mergeable; the
CAS is rsync-friendly.

`notes` — annotations by humans or agents on a project, experiment, or run: `id` (UUID),
exactly one of `project_id`/`experiment_id`/`run_id` non-null (CHECK-enforced), `content`,
`created_at`, `updated_at`.

`run_logs` — optional captured stdout/stderr, populated by `etc run exec`: `(run_id, seq)` primary
key, `stream`, `logged_at`, `line`. Off by default; log capture is a firehose and belongs in the
tracker only when you ask for it.

### 4.6 RL-specific consequences

No table above says "RL", deliberately — but the design decisions are RL-driven:
1. Multiple time axes: co-logged axis metrics + `default_axis` (4.3). Train vs eval is a naming
   convention (`train/*`, `eval/*`) documented in metric descriptions, not a schema fork.
2. Episodes: log per-episode quantities (`episode/return`, `episode/length`) against the `episode`
   axis. No episode table; episodes are just another axis.
3. Seeds: `runs.seed` plus experiment grouping gives the UI what it needs for mean/IQM/CI bands
   across seed-replicates [8].
4. Preemption and resume: to resume a run from a checkpoint at step k after logging past k, run
   `etc run resume <run> --truncate-after <k>` — it deletes samples with step > k, then logging
   continues on the same run id. Without truncation, the idempotent primary key would silently
   keep stale samples from the dead branch of the run.
5. Rollout videos: step-indexed `video` artifacts, rendered inline on the run page.

## 5. Ingestion

Three paths, slowest to fastest:
1. One-shot CLI: `etc log -r <run> --step 100 train/loss=0.42 env_step=51200`. Process-spawn cost
   ~ms; fine for episodic logging or shell-based experiments.
2. Streaming: the training process spawns `etc log --stdin` once and writes JSONL lines
   (`{"step":100,"metrics":{"train/loss":0.42,"env_step":51200}}`) to its stdin. etc batches rows
   into transactions (flush every 100 ms or 1000 rows, whichever first), updates summaries and the
   run heartbeat on flush. Works from any language that can spawn a subprocess; this is the v1
   answer to "where is the Python SDK?".
3. Direct HTTP: POST the same JSONL to the server's sync endpoint, for environments where running
   a binary is undesirable.

SQLite settings for the write path: WAL mode [10], `synchronous=NORMAL`, one writer connection,
prepared statements reused across batches. Batched transactional inserts into a WITHOUT ROWID
table comfortably sustain 10^5+ rows/sec [9] — orders of magnitude above any real training loop's
logging rate.

Environment variable conventions, so training code needs no argument plumbing: `ETC_DB`,
`ETC_PROJECT`, `ETC_RUN_ID`, `ETC_REMOTE`, `ETC_API_KEY`.

## 6. Merge and sync

Merging is the core primitive, inherited from ken [1] and promoted: `etc sync` is a merge over
HTTP, and `etc merge -f other.db` is the same merge applied to a local file. Both run as a single
transaction per batch.

Merge semantics by table class:
1. Definitions (`metrics`, `artifact_kinds`): matched by name. Same name + same description →
   skip. Same name + different description → conflict, because descriptions are specifications
   (ken's rule [1]). Default abort; `--force` keeps the target's description; `--check` reports
   only.
2. Containers (`projects`, `experiments`): matched by name (name is the human identity). When
   names match but UUIDs differ — two hosts independently created project `atari` — source rows
   are rewritten to the target's id during merge. UUIDs are identity for rows created once and
   synced everywhere; names are identity for things humans create independently and expect to be
   the same thing.
3. Leaves (`runs`, `artifacts`, `notes`): UUID identity is authoritative; `INSERT OR IGNORE`.
4. Facts (`samples`, `summaries`, `run_config`, `run_tags`, `run_logs`): natural composite keys;
   `INSERT OR IGNORE`, target wins. Summaries are recomputed for any (run, metric) touched.

Sync protocol:
1. Handshake: `GET /api/v1/sync/info` → server schema version and per-table cursors for the
   client's runs. A server refuses data from a client with a newer schema and tells it so.
2. Push: `POST /api/v1/sync` with a body of JSONL batches (`{"table":"samples","rows":[...]}`),
   applied under the merge semantics above; response reports per-table accepted counts.
3. The client tracks per-remote, per-table high-water marks in a local `sync_state` table as an
   optimization only. Correctness never depends on cursors: idempotent merge means re-sending
   anything is safe. `etc sync` retries dumb and hard.
4. Artifacts: metadata syncs in the batch; blobs upload separately by sha256
   (`PUT /api/v1/objects/<sha256>`), skipped when the server already has the hash.

Live mode: `etc log --stdin --sync` flushes batches to the remote on the same cadence as local
commits, degrading silently to local-only when the network fails and catching up on reconnect.

## 7. Server

`etc serve --data-dir <dir> --addr <host:port>`. Data layout:

```
<data-dir>/
  server.db            control plane: users, keys, sessions
  projects/<uuid>.db   one data-plane database per project, exact client schema
  objects/<aa>/<sha>   artifact CAS
```

One SQLite file per project keeps writer contention per-project, makes archival `cp`, and makes
"download your project" literal: `GET /api/v1/projects/<id>/download` streams the database file
(via the SQLite backup API for a consistent snapshot). Data liberation is a feature, not a
grudging export [2].

### 7.1 Authentication

Control-plane schema: `users` (id, username unique, password_hash, created_at, disabled_at),
`api_keys` (id, user_id, name, token_prefix, token_hash, created_at, last_used_at, revoked_at),
`sessions` (id, user_id, token_hash, created_at, expires_at).

1. Passwords: argon2id [11] via Zig's `std.crypto.pwhash.argon2` — no external crypto dependency.
2. API keys: 256-bit random, shown once at creation, stored as SHA-256 hashes; a short stored
   prefix supports identification. Sent as `Authorization: Bearer <key>`; constant-time
   comparison. Used by `etc sync` and any programmatic access.
3. Browser sessions: opaque random cookie (HttpOnly, Secure, SameSite=Lax), server-side session
   row, CSRF token on mutating form posts.
4. No self-serve signup: `etc user add` / `etc key issue` are admin CLI commands run against the
   server data dir. Lab-scale, deliberately.
5. v1 authorization is coarse: any authenticated user reads and writes any project. Per-project
   roles are future work; the ACL lands in server.db without touching data-plane files.
6. TLS terminates at a reverse proxy (Caddy, nginx). Serving TLS from Zig's std is not yet a
   battle-tested story, and etc should not pretend otherwise.

### 7.2 HTTP API

JSON over HTTP, `/api/v1/`:
1. Sync and objects endpoints (section 6).
2. `GET /projects`, `GET /projects/<id>/runs?tag=&state=&config.optimizer.lr=<1e-3&sort=summary.eval/return`
   — run listing driven entirely by `run_config.value_num` and `summaries`.
3. `GET /runs/<id>` — config, summaries, tags, artifacts, notes.
4. `GET /runs/<id>/history?metric=train/loss&x=env_step&points=1500` — the chart feed. The server
   joins the metric against the axis on `step` and downsamples to ≤ points buckets using min/max
   binning per bucket (spikes survive; a loss explosion you smoothed away is a lie). LTTB [12] is
   the documented alternative if binning proves visually inadequate.
5. `GET /experiments/<id>/aggregate?metric=eval/return&x=env_step&stat=mean|iqm&band=std|ci95` —
   multi-seed aggregation done server-side [8].

Implementation: Zig `std.http.Server` with a small thread pool; a lab-scale tracker does not need
more. If std proves insufficient, http.zig [14] is the fallback — vendored, like everything else.

### 7.3 Web UI

Server-rendered HTML with embedded CSS/JS (`@embedFile`); charts by vendored uPlot [15] (~50 KB,
built for exactly this: dense time series). No CDN, no external requests, works on an airgapped
cluster network.

Views:
1. Projects → run table: sortable/filterable by config columns, summary columns, tags, state;
   stale-heartbeat runs flagged.
2. Run page: charts grouped by metric namespace (`train/`, `eval/`, `sys/`), x-axis selector
   (step, wall time, or any axis metric), config, artifacts with inline video playback, notes.
3. Compare view: overlay N runs; group by experiment; aggregate bands (mean/IQM ± std/CI) for
   seed groups [8].

## 8. CLI

ken's discoverability rules [1]: every command and subcommand answers `-h` scoped to itself;
every database-touching command accepts `-D/--db <path>` (default: platform data dir, e.g.
`~/Library/Application Support/etc/etc.db`); query commands emit JSON for machine consumption.

```
etc init                                    create/upgrade a database
etc project  create|list|show|download
etc experiment create|list|show
etc run      start|finish|resume|exec|list|show
etc log      [-r RUN] [--step N] k=v ...    one-shot logging
etc log      --stdin [--sync]               streaming logging
etc metric   define|list                    write those descriptions
etc artifact add|get|list
etc note     add|list
etc tag      add|rm
etc query    runs|samples ...               filtered JSON output
etc sync     [--remote URL] [--check]
etc merge    -f <source.db> [--check|--force|--nocheck]
etc serve    --data-dir DIR --addr ADDR
etc user     add|list|disable               server admin
etc key      issue|revoke|list              server admin
etc skill                                   generate agent skills (ken heritage)
```

`etc run exec -- python train.py` wraps a command: creates the run, captures git provenance and
environment, streams stdout/stderr to `run_logs` if asked, records exit status, finishes the run
with the right state on exit or signal.

## 9. Build and vendoring

1. Zig ≥ 0.16, same toolchain baseline as ken [1].
2. SQLite amalgamation [9] vendored at `lib/sqlite3.c` / `lib/sqlite3.h`, compiled into the module
   with `-DSQLITE_OMIT_LOAD_EXTENSION`, `link_libc = true` — ken's exact arrangement.
3. Web assets (HTML templates, CSS, JS, uPlot [15]) vendored in-tree and embedded with
   `@embedFile`. The release artifact is one file.
4. No fetched Zig dependencies in v1. Anything adopted later (e.g. http.zig [14]) gets vendored.
5. `zig build` / `zig build run` / `zig build test`; cross-compile with
   `-Dtarget=x86_64-linux-musl` (or aarch64) for static cluster binaries.

## 10. Migrations and compatibility

1. `meta` table holds `schema_version`. Every release ships forward-only migrations from every
   prior version, and — ken's guarantee [1] — these migrations never fail.
2. A binary opening a newer-schema database refuses with a clear "upgrade etc" error rather than
   guessing.
3. The sync handshake carries schema versions; the server must be at least as new as the client.
   Upgrade servers first.
4. `samples` is append-only in normal operation; migrations avoid rewriting it wherever possible
   because it is the table with a billion rows in it.

## 11. Security posture

1. All API and UI access authenticated; no anonymous read in v1.
2. Secrets at rest are hashes (argon2id for passwords [11], SHA-256 for high-entropy tokens).
3. SQL exclusively through bound parameters (the ken wrapper's `execParams` pattern).
4. Artifact CAS paths are derived from hashes, never from user-supplied names; downloads set
   `Content-Disposition` from sanitized names.
5. Config values and notes render HTML-escaped; the UI ships a restrictive Content-Security-Policy
   (trivial, since everything is same-origin by construction).
6. TLS via reverse proxy (7.1).

## 12. Future work

1. Per-project roles and teams.
2. Thin Python client (wraps `etc log --stdin` or the HTTP API).
3. Sweeps: experiment-level config search tracked natively.
4. Reports: saved chart/note compositions, W&B-reports-like [2].
5. Metric compaction policies for multi-year retention (downsample cold runs in place).
6. BLOB(16) UUID storage if text UUIDs measurably bloat hot tables.

## References

[1] ken — catalog of research literature; source of etc's schema philosophy, merge semantics, CLI
    discoverability, and build arrangement. https://github.com/zomglings/ken
[2] Weights & Biases documentation. https://docs.wandb.ai/
[3] W&B `define_metric` — custom x-axes for logged metrics.
    https://docs.wandb.ai/guides/track/log/customize-logging-axes
[4] MLflow Tracking. https://mlflow.org/docs/latest/tracking.html
[5] TensorBoard. https://www.tensorflow.org/tensorboard
[6] Aim — open-source experiment tracker. https://github.com/aimhubio/aim
[7] Sacred — experiment configuration and reproducibility. https://github.com/IDSIA/sacred
[8] Agarwal et al., "Deep Reinforcement Learning at the Edge of the Statistical Precipice",
    NeurIPS 2021 (rliable). https://arxiv.org/abs/2108.13264
[9] SQLite amalgamation. https://sqlite.org/amalgamation.html ; insert throughput and
    transactions: https://www.sqlite.org/faq.html#q19
[10] SQLite write-ahead logging. https://sqlite.org/wal.html
[11] Argon2, RFC 9106. https://www.rfc-editor.org/rfc/rfc9106
[12] Sveinn Steinarsson, "Downsampling Time Series for Visual Representation" (LTTB), University
    of Iceland, 2013. https://skemman.is/handle/1946/15343
[13] Litestream — SQLite streaming replication. https://litestream.io/
[14] http.zig — HTTP server library for Zig. https://github.com/karlseguin/http.zig
[15] uPlot — fast time-series charting. https://github.com/leeoniya/uPlot
