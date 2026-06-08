# CLAUDE.md — AI Meeting Assistant

Privacy-first desktop app (macOS + Windows). Passively captures meeting audio (Teams/Meet/Webex), transcribes locally in real time, stores locally, then generates an editable prompt-driven summary. **Fork of [Meetily](https://github.com/Zackriya-Solutions/meetily)** (Tauri, MIT).

Specs are source of truth — read before coding, and if code disagrees, docs win (or raise it):
`ai-meeting-assistant-design.md` (behavior) · `tauri-meetily-implementation-plan.md` (phases, gaps G1–G10, schema) · `reuse-research.md` (reuse rationale).

## Working style (biases toward caution; use judgment on trivial tasks)

- **Think before coding.** State assumptions; if multiple readings exist, surface them — don't pick silently. If unclear, stop and ask. If a simpler way exists, say so and push back.
- **Simplicity first.** Minimum code that solves the task; nothing speculative. No unrequested abstractions/config/error-handling. If 200 lines could be 50, rewrite.
- **Surgical changes.** Touch only what the request needs; every changed line must trace to it. Match existing style. Don't refactor or reformat untouched code; remove only orphans YOUR change created; flag pre-existing dead code, don't delete it.
- **Goal-driven.** Turn the task into a verifiable check ("fix bug" → write a failing test, then make it pass), state a brief plan with per-step verification, and loop until it passes.

## Implementation_Tracker.md (read first, update last)

Maintain `Implementation_Tracker.md` as the resume point across sessions. **At session start, read it before anything else; at session end (or after any meaningful change), update it.** Keep it short and current — prune stale notes, don't append-only.

It must always answer "where am I and what's next" for a fresh agent:
- Current phase + gap (e.g. Phase 3 / G2 map-reduce) and overall status.
- Done · In progress · Next step (one concrete action).
- Key decisions & deviations from the specs (with one-line why).
- Open blockers / questions awaiting the user.

If it conflicts with the specs, the specs win — fix the tracker.

## Don't assume (the costly mistakes)

- **No "virtual speaker," no virtual audio driver.** Capture = passive OS loopback (WASAPI / Core Audio process-tap / ScreenCaptureKit). Never touch the playback path — the user hears the meeting bit-identical and uninterrupted. See design §2.1.
  - **Meetily upstream currently uses BlackHole (macOS) for system audio.** This is a virtual driver — we must replace it with ScreenCaptureKit process-tap in Phase 1 (G6/G7). Do not add or recommend BlackHole in any new code.
- **Local-first; cloud is opt-in.** No network call on a default path. Cloud STT/LLM only when a provider is explicitly enabled, after a cost estimate + budget check. Local jobs are never blocked by budget logic.
- **Never summarize a 100h transcript in one call** — exceeds every context window. Use map-reduce (G2) + resumable job queue (G3). 1h may be single-pass.
- **Secrets → OS keychain** (Keychain/DPAPI), referenced by the registry. Never in `providers.*` or git.
- **Meetily PRO ≠ available.** Diarization, auto-join, advanced PDF/DOCX export are paid. v1 = "me vs. others" tagging + markdown only. Don't depend on PRO code.
- **Crash-safe by contract.** Transcript = append-only + WAL; audio = segmented Opus + per-segment fsync. Never buffer a whole meeting and flush at the end.
- **"Done" = phase Exit criterion passes on BOTH macOS and Windows.**
- **`backend/` is archived legacy.** The Python/FastAPI backend under `backend/` is unsupported — do not add code there, do not restart its server, do not reference its endpoints.

## Stack & layout

- Tauri single app. `frontend/` = Next.js + TS, **UI only** — calls Tauri commands, renders events, holds no business logic.
- Rust core: Audio Engine · Transcription Engine (whisper.cpp / Parakeet) · Summary Engine (Ollama + cloud adapters) · Database (SQLite) · new Job Queue. **All cross-OS work (capture, ML, DB, jobs) lives here, not the frontend.**
- Storage: SQLite (wrap in SQLCipher) + segmented Opus on disk + FTS5 search.

## Build commands

```bash
# All scripts live in frontend/ — run from there
cd frontend

# macOS
./clean_run.sh              # clean build + run (info logging)
./clean_run.sh debug        # with debug logging
./clean_build.sh            # production build

# Windows
clean_run_windows.bat
clean_build_windows.bat

# Or manually (from frontend/)
pnpm install
pnpm run tauri:dev          # dev mode (auto-detects GPU)
pnpm run tauri:build        # production build

# GPU variants
pnpm run tauri:dev:metal    # macOS Metal
pnpm run tauri:dev:cuda     # NVIDIA CUDA
pnpm run tauri:dev:vulkan   # AMD/Intel Vulkan
pnpm run tauri:dev:cpu      # CPU-only

# Rust debug logging
RUST_LOG=debug ./clean_run.sh
RUST_LOG=app_lib::audio=debug ./clean_run.sh   # audio module only
```

## Key file locations

| Area | File |
|---|---|
| Tauri entry / command registration | `frontend/src-tauri/src/lib.rs` |
| Audio pipeline (mixing, VAD) | `frontend/src-tauri/src/audio/pipeline.rs` |
| Recording orchestration | `frontend/src-tauri/src/audio/recording_manager.rs` |
| Audio file writing | `frontend/src-tauri/src/audio/recording_saver.rs` |
| macOS system capture | `frontend/src-tauri/src/audio/capture/core_audio.rs` |
| Windows capture | `frontend/src-tauri/src/audio/devices/platform/windows.rs` |
| Database module | `frontend/src-tauri/src/database/mod.rs` |
| Whisper engine | `frontend/src-tauri/src/whisper_engine/whisper_engine.rs` |
| Main UI | `frontend/src/app/page.tsx` |
| Global state (React) | `frontend/src/components/Sidebar/SidebarProvider.tsx` |
| Tauri config | `frontend/src-tauri/tauri.conf.json` |

## Conventions

- **Pluggable behind interfaces.** New STT/LLM/TTS = implement `STTProvider`/`LLMProvider`/`TTSProvider` + registry entry. No core changes, no hard-coded SDK calls on a path.
- **Additive DB migrations** (add tables/columns; don't alter Meetily's) → stays mergeable upstream. Each migration gets a forward test on a populated DB.
- **Fork discipline.** Keep engine modules close to upstream; isolate new work; log every divergence in `UPSTREAM.md`.
- **Never run `cargo fmt` on upstream Rust files.** Upstream does not enforce rustfmt; reformatting 100+ files creates noise diffs and merge conflicts. Only format files you authored. CI does not run `cargo fmt --check`. If a linter/IDE offers to format-on-save, disable it for `frontend/src-tauri/src/` and `llama-helper/`.
- **Source tags are first-class.** Every transcript segment carries `source` = `local` (mic) / `remote` (loopback). Don't drop or merge.
- **Cost tracked.** Cloud jobs estimate `tokens × price` first; record `tokens_in/out`, `cost_estimate`; respect `budget_cap` / `spent_this_month`.
- **Style:** existing repo linters (rustfmt + clippy, eslint/prettier). Match surrounding code; don't reformat untouched files.
- **Error handling:** Rust uses `anyhow::Result`; frontend uses try-catch with user-friendly messages.
- **Audio naming:** use "microphone" and "system" — not "input"/"output".
- **Shared state:** `Arc<RwLock<T>>` for shared mutable state across async tasks; `Arc<AtomicBool>` for simple flags.
- **Hot-path logging:** use `perf_debug!()` / `perf_trace!()` macros — zero cost in release builds, verbose in debug.
- **New Tauri command pattern:** define with `#[tauri::command]` → register in `tauri::generate_handler![...]` in `lib.rs` → call via `invoke()` from frontend.

## Audio pipeline facts (critical for Phase 1)

The existing pipeline has **two parallel paths**:
- **Recording path**: pre-mixed RMS-ducked audio → `RecordingSaver` (writes to file)
- **Transcription path**: VAD-filtered speech only → `WhisperEngine` (~70% load reduction)

Expects **48 kHz** sample rate; resampling happens at capture time. Whisper models are loaded once and cached — changing models requires app restart. Full audio module map: see `docs/meetily-reference.md`.

## Build & test

Use the repo's own scripts — don't invent a build. Verify a real capture→transcribe→summarize round-trip on both OSes before claiming a build works. Gates blocking "done":
- Capture: bit-identical pass-through; non-empty dual-channel. Manual matrix on Teams/Meet/Webex.
- Crash recovery: kill mid-meeting and mid-summarization → valid transcript/archive, jobs resume near failure.
- Summarizer: chunk-boundary unit tests + e2e on 1h/10h/100h; bounded memory; budget respected.
- Providers: contract tests vs a fake adapter + one real local + one real cloud each.

## Platform notes

- **macOS:** ScreenCaptureKit requires macOS 13+. Needs **microphone + screen recording** permissions. Metal + CoreML GPU automatically enabled.
- **Windows:** WASAPI exclusive mode can conflict with other apps — handle the `AUDCLNT_E_DEVICE_IN_USE` error. CUDA/Vulkan via Cargo features. Requires Visual Studio Build Tools (C++ workload).
- **Paths:** always use Tauri path APIs (`downloadDir`, `appDataDir`, etc.) — never hardcode.

## Security

Local-only by default; UI must signal when anything leaves the machine. SQLCipher DB, encrypted audio, keys in keychain. Show a consent reminder (inform, not legal guarantee). Per-meeting delete + global purge must fully remove audio + transcript + reports.

## When unsure

Check the three specs first. Open questions (flag, don't guess): hardware floor / default model sizes, v1 export formats, Meet-in-browser end-detection, consent-UX scope. Full open question list: `docs/open-questions.md`.
