# Code Review Report — branch `mariadb-support`

Date: 2026-06-09
Scope: `git diff master...HEAD` (710 lines) — localtests Docker replica-test harness rework: docker-compose moved to `script/`, per-flavor config dirs (`mysql-5.7`, `mysql-8.0`, `mysql-8.4`, `percona-server-8.0`), TLS certs, rewritten `script/docker-gh-ost-replica-tests` and `localtests/test.sh`, changed mysql wrapper scripts.

Method: 7 independent finder angles (line-by-line, removed-behavior, cross-file, reuse, simplification, efficiency, altitude) → dedup → 12 candidates → 1-vote adversarial verify each. 0 refuted. Top 10 below, ranked most-severe first; findings 1–5 CONFIRMED breakage, 6–8 PLAUSIBLE, 9–10 CONFIRMED maintenance traps.

## Findings

### 1. CI 5.7 matrix leg dead on arrival — flavor dir missing [CONFIRMED]
`.github/workflows/replica-tests.yml:13`

CI matrix image `mysql/mysql-server:5.7.41` resolves to flavor dir `script/docker/mysql-server-5.7`, which doesn't exist — `compose_setup` hard-exits, so every 5.7 CI job fails at setup (and teardown, since `compose_setup` runs before `down` too).

**Failure scenario:** `compose_setup` strips `mysql/mysql-server:5.7.41` to repo basename `mysql-server` + `5.7` → `CONF_PATH=script/docker/mysql-server-5.7` → "No config dir for image" + exit 1. Only `mysql-5.7` dir exists (its `flavor.env` targets official `mysql:5.7`). Workflow untouched by this branch.

**Fix:** update the workflow matrix to `mysql:5.7` (or add an alias map / explicit `TEST_MYSQL_FLAVOR` override).

### 2. docker-exec fallback drops `-uroot` — Percona auth failure [CONFIRMED]
`script/gh-ost-test-mysql-master:15`, `script/gh-ost-test-mysql-replica:15`

The docker-exec fallback (`docker exec -i -e MYSQL_PWD=opensesame mysql-primary mysql -h 127.0.0.1 "$@"`) omits the explicit `-uroot` that the native-client branch passes, so the client logs in as the container's OS user. Percona images end their Dockerfile with `USER mysql`.

**Failure scenario:** `GH_OST_TEST_USE_DOCKER=1` (or no host mysql client) + `TEST_MYSQL_IMAGE=percona/percona-server:8.0.41-32`: client defaults to MySQL user `mysql` with root's `MYSQL_PWD` → ERROR 1045 on every wrapper call; `setup()` dies at `poll_mysql` despite a healthy server. Works on `mysql:*` images only by accident (exec defaults to root).

**Fix:** add `-uroot` to the docker-exec branch.

### 3. Healthcheck gives mysqld only ~25s first-boot before `--wait` aborts [CONFIRMED]
`script/docker-compose.yml:15`

New healthcheck (interval 5s, start_period 10s, no `retries` override → default 3) plus `docker compose up -d --wait` under `set -euo pipefail`: container marked unhealthy at ~t≈25s, `--wait` exits non-zero, `setup()` dies before the more generous 30s `poll_mysql` loop ever runs.

**Failure scenario:** `mysql-5.7/flavor.env` forces `linux/amd64` → qemu emulation on arm64 hosts, where fresh-datadir init routinely takes 60s+ → `up` fails on exactly the platform `flavor.env` was added to support.

**Fix:** raise `retries` (~10) or `start_period` (e.g. 120s).

### 4. `down` without `-t` leaks the toxiproxy container — regression [CONFIRMED]
`script/docker-gh-ost-replica-tests:107`

`COMPOSE_FILE` includes `docker-compose.toxiproxy.yml` only when the current invocation's `$2 == '-t'`. Old teardown unconditionally ran `docker stop/rm mysql-toxiproxy || true`.

**Failure scenario:** `up -t` then plain `down`: `docker compose down -v` (no `--remove-orphans`) treats `mysql-toxiproxy` as an orphan, warns, leaves it running with ports 8474/23308 bound and stale toxics configured — breaks the next `-t` run and port users.

**Fix:** add `--remove-orphans` to teardown.

### 5. Throttle default 0 silently kills DML-replay test coverage [PLAUSIBLE]
`localtests/test.sh:180`

throttle-query changed from hardcoded `< 5` (guaranteed ~5s throttle window) to `< ${THROTTLE_SECONDS:-0}` (always false → never throttles). Nothing sets `THROTTLE_SECONDS` — not `.github/workflows/replica-tests.yml`, not the driver.

**Failure scenario:** ~72 test cases drive concurrent DML via `create event ... every 1 second`; previously the 5s window guaranteed events fired during the migration. Now binlog DML-replay coverage is probabilistic (depends on gh-ost overhead exceeding 1s before cut-over). Tests still pass as pure row-copy checks on fast runners, masking future DML-replay regressions.

**Fix:** export a nonzero `THROTTLE_SECONDS` in CI, or default it > 0.

### 6. Replica terminology by `8.4` substring match — wrong for MariaDB and MySQL 9.x [PLAUSIBLE]
`localtests/test.sh:100`

`[[ $mysql_version =~ "8.4" ]]` picks replica/slave wording and `Seconds_Behind_Source`/`Master` column.

**Failure scenario:** on this mariadb-support branch: MariaDB 11.8.4 contains `8.4` → picks `Seconds_Behind_Source`, a column MariaDB's `SHOW REPLICA STATUS` lacks → catch-up poll's grep matches nothing and exits immediately every test. MySQL 9.x doesn't contain `8.4` → `stop slave` errors (statement removed in 8.4+).

**Fix:** declare terminology per flavor dir (e.g. in `flavor.env`, which the driver already sources) instead of a second version sniff in shared code; canonical mapping already exists in `go/mysql/replica_terminology_map.go`.

### 7. `wait_replica_caught_up` swallows GTID-wait errors and timeout [PLAUSIBLE]
`localtests/test.sh:137`

`DO WAIT_FOR_EXECUTED_GTID_SET(...)` discards the timeout return value by construction (and the redirect hides errors); on MariaDB the `@@global.gtid_executed` fetch fails silently (`2>/dev/null`) → `master_gtid` empty → every test degrades to bare `sleep 1`, the racy baseline this function was written to eliminate.

**Failure scenario:** MariaDB (branch target): flat 1s sleep per test (~70s/run wasted) AND replica may still be stale → flaky "table not found"/checksum failures with zero diagnostics. Even on MySQL a 10s wait timeout is invisible (`DO` discards return; mysql exits 0).

**Fix:** `SELECT` the wait result and check it; warn on fallback; add MariaDB branch via `MASTER_GTID_WAIT(@@gtid_binlog_pos, 10)`.

### 8. `SSL_CERT_FILE` exported globally — replaces trust store for all children [PLAUSIBLE]
`script/docker-gh-ost-replica-tests:149`

`compose_setup` exports `SSL_CERT_FILE=script/docker/certs/ca.pem` for every subcommand; only sysbench needs it. Go honors `SSL_CERT_FILE` on Linux.

**Failure scenario:** vendored deps shield module downloads, but `go.mod` declares `go 1.25.9` and CI uses the runner's preinstalled Go with no setup-go step — if older, the GOTOOLCHAIN auto-download over HTTPS fails with `x509: certificate signed by unknown authority` under the test-only CA. Environment-dependent.

**Fix:** scope the export to the sysbench invocation (or append the test CA to the system bundle).

### 9. Nine per-flavor files are byte-identical triplicates [CONFIRMED]
`script/docker/{mysql-8.0,mysql-8.4,percona-server-8.0}/`

`common.cnf`, `create_user.sql`, `start_replication.sql` md5-identical across the three 8.x dirs; `compose_setup` has no shared-defaults fallback (single `CONF_PATH`, hard-fail if missing).

**Cost:** adding MariaDB = 4th full copy; any grant/cnf/replication tweak must be hand-copied into 4+ dirs, and a missed copy silently gives one flavor different test conditions masquerading as version incompatibilities.

**Fix:** `docker/default/` dir + per-file fallback in `compose_setup` (mysql-5.7 is the only divergent flavor today).

### 10. Password literal hardcoded in 6 client call sites despite new `.env` [CONFIRMED]
`script/.env:1`

`.env` declares `MYSQL_ROOT_PASSWORD=opensesame` (consumed only by compose), but the literal stays in `gh-ost-test-mysql-master:12,15`, `gh-ost-test-mysql-replica:12,15`, `localtests/test.sh:241,255` (sysbench) with no env indirection.

**Cost:** changing the password in `.env` — the edit the file invites — boots containers with the new password while every client still sends `opensesame` → all tests fail with access-denied pointing at MySQL, not at the stale literals.

**Fix:** wrappers source `script/.env` or use `${MYSQL_ROOT_PASSWORD:-opensesame}`.

## Cut by the 10-finding cap (confirmed, minor)

- `script/docker-gh-ost-replica-tests:98` — function `ps()` shadows the system `ps` binary inside the script; only the subcommand dispatch calls it today, but rename to `compose_ps` (note `localtests/test.sh` uses real `ps -p` — unaffected since the function isn't exported).
- `script/docker-gh-ost-replica-tests:37-51` — `mysql-source()`/`mysql-replica()` are dead code (zero callers after setup rewrite) and still carry the stale `[[ $TEST_MYSQL_IMAGE =~ "mysql:8.4" ]]` `--ssl-mode` special-casing the flavor-dir scheme replaced. Delete both.

## Machine-readable findings

```json
[
  {"file": ".github/workflows/replica-tests.yml", "line": 13, "summary": "CI matrix image 'mysql/mysql-server:5.7.41' resolves to flavor dir script/docker/mysql-server-5.7, which doesn't exist — compose_setup hard-exits, so every 5.7 CI job fails at setup (and teardown).", "failure_scenario": "compose_setup strips 'mysql/mysql-server:5.7.41' to repo basename 'mysql-server' + '5.7' → CONF_PATH=script/docker/mysql-server-5.7 → \"No config dir for image\" + exit 1. Only mysql-5.7 dir exists. Workflow untouched by this branch → 5.7 matrix leg dead on arrival."},
  {"file": "script/gh-ost-test-mysql-master", "line": 15, "summary": "docker-exec fallback drops the explicit -uroot the native branch passes, so the client logs in as the container's OS user — Percona images run as USER mysql → access denied on every wrapper call. Same bug in script/gh-ost-test-mysql-replica:15.", "failure_scenario": "GH_OST_TEST_USE_DOCKER=1 (or no host mysql client) + TEST_MYSQL_IMAGE=percona/percona-server:8.0.41-32: client defaults to MySQL user 'mysql' with root's MYSQL_PWD → ERROR 1045 everywhere; setup() dies at poll_mysql despite healthy server."},
  {"file": "script/docker-compose.yml", "line": 15, "summary": "New healthcheck (interval 5s, start_period 10s, default retries 3) gives mysqld only ~25s of first-boot init before docker compose up --wait marks it unhealthy and aborts setup under set -e.", "failure_scenario": "mysql-5.7 flavor.env forces linux/amd64 → qemu emulation on arm64 hosts, where fresh-datadir init routinely takes 60s+. Container unhealthy at ~25s, --wait exits non-zero, setup() dies before the 30s poll_mysql loop runs."},
  {"file": "script/docker-gh-ost-replica-tests", "line": 107, "summary": "down without -t no longer removes the toxiproxy container that up -t created; old teardown stopped/removed it unconditionally — regression.", "failure_scenario": "up -t then plain down: docker compose down -v (no --remove-orphans) treats mysql-toxiproxy as an orphan and leaves it running with ports 8474/23308 bound and stale toxics — breaks next -t run."},
  {"file": "localtests/test.sh", "line": 180, "summary": "throttle-query default changed from hardcoded '< 5' (guaranteed ~5s throttle window) to '< ${THROTTLE_SECONDS:-0}' (never throttles), and nothing sets THROTTLE_SECONDS — deterministic DML-replay window the ~72 'every 1 second' event tests rely on is gone.", "failure_scenario": "In CI '< 0' is always false → no throttle; binlog DML-replay coverage becomes probabilistic. Tests still pass as pure row-copy checks on fast runners, silently masking future DML-replay regressions."},
  {"file": "localtests/test.sh", "line": 100, "summary": "Replica terminology picked by substring match [[ $mysql_version =~ \"8.4\" ]] — wrong for MariaDB x.8.4 (matches, but MariaDB lacks Seconds_Behind_Source column → wait loop silently no-ops) and MySQL 9.x (no match → 'stop slave' errors).", "failure_scenario": "MariaDB 11.8.4 contains '8.4' → grep Seconds_Behind_Source finds nothing → start_replication catch-up poll exits immediately every test. Declare terminology in flavor.env instead of a second version sniff in shared code."},
  {"file": "localtests/test.sh", "line": 137, "summary": "wait_replica_caught_up discards all errors and the GTID wait result: DO WAIT_FOR_EXECUTED_GTID_SET discards the timeout return value by construction, and on MariaDB the @@global.gtid_executed fetch fails silently → every test degrades to bare 'sleep 1'.", "failure_scenario": "MariaDB: flat sleep 1 per test (~70s/run wasted) AND replica may still be stale → flaky failures with zero diagnostics. Even on MySQL a 10s wait timeout is invisible (DO discards return; mysql exits 0)."},
  {"file": "script/docker-gh-ost-replica-tests", "line": 149, "summary": "compose_setup exports SSL_CERT_FILE=script/docker/certs/ca.pem globally — replacing the trusted CA set for every child process (go build, curl) when only sysbench needs it.", "failure_scenario": "go.mod declares go 1.25.9 and CI uses the runner's preinstalled Go with no setup-go — if older, GOTOOLCHAIN auto-download over HTTPS fails with 'x509: certificate signed by unknown authority' under the test-only CA."},
  {"file": "script/docker/mysql-8.4/common.cnf", "line": 1, "summary": "Nine per-flavor files are byte-identical triplicates (md5-verified): mysql-8.0/8.4/percona-server-8.0 share identical common.cnf, create_user.sql, start_replication.sql; compose_setup has no shared-defaults fallback.", "failure_scenario": "Adding MariaDB = 4th full copy; any tweak must be hand-copied into 4+ dirs, and a missed copy silently gives one flavor different test conditions masquerading as version incompatibilities."},
  {"file": "script/.env", "line": 1, "summary": "New .env declares MYSQL_ROOT_PASSWORD=opensesame as source of truth, but the literal stays hardcoded in 6 client call sites (gh-ost-test-mysql-master:12,15, -replica:12,15, test.sh:241,255) with no env indirection.", "failure_scenario": "Changing the password in .env boots containers with the new password while every wrapper and sysbench still sends 'opensesame' → all tests fail with access-denied pointing at MySQL, not at the stale literals."}
]
```
