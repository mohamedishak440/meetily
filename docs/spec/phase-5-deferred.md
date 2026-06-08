# Phase 5 — Deferred Features

**Status:** Deferred (not in v1 scope)  
**Depends on:** Phase 4 complete (v1 shipped)  
**Prerequisite for starting:** explicit decision to resume

---

## Overview

These features are out of v1 scope by design. They are captured here so that the work is not lost, but nothing here should be built until the v1 exit criterion (Phase 4) passes on both platforms.

---

## D1 — Advanced export (PDF / DOCX)

**Context:** Meetily PRO gates PDF/DOCX export. v1 ships markdown only.

**When to revisit:** user demand + decision to build vs license from Meetily PRO.

**Implementation sketch:**
- `export_pdf`: pass `body_md` through `pandoc` (bundled binary) or use a Rust PDF crate (`printpdf` / `typst`). Typst is the stronger choice — handles markdown, produces clean output, MIT-licensed.
- `export_docx`: `pandoc` from markdown to docx. Simpler than native docx generation.
- Both calls are new Tauri commands; no changes to the summarizer or job queue.

---

## D2 — Optional TTS (text-to-speech playback of summaries)

**Context:** design §4.8 marks TTS as optional and off-by-default.

**When to revisit:** if users want to "listen" to their meeting summary (e.g., during commute).

**Implementation sketch:**
- Add `TTSProvider` interface (see design §4.8).
- Local: [Piper](https://github.com/rhasspy/piper) — fast, MIT, on-device. Rust bindings exist.
- Cloud: OpenAI TTS / ElevenLabs — adapters behind the existing provider registry.
- A new Tauri command `tts_synthesize({ summaryId, voice })` returns an audio file path; frontend plays it.
- Budget guard already covers cloud TTS cost tracking (extend `providers.cost_per_char`).

---

## D3 — Mobile (iOS / Android)

**Context:** design explicitly defers mobile. The Rust core is shell-agnostic, making this feasible later.

**When to revisit:** when there is a concrete requirement (e.g., record calls on a phone).

**Implementation sketch (two paths):**

**Path A — Tauri 2 mobile:** Tauri 2 supports iOS and Android builds. The existing Tauri IPC would carry over; only the capture layer needs a new OS-specific impl (iOS: `AVAudioSession` + `AudioUnit`, limited to mic-only or shared audio if the OS allows; Android: `AudioRecord`). Likely mic-only capture — neither iOS nor Android exposes a public loopback API.

**Path B — React Native shell over the Rust engine:** the engine binary is a sidecar exposed via a local socket (as designed); an RN mobile app connects to it over the IPC seam. Same engine, different shell.

**Hard constraint:** mobile capture will be mic-only (no system loopback on iOS/Android without special entitlements). The "me vs. others" distinction is lost on mobile unless the user is on a speakerphone and both sides are in the mic.

---

## D4 — Calendar integration / auto-record scheduling

**Context:** design §2.4 lists this as a v2 candidate.

**When to revisit:** if users want hands-free auto-start when a calendar event begins.

**Implementation sketch:**
- Subscribe to system calendar events (macOS `EventKit`, Windows Calendar COM API).
- On event start: if event is a meeting (location contains a meeting URL or known conferencing app), auto-start capture.
- Show a "Starting to record: [event title]" notification with a Cancel button.
- This is a new Rust module; no changes to the capture engine itself.

---

## D5 — Encrypted cloud sync / multi-device

**Context:** explicitly out of scope (local-only is a hard NFR).

**When to revisit:** only if the product pivots to multi-user or team use.

**Implementation sketch:** end-to-end-encrypted sync of the SQLCipher DB and audio files to a user-controlled storage backend (S3-compatible, iCloud Drive, etc.). Keys never leave the device — only ciphertext synced. This is a significant architecture change; model it as a separate backend module.

---

## D6 — Speaker diarization (beyond "me vs. others")

**Context:** full diarization is a Meetily PRO feature. v1 only does local/remote tagging.

**When to revisit:** when a good local diarization model (e.g., pyannote.audio ONNX export) is available with an acceptable license, or if we license from Meetily PRO.

**Implementation sketch:**
- Add a `speaker_id` column to `transcript_segments`.
- Run a diarization model as a post-processing step (not live — batch after capture).
- The model assigns speaker labels; UI shows color-coded speaker tracks.
- No changes to capture or summarization paths.

---

## Tracking

These items should be revisited when:
- v1 is shipped and stable on both platforms.
- User feedback identifies clear demand.
- A team decision is made to expand scope.

Open a new spec file under `docs/spec/` for whichever deferred item is chosen next.
