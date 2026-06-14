# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

gh-ost is a triggerless online schema migration tool for MySQL. It applies a table `ALTER` by creating a "ghost" table, backfilling existing rows, and tailing the binary log to replay ongoing DML onto the ghost table, then atomically swaps it in at cut-over.

## Commands

```bash
script/build          # build bin/gh-ost (bootstraps a pinned Go toolchain under .gopath)
script/test            # gofmt check + build + run all Go unit tests
script/lint            # golangci-lint (config: .golangci.yml)
```

Run a single unit test (the scripts vendor Go into `.gopath`; for ad-hoc runs use the repo's Go directly):

```bash
go test ./go/logic/ -run TestMigrator -v
go test ./go/... -v          # all unit tests
```

`script/test` passes extra args through to `go test` (e.g. `script/test -run TestFoo`). Code must be `gofmt -s` clean — `script/test` fails on any diff.

### Replica integration tests (`localtests/`)

These run real migrations against a primary/replica MySQL pair in Docker. Each subdirectory of `localtests/` is one test case (see `doc/local-tests.md`).

```bash
script/docker-gh-ost-replica-tests up          # start primary+replica containers (TEST_MYSQL_IMAGE overrides image)
script/docker-gh-ost-replica-tests run         # build gh-ost and run all cases
script/docker-gh-ost-replica-tests run 'name$' # run cases whose path matches the grep pattern
script/docker-gh-ost-replica-tests down        # tear down
```

Each case runs gh-ost with `--test-on-replica` (migrates on the replica, then swaps the table back) and checksums the original vs ghost table. A case directory may contain: `create.sql` (required, seeds the table on the master), `extra_args`, `before.sql`/`after.sql`/`destroy.sql`, `expect_failure`, `expect_table_structure`, `orig_columns`/`ghost_columns`/`order_by`, `ignore_versions`, `sql_mode`, `gtid_mode`, or a custom `test.sh`.

Test-harness env vars: `THROTTLE_SECONDS` (artificial copy-throttle window, default 0), `GH_OST_TEST_USE_DOCKER=1` (force `docker exec` instead of the host `mysql` client in the wrappers). The MySQL containers present a test cert from `script/docker/certs/` so OpenSSL clients (sysbench) can connect over TLS to the `caching_sha2_password` users.

## Architecture

Entry point `go/cmd/gh-ost/main.go` parses ~100 flags into a `base.MigrationContext`, then hands off to `logic.Migrator`.

- **`go/base/context.go` — `MigrationContext`**: the single shared-state object threaded through every component (connection configs, the chosen unique key, column lists, throttle state, row-copy progress, cut-over settings). Most behavior is driven by fields set here.
- **`go/logic/migrator.go` — `Migrator`**: the orchestrator. Owns the overall state machine: inspect → create ghost table → start binlog streaming → copy rows in chunks while applying live DML → wait for cut-over readiness → cut over. Runs the row-copy and binlog-apply loops and the status/heartbeat reporting.
- **`go/logic/inspect.go` — `Inspector`**: reads `information_schema`, validates the table/replica, and **picks the shared unique key** used for chunking (this choice drives the `force index` in every copy query). Runs against the replica under `--test-on-replica`.
- **`go/logic/applier.go` — `Applier`**: writes side. Creates the ghost (`_gho`) and changelog (`_ghc`) tables, executes chunked `INSERT ... SELECT` row copies, applies DML events, and performs the cut-over (magic-table lock + atomic rename). Also creates the checkpoint table.
- **`go/logic/streamer.go` + `go/binlog/`**: tails the binary log (via go-mysql), decodes row events, and feeds them as DML to the applier. `streamer.go` routes events; `binlog/` decodes them.
- **`go/logic/throttler.go` — `Throttler`**: continuously evaluates throttle conditions (replica lag, `--max-load`/`--critical-load`, `--throttle-query`, flag/HTTP controls) and gates row copy.
- **`go/logic/server.go`**: the interactive command server (unix socket / TCP) for runtime control (`status`, `throttle`, `unpostpone`, etc. — see `doc/interactive-commands.md`).
- **`go/logic/checkpoint.go`**: `--checkpoint`/`--resume` support — persists copy progress and binlog coords to a checkpoint table so an interrupted migration can resume.
- **`go/sql/builder.go`**: generates all the migration SQL (chunked range selects, DML replay). `go/sql/types.go` and `encoding.go` handle column-type and value encoding across the original/ghost boundary.
- **`go/mysql/`**: connection config, `InstanceKey` host identity, GTID helpers, and MySQL-vs-MariaDB replica terminology (`replica_terminology_map.go`).

**Cut-over** is the subtle part (`doc/cut-over.md`): gh-ost uses a "magic table" + voluntary lock so the rename blocks until all in-flight binlog events up to the lock point are applied, guaranteeing no lost rows. Under `--test-on-replica` it stops replication, performs the same cut-over, then swaps back — and leaves replication stopped (the integration harness restarts it per test).

Deep-dive docs live in `doc/` (e.g. `triggerless-design.md`, `shared-key.md`, `throttle.md`, `cut-over.md`, `resume.md`, `command-line-flags.md`).
