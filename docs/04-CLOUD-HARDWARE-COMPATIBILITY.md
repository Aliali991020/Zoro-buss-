# 04 — Cloud & Hardware Compatibility Report

**Date:** 2026-11-15 · **Status:** Research snapshot (free tiers change without notice — re-verify before each phase that depends on a quota)
**Rules applied:** Google Drive provides **storage only, never compute/render** · GitHub Actions is **temporary batch compute only, never a permanent server or GPU worker** · **no unlimited free cloud GPU is promised** · paid services require explicit owner approval + hard cap.

---

## 1. Given hardware constraints

| Machine | Specs (measured) | Role in the design |
|---|---|---|
| Owner's Windows 10 laptop | low-end (given) | **Optional client only** — Discord app + occasional admin. Must be able to stay 100 % off while jobs run (acceptance test A). Never a production worker. |
| Dev/CI sandbox (this session) | 2 cores / 3.8 GB RAM / 21 GB disk · Python 3.11 · Node 22 · **no Docker, no ffmpeg** | Tests + docs only; mirrors the low-end constraint, so the stack must be testable without containers (embedded-Postgres fallback) |

**Design consequence:** control plane + at least one CPU worker must live on an always-on cloud host; everything else (batch renders, GPU experiments) must tolerate being ephemeral.

## 2. Free compute providers — verified comparison (Nov 2026)

| Provider | Free allowance (published) | Always-on? | Card? | Real catches | Verdict for this system |
|---|---|---|---|---|---|
| **Oracle Cloud Always Free** | 2 ARM OCPU + 12 GB RAM (allowance **halved from 4 OCPU/24 GB on 15 Jun 2026**), 2 × AMD micro VMs, 200 GB block storage [1][2] | Yes — but **idle instances can be reclaimed**; ARM capacity in popular regions frequently out-of-stock; home region fixed at signup [1] | Yes | provisioning may need days of retries | **Primary target** for bot + coordinator + Postgres + queue + CPU worker |
| **Render** | 750 instance-hours/workspace/month; free Postgres expires in ~30–90 days; 512 MB / 0.1 CPU [1][3] | **No** — spins down after 15 min idle, ~1 min cold start; free bandwidth cut to 5 GB/mo (Apr 2026) [3] | No (web tier) | self-labeled "not for production" | fallback for a sleeping web/API endpoint only — **not** the always-on coordinator |
| **Fly.io** | **No free tier** (ended 7 Oct 2024); trial = 2 machine-hours or 7 days [1][3] | trial only | Yes | cheapest machine ≈ $1.94/mo [1] | unusable at $0 |
| **GitHub Actions** | free minutes on public repos | ephemeral (job-scoped) | No | **explicitly forbidden as a permanent server/GPU worker** (owner rule + spec §1) | CI tests, scheduled `pg_dump`, optional one-off batch jobs |
| **Kaggle Notebooks** | ≈ 30 GPU-hours/week (1×P100 or 2×T4), 12 h/session, 20 GB disk, no card [4][5] | **No** — session-bound, weekly quota | No | no SSH/root; not a hosting platform | manual GPU experiments only — **never a production dependency** |
| **Google Colab (free)** | T4, **undisclosed** dynamic quota, ≤12 h session, 90-min idle timeout; scheduling removed Mar 2026 [4][5] | No | No | quota opaque; cut-offs last days | manual experiments only |
| **Lightning AI** | monthly free GPU credits; free studios restart every 4 h (storage persists) [5] | semi | No | restart cadence | optional experiments |
| **AWS/Azure/GCP trials** | 6–12-month trials / $300 credits [4] | trial-limited | Yes | auto-charge after trial unless cancelled | excluded from $0 plan |

**Truthful statement (repeated in every doc):** *there is no free 24/7 GPU server in 2026.* GPU access is quota-limited and session-bound. The production path is therefore CPU-based assembly (FFmpeg on the VM) with stock/graphics/motion fallbacks; optional GPU experiments never sit in the critical path.

## 3. Free LLM APIs — verified comparison (Nov 2026)

| Provider | Free allowance | Card? | Planned role |
|---|---|---|---|
| **Google Gemini API** | Flash / Flash-Lite / 2.5 Pro free within per-model RPM+RPD limits shown in AI Studio; free-tier data may improve Google products [6][7][8] | No | **primary** script/research/classification |
| **Groq** | ≈30 req/min, 1,000 req/day, 200K tokens/day per model; Whisper free (≈2K req/day) [6][7][8] | No | fast fallback + ASR for caption verification |
| **OpenRouter `:free`** | 20/min; 50/day (1,000/day after a one-time $10 credit purchase) [6][7][8] | No | multi-model escape hatch (rotating lineup) |
| **Cloudflare Workers AI** | 10,000 Neurons/day [7][8] | No | optional |
| **Z.ai GLM Flash** | Flash models free [6][8] | No | optional |
| **LLM7.io** | up to 1M tokens/day (2 req/s) [8] | No | optional bulk summarization |
| **Pollinations** | plain-HTTP text/image/audio/video; 1 req/15 s anonymous (watermarks possible) [8] | No | experimental only |
| **Local Ollama** | unlimited on owner hardware | — | dev-only; low-end laptop is not a production path |

**Capacity view:** Gemini + Groq + OpenRouter together provide thousands of free requests/day — sufficient for script/research stages **if** we cache aggressively and meter every call against per-project and global caps (Phase 1 budget ledger). All keys live in the host secret manager; never in git, logs or Discord.

## 4. Storage: Google Drive 5TB (owner-specified)

| Topic | Decision |
|---|---|
| Role | **Artifacts, source assets, archives and backups only** — exactly per spec §4 |
| Forbidden uses | live database, PostgreSQL data dir, WAL, lock files, active queue state — **never** (spec §2.4) |
| Access method | official Google Drive API (preferred) or rclone OAuth; tokens in secret manager, encrypted at rest |
| Layout | `/Universal-AI-Business-OS/00-system-backups/ 01-projects/{id}/… 02-channel-brand-assets/ 03-client-deliverables/ 04-digital-products/ 05-exports/` |
| Metadata | in PostgreSQL, not just filenames: content hashes, licence/source records |
| Lifecycle | temp renders expire; originals never auto-deleted; destructive ops need confirmation |
| Quota alerts | 70 % / 85 % / 95 % → Discord notifications |
| Links | private by default; temporary least-privilege share links only |
| Tests (acceptance F) | upload · download · list · checksum verify · interrupted-transfer resume · quota warning · backup + **tested restore** |
| Current status | **BLOCKED on owner action B2** — Google OAuth client not yet created; the connected Google Drive app in this session is OFF (no Drive operations attempted in Phase 0) |

If Drive proves unsuitable for hot worker scratch, it remains archive/deliverable storage while a small optional S3-compatible bucket (only if ever approved/paid) takes hot artifacts — the adapter interface stays identical.

## 5. Zero-cost production tooling (licence check required per tool at integration)

| Need | Candidate | Licence posture |
|---|---|---|
| Video assembly | FFmpeg | LGPL/GPL — verify build flags; open source, free |
| Motion graphics | Remotion **or** SVG/HTML frame render | Remotion has a company license for larger orgs — evaluate at Phase 3; SVG/HTML path is unambiguously free |
| TTS | local neural TTS (Piper-class) first; free-tier cloud TTS as fallback | MIT/Apache-class expected — verify per voice model |
| Stock media | Pexels / Pixabay-class APIs | free licences with different attribution rules → recorded in `asset_sources` |
| ASR verification | Groq Whisper (free tier) | free tier, metered |
| Music/SFX | CC0/free-licence libraries | attribution ledger |
| DB/queue | PostgreSQL 16 (+ optional Redis/RabbitMQ later) | open source |
| Framework | FastAPI, SQLAlchemy, Alembic, pydantic, httpx, pytest, ruff | open source, small |
| Bot | discord.py / official Interactions API | open source (Phase 2) |
| Storage | Google Drive API / rclone | free API quota; OAuth |

**Nothing above is installed yet** — Phase 1 installs only the framework row (justified, small).

## 6. Cost model (truthful scenarios)

| Scenario | Monthly cost | What works | What breaks |
|---|---|---|---|
| **A. Zero-budget target (Phases 1–3)** | **$0** | Oracle Always Free VM runs bot+coordinator+Postgres+queue+CPU worker; free LLM tiers; GitHub Actions CI; Drive storage; stock/graphics visuals | Oracle reclaims idle VM or ARM capacity vanishes → need fallback B; free LLM caps throttle heavy days; **no GPU video-gen** |
| **B. Zero-budget fallback (no Oracle)** | **$0** | Render free sleeping API + GitHub Actions batch + Drive | wake latency may fail laptop-independence expectations; very small RAM |
| **C. Minimum viable always-on (only with explicit approval)** | ≈$4–7/mo | small VPS running the same compose stack; everything else still free-tier | **not activated** without owner approval + hard cap |
| **D. GPU / advanced video models** | not $0 today | — | stays disabled; no free always-on GPU exists; Seedance-class models are paid — activation needs approval |

**Enforced controls (Phase 1):** per-project + global caps · per-provider daily request budgets · pause-and-notify at thresholds · default-disabled paid providers · full `cost_ledger` (even $0 usage recorded as units) — never silent charges (acceptance test G).

## 7. Recommended (provisional) deployment

```
Owner phone ──Discord──▶ [Oracle Always Free VM]
                           ├── Discord bot (Phase 2)
                           ├── Coordinator API + Scheduler
                           ├── PostgreSQL + Queue (containerized, same VM)
                           ├── CPU worker #1 (same VM; more VMs later)
                           └── Storage adapter ──▶ Google Drive 5TB (artifacts/backups)
GitHub Actions ── CI, scheduled pg_dump, optional batch renders (ephemeral only)
Kaggle / Colab ── manual experiments only (never in production path)
```

All containerized behind a deployment abstraction → if Oracle changes terms again, the same stack moves to another Docker-capable host without code changes.

## 8. Owner actions this report depends on

1. **B2** — create Google OAuth client for Drive (steps in `05` §8) and enable the Drive app when ready.
2. **B3** — sign up for Oracle Always Free (or approve alternative in scenario C).
3. Re-verify quota snapshots before Phases 3 and 4 (free tiers shift often).

## 9. Evidence sources

1. [Best Free VPS & Cloud Server Tiers in 2026 — mrplanb.com](https://www.mrplanb.com/datacenter/free-vps-tiers)
2. [Best Free Docker Hosting in 2026 — xcloud.host](https://xcloud.host/best-free-docker-hosting/)
3. [State of Free Hosting 2026 — snapdeploy.dev](https://snapdeploy.dev/state-of-free-hosting)
4. [Google Colab Alternatives: 8 GPU Clouds Compared 2026 — spheron.network](https://www.spheron.network/blog/google-colab-alternatives-8-gpu-clouds-compared-2026/)
5. [Kaggle vs Colab in 2026 — fastero.com](https://fastero.com/blog/kaggle-vs-colab)
6. [7 Free LLM APIs in 2026 — gramosoft.tech](https://blog.gramosoft.tech/free-llm-apis/)
7. [Free AI and LLM API keys in 2026 — byoklist.com](https://byoklist.com/guides/free-ai-api-keys)
8. [9 free LLM APIs compared: rate limits, card rules and the catch — dev.to](https://dev.to/aifree/9-free-llm-apis-compared-rate-limits-card-rules-and-the-catch-2026-3hhj)
