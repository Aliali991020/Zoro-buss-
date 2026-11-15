# 02 — Implementation Gap Report

**Date:** 2026-11-15 · **Labels:** IMPLEMENTED (code+passing test) · PARTIAL · PLANNED · BLOCKED(reason)
**Ground truth:** the workspace contains only `README.md`; the claimed v2.1.0 source is unavailable (see `00-REPOSITORY-AUDIT.md`). Nothing may be labeled working on the strength of documentation alone.

---

## 0. Headline numbers

| Metric | Value |
|---|---|
| Source files in repo | 1 (`README.md`) — plus this Phase 0 document set |
| Python modules / tests / dependencies | 0 / 0 / 0 |
| Claimed v2.1.0 codebase located | **No — BLOCKED (source unavailable)** |
| IMPLEMENTED | **1** — Phase 0 documents |
| PARTIAL | 0 |
| PLANNED | ~22 |
| BLOCKED | ~24 |

---

## 1. Production engine — feature-by-feature (24 steps, spec §5)

| # | Feature | Status | Gap to close | Target phase |
|---|---|---|---|---|
| 1 | Request intake (typed, validated, persisted) | PLANNED | `VideoRequest` schema + endpoint | 1–3 |
| 2 | Niche/channel selection | PLANNED | channel model + binding UI/commands | 3 (schema in 1) |
| 3 | Topic research | PLANNED | free-LLM adapter + research task type | 3 |
| 4 | Source collection & citations | PLANNED | citation records, source adapter | 3 |
| 5 | Fact verification | PLANNED | claim extraction + cross-check stage, evidence table | 3 |
| 6 | Script generation | PLANNED | prompt templates, language support, token budgets | 3 |
| 7 | Story structure & hook | PLANNED | per-format beat templates (Shorts/long/doc) | 3 |
| 8 | Character & visual bible | PLANNED | bible schema + consistency tokens across scenes | 3 |
| 9 | Storyboard generation | PLANNED | scenes→shots JSON (camera, transitions) | 3 |
| 10 | Scene/shot planning | PLANNED | timing, aspect, caption windows | 3 |
| 11 | Asset discovery/generation | PLANNED | stock APIs (Pexels/Pixabay-class), image gen if free, graphics renderer; licence ledger | 3 |
| 12 | Voiceover (TTS) | PLANNED | TTS adapter — local neural TTS or free-tier cloud; licence audit first | 3 |
| 13 | Music & SFX | PLANNED | free-licence libraries + attribution ledger | 3 |
| 14 | Animation / motion graphics | PLANNED | template-based motion (Remotion or FFmpeg filters) | 3, 8 |
| 15 | Scene rendering | PLANNED | cloud worker with declared resources | 3 |
| 16 | FFmpeg assembly | PLANNED | FFmpeg worker (binary absent today — install + smoke test first) | 3 |
| 17 | Subtitles & captions | PLANNED | script-timed + ASR verification (Groq Whisper free tier) | 3 |
| 18 | Thumbnail generation | PLANNED | template compositor | 3 |
| 19 | Technical QC | PLANNED | ffprobe stream checks, duration/aspect/loudness rules | 3 |
| 20 | Human review | PLANNED | Discord approval card (bot = Phase 2; gate API = Phase 1) | 1–3 |
| 21–22 | Publish prep + owner approval | PLANNED | metadata, licence check, draft package, confirm flow | 4 |
| 23 | Publishing (official APIs) | PLANNED | YouTube Data API adapter, OAuth scopes, idempotency, verify | 4 |
| 24 | Analytics & learning | PLANNED | metrics import → reports → topic feedback loop | 4 |

**Cross-cutting gaps for every stage:** typed input/output JSON schemas · validation · idempotency keys · checkpoint/resume · per-stage retry policy · audit events · budget metering · resource declarations (CPU/RAM/GPU/duration/storage).

## 2. Control plane & infrastructure

| Feature | Status | Gap | Phase |
|---|---|---|---|
| PostgreSQL schema + Alembic migrations | PLANNED | 15 tables, indexes, downgrade drill | 1 |
| Durable queue (deps, retry, DLQ, cancel) | PLANNED | broker interface + PG `SKIP LOCKED` impl | 1 |
| Coordinator API (jobs/tasks/events/health/pause) | PLANNED | FastAPI app + auth + validation | 1 |
| Worker registry: register/heartbeat/lease/recovery | PLANNED | runtime + recovery tests | 1 |
| Object storage abstraction | PLANNED | interface + local FS impl; **Google Drive adapter** pending OAuth (owner) | 1 |
| Budget ledger + hard caps + paid-provider gate | PLANNED | cost tables + enforcement test | 1 |
| Backups + tested restore | PLANNED | `pg_dump` cron → Drive + restore drill | 1 |
| Capability-aware scheduling (CPU/GPU/RAM) | PLANNED | scoring in registry | 1 tail |
| Temporary/ephemeral workers | PLANNED | lease expiry handles | 1 |
| Discord bot: 39 slash commands, allowlist, buttons, approval cards, alerts, emergency stop | PLANNED | entire bot (spec §3) | 2 |
| Hermes/conversational interface | PLANNED | interface stub only; no fake agents | 2+ |

## 3. Visual engine & AI models

| Feature | Status | Gap |
|---|---|---|
| 8 visual styles (3D cartoon, 3D game-style, cinematic, historical doc, illustrated, motion graphics, maps/data-viz, educational) | PLANNED | style registry data + resolver |
| Multi-scene consistency (characters, environments, camera, transitions) | PLANNED | bible tokens + asset reuse graph |
| Fallback chain: AI video → stock → original graphics → motion templates | PLANNED | ordered fallback in asset stage |
| Free AI video generation integration | **BLOCKED** | no verified, free, commercially-usable provider; will integrate only when one passes tests |
| Seedance / any advanced video model | **BLOCKED** | no working tested integration — claiming it is forbidden |
| Universal text-to-video / 3D scene generation | **BLOCKED** | same; prototype-era claims cannot be verified (source unavailable) |
| Semantic visual continuity QA | PLANNED | Phase 8 |

## 4. Community, publishing, business (spec §6–§8)

| Feature | Status | Gap | Phase |
|---|---|---|---|
| Comment inbox (official APIs only) | PLANNED | platform adapters + normalization | 5 |
| Classification (praise/question/complaint/spam/misinfo/business) | PLANNED | LLM classifier + rules | 5 |
| Reply drafting + owner approval + restricted auto-reply | PLANNED | approval flow; never for legal/medical/financial/complaint/misinfo/safety/sponsorship | 5 |
| FAQ detection from repeated questions | PLANNED | 5 |
| YouTube publishing adapter (capability matrix, scopes, limits) | PLANNED | 4 |
| TikTok/Instagram/Facebook adapters | PLANNED | documented capability matrices; unsupported ops labeled, no bypass | 4–6 |
| Analytics import + Discord reports | PLANNED | 4 |
| Multi-channel/brand ops | PLANNED | 6 |
| Client CRM / proposals / delivery / invoices | PLANNED | 7 |
| Digital product catalog + orders | PLANNED | 7 |
| Affiliate links + commission reporting | PLANNED | 7 |
| Sponsorship CRM | PLANNED | 7 |
| Revenue/expense ledger, per-channel P&L, cashflow (actuals only) | PLANNED | 7 (tables in 1) |

## 5. Tool registry (spec §9)

| Requirement | Status | Gap |
|---|---|---|
| Registry schema with full metadata (licence, commercial use, hardware, API, free-tier limits, fallback, status) | PLANNED | Phase 1: table + ~10 verified entries |
| Discovery/evaluation workflow (verify, licence read, security scan, isolated test, owner approval, version+checksum, uninstall path) | PLANNED | documented checklist + `/tools evaluate` (Phase 2 surface) |
| 100+ entries | PLANNED | accumulated over Phases 3–8 — **no bulk downloads, no empty placeholder entries** |
| Provider adapter architecture (swap providers without pipeline changes) | PLANNED | adapter protocols per category |

## 6. Agents (spec §10)

| Requirement | Status | Gap |
|---|---|---|
| Typed agent registry (narrow tasks, typed I/O, allowed/denied tools, budget/time limits, retry, audit, escalation) | PLANNED | manifest schema in Phase 1; wiring in Phase 3 |
| 21 named roles (researcher…tool evaluator) | PLANNED | roles become real only when their tasks/inputs exist — **no role-description-only fake agents** |
| Unrestricted shell/payment/publish/delete agents | **BLOCKED (by design)** | denied by policy |

## 7. Security & copyright (spec §10–§11)

| Requirement | Status | Notes |
|---|---|---|
| No credentials in git/logs/Discord | ✅ today (nothing to leak) | enforced via `.env.example`, secret manager, log redaction (Phase 1) |
| OAuth tokens encrypted at rest | PLANNED | Google Drive adapter phase |
| Asset source + licence recording; incompatible licences rejected | PLANNED | `asset_sources` table + QC check |
| Downloaded-asset validation (hash/malware) | PLANNED | ingest step |
| No copying other creators' scripts/characters | ✅ policy | prompt + QC checks Phase 3 |
| Human approval before public publishing | PLANNED | gates API Phase 1, cards Phase 2, enforced Phase 4 |
| No fake engagement / scraping / DRM bypass / license circumvention | ✅ policy | forbidden by owner rules + spec; code review checklist |
| Budget caps, emergency stop, rate limits, audit trail | PLANNED | Phase 1 (caps/pause/audit), Phase 2 (Discord surface) |
| Backup + tested restore; data export/deletion | PLANNED | Phase 1 restore drill; export Phase 6+ |

## 8. Acceptance tests (spec §14) — current status

| Test | Status | Depends on |
|---|---|---|
| A Laptop independence | PLANNED | Phases 1–2 |
| B Cloud rendering + checksum + Discord link | PLANNED | Phase 1 storage + Phase 3 render |
| C Recovery (no duplicate publish / lost task) | PLANNED | Phase 1 leases/retries |
| D Discord security | PLANNED | Phase 2 |
| E Comment management | PLANNED | Phase 5 |
| F Storage upload/download/quota/backup | PLANNED | Phase 1 (Drive adapter after OAuth) |
| G Cost controls | PLANNED | Phase 1 budget ledger |
| H Quality (streams, captions, approval recorded) | PLANNED | Phase 3 QC |

**0 of 8 passing today.**

## 9. Verified tool & provider comparison (research, 2026-11 — re-verify per phase)

### 9.1 Always-on compute

| Provider | Free allowance (as published) | Always-on? | Card? | Catch | Verdict |
|---|---|---|---|---|---|
| Oracle Always Free | 2 ARM OCPU/12 GB (halved from 4/24 in Jun 2026) + 2 AMD micro VMs + 200 GB [R1] | Yes, but idle reclamation + ARM capacity shortages [R1][R2] | Yes | region fixed at signup | **Primary target** |
| Render | 750 h/mo; 512 MB/0.1 CPU; free PG expires [R1][R3] | No — 15-min sleep, ~1 min wake [R3] | No | bandwidth cut to 5 GB/mo (Apr 2026) [R3] | fallback for web only |
| Fly.io | none — free tier ended Oct 2024 [R1][R3] | trial only | Yes | ~$1.94/mo minimum | not $0 |
| GitHub Actions | free minutes on public repos | ephemeral | No | **forbidden as permanent server/GPU** (owner rule) | CI + temporary batch only |
| Kaggle | ~30 GPU-h/week, 12 h sessions [R4][R5] | No | No | no SSH/root; quota weekly | experiments only |
| Colab free | T4, opaque quota, 12 h / 90-min idle [R4][R5] | No | No | no scheduling since Mar 2026 [R5] | experiments only |

### 9.2 Free LLM APIs (script/research/classification)

| Provider | Free allowance (Nov 2026) | Card? | Role |
|---|---|---|---|
| Google Gemini | Flash/Flash-Lite/2.5 Pro within per-model limits [R6][R7][R8] | No | primary script/research |
| Groq | ~30 rpm, 1,000 req/day, 200K tok/day per model + Whisper [R6][R7][R8] | No | fast fallback + STT |
| OpenRouter `:free` | 20/min, 50/day (1,000/day after $10 credits) [R6][R7][R8] | No | model escape hatch |
| Cloudflare Workers AI | 10K Neurons/day [R7][R8] | No | optional |
| LLM7.io | up to 1M tokens/day [R8] | No | optional bulk |
| Pollinations | plain-HTTP text/image/audio/video; 1 req/15 s anon [R8] | No | experimental only |

### 9.3 Production tooling (licence to verify per tool at integration)

FFmpeg (assembly) · Remotion or SVG/HTML render (motion) · local neural TTS (voice) · Pexels/Pixabay-class APIs (stock) · Groq Whisper (ASR verify) · PostgreSQL/Redis/RabbitMQ (data/queue) · FastAPI/SQLAlchemy/Alembic/pytest (framework) · discord.py or Interactions API (Phase 2) · rclone/Google Drive API (storage) — all open-source or free-tier; **none installed yet**.

## 10. Security & copyright risk register

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Free-tier quota exhaustion mid-job | High | job stalls | cooldown + truthful queued state + owner alert; multi-provider fallback |
| Oracle reclaims idle VM / capacity vanishes | Medium | control plane down | deployment abstraction + backups; wake/replace playbook |
| Licence-incompatible asset shipped | Medium | copyright strike | `asset_sources` ledger + QC gate before publish |
| Secret leak (token in log/PR) | Medium | account compromise | secret manager, redaction, `.env` git-ignored, CI secret scan |
| Paid API accidentally activated | Low | charges | hard budget gate + default-disabled providers + owner approval |
| Duplicate publish after retry | Medium | channel policy risk | idempotency keys + publish verify step |
| Third-party tool malware/supply chain | Medium | compromise | isolated evaluation, no auto-exec, dependency scanning |
| Fake-engagement / scraping pressure | — | policy breach | structurally absent; prohibited in checklist |
| Prompt/PII in logs | Medium | privacy | log redaction; no customer PII in Phase 1–4 |

## 11. References (verified sources for §9)

- **[R1]** [Best Free VPS & Cloud Server Tiers in 2026](https://www.mrplanb.com/datacenter/free-vps-tiers)
- **[R2]** [Best Free Docker Hosting 2026](https://xcloud.host/best-free-docker-hosting/)
- **[R3]** [State of Free Hosting 2026](https://snapdeploy.dev/state-of-free-hosting)
- **[R4]** [Google Colab Alternatives: 8 GPU Clouds 2026](https://www.spheron.network/blog/google-colab-alternatives-8-gpu-clouds-compared-2026/)
- **[R5]** [Kaggle vs Colab 2026](https://fastero.com/blog/kaggle-vs-colab)
- **[R6]** [7 Free LLM APIs in 2026](https://blog.gramosoft.tech/free-llm-apis/)
- **[R7]** [Free AI & LLM API keys 2026](https://byoklist.com/guides/free-ai-api-keys)
- **[R8]** [9 free LLM APIs compared 2026](https://dev.to/aifree/9-free-llm-apis-compared-rate-limits-card-rules-and-the-catch-2026-3hhj)

## 12. Biggest risks (ranked)

1. **Missing v2.1.0 source** — if it surfaces, estimates change materially (owner action B1).
2. **Free-tier volatility** (2026 saw multiple free tiers shrink) — provider abstraction + fallbacks.
3. **No free production-grade video model** — visual engine leans on stock/graphics/motion; over-claiming forbidden.
4. **Google Drive OAuth not yet provisioned** (owner action B2) — storage live tests blocked until then.
5. **Single-VM PostgreSQL** — acceptable at start; backups mandatory from day one.
