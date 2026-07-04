# AGENTS.md

Guidance for AI agents working in this repository. CLAUDE.md is a symlink to this file. Adapted
from ken (https://github.com/zomglings/ken) and clacks (https://github.com/downstairs-dawgs/clacks).

Some proverbs that guide our development of programs:
- Slow is smooth and smooth is fast.
- Measure twice, cut once.

## Communication

In all your writing, be terse, concise, precise, and information dense. Your writing is meant to
be consumed by both humans and AI.

Do not overuse Markdown formatting, but do use some amount of it. Headings, newlines, code blocks
are good. Excessive use of bold, italics, and strikethroughs is bad.

No flattery. No emojis. Do not waste the user's time or your tokens.

## Background

`etc` is an experiment tracker for machine learning experiments, with a focus on reinforcement
learning. It is a thin wrapper around a database. The database schema is the most important aspect
of the program.

design.md is the authoritative design document. Read it before making changes; keep it current as
the implementation evolves. The essentials:

- We use SQLite (vendored). Databases are files: shareable by copying, combinable by merging.
- Local-first: experiment hosts log to a local database; syncing to the server is an idempotent
  merge. The server is optional.
- `etc serve` is an authenticated web service that ingests experiment data and serves
  visualizations.
- The hot `samples` table is keyed `(run_id, metric, step)`; re-ingestion is idempotent by
  construction. Time axes (env steps, episodes, gradient updates) are themselves metrics.
- `description` columns on definition tables (metrics, artifact kinds) are specifications for both
  humans and AI agents. Write them carefully.
- Schema migrations across `etc` versions must never fail.
- Every CLI command that operates against a database accepts the global `-D/--db <path>` flag, and
  every command and subcommand answers `-h`/`--help` scoped to itself.

## Technology

This is a Zig project. The SQLite amalgamation is vendored in `lib/` and compiled into the module
with `-DSQLITE_OMIT_LOAD_EXTENSION`. No fetched dependencies: anything adopted gets vendored.

Standard commands:
- `zig build` — build the project
- `zig build run` — build and run
- `zig build test` — run all tests
- `zig test src/<file>.zig` — run tests in a single file

The user is still new to Zig (`etc` is his second Zig program, after ken). Guide him through the
experience in a forthright and candid manner.

## Working practices

- Use CLI tools instead of complicated MCP servers etc. whenever possible.
- Make as much of your work reproducible as you can. Store artifacts (files, etc.) and scripts for
  how to make use of them. Always make a directory (even a temporary one) for each group of files,
  unless one has already been specified.
- Always respect .gitignore.
- NEVER amend commits. Always create new commits. No exceptions.
- NEVER rebase. Use merge commits to integrate changes.
- Do not claim credit on commit messages and PR descriptions.
- Before committing, run all checks:

```
zig fmt --check build.zig src/
zig build
zig build test
```
