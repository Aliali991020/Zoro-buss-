# Phase 0 — Hardware & Free-Cloud Compatibility Report

**Date:** 2026-11-15 · **Status:** Research complete; selections provisional pending owner signup

---

## 1. Owner hardware constraint (given)

- Owner's machine: **low-end Windows 10 laptop**.
- Hard requirement: laptop may be **completely off** while jobs continue (spec §3.3,
  acceptance test A). Heavy rendering must never depend on it.
- Development sandbox (this repo's CI/agent environment) is also low-end: 2 cores / 3.8 GB
  RAM / 21 GB disk, **no Docker, no ffmpeg** — so the design must be testable on small
  machines and deployable to a cloud VM.

**Implication:** the control plane + at least one CPU worker must live on an always-on cloud
host. The laptop's only roles: Discord client, occasional admin/dev.

## 2. Free compute options — current evidence (Nov 2026)

| Provider | What's free (as published) | Always-on? | Card? | Key catch | Verdict for us |
|---|---|---|---|---|---|
| **Oracle Cloud Always Free** | 2 ARM OCPU + 12 GB RAM (allowance **halved from 4 OCPU/24 GB in June 2026**), plus 2 AMD micro VMs, 200 GB block storage [1][2] | Yes, but idle instances can be reclaimed; ARM capacity in popular regions often exhausted [1] | Yes | Home region fixed at signup; provisioning may need retries | **Primary target** for coordinator + Postgres + queue + CPU worker |
| **Render** | 750 instance-hours/workspace/month; free Postgres expires (~30–90 days) [1][3] | No — sleeps after 15 min idle, ~1 min cold start; free tier 512 MB / 0.1 CPU, bandwidth cut to 5 GB/mo Apr 2026 [3] | No (web services) | "Not for production" per its own terms; DB expiry | Fallback for web UI/API only, not for the always-on coordinator |
| **Fly.io** | **No free tier** — ended Oct 2024; trial = 2 machine-hours or 7 days [1][3] | Trial only | Yes | Cheapest machine ~$1.94/mo [1] | Not usable at $0 |
| **GitHub Actions** | Free minutes on public repos | Ephemeral (job-based) | No | Spec explicitly forbids treating it as an always-on server | Temporary batch jobs only (tests, backups, periodic renders) |
| **Kaggle Notebooks** | ~30 GPU-hours/week (P100 or 2×T4), 12 h sessions, 20 GB disk [4][5] | No — session-bound | No | No root/SSH; not for hosting; quota resets weekly | Optional experiments only, never a production dependency |
| **Google Colab (free)** | T4, dynamic undisclosed limits, ≤12 h sessions, 90 min idle timeout [4][5] | No | No | Quota opaque; no scheduling since Mar 2026 [5] | Experiments only |
| **Lightning AI** | Monthly free GPU credits; studios restart every 4 h on free plan [5] | Semi (storage persists) | No | Restart cadence | Optional experiments |
| **Google Cloud / AWS / Azure** | $300 trial credits / 6–12 month trials [4] | Trial-limited | Yes | Trial ends → charges | Not part of $0 plan |

**Honest statement:** there is **no** free 24/7 GPU server in 2026. GPU work is quota-limited
and session-bound. The architecture therefore treats GPU/video-gen as *optional*, with
CPU-based assembly (FFmpeg) + stock/graphics/motion fallbacks as the default production path.

## 3. Free LLM APIs (for script/research/agents) — current evidence

| Provider | Free allowance (Nov 2026) | Card? | Notes for our use |
|---|---|---|---|
| **Google Gemini API** | Flash/Flash-Lite/2.5 Pro free within per-model RPM/RPD limits (shown in AI Studio); free-tier data may be used to improve products [6][7][8] | No | **Primary** for script/research stages; per-project rate limits enforced by our budget layer |
| **Groq** | ~30 req/min, 1,000 req/day, 200K tokens/day per model; Whisper also free [6][7][8] | No | Fast fallback + STT for caption verification |
| **OpenRouter `:free` models** | 20/min; 50/day (1,000/day after $10 lifetime credits) [6][7][8] | No | Multi-model escape hatch; rotating lineup |
| **Cloudflare Workers AI** | 10,000 Neurons/day [7][8] | No | Optional |
| **Z.ai GLM Flash** | Flash models priced free [6][8] | No | Optional |
| **LLM7.io** | up to 1M tokens/day (2 req/s) [8] | No | Optional bulk summarization |
| **Pollinations** | text/image/audio/video over plain HTTP; anonymous 1 req/15 s [8] | No | Experimental only (watermarks possible) |
| **Local Ollama** | unlimited on our own hardware | — | Only for dev machines; laptop is low-end → not a production path |

**Combined realistic budget:** thousands of free requests/day across Gemini + Groq +
OpenRouter — enough for script/research/agent stages if we cache aggressively. All keys go
into the host secret manager; the cost ledger records usage against caps even at $0.

## 4. Free media & production tooling (no-cost stack)

| Need | Candidate (licence to verify per tool) | Status |
|---|---|---|
| Video assembly | **FFmpeg** (LGPL/GPL — build flags matter) | PLANNED — install on worker, smoke test Phase 1/3 |
| Scripted motion graphics | Remotion (company license for orgs >3 people — check) / HTML+CSS render / SVG animation | PLANNED — evaluate in Phase 3 |
| TTS | Local neural TTS (e.g. Piper-class, MIT-style — verify each voice's licence) / free-tier cloud TTS | PLANNED — Phase 3, licence audit first |
| Images/stock | Pexels/Pixabay/Unsplash-class APIs (free licences, attribution rules differ) | PLANNED — Phase 3, licence ledger required |
| Subtitles | script-timed generation + ASR verification (Groq Whisper free tier) | PLANNED — Phase 3 |
| Music/SFX | CC0/free-licence libraries with recorded attribution | PLANNED — Phase 3 |
| DB/queue | PostgreSQL, Redis (both open source) | Phase 1 |
| Orchestration | FastAPI, SQLAlchemy, Alembic, pytest | Phase 1 |
| Discord | discord.py / official Interactions API | Phase 2 |

## 5. Storage: the TB question (BLOCKED on owner answer)

- Spec: owner has **~5 TB** cloud storage; **provider must be confirmed** — do not assume
  Google Drive.
- Design: provider-neutral object-storage interface (upload/download/list/checksum/quota/
  temporary links/licence metadata) with the first concrete adapter chosen once confirmed.
- If Drive: official API or rclone OAuth, tokens in secret manager, **artifacts/backups only
  — never live DB/queue** (spec §4).
- Quota alerts at 70/85/95%.

## 6. Cost model (truthful)

| Scenario | Monthly cost | What you get / what breaks |
|---|---|---|
| **A. Zero-budget prototype** (Phase 1–3 target) | **$0** | Oracle Always Free VM (if signup succeeds) runs coordinator+Postgres+queue+CPU worker; free LLM tiers for text; GitHub Actions for CI; stock/graphics for visuals. *Breaks if:* Oracle reclaims idle VM or ARM capacity vanishes; free LLM caps throttle heavy days; no GPU video-gen. |
| **B. Zero-budget, no Oracle** (fallback) | **$0** | Render free web service sleeps (bot/commands wake it); GitHub Actions for batch; *breaks:* no always-on guarantee — laptop-independence test may fail on wake latency. |
| **C. Minimum viable always-on (if owner ever approves spend)** | ~$4–7/mo | Small VPS (post-trial) with Postgres+Redis; everything else still free-tier. **Not activated without explicit approval + hard cap.** |
| **D. GPU video generation** | not $0 today | No free always-on GPU exists; paid GPU/API costs stay disabled (spec §11). |

**Cost controls in code (Phase 1):** per-project and global caps, per-provider daily request
budgets, pause-and-notify on threshold — never silent charges (acceptance test G).

## 7. Recommended deployment (provisional)

```
Owner phone ──Discord──▶ [Oracle Always Free VM]
                           ├── bot (Phase 2) + coordinator API + scheduler
                           ├── PostgreSQL + queue (same VM, containerized)
                           ├── CPU worker #1 (same VM; more VMs later)
                           └── storage adapter ──▶ owner's 5TB (provider TBC)
GitHub Actions ── CI tests, scheduled backups, optional batch renders (ephemeral)
Kaggle/Colab ── manual experiments only (never in the critical path)
```

All of it is swap-ready: containers + provider-neutral interfaces mean if Oracle fails in
2027, the same compose stack runs elsewhere.

## 8. Evidence sources

1. [Best Free VPS & Cloud Server Tiers in 2026 (mrplanb.com)](https://www.mrplanb.com/datacenter/free-vps-tiers)
2. [Best Free Docker Hosting in 2026 (xcloud.host)](https://xcloud.host/best-free-docker-hosting/)
3. [State of Free Hosting 2026 (snapdeploy.dev)](https://snapdeploy.dev/state-of-free-hosting)
4. [Google Colab Alternatives: 8 GPU Clouds Compared 2026 (spheron.network)](https://www.spheron.network/blog/google-colab-alternatives-8-gpu-clouds-compared-2026/)
5. [Kaggle vs Colab in 2026 (fastero.com)](https://fastero.com/blog/kaggle-vs-colab)
6. [7 Free LLM APIs in 2026 (gramosoft.tech)](https://blog.gramosoft.tech/free-llm-apis/)
7. [Free AI and LLM API keys in 2026 (byoklist.com)](https://byoklist.com/guides/free-ai-api-keys)
8. [9 free LLM APIs compared 2026 (dev.to)](https://dev.to/aifree/9-free-llm-apis-compared-rate-limits-card-rules-and-the-catch-2026-3hhj)

*Free tiers change without notice — this report is a snapshot; re-verify before each phase
that depends on a specific quota.*
