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
- [x] Created `.github/workflows/ci.yml` (lint gate: rustfmt + clippy + eslint on push/PR; added cmake install for macOS)
- [x] Updated `CLAUDE.md` (build commands corrected to `cd frontend/`; file map; Meetily conventions; BlackHole conflict note)
- [x] Created `docs/meetily-reference.md` (upstream internals detail)
- [x] Ran `cargo fmt` on entire workspace (112 files; upstream had no fmt enforcement)
- [x] Fixed 2 clippy warnings in `build/ffmpeg.rs` (unused import; `last()` → `next_back()`)

## In progress

- [ ] Manual smoke test (macOS Apple Silicon)

## Next step

1. **Manual smoke test (macOS Apple Silicon):**
   - `cd frontend && pnpm install && pnpm run tauri:dev`
   - Record 30 seconds → verify transcript appears → verify Ollama summary generates
   - Confirm `core_audio.rs` uses SCK process-tap, not BlackHole; update `UPSTREAM.md` build notes
2. **CI gate:** verify `ci.yml` goes green on GitHub Actions.
3. **Tag** passing commit as `v0-baseline`.
4. Update tracker: Phase 0 → done; Phase 1 → in progress.

---

## Key decisions & deviations

| Area | Decision | Why |
|---|---|---|
| BlackHole / macOS audio | Flagged in UPSTREAM.md as a known conflict. SCK path appears to exist in v0.4.0; needs smoke-test confirmation. Phase 1 (G6/G7) will replace any virtual-driver path with SCK process-tap. | CLAUDE.md constraint: no virtual driver |
| CI workflows | Added `ci.yml` (lint/check on PR + `brew install cmake`). Kept Meetily's existing `build-macos.yml` / `build-windows.yml` (release/`workflow_dispatch` only) unchanged. | Full binary builds are too slow for PR gates; cmake needed for whisper-rs-sys/llama-cpp |
| CLAUDE.md | Replaced Meetily's 17KB CLAUDE.md. Extracted useful content to `docs/meetily-reference.md`. Build paths corrected to `cd frontend/`. | Our project instructions must take precedence |
| Branch name | `our-baseline` (not `main`). Our `main` maps to upstream `main`. `our-baseline` is where Phase 0–4 work lands. | Keeps upstream `main` clean for cherry-picks |
| cargo fmt on whole workspace | Ran `cargo fmt` across 112 files. Noted in UPSTREAM.md. | CI gate requires passing `fmt --check`; upstream does not enforce fmt |

---

## Open blockers

- **Smoke test not run yet.** Cannot confirm Phase 0 exit until done manually (requires macOS + Ollama).
- **Q1–Q4 unanswered** (see `docs/open-questions.md`). Q1 (hardware floor) blocks Phase 1 model choice; Q3 (Meet end-detect) and Q4 (consent UX) block Phase 4 design.
