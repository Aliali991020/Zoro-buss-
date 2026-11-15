# Phase 0 — Prioritized Development Roadmap

**Date:** 2026-11-15 · **Status:** PLANNED — owner approval required before Phase 1
**Working method:** small verifiable phases → explain → inspect → modify → test → fix →
document → show changed files → report done/remaining (mission §11).

---

## Priority model

- **P0** — unblocks everything else (do first)
- **P1** — required for first real video
- **P2** — required for operating the business
- **P3** — scale & ecosystem

---

## Phase 0 — Audit & safety ✅ DONE (this deliverable)

Deliverables: repo audit · architecture diagram · gap report · roadmap · hardware/free-cloud
report · exact Phase 1 plan. **No production deployment.**

Exit criteria: owner reviews and approves Phase 1 (mission §12).

---

## Phase 1 — Cloud foundation (P0) ← AWAITING APPROVAL

Scope (details in `06-phase-1-plan.md`):

1. Repo scaffolding: `pyproject.toml`, package layout, lint/test tooling.
2. PostgreSQL schema + Alembic migrations (projects, channels, jobs, tasks, dependencies,
   worker leases, approvals, audit events, cost ledger, asset sources, tool registry).
3. Durable queue with dependencies, retries/backoff, dead-letter, cancellation
   (broker interface; Postgres SKIP LOCKED first, Redis/RabbitMQ only if measured need).
4. Coordinator API (FastAPI): job CRUD, status, event log, health, emergency pause flag,
   budget caps.
5. Worker runtime: registration, heartbeat, lease acquisition/extension, idempotent claim,
   typed task adapters, graceful shutdown, crash recovery.
6. Object storage abstraction: interface + local-FS implementation + checksum/quota tests
   (Drive/S3 adapters wait on provider confirmation).
7. Tool registry schema + first ~10 verified entries (licence-checked).
8. Tests: unit + Postgres integration (embedded PG fallback) + CI workflow.
9. Deploy dev environment on the chosen free cloud VM (owner account), with backups.

**Definition of done:** tests green; a scripted end-to-end "hello job" runs
coordinator↔worker↔storage on the cloud VM with the laptop off; recovery test C passes in
miniature; rollback = revert merge + `alembic downgrade`.

**Estimated size:** ~15–25 files, no huge dependencies.

## Phase 2 — Discord control plane (P0/P1)

Bot registration instructions → allowlist → core commands
(`/status /queue /projects /video create|status|approve|reject /worker status /budget
/costs /system pause|resume /help`) → approval cards → notifications → emergency stop.
Maps to acceptance test D.

## Phase 3 — One end-to-end video (P1)

First vertical slice using free providers only: intake → research → script → storyboard →
assets (stock/graphics fallback) → TTS → assembly (FFmpeg worker) → captions → thumbnail →
QC → (approval gate) → artifact to storage → Discord link. **Render on cloud worker, not the
laptop.** Maps to acceptance tests A, B, H. 30–60 s original video.

## Phase 4 — Publishing & analytics (P1)

YouTube adapter (official API, OAuth scopes, draft→preview→license check→owner approval→
publish→verify), analytics import, Discord reports. Test E partially, spec §7 capability
matrix per platform.

## Phase 5 — Community management (P2)

Comment inbox (official API only), classification, draft replies, approval flow, restricted
auto-reply after validation. Never: fake engagement, scraping, auto-replies on sensitive
categories.

## Phase 6 — Multi-channel operations (P2)

Multiple channel configs/brands, separate schedules & analytics, shared templates.

## Phase 7 — CRM & monetization (P2)

Client pipeline, proposals, delivery, invoices, product catalog, affiliate tracking,
revenue/expense ledger with actuals-only reporting (5 separate business modules).

## Phase 8 — Advanced production & tool ecosystem (P3)

Remaining visual styles, more adapters, tool registry toward 100+ **verified** entries,
optional GPU providers (only real ones), distributed worker scaling, semantic visual
continuity QA.

---

## Cross-phase engineering standards (every phase)

1. Explain → inspect → modify → test → fix → docs → changed-files list → done/remaining.
2. Feature labels truthful: IMPLEMENTED only with a passing test.
3. No placeholder buttons, fake agents, or doc-only features.
4. No paid API/hosting without explicit owner approval + cap.
5. Work on branch → PR → review (never direct push to `main`).
6. Each phase ends with: changed files, commands run, test results, security/cost notes,
   known limitations, rollback steps, next-phase proposal (spec §15).

## Dependency map (simplified)

```
Phase0 ─▶ Phase1 ─┬─▶ Phase2 ─▶ Phase3 ─▶ Phase4 ─▶ Phase5
                  │              │
                  │              └─▶ Phase6 ─▶ Phase7
                  └─▶ storage provider answer ─┘   │
                                                   └─▶ Phase8 (continuous)
```

## What could change the plan

- **Owner supplies original v2.1.0 repo** → Phase 1 grows a migration step; some adapters
  ported instead of rebuilt.
- **Owner confirms a paid budget** → optional managed Postgres/queue; still capped.
- **Free tiers shrink further** → VM provider swap via deployment abstraction; design
  already isolates it.
