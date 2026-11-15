# Universal AI Media & Business OS

> **Current status: Phase 0 complete — audit & planning only. No production features are implemented yet.**
> Nothing in this repository should be considered working until its integration test says so.

## What this will be

A cloud-first, Discord-controlled AI content production platform: natural-language video
requests in → planned, researched, scripted, voiced, assembled, QC'd, human-approved videos
out — running on cloud workers so the owner's laptop can stay off. Plus the surrounding
business modules (channels, publishing, comments, CRM, revenue ledger, tool registry).

Full requirements: see the master specification
(`Universal_AI_Media_Business_OS_Arena_Master_Spec.md`, owner-provided).

## Honest feature labels

| Label | Meaning |
|---|---|
| IMPLEMENTED | code exists **and** a test proves it |
| PARTIAL | some code exists, incomplete |
| PLANNED | specified, no code yet |
| BLOCKED | cannot proceed (reason recorded) |

**Today: 1 IMPLEMENTED (the Phase 0 documents below), 0 PARTIAL, ~24 PLANNED, ~21 BLOCKED.**

## Phase 0 deliverables (IMPLEMENTED)

| Doc | Contents |
|---|---|
| [`docs/phase0/01-repository-audit.md`](docs/phase0/01-repository-audit.md) | Complete repository audit — including the critical finding that the claimed v2.1.0 source repo was not found |
| [`docs/phase0/02-architecture.md`](docs/phase0/02-architecture.md) | Target architecture diagram + component responsibilities + Phase 1 infra decisions |
| [`docs/phase0/03-gap-report.md`](docs/phase0/03-gap-report.md) | Feature-by-feature implementation gap report (all 21 production stages + cross-cutting) |
| [`docs/phase0/04-roadmap.md`](docs/phase0/04-roadmap.md) | Prioritized Phases 0–8 roadmap with dependency map and per-phase engineering standards |
| [`docs/phase0/05-hardware-free-cloud.md`](docs/phase0/05-hardware-free-cloud.md) | Hardware & free-cloud compatibility report (2026 free-tier evidence, cost model) |
| [`docs/phase0/06-phase1-plan.md`](docs/phase0/06-phase1-plan.md) | Exact file-level plan for Phase 1 (cloud foundation) — **awaiting owner approval** |

## Key facts

- **Repository audit (2026-11-15):** the workspace contained only this README. The claimed
  existing codebase (`universal-ai-creator-studio` / `faceless-factory` v2.1.0) does **not**
  exist on the owner's GitHub account or on disk → all "existing" features are BLOCKED until
  the source is provided; otherwise the build is greenfield guided by the spec.
- **Zero budget by default.** No paid API, hosting or subscription is activated without the
  owner's explicit approval and a hard spending cap.
- **Discord is the control plane** (Phase 2): slash commands, approval cards, alerts,
  emergency stop. The laptop is optional.
- **No fake claims:** no universal text-to-video/3D generation, no always-on free GPU, no
  guaranteed revenue figures, no fake engagement — ever.

## Development

```bash
python3 -m venv .venv && . .venv/bin/activate
pip install -e ".[dev]"     # available from Phase 1 onward
ruff check src tests        # from Phase 1
pytest -q                   # from Phase 1
```

Phases follow the same loop every time: explain → inspect → modify → test → fix →
document → show changed files → report done/remaining. Work happens on feature branches
via pull requests; never direct pushes to `main`.
