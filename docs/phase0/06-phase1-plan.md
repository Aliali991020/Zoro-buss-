# Phase 0 — Exact Changes Planned for Phase 1 (Cloud Foundation)

**Status:** PROPOSED — **waiting for owner approval** (mission §12: no Phase 1 code until approved).
**Scope discipline:** this is the full Phase 1 surface. No Discord bot, no video pipeline,
no publishing — those are Phases 2–4.

---

## 1. What Phase 1 delivers

A **running, tested cloud foundation**: PostgreSQL + durable queue + coordinator API +
worker runtime with leases/heartbeats/retries/recovery + object-storage abstraction +
health/status/event log + budget caps + tool-registry schema — deployed to a free cloud VM,
verified with an end-to-end "hello job" while the owner's laptop is off (mini acceptance
test A), with backups.

## 2. Proposed file tree (exact)

```
Zoro-buss-/
├── pyproject.toml                 # package `studio`, deps pinned, [dev]=pytest,ruff
├── .env.example                   # every required env var documented, no secrets
├── .gitignore                     # .env, *.db, venv, artifacts, __pycache__
├── README.md                      # rewritten: honest status + quickstart
├── docker-compose.yml             # postgres + coordinator + worker (cloud VM)
├── Dockerfile                     # single image, both roles via entrypoint
├── alembic.ini
├── migrations/                    # Alembic revisions (0001 initial schema …)
├── src/studio/
│   ├── __init__.py                # __version__ = "0.1.0"
│   ├── config.py                  # pydantic-settings; env-driven; safe defaults
│   ├── db.py                      # SQLAlchemy engine/session; Postgres-first
│   ├── models.py                  # projects, channels, jobs, tasks, dependencies,
│   │                              # worker_leases, approvals, audit_events,
│   │                              # cost_ledger, asset_sources, tools, settings
│   ├── schemas.py                 # pydantic request/response + task I/O envelopes
│   ├── queue/
│   │   ├── base.py                # Broker protocol (enqueue, claim, ack, fail, dlq)
│   │   └── pg_broker.py           # FOR UPDATE SKIP LOCKED + LISTEN/NOTIFY
│   ├── coordinator/
│   │   ├── app.py                 # FastAPI: /health, /jobs, /tasks, /workers,
│   │   │                          # /events, /budget, /storage/*, /system/pause
│   │   ├── jobs.py                # lifecycle, dependency resolution, resume
│   │   ├── budgets.py             # caps, thresholds, pause-on-exceeded
│   │   └── audit.py               # append-only event writer
│   ├── worker/
│   │   ├── runtime.py             # register→loop→claim→lease-extend→ack/fail
│   │   ├── registry.py            # capability match, heartbeat, lease expiry, recovery
│   │   ├── retry.py               # exponential backoff + jitter + max attempts + DLQ
│   │   └── tasks/
│   │       ├── base.py            # TypedTask protocol: spec(), validate(), run()
│   │       └── hello.py           # the one real Phase-1 task (writes artifact)
│   ├── storage/
│   │   ├── base.py                # ObjectStore interface (put/get/list/stat/checksum/
│   │   │                          # quota/delete-with-confirm/short-link)
│   │   └── local.py               # local-FS implementation (tests + VM scratch)
│   ├── tools/
│   │   ├── registry.py            # tool records, licence fields, status workflow
│   │   └── seed.py                # first ~10 verified entries (licence-checked)
│   └── logging.py                 # structured logs, redaction of secrets
├── tests/
│   ├── conftest.py                # embedded-Postgres fixture (skip-with-reason fallback)
│   ├── test_models.py
│   ├── test_queue.py              # deps, claim exclusivity, retry, DLQ, cancel
│   ├── test_coordinator_api.py
│   ├── test_worker_lease_recovery.py
│   ├── test_budget_enforcement.py
│   ├── test_storage_local.py      # checksum, quota warning, resume interrupted upload
│   └── test_hello_e2e.py          # coordinator↔worker↔storage on one machine
├── deploy/
│   ├── README.md                  # zero-cost VM setup steps (Oracle), secrets via env
│   ├── bootstrap.sh               # install docker, clone, .env from example, compose up
│   └── backup.sh                  # pg_dump → timestamped → storage adapter
└── docs/phase0/                   # (already committed) the six Phase-0 deliverables
```

**~35–45 files total, no placeholder junk** — every file has a test that exercises it.

## 3. Database schema (initial migration `0001`)

Tables (columns abbreviated; full DDL written during implementation):

| Table | Purpose | Key columns |
|---|---|---|
| `projects` | tenancy boundary | id, name, channel_id, status, budget_cents, created_at |
| `channels` | brand profiles | id, name, language, niche, style_profile |
| `jobs` | top-level production runs | id, project_id, type, state, idempotency_key, checkpoint, requested_by |
| `tasks` | stage units | id, job_id, type, state, attempt, max_attempts, input_json, output_json, not_before |
| `task_dependencies` | DAG edges | pred_task_id, succ_task_id |
| `workers` | registered workers | id, kind, capabilities_json, cpu, ram_mb, gpu, host, status |
| `worker_leases` | claim/heartbeat | task_id, worker_id, lease_expires_at, heartbeat_at |
| `approvals` | human gates | id, job_id, kind, state, decided_by, decided_at, reason |
| `audit_events` | append-only log | id, actor, action, subject, result, at, payload_json |
| `cost_ledger` | usage/costs | id, project_id, provider, units, cost_cents, at |
| `asset_sources` | licence provenance | id, asset_uri, source, licence, hash, checked_at |
| `tools` | tool registry | name, repo_url, category, licence, commercial_use, hw_reqs, api_reqs, free_tier_limits, fallback, status |
| `settings` | system flags | key, value (e.g. `system.paused`) |

Indexes: `(state, not_before)` on tasks, partial unique on `jobs.idempotency_key`,
`lease_expires_at` on leases, `(project_id, at)` on cost_ledger.

## 4. Queue semantics (test-enforced)

- Claim = atomic `SELECT … FOR UPDATE SKIP LOCKED` + lease row; **two workers never run the
  same task** (tested).
- Dependencies: successor becomes claimable only when all predecessors `SUCCEEDED`.
- Retry: exponential backoff + jitter; `attempt ≥ max_attempts` → dead-letter state, visible
  via API; manual requeue endpoint.
- Cancellation propagates to pending successors; running task cancelled at next heartbeat.
- `system.paused` flag stops all dispatch (API + DB) — Discord will flip it in Phase 2.

## 5. Worker contract (test-enforced)

1. `POST /workers/register` → worker id + capabilities.
2. Loop: claim matching task → lease 30 s TTL → heartbeat every 10 s → ack/fail.
3. Crash mid-task: lease expires → registry requeues with `attempt+1` (recovery test).
4. Graceful shutdown: finish current task or release lease within timeout.
5. Every task run writes an `audit_events` row + cost ledger entry (even $0 = units).

## 6. Coordinator API surface (Phase 1)

```
GET  /health                     → db, broker, storage, paused flag
GET/POST /jobs                   → list / create (validated input, idempotency key)
GET  /jobs/{id}                  → state + tasks + events
POST /jobs/{id}/cancel
GET  /tasks?state=               → queue visibility
GET  /workers                    → registry + live leases
GET  /events?subject=            → audit trail
GET/PUT /budget                  → caps, current spend, thresholds
POST /system/pause | /resume     → emergency stop (owner-only in Phase 2)
GET  /storage/quota              → adapter quota (local impl returns disk stats)
GET  /tools                      → registry listing
```

Auth in Phase 1: bearer token from env (short-lived internal). Full RBAC lands Phase 2.

## 7. Dependencies (justified, small)

| Package | Why | Size risk |
|---|---|---|
| fastapi, uvicorn | API + async | low, standard |
| sqlalchemy≥2, alembic | ORM + migrations | low |
| pydantic, pydantic-settings | typed schemas/config | low |
| psycopg[binary] | Postgres driver | low |
| httpx | worker→API calls | low |
| pytest, pytest-asyncio, ruff | test/lint | dev-only |
| pgserver (dev extra) | embedded Postgres for integration tests without Docker | dev-only, skipped if unavailable |

**Not installed:** Discord libs (Phase 2), ffmpeg bindings (Phase 3), ML/GPU stacks (never
by default), anything requiring paid keys.

## 8. Exact commands Phase 1 will run (and report)

```bash
python3 -m venv .venv && . .venv/bin/activate
pip install -e ".[dev]"
ruff check src tests
pytest -q                          # must be green (integration skips = explicit, not silent)
# on cloud VM:
./deploy/bootstrap.sh              # docker compose up -d
./deploy/backup.sh                 # pg_dump → storage adapter → verify restore in drill
# acceptance mini-test:
curl -s localhost:8000/health      # ok
# create hello job via API → stop laptop → worker completes → artifact checksum matches
```

## 9. Testing strategy

- **Unit:** models, backoff math, budget caps, storage checksums.
- **Integration (real Postgres):** queue concurrency, lease expiry recovery, dependency
  gating, API round-trips. Uses embedded PG; if unavailable, tests **skip with a loud
  reason** (never silently pass).
- **E2E:** `test_hello_e2e.py` = create job → worker executes → artifact in storage →
  events recorded → audit complete.
- **CI:** GitHub Actions runs `ruff + pytest` on every push to the feature branch.
- **Rollback drill:** `alembic downgrade base` + `alembic upgrade head` on a scratch DB.

## 10. Security & cost implications

- No secrets committed: `.env.example` only; real values injected on the VM.
- Bearer auth + input validation on every endpoint; structured logs with redaction.
- Budget caps enforced in code; exceeding cap → pause + audit event (test G foundation).
- Cost this phase: **$0** (free VM + open-source deps). No paid API calls in Phase 1.
- Licence review for every new dependency recorded in the PR description.

## 11. Known limitations (honest)

- Single-VM Postgres = single point of failure until backups + restore drill pass.
- Broker is Postgres-based; high-throughput Redis/RabbitMQ upgrade is designed-for but
  not implemented until measured need.
- No Discord, no video, no publishing yet — by design.
- Storage adapter ships local-FS only; Drive/S3 pending the owner's 5TB provider answer.
- If the original v2.1.0 repository surfaces mid-phase, a migration task is inserted.

## 12. Rollback plan

1. Phase 1 lands on a feature branch → PR → merge only when tests green.
2. Revert merge commit → app returns to Phase-0-only state (docs remain).
3. DB: `alembic downgrade` (no destructive production data exists yet).
4. VM: `docker compose down` removes services; volumes optional-keep.

## 13. Deliverables Phase 1 must produce at completion (spec §15)

changed files · commands executed · test results · security & cost notes · known
limitations · rollback instructions · next-phase proposal (Phase 2: Discord control plane).
