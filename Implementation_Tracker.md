# Implementation Tracker

Resume point for every session. Read this first; update after every meaningful change.

---

## Current state

**Phase:** 0 — Fork & Baseline  
**Status:** In progress — Phase 0 scaffolding committed; smoke test not yet run

---

## Done

- [x] Forked `Zackriya-Solutions/meetily` → `mohamedishak440/meetily`
- [x] Cloned fork into working directory; checked out `v0.4.0` as branch `our-baseline`
- [x] Created `UPSTREAM.md` (pinned commit `0281737`, known conflict: BlackHole vs SCK)
- [x] Created `docs/open-questions.md` (Q1–Q4)
- [x] Created `docs/spec/` (phase 0–5 specs written in prior session)
- [x] Created `.github/workflows/ci.yml` (lint gate: rustfmt + clippy + eslint on push/PR)
- [x] Updated `CLAUDE.md` with build commands, file map, Meetily conventions, and BlackHole conflict note
- [x] Created `docs/meetily-reference.md` (upstream internals detail)

## In progress

- [ ] Commit and push `our-baseline` branch to fork

## Next step

1. Commit all Phase 0 additions and push to `mohamedishak440/meetily` on `our-baseline`.
2. **Manual smoke test (must do on macOS Apple Silicon):**
   - Install deps: follow `docs/BUILDING.md`
   - Run `./clean_run.sh`
   - Record 30 seconds of audio → verify transcript appears → verify Ollama summary generates
   - If BlackHole appears as a requirement, confirm whether `core_audio.rs` already uses SCK process-tap or still needs a virtual device; update `UPSTREAM.md` accordingly
3. **CI gate:** verify `ci.yml` goes green on the pushed branch (lint only — no binary build in CI).
4. **Tag** the passing commit as `v0-baseline`.
5. Update this tracker: Phase 0 → done; Phase 1 → in progress.

---

## Key decisions & deviations

| Area | Decision | Why |
|---|---|---|
| BlackHole / macOS audio | Flagged in UPSTREAM.md as a known conflict. SCK path appears to exist in v0.4.0; needs smoke-test confirmation. Phase 1 (G6/G7) will replace any virtual-driver path with SCK process-tap. | CLAUDE.md constraint: no virtual driver |
| CI workflows | Added `ci.yml` (lint/check on PR). Kept Meetily's existing `build-macos.yml` / `build-windows.yml` (release/`workflow_dispatch` only) unchanged. | Full binary builds are too slow for PR gates |
| CLAUDE.md | Replaced Meetily's 17KB CLAUDE.md. Extracted useful content to `docs/meetily-reference.md`. | Our project instructions must take precedence |
| Branch name | `our-baseline` (not `main`). Our `main` maps to upstream `main`. `our-baseline` is where Phase 0–4 work lands. | Keeps upstream `main` clean for cherry-picks |

---

## Open blockers

- **Smoke test not run yet.** Cannot confirm Phase 0 exit until done manually (requires macOS + Ollama).
- **Q1–Q4 unanswered** (see `docs/open-questions.md`). Q1 (hardware floor) blocks Phase 1 model choice; Q3 (Meet end-detect) and Q4 (consent UX) block Phase 4 design.
