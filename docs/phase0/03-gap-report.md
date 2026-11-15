# Phase 0 — Detailed Implementation Gap Report

**Date:** 2026-11-15 · **Legend:** IMPLEMENTED · PARTIAL · PLANNED · BLOCKED(reason)
**Ground rule:** nothing is marked IMPLEMENTED unless code + a passing test exist.

---

## 0. Headline

| Metric | Value |
|---|---|
| Source files in repo | 1 (`README.md`, 13 bytes) |
| Python modules | 0 |
| Tests | 0 |
| Dependencies declared | 0 |
| Claimed v2.1.0 codebase located | **No** (BLOCKED — see audit §3) |
| Features IMPLEMENTED today | **1** — the Phase 0 document set itself |
| Features PLANNED | ~24 |
| Features BLOCKED | ~21 (mostly "prior code unavailable" + no-free-GPU/3D truth) |

---

## 1. Production engine (21 stages per mission §3)

| # | Stage | Status | Gap to close | Phase |
|---|---|---|---|---|
| 1 | Request intake | PLANNED | typed `VideoRequest` model, validation, persistence | 1–3 |
| 2 | Topic discovery | PLANNED | LLM adapter (free tier), niche/channel binding | 3 |
| 3 | Research & source collection | PLANNED | web/search adapter, citation records | 3 |
| 4 | Fact checking | PLANNED | claim extraction + source cross-check stage, evidence table | 3 |
| 5 | Script generation | PLANNED | prompt templates, language support, token budget | 3 |
| 6 | Story structure | PLANNED | hook/beats/pacing templates per format (Shorts, long, doc) | 3 |
| 7 | Visual style selection | PLANNED | 8 style profiles as data files + resolver | 3 |
| 8 | Character/environment bible | PLANNED | bible schema, consistency tokens reused across scenes | 3 |
| 9 | Storyboard generation | PLANNED | scene list → shot list JSON with camera/transition fields | 3 |
| 10 | Scene/shot planning | PLANNED | timing, aspect ratio, caption window planning | 3 |
| 11 | Asset discovery/generation | PARTIAL→PLANNED | adapters: stock APIs (Pexels/Pixabay), image gen, graphics renderer; licence ledger | 3 |
| 12 | Voice generation | PLANNED | TTS adapter (free-tier/local: e.g. Piper/edge-style TTS — evaluate licences) | 3 |
| 13 | SFX & music | PLANNED | free-licence audio library + attribution ledger | 3 |
| 14 | Animation/motion graphics | PLANNED | template-based motion (Remotion or FFmpeg filters) | 3, 8 |
| 15 | Scene rendering | PLANNED | **cloud worker** with resource declarations | 3 |
| 16 | Video editing/assembly | PLANNED | FFmpeg pipeline worker (install + smoke test first) | 3 |
| 17 | Subtitles/captions | PLANNED | ASR or script-derived timing, SRT burn-in, sync QC | 3 |
| 18 | Thumbnail | PLANNED | template compositor | 3 |
| 19 | Technical QC | PLANNED | ffprobe stream checks, duration/aspect/loudness rules | 3 |
| 20 | Human approval | PLANNED | Discord approval card (Phase 2 gate in pipeline) | 2–3 |
| 21 | Publishing prep | PLANNED | metadata, licence check, draft package | 4 |
| — | Publishing | PLANNED | YouTube Data API adapter (official scopes), idempotency | 4 |
| — | Analytics & learning | PLANNED | import metrics → channel reports → topic feedback loop | 4 |

**Cross-cutting gaps (all stages):** typed input/output schemas, JSON-schema validation,
idempotency keys, checkpoint/resume, per-stage retry policy, audit events, budget metering.

## 2. Tool discovery & registry (mission §4)

| Requirement | Status | Gap |
|---|---|---|
| Registry data model (name, repo, category, licence, commercial use, hardware, API needs, free-tier limits, fallback, integration status) | PLANNED | schema + seed of ~10 verified entries in Phase 1; **no bulk downloads** |
| Tool adapter architecture (provider swap without pipeline rewrite) | PLANNED | adapter protocol + registry-driven routing |
| Discovery workflow (verify repo, licence, security scan, isolated test, owner approval) | PLANNED | documented checklist enforced by `/tools evaluate` |
| 100+ entries | PLANNED | accumulated over Phases 3–8, ~5–10 real integrations per phase |
| "Reverse engineering paid software" | **BLOCKED (policy)** | spec §10 forbids bypassing provider restrictions; replaced by: official free tiers, open-source alternatives, clean-room API interop |

## 3. Universal visual engine (mission §5)

| Requirement | Status | Gap |
|---|---|---|
| 8 visual styles as selectable profiles | PLANNED | style registry data + resolver |
| Multi-scene production, character/environment consistency | PLANNED | bible tokens + asset reuse graph |
| Camera instructions, transitions | PLANNED | storyboard schema fields |
| Fallback: stock → original graphics → motion templates | PLANNED | ordered fallback chain in asset stage |
| Universal text-to-video / 3D generation | **BLOCKED** | no verified free commercial-grade provider; will not be claimed |
| Seedance-class model parity claims | **BLOCKED (truthfulness)** | we integrate real providers when available; no competitive claims without tests |

## 4. Automation & agent architecture (mission §6)

| Requirement | Status | Gap |
|---|---|---|
| Task dependencies, worker leases, retry/backoff, persistent history | PLANNED | Phase 1 core (queue + registry + Postgres) |
| Agent role registry with typed I/O, budgets, deny-lists | PLANNED | agent manifest schema; **no fake role-only agents** |
| Provider routing + budget enforcement + approval gates | PLANNED | Phase 1 (budget/gates), Phase 3 (providers) |
| Job recovery, logging, monitoring | PLANNED | Phase 1 (event log + health + recovery tests) |
| Hermes / Discord integration interfaces | PLANNED | Phase 2 (Discord); Hermes = interface stub only until real |

## 5. Cloud-first compute (mission §7)

| Requirement | Status | Gap |
|---|---|---|
| Lightweight control plane on always-on cloud | PLANNED | Phase 1 deploy target (see cloud report) |
| Independent workers, capability detection, resource-aware scheduling | PLANNED | Phase 1 worker registry; scheduler scoring in Phase 1 tail |
| Temporary worker support | PLANNED | ephemeral worker registration + lease expiry |
| Cloud storage integration | PLANNED | neutral adapter; **provider must be confirmed by owner** |
| DB backups + recovery after worker failure | PLANNED | pg_dump cron + restore drill (acceptance test C) |
| Honest free-compute posture | ✅ in docs | free tiers documented as limited/ephemeral |

## 6. Revenue streams (mission §8)

All five streams: **PLANNED** as separate business modules (Phase 7) — channel analytics,
distribution, client services/CRM, digital products, affiliates. Revenue ledger must record
actuals only; no forecasts (spec §8).

## 7. Zero-budget policy (mission §9)

| Requirement | Status | Gap |
|---|---|---|
| Free/open tools priority | ✅ policy written | enforce via tool registry default-allow list |
| No paid activation without approval | PLANNED | Phase 1 budget ledger + provider kill-switch |
| Cost tracking + hard limits | PLANNED | Phase 1 `costs` table + cap enforcement test |

## 8. Security & copyright (mission §10)

| Requirement | Status | Gap |
|---|---|---|
| No credential exposure | ✅ (nothing to leak) | `.env.example`, secret manager docs in Phase 1 |
| Asset source + licence recording | PLANNED | `asset_sources` table from Phase 1 |
| Incompatible licence rejection | PLANNED | licence allowlist check in asset stage |
| No copying creators' scripts/characters | ✅ policy | prompt + QC checks in Phase 3 |
| Approval before publishing | PLANNED | Phase 2 approval cards (hard gate) |
| Downloaded-asset validation | PLANNED | hash + malware-scan step in asset ingest |
| No fake engagement | ✅ policy | forbidden in code review checklist |

## 9. Discord control plane (spec §3) — all PLANNED (Phase 2)

39 slash commands specified; bot registration, allowlist, buttons/menus, approval cards,
progress messages, alerts, emergency pause. Gap: entire bot — plus the Phase 1 API must
expose the endpoints those commands call.

## 10. Acceptance tests (spec §14)

| Test | Status | Depends on |
|---|---|---|
| A Laptop independence | PLANNED | Phase 1–2 |
| B Cloud rendering + checksum + link | PLANNED | Phase 1 storage + Phase 3 render |
| C Recovery (no dup publish/lost task) | PLANNED | Phase 1 leases/retries |
| D Discord security | PLANNED | Phase 2 |
| E Comment management | PLANNED | Phase 5 |
| F Storage (upload/download/quota/backup) | PLANNED | Phase 1 (pending provider) |
| G Cost controls | PLANNED | Phase 1 budget ledger |
| H Quality (streams, captions, approval recorded) | PLANNED | Phase 3 QC stage |

## 11. Biggest risks (ranked)

1. **Missing source repository** — if the owner actually has v2.1.0 code somewhere, Phase 1
   estimates double/halve depending on reuse. Action: owner supplies it or confirms greenfield.
2. **Free-tier volatility** — 2026 saw Oracle halve ARM allowance, Render cut bandwidth,
   Fly.io end free tier. Mitigation: provider abstraction + documented fallbacks (report 05).
3. **5TB provider unknown** — storage adapter blocked on this answer.
4. **No free production-grade video model** — visual engine must lean on stock/graphics/
   motion templates; must never over-claim.
5. **Single-VM Postgres** — acceptable at this scale; backups mandatory from day one.
