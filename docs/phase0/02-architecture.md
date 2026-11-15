# Phase 0 — Target Architecture (Cloud-First, Discord-Controlled)

**Date:** 2026-11-15 · **Status:** PLANNED (design approved → Phase 1 implements it)

---

## 1. Design principles (from the master spec)

1. **The owner's laptop is optional.** Coordinator, workers, scheduling, rendering, storage
   sync and notifications run in the cloud. The laptop may be 100% powered off.
2. **Discord is the primary control plane** — phone-first slash commands, approval cards,
   alerts, emergency stop.
3. **Truthful states.** A job is never "rendering" unless a worker actually holds its lease.
4. **Zero budget by default.** Paid providers stay disabled until the owner approves a cap.
5. **Replaceable providers.** Every external service sits behind an adapter interface.

## 2. Logical architecture diagram

```
                        ┌──────────────────────────────┐
                        │  OWNER (any phone / PC)      │
                        │  Discord app — may be OFF    │
                        └──────────────┬───────────────┘
                                       │ slash commands / buttons
                                       ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  CLOUD CONTROL PLANE (always-on, $0 target: Oracle Always Free VM or equiv.)│
│                                                                              │
│  ┌─────────────────────┐   ┌──────────────────────────────────────────────┐  │
│  │ Discord Bot Gateway │──▶│  Command Router + Auth (allowlist, roles,    │  │
│  │ (interactions only) │   │  ephemeral replies, confirm-gated actions)   │  │
│  └─────────────────────┘   └───────────────┬──────────────────────────────┘  │
│                                            ▼                                 │
│  ┌──────────────────────────────────────────────────────────────────────┐    │
│  │ Coordinator API (FastAPI)                                            │    │
│  │  • create/inspect/cancel jobs  • approval gates  • budget enforcement│    │
│  │  • audit event log             • health endpoint • webhook receivers  │    │
│  └───────┬───────────────────────────────┬──────────────────────────────┘    │
│          │                               │                                   │
│          ▼                               ▼                                   │
│  ┌───────────────────┐         ┌──────────────────────┐                      │
│  │ PostgreSQL        │         │ Queue Broker         │  (selection:         │
│  │ projects, tasks,  │◀───────▶│ durable jobs, deps,  │   Phase 1 decision,  │
│  │ leases, approvals,│         │ retries, dead-letter │   see §5)            │
│  │ audit, ledger     │         └──────────┬───────────┘                      │
│  └───────────────────┘                    │ dispatch                         │
│          ▲                                ▼                                  │
│  ┌───────┴────────┐            ┌──────────────────────┐                      │
│  │ Scheduler      │            │ Worker Registry      │                      │
│  │ cron/interval  │            │ heartbeats, leases,  │                      │
│  │ analytics,     │            │ capability matching, │                      │
│  │ comment polls  │            │ job recovery         │                      │
│  └────────────────┘            └──────────┬───────────┘                      │
└───────────────────────────────────────────┼──────────────────────────────────┘
                                            │ lease + heartbeat
             ┌──────────────────────────────┼───────────────────────────────┐
             ▼                              ▼                               ▼
   ┌──────────────────┐          ┌──────────────────┐            ┌──────────────────┐
   │ CPU worker       │          │ Render worker    │            │ API workers      │
   │ research, script │          │ ffmpeg assembly, │            │ LLM (free tier), │
   │ facts, metadata  │          │ captions, thumbs │            │ TTS, image gen,  │
   │ (ephemeral OK)   │          │ (cloud VM / spot)│            │ stock media APIs │
   └────────┬─────────┘          └────────┬─────────┘            └────────┬─────────┘
            │                             │                               │
            └─────────────────────────────┼───────────────────────────────┘
                                          ▼
                     ┌────────────────────────────────────────┐
                     │ Object Storage Adapter (provider-neutral)│
                     │  • owner's 5TB cloud (provider TBC)      │
                     │  • optional S3-compatible bucket         │
                     │  • worker scratch (ephemeral)            │
                     │  checksums, licences, quota alerts       │
                     └───────────────────┬────────────────────┘
                                         ▼
                     ┌────────────────────────────────────────┐
                     │ Discord notifications: progress,        │
                     │ approval cards, artifact links, alerts  │
                     └────────────────────────────────────────┘

  NOT in the cloud path (by policy): owner's laptop, Google Drive-as-database,
  GitHub Actions as always-on server, any paid API without approved budget.
```

## 3. Production pipeline (typed, resumable stages)

Each stage is a pure-ish function `StageInput → StageOutput` registered as a task type with
schema validation, idempotency key, retry policy and checkpoint persistence. A crashed job
resumes at the last completed stage.

```
[1] Request intake ─▶ [2] Topic discovery ─▶ [3] Research/sources ─▶ [4] Fact check
        │                                                                │
        ▼                                                                ▼
[5] Script generation ─▶ [6] Story structure ─▶ [7] Visual style ─▶ [8] Character/env bible
        │                                                                │
        ▼                                                                ▼
[9] Storyboard ─▶ [10] Scene/shot plan ─▶ [11] Assets (gen|stock|graphics)
        │                                                            │
        ▼                                                            ▼
[12] Voice (TTS) ─▶ [13] Music/SFX ─▶ [14] Animation/motion ─▶ [15] Scene render
        │                                                            │
        ▼                                                            ▼
[16] FFmpeg assembly ─▶ [17] Subtitles ─▶ [18] Thumbnail ─▶ [19] Technical QC
        │                                                            │
        ▼                                                            ▼
[20] HUMAN APPROVAL (Discord card, hard gate) ─▶ [21] Publish prep ─▶ [22] Owner approve
                                                                                │
                                                                                ▼
                                                            [23] Publish (official API)
                                                                                │
                                                                                ▼
                                                            [24] Analytics + learning
```

**Visual style engine** (styles, not magic): 3D cartoon · 3D game-style · cinematic ·
historical documentary · illustrated · motion graphics · maps/data-viz · educational
explainer. Every style declares its asset sources; when free generative video is unavailable
the engine falls back to **stock footage → original graphics → motion templates** in that
order. Universal text-to-video / 3D generation is explicitly **not claimed**.

## 4. Component responsibilities

| Component | Owns | Does NOT own |
|---|---|---|
| Discord Bot Gateway | interaction parsing, allowlist, confirm flows | business logic, secrets display |
| Command Router / Auth | RBAC, ephemeral vs public replies, rate limits | task execution |
| Coordinator API | job/task lifecycle, approvals, budgets, audit | heavy compute |
| Queue Broker | durability, deps, retries, DLQ, cancellation | scheduling policy |
| Worker Registry | leases, heartbeats, capability match, recovery | task semantics |
| Scheduler | cron/interval triggers (production, analytics, polls) | ad-hoc commands |
| Workers | executing one task type with declared resources | cross-task state |
| Object Storage Adapter | upload/download/list/checksum/quota/licence metadata | live DB files |
| Agent Orchestrator | planning + routing of pipeline stages to workers | unrestricted shell |
| Tool Registry | catalogue, licence, integration status, fallbacks | auto-installing tools |

## 5. Phase 1 infrastructure decisions (to be validated by tests)

| Decision | Choice for Phase 1 | Why | Escape hatch |
|---|---|---|---|
| Control-plane host | **Oracle Cloud Always Free VM** (2 OCPU / 12 GB ARM as of mid-2026) — owner signs up; alternatives in `05-hardware-free-cloud.md` | only credible $0 always-on option left in 2026 | any Docker-capable host via `deploy/` abstraction |
| Database | **PostgreSQL 16** | spec-mandated for multi-host; SQLite only for unit tests | — |
| Queue | **Decision gate in Phase 1:** start with **PostgreSQL `FOR UPDATE SKIP LOCKED` + LISTEN/NOTIFY** in the same VM; promote to Redis (RQ/Arq) or RabbitMQ only when worker count/throughput demands it | zero extra services on a free VM; one durability domain; measured before adding moving parts | `broker/` interface allows Redis/RabbitMQ drop-in |
| Coordinator API | FastAPI + uvicorn | typed, OpenAPI, async, huge ecosystem | — |
| ORM/migrations | SQLAlchemy 2 + Alembic | standard, Postgres-first | — |
| Local integration tests | embedded PostgreSQL (`pgserver`) or skip-with-reason when unavailable | this sandbox has no Docker; queue semantics must still be tested against **real** Postgres | CI runs the same suite on Postgres service container |
| Object storage Phase 1 | local FS implementation of the neutral interface + checksum tests; Drive/S3 adapters in Phase 1 tail pending owner's provider answer | provider unknown (spec forbids assuming Drive) | rclone OAuth adapter planned |
| Secrets | environment via host secret manager (`.env.example` committed, real `.env` git-ignored) | spec §11 | — |
| Discord | **not in Phase 1** — Phase 2 | spec phase order | — |

## 6. Security architecture

- Default-deny Discord allowlist (user IDs + guild/channel IDs) enforced at the router.
- Confirmation required for: publish, spend, delete, settings changes, replies — surfaced as
  button-based approval cards with preview + source links.
- Audit event for every external action (who/what/when/result) in PostgreSQL.
- Secrets never in git, logs, or Discord messages; OAuth tokens encrypted at rest.
- Emergency `/system pause` halts dispatch globally (Phase 2) — implemented in Phase 1 as an
  API flag the bot will call.
- Budget ledger with hard caps; exceeding a cap pauses expensive tasks + notifies, never
  silently charges.

## 7. Deployment topology (zero-budget target)

```
[Owner phone: Discord] ──▶ [Cloud VM: bot + coordinator + Postgres + queue + 1 CPU worker]
                                   │
                                   ├─▶ [storage adapter] ─▶ owner's 5TB cloud / S3-compatible
                                   │
                                   ├─▶ [GitHub Actions] ── temporary batch compute only
                                   │                        (NOT always-on; spec §1)
                                   │
                                   └─▶ [Kaggle/Colab free GPU] ─ optional, session-bound,
                                        for experiments only — never a production dependency
```

The same compose file scales to multiple worker hosts later (broker + Postgres already
network-reachable). Every service is containerized so the hosting provider can be swapped
without touching application code.

## 8. What is deliberately out of scope for the diagram

- Paid GPU clouds, paid LLM APIs, paid schedulers → disabled by default.
- Browser automation / scraping to bypass platform rules → prohibited.
- Google Drive or synced folders as live database/queue → prohibited by spec.
- "Autonomous agents" with shell/publish/delete rights → agents get typed tools only.
