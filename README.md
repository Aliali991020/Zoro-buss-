# Universal AI Media & Business OS

> **Current status: Phase 0 complete — audit & planning only. No production features are implemented yet.**
> Nothing in this repository is considered working until an integration test proves it.

## What this will be

A cloud-first, Discord-controlled AI content production platform: natural-language video
requests in → planned, researched, fact-checked, scripted, story boarded, voiced, assembled,
QC'd, human-approved, published and analyzed — running on persistent cloud workers so the
owner's laptop can stay off. Plus the surrounding business modules: multi-channel operations,
comment management, CRM, five revenue streams and a 100+ verified tool registry.

Authoritative requirements: the owner-provided
`Universal_AI_Media_Business_OS_Arena_Master_Spec.md`.

## Honest feature labels

| Label | Meaning |
|---|---|
| IMPLEMENTED | code exists **and** a test proves it |
| PARTIAL | some code exists, incomplete |
| PLANNED | specified, no code yet |
| BLOCKED | cannot proceed (reason recorded) |

**Today: 1 IMPLEMENTED (the Phase 0 documents), 0 PARTIAL, ~22 PLANNED, ~24 BLOCKED.**

## Phase 0 deliverables (the only IMPLEMENTED features)

| File | Contents |
|---|---|
| [`docs/00-REPOSITORY-AUDIT.md`](docs/00-REPOSITORY-AUDIT.md) | GitHub account/repo verification, master-spec inspection, truthful feature classification, verified-vs-unknown, blockers |
| [`docs/01-SYSTEM-ARCHITECTURE.md`](docs/01-SYSTEM-ARCHITECTURE.md) | Mermaid architecture diagram, Discord control plane, cloud coordinator/queue/workers, Google Drive 5TB storage design, 24-step pipeline, DB & queue design, recovery/backup, security |
| [`docs/02-IMPLEMENTATION-GAP-REPORT.md`](docs/02-IMPLEMENTATION-GAP-REPORT.md) | Feature-by-feature status, verified tool/provider comparison, security & copyright risk register |
| [`docs/03-DEVELOPMENT-ROADMAP.md`](docs/03-DEVELOPMENT-ROADMAP.md) | Phases 0–8 with complexity estimates, dependencies and dependency map |
| [`docs/04-CLOUD-HARDWARE-COMPATIBILITY.md`](docs/04-CLOUD-HARDWARE-COMPATIBILITY.md) | Realistic 2026 free-tier limits, hardware constraints, $0 cost model, evidence sources |
| [`docs/05-PHASE-1-IMPLEMENTATION-PLAN.md`](docs/05-PHASE-1-IMPLEMENTATION-PLAN.md) | Exact Phase 1 file changes, schema, semantics, acceptance tests A1–A11 — **awaiting owner approval** |

## Key facts & hard rules

- **Repository audit (2026-11-15):** no `universal-ai-creator-studio` / `faceless-factory`
  repository exists on the owner's GitHub account. The accessible repository contained only
  this README. **The owner reports a source archive `universal-ai-creator-studio-v2.1.0.zip`
  exists — it is not present in this environment (exhaustive search). Upload pending; the
  migration-vs-greenfield decision stays open until it is inspected (audit §2.1).** Nothing
  is attributed to that unavailable codebase.
- **Google Drive = storage/archives only** — never compute or rendering.
- **GitHub Actions = temporary batch/CI only** — never a permanent server or GPU worker.
- **No unlimited free GPU is promised** — no free 24/7 GPU exists in 2026.
- **No bypass of paid licensing, DRM, authentication or API restrictions** — legitimate APIs
  and open-source alternatives only.
- **No capability claims without a tested integration** — Seedance or any advanced video model
  is not "integrated" unless a working test says so.
- **Zero budget by default** — paid services stay disabled until explicit owner approval with
  a hard spending cap.
- **No fake agents, placeholder skills or bulk empty files.** No credentials in git, logs or
  Discord. No fake engagement — ever.

## Development

```bash
python3 -m venv .venv && . .venv/bin/activate
pip install -e ".[dev]"     # available from Phase 1 onward
ruff check src tests        # from Phase 1
pytest -q                   # from Phase 1
```

Every phase follows: explain → inspect → modify → test → fix → document → show changed
files → report done/remaining. Work happens on feature branches via pull requests;
never direct pushes to `main`.
