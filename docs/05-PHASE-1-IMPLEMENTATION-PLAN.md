# 05 — Phase 1 Implementation Plan (Cloud Foundation)

**Status:** PROPOSED — **STOP: no Phase 1 code until the owner explicitly approves this plan.**
**In scope:** cloud foundation only. **Out of scope:** Discord bot (Phase 2) · video pipeline (Phase 3) · publishing (Phase 4) · any paid service.

---

## 1. Objective

A running, tested cloud foundation: **PostgreSQL + durable queue + coordinator API + worker runtime (leases/heartbeats/retries/recovery) + object-storage abstraction (Google Drive adapter once OAuth exists) + budget caps + tool-registry schema**, deployed on a free cloud VM, verified end-to-end with the owner's laptop off (mini acceptance test A/C/F/G).

## 2. Exact file changes (Phase 1 file tree)

```
Zoro-buss-/
├── pyproject.toml                    # package `studio`; pinned deps; [dev] = pytest, ruff, pgserver
├── .env.example                      # every required env var documented — NO secret values
├── .gitignore                        # .env, *.db, venv/, artifacts/, __pycache__/
├── README.md                         # updated: honest status + quickstart (docs stay truthful)
├── Dockerfile                        # one image; role selected via entrypoint (coordinator|worker)
├── docker-compose.yml                # postgres + coordinator + worker (cloud VM)
├── alembic.ini
├── migrations/
│   └── 0001_initial_schema.py        # forward + reversible; downgrade drill in tests
├── src/studio/
│   ├── __init__.py                   # __version__ = "0.1.0"
│   ├── config.py                     # pydantic-settings, env-driven, safe defaults
│   ├── db.py                         # SQLAlchemy engine/session; Postgres-first
│   ├── models.py                     # 15 tables (see §3)
│   ├── schemas.py                    # pydantic envelopes: JobInput, TaskIO, API models
│   ├── logging.py                    # structured logs + secret redaction
│   ├── queue/
│   │   ├── base.py                   # Broker protocol: enqueue/claim/ack/fail/cancel/dlq
│   │   └── pg_broker.py              # FOR UPDATE SKIP LOCKED + LISTEN/NOTIFY
│   ├── coordinator/
│   │   ├── app.py                    # FastAPI routes (see §5)
│   │   ├── jobs.py                   # lifecycle, dependency resolution, resume-at-checkpoint
│   │   ├── budgets.py                # caps, thresholds, pause-on-exceeded, provider gates
│   │   └── audit.py                  # append-only event writer
│   ├── worker/
│   │   ├── runtime.py                # register → loop → claim → lease-extend → ack/fail
│   │   ├── registry.py               # capability match, heartbeat, expiry reaping, recovery
│   │   ├── retry.py                  # exponential backoff + jitter + max attempts → DLQ
│   │   └── tasks/
│   │       ├── base.py               # TypedTask protocol: spec() validate() run()
│   │       └── hello.py              # the one real Phase-1 task: writes + checksums an artifact
│   ├── storage/
│   │   ├── base.py                   # ObjectStore: put/get/list/stat/checksum/quota/confirm-delete/temp-link
│   │   ├── local.py                  # local-FS implementation (tests + VM scratch)
│   │   └── gdrive.py                 # Google Drive adapter (added once owner OAuth exists)
│   └── tools/
│       ├── registry.py               # tool records, licence fields, status workflow
│       └── seed.py                   # first ~10 verified entries (licence-checked, no bulk)
├── tests/
│   ├── conftest.py                   # embedded-Postgres fixture; loud skip-if-unavailable
│   ├── test_models.py
│   ├── test_queue.py                 # deps, claim exclusivity, retry, DLQ, cancel
│   ├── test_coordinator_api.py
│   ├── test_worker_lease_recovery.py
│   ├── test_budget_enforcement.py
│   ├── test_storage_local.py         # checksum, quota warning, interrupted-upload resume
│   ├── test_tool_registry.py
│   └── test_hello_e2e.py             # coordinator↔worker↔storage, one machine
├── .github/workflows/ci.yml          # ruff + pytest on every push (CI, not a server)
└── deploy/
    ├── README.md                     # zero-cost VM setup (Oracle), secrets via env manager
    ├── bootstrap.sh                  # install docker, clone, compose up, migrations
    └── backup.sh                     # pg_dump → Drive 00-system-backups/ → restore drill
```

**Total ≈ 35–45 files. Every file is exercised by a test — no placeholders, no empty skills, no fake agents.**

**Changed files vs today:** everything above is *new* (repo currently has only `README.md` + `docs/`). No existing working code is modified because none exists.

## 3. Database schema (migration `0001`)

| Table | Purpose | Key columns |
|---|---|---|
| `projects` | tenancy + budgets | id, name, status, budget_cents, created_at |
| `channels` | brand/niche/language/format profiles | id, name, language, niche, style_profile |
| `jobs` | production runs | id, project_id, type, state, idempotency_key UNIQUE, checkpoint JSON, requested_by |
| `tasks` | stage units | id, job_id, type, state, attempt, max_attempts, input/output JSON, not_before, priority |
| `task_dependencies` | DAG edges | pred_task_id, succ_task_id |
| `workers` | registry | id, kind, capabilities JSON, cpu, ram_mb, gpu, host, status, last_seen |
| `worker_leases` | claim safety | task_id UNIQUE, worker_id, lease_expires_at, heartbeat_at |
| `approvals` | human gates | id, job_id, kind, state, decided_by, decided_at, reason |
| `audit_events` | append-only log | id, actor, action, subject, result, at, payload JSON |
| `cost_ledger` | usage & spend | id, project_id, provider, units, cost_cents, at |
| `asset_sources` | copyright provenance | id, asset_uri, source, licence, hash, checked_at |
| `tools` | 100+ registry | name, repo_url, category, description, install_notes, os_compat, hw_reqs, licence, commercial_use, api_reqs, free_tier_limits, fallback, status |
| `comment_inbox` | Phase 5 ready | id, platform, external_id, text, classification, reply_state |
| `revenue_records` | Phase 7 ready | id, module, ref, amount_cents, kind(actual\|pipeline), at |
| `settings` | system flags | key, value (`system.paused`, provider enablement) |

**Indexes:** `(state, not_before)` on tasks · partial unique `jobs.idempotency_key` · `worker_leaves.lease_expires_at` · `(project_id, at)` on cost_ledger · append-only trigger on audit_events.

## 4. Queue & worker semantics (test-enforced)

1. **Claim exclusivity:** `SELECT … FOR UPDATE SKIP LOCKED` + lease row — *two workers can never run the same task* (asserted in `test_queue.py`).
2. **Dependencies:** successor claimable only when all predecessors `SUCCEEDED`.
3. **Retry:** exponential backoff + jitter; `attempt ≥ max_attempts` → dead-letter state, listed via API, manual requeue endpoint.
4. **Cancellation:** propagates to pending successors; running task released at next heartbeat.
5. **Worker loop:** `POST /workers/register` → claim matching task → lease TTL 30 s → heartbeat 10 s → ack/fail; graceful shutdown releases lease.
6. **Crash recovery:** lease expiry ⇒ registry requeues with `attempt+1` (asserted in `test_worker_lease_recovery.py`).
7. **Emergency stop:** `settings['system.paused']=true` stops all dispatch (API flag; Discord button wires to it in Phase 2).
8. **Every run** writes an `audit_events` row + `cost_ledger` entry (even $0 = recorded units).

## 5. Coordinator API surface (Phase 1)

```
GET  /health                  → db, broker, storage, paused flag, version
GET/POST /jobs                → list / create (schema-validated, idempotency key)
GET  /jobs/{id}               → state + tasks + recent events
POST /jobs/{id}/cancel
GET  /tasks?state=            → queue visibility (truthful)
GET  /workers                 → registry + live leases
GET  /events?subject=         → audit trail
GET/PUT /budget               → caps, spend-to-date, thresholds
POST /system/pause | /resume  → emergency stop
GET  /storage/quota           → adapter quota (local impl = disk stats)
GET  /tools                   → registry listing
POST /workers/{id}/heartbeat  → lease extension
```

**Auth Phase 1:** static bearer token from secret manager (short-lived internal trust). Full RBAC + Discord identity mapping arrives Phase 2.

## 6. Dependencies (justified, small)

| Package | Why | Risk |
|---|---|---|
| fastapi, uvicorn | async API + OpenAPI | low |
| sqlalchemy≥2, alembic | ORM + reversible migrations | low |
| pydantic, pydantic-settings | typed schemas/config | low |
| psycopg[binary] | Postgres driver | low |
| httpx | worker→API | low |
| pytest, pytest-asyncio, ruff | test/lint (dev) | dev-only |
| pgserver (dev extra) | embedded Postgres for integration tests **without Docker** (this sandbox has none); loud skip if unavailable | dev-only |

**Explicitly NOT installed in Phase 1:** discord libs, ffmpeg bindings, ML/GPU stacks, anything needing paid keys.

## 7. Acceptance tests for Phase 1 (must pass before Phase 1 is called complete)

| # | Test | Evidence |
|---|---|---|
| A1 | `ruff check src tests` → clean | CI log |
| A2 | `pytest -q` → **green**; any skipped integration test prints a loud reason (never silent) | CI log |
| A3 | Migration drill: `alembic upgrade head` → `downgrade base` → `upgrade head` on scratch DB | test output |
| A4 | Queue: dependency gating, claim exclusivity (concurrent workers), retry→DLQ, cancel | `test_queue.py` |
| A5 | Recovery: kill worker mid-task → lease expiry → requeue → no duplicate side effects | `test_worker_lease_recovery.py` |
| A6 | Budget: exceeding cap pauses dispatch + writes audit event; paid providers default-disabled | `test_budget_enforcement.py` |
| A7 | Storage: upload/download/list/checksum match + interrupted-upload resume + quota stats | `test_storage_local.py` |
| A8 | **E2E hello-job:** create job via API → worker claims → artifact lands in storage with matching checksum → audit trail complete | `test_hello_e2e.py` |
| A9 | **Cloud VM drill (mini test A):** deploy via `deploy/bootstrap.sh` → create job → laptop off → completion + checksum verified → `/status` and `/queue` truthful throughout | manual + logs |
| A10 | **Backup/restore drill:** `deploy/backup.sh` → restore into clean DB → row counts match (foundation of acceptance F) | drill log |
| A11 | Secrets: `git log -p` contains no secret material; `.env` ignored; `.env.example` complete | CI grep step |

## 8. Owner actions required during Phase 1

| ID | Action | Needed for |
|---|---|---|
| B2 | Create a Google Cloud project → enable Drive API → OAuth client (Desktop/Web) → put client id/secret in VM secret manager; enable the Drive app in this session | live `gdrive.py` tests (A7/F) — until then the adapter tests run against local FS only |
| B3 | Sign up Oracle Always Free (or choose fallback host) → run `deploy/bootstrap.sh` | A9 |

Steps for B2 (official flow): console.cloud.google.com → new project → APIs & Services → enable "Google Drive API" → OAuth consent screen (External, add yourself as test user) → Credentials → Create OAuth client ID → redirect URI for the chosen rclone/Desktop flow → store values in the VM's secret manager (never chat, never git).

## 9. Security & cost implications

- No secrets committed (A11) · bearer auth + input validation on all endpoints · log redaction.
- Budget caps enforced in code; breach → pause + notify (test A6) — no silent charges.
- Google Drive limited to artifacts/backups; **never** DB/WAL/queue files.
- Phase 1 cost: **$0** (free VM + open-source deps). Zero paid API calls in this phase.
- Licence of every new dependency recorded in the PR description.

## 10. Known limitations (honest)

- Single-VM PostgreSQL = single point of failure until A10 passes repeatedly.
- Postgres broker is a deliberate Phase-1 simplicity choice; Redis/RabbitMQ promotion only on measured need (interface ready).
- No Discord, no video, no publishing yet — by design.
- Drive adapter live-tests blocked on owner action B2.
- **Migration-vs-greenfield gate OPEN:** the owner reports a source archive
  `universal-ai-creator-studio-v2.1.0.zip`; it is absent from this environment (searched
  2026-11-15). Per audit §2.1, the ZIP must be uploaded and inspected — producing the
  per-module MIGRATE/REFACTOR/REPLACE/DISCARD verdicts — **before** this plan is approved.
  If reusable modules exist, a migration task is inserted here and §2's file tree adjusts;
  if the archive confirms staleness, this plan proceeds as written on explicit owner
  confirmation.

## 11. Rollback plan

1. Phase 1 lands on the feature branch → PR → merge only when A1–A8 green.
2. Revert merge commit → repo returns to Phase-0 docs-only state.
3. DB: `alembic downgrade base` (no production data exists yet).
4. VM: `docker compose down` (volumes optional-keep).

## 12. Definition of done → report (spec §15)

When A1–A11 pass, Phase 1 produces: changed files · commands executed · test results · security & cost notes · known limitations · rollback instructions · next-phase proposal (**Phase 2 — Discord control plane**).

---

**STOP. Phase 0 complete. Awaiting explicit owner approval to begin Phase 1.**
