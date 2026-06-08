# AI Meeting Assistant — System Design

**Version:** 1.0 · **Date:** 2026-06-07 · **Status:** Draft for review
**Targets:** macOS + Windows desktop (deferred: iOS, Android)
**Profile chosen:** Hybrid processing · Local-only storage · Personal single-user · Low cloud budget · Pluggable vendors

---

## 1. Summary

A desktop application that silently captures the audio of an online meeting (Teams, Google Meet, Webex, or any app), transcribes it in near-real-time, archives the audio locally, and — when the meeting ends — generates an editable, prompt-driven summary report. All capture is passive: the user continues to hear the meeting through their normal speakers, unmodified.

The design corrects one assumption from the brief (see §2.1): we do **not** add an in-app "virtual speaker" that you select inside Teams/Meet/Webex. That mechanism cannot capture remote participants or your own mic from inside those apps. Instead we use the operating system's native **audio loopback / process-tap** facilities to copy the system audio mix non-intrusively. A virtual audio driver is retained only as a fallback for older OS versions.

The pipeline, storage, and AI vendors are built around a **pluggable adapter architecture** so that speech-to-text (STT), large language model (LLM), and text-to-speech (TTS) providers — local or cloud — can be swapped via configuration without code changes.

---

## 2. Requirements

### 2.1 Critical correction to requirement #2/#3 — how audio is actually captured

The brief asks the app to "add another virtual speaker selectable inside Teams/Meet/Webex" and to "duplicate meeting speaker output." This does not work as stated, for three reasons:

1. A virtual **speaker** (output device) only receives audio that an app deliberately plays *to it*. Teams/Meet/Webex play remote participants to your **default** output; they will not route a copy to an arbitrary device just because it exists.
2. Even if you forced the conferencing app's output to a virtual device, you would capture **only the remote participants** — never your own microphone — so the transcript would be missing half the conversation.
3. Selecting a non-real speaker inside the meeting app risks the user not hearing anything (the original requirement #3 — "pass original output through unmodified" — would be violated).

**Corrected model — OS-native loopback (no in-app device selection required):**

| OS | Mechanism | Captures | Notes |
|---|---|---|---|
| macOS 14.2+ | Core Audio **Process Tap** (`AudioHardwareCreateProcessTap`) | Per-process / system render mix | No kernel extension, no virtual driver. Tap is read-only; original playback is untouched. |
| macOS 13+ | **ScreenCaptureKit** audio capture | System mix | Public since macOS 13; the baseline tap-free path before process taps existed. |
| macOS ≤12 | Virtual audio driver (HAL plugin, e.g. BlackHole-style) as an aggregate device | System mix | Fallback only; requires user install + permission. |
| Windows 10 2004+ / 11 | **WASAPI loopback** (`IAudioClient` + `AUDCLNT_STREAMFLAGS_LOOPBACK`), optionally **process loopback** | Render endpoint mix (what you hear) | No virtual driver. Loopback is a passive copy; playback continues unmodified. |
| Windows older | Virtual audio driver fallback | System mix | Fallback only. |

The user's microphone is captured separately via the standard input API (AVAudioEngine on macOS, WASAPI capture on Windows) and **mixed into a second channel** so the transcript distinguishes "me" from "others."

This is strictly better than the virtual-speaker idea: passive, captures both sides, and the user keeps hearing the meeting through their real default device because we only *read* the stream.

> The rest of this document keeps the brief's intent ("duplicate the meeting audio, transcribe the copy, store it, leave the original alone") but implements it through loopback rather than a virtual speaker.

### 2.2 Functional requirements

- **FR1 — Passive capture.** Capture system audio (all participants) plus local mic with zero audible effect on the live meeting.
- **FR2 — Real-time transcription.** Stream the captured audio to an STT provider and persist a timestamped, speaker-attributed transcript as the meeting runs.
- **FR3 — Audio archival.** Store the captured audio locally in a compact format alongside the transcript.
- **FR4 — End-of-meeting trigger.** Detect meeting end (loopback source goes silent/closes, conferencing app process exits, or user stops) and kick off report generation.
- **FR5 — Summarization at scale.** Summarize transcripts of 1h, 10h, and 100h durations into a structured report.
- **FR6 — Editable prompt.** The summarization prompt is stored locally, loaded at run time, and editable by the user (with templates and versioning).
- **FR7 — Pluggable vendors.** STT, LLM, and (optional) TTS providers are configurable — local or cloud — with per-provider model, endpoint, and key settings.
- **FR8 — Library & review.** Browse past meetings, play back audio, read/search transcripts, regenerate reports with a different prompt or model.

### 2.3 Non-functional requirements

- **NFR1 — Privacy.** Local-only storage; nothing leaves the machine unless a cloud provider is explicitly enabled. Audio/transcripts encrypted at rest.
- **NFR2 — Low cloud cost.** Default to local models; cloud is opt-in with visible cost estimates and budget caps.
- **NFR3 — Low live overhead.** Capture + streaming STT must not noticeably load the machine during a call (target < 1 CPU core for capture + local streaming STT on a modern laptop).
- **NFR4 — Reliability for long jobs.** A 100h summarization must be resumable and crash-safe; transcripts must survive an app crash mid-meeting.
- **NFR5 — Cross-platform reuse.** Core engine shared across macOS/Windows today and reusable for mobile later; only the capture layer is OS-specific.
- **NFR6 — Modularity.** Components are swappable behind interfaces (the explicit "pluggable" constraint).

### 2.4 Out of scope (v1)

- Real-time bot that *joins* the meeting as a participant (we tap audio locally, not via meeting APIs/bots).
- Live translation, live captions overlay, or speaker diarization beyond "me vs. others."
- Cloud sync, multi-user accounts, team sharing, or a server backend.
- Mobile (iOS/Android) builds — architecture leaves room but no implementation.
- Video capture, screen recording, or slide extraction.
- Calendar integration / auto-join / auto-record scheduling (candidate for v2).
- Legal consent enforcement (we surface notices; we do not police jurisdiction — see §9).

---

## 3. High-level architecture

We split the app into a thin **UI shell** (React Native for desktop) and a headless **Core Engine** (native, reusable) that does capture, transcription, storage, and AI orchestration. They talk over a local IPC channel. This keeps the heavy, OS-specific, and reusable logic out of the UI layer — directly satisfying the "modular / pluggable / reusable" constraint.

```
┌──────────────────────────────────────────────────────────────────┐
│  UI Shell  —  React Native (react-native-macos / -windows)        │
│  Meeting controls · live transcript · library · prompt editor ·   │
│  provider & budget settings                                       │
└───────────────▲───────────────────────────────┬──────────────────┘
                │   local IPC (JSON-RPC over      │  events (level,
                │   UNIX socket / named pipe)      │  partial txt, job %)
┌───────────────┴───────────────────────────────▼──────────────────┐
│  CORE ENGINE  (Rust, headless, cross-platform)                    │
│                                                                   │
│  ┌─────────────┐   ┌──────────────┐   ┌──────────────────────┐    │
│  │ Capture     │──▶│ Audio Router │──▶│ Archive Writer       │    │
│  │ (OS native) │   │  (tee/fanout)│   │ (Opus/FLAC → disk)   │    │
│  └─────────────┘   └──────┬───────┘   └──────────────────────┘    │
│         ▲                 │                                        │
│   loopback + mic          ▼                                        │
│                    ┌──────────────┐   ┌──────────────────────┐    │
│                    │ STT Adapter  │──▶│ Transcript Store     │    │
│                    │ (provider)   │   │ (SQLite, append-only)│    │
│                    └──────────────┘   └──────────┬───────────┘    │
│                                                  │                 │
│   ┌────────────────┐   ┌──────────────┐   ┌──────▼───────────┐    │
│   │ Prompt Store   │──▶│ Summarizer   │◀──│ Job Queue        │    │
│   │ (editable)     │   │ (map-reduce) │   │ (resumable)      │    │
│   └────────────────┘   └──────┬───────┘   └──────────────────┘    │
│                               ▼                                    │
│   ┌────────────────┐   ┌──────────────┐   ┌──────────────────┐    │
│   │ LLM Adapter    │   │ Report Store │   │ TTS Adapter (opt)│    │
│   │ (provider)     │   │ (md/pdf)     │   │ (provider)       │    │
│   └────────────────┘   └──────────────┘   └──────────────────┘    │
│                                                                   │
│   Provider Registry  ◀── config file (vendors, models, keys, $cap)│
└───────────────────────────────────────────────────────────────────┘
                    │
          Local encrypted storage (SQLite + audio files + config)
```

**Why a separate Core Engine instead of doing it all in React Native:** RN cannot capture system audio or run local ML inference — those need native code. Putting them behind a single headless engine (recommended: **Rust**, via `cpal`/native bindings + `whisper.cpp`/`llama.cpp` FFI) means the same engine binary backs the desktop app now and can be reused under a future mobile shell, a CLI, or tests. The RN shell becomes replaceable, which is the modularity the brief asks for. (Acceptable alternative: a C++ core, or a Tauri shell instead of RN — see §10 trade-offs.)

---

## 4. Component deep-dive

### 4.1 Capture layer (OS-specific, behind one interface)

A single `AudioSource` trait/interface with per-OS implementations:

```
interface AudioSource {
  listDevices() -> [Device]
  start(config: { systemLoopback: bool, mic: bool, sampleRate, channels }) -> Stream
  stop()
  onFrames(callback)        // raw PCM frames + timestamps + source tag
}
```

- **macOS impl:** Core Audio process tap / ScreenCaptureKit for system mix; AVAudioEngine for mic. Requests microphone + screen/audio-recording permission on first run.
- **Windows impl:** WASAPI loopback for system mix; WASAPI capture for mic.
- Two logical channels are tagged at the source: `remote` (loopback) and `local` (mic), enabling "me vs. others" attribution without full diarization.

**Pass-through guarantee (FR1/requirement #3):** loopback/tap reads a copy of the render stream; it never sits in the playback path, so the user's audio is bit-identical and uninterrupted. No mixing back, no added latency to live audio.

### 4.2 Audio Router (tee / fan-out)

Takes the captured PCM and fans it to two consumers with independent back-pressure:

1. **Archive branch** — buffered, encoded to a compact speech codec and written to disk.
2. **STT branch** — resampled to 16 kHz mono, framed into ~100–500 ms chunks, pushed to the STT adapter.

A bounded ring buffer decouples capture (real-time, must never block) from STT (variable latency). If STT falls behind, the router drops STT frames *before* it ever drops archive frames — the recording is the source of truth and can be re-transcribed later.

### 4.3 Archive Writer

- **Default codec: Opus @ ~24 kbps mono** — speech-optimized, tiny. (See §6 sizing.)
- Optional FLAC/WAV for users who want lossless.
- Files rotated per meeting; segmented (e.g., 10-min segments) so a crash loses at most one segment and playback can seek quickly.
- Each segment fsync-checkpointed; segment index stored in SQLite.

### 4.4 STT Adapter (pluggable)

`STTProvider` interface (streaming + batch):

```
interface STTProvider {
  capabilities() -> { streaming, languages, diarization, costPerMinute }
  transcribeStream(audioChunks) -> AsyncIterator<TranscriptEvent>   // partial + final
  transcribeBatch(audioFile) -> Transcript                          // for re-runs / 100h
}
```

- **Local default:** `whisper.cpp` (e.g., base/small/medium depending on hardware) — zero marginal cost, fully private. Used for live streaming and as the budget-free path.
- **Cloud optional:** Deepgram / AssemblyAI / OpenAI Whisper API adapters — higher accuracy/speed, used only when toggled, with per-minute cost shown.
- Output is written to the Transcript Store as **final** segments; partials are streamed to the UI but not persisted.

### 4.5 Transcript Store

- SQLite, **append-only** segment table (see §5). Append-only means a crash mid-meeting leaves a valid, queryable transcript up to the last flush.
- Full-text search (SQLite FTS5) over segment text for the library.
- Write-Ahead Logging (WAL) for crash safety; periodic checkpoint.

### 4.6 Prompt Store (editable — FR6)

- Prompts stored as rows: `{id, name, body, variables, version, is_default, updated_at}`.
- Ships with seeded templates (e.g., "Action-items & decisions", "Detailed minutes", "Exec summary").
- The summarizer loads the selected prompt at run time; users edit in the UI; edits create a new **version** (old versions retained so a report can be reproduced).
- Variables (e.g., `{{meeting_title}}`, `{{participants}}`, `{{date}}`) are interpolated before sending to the LLM.

### 4.7 Summarizer (handles 1h → 100h via map-reduce)

A 100-hour meeting is far beyond any model's context window, so summarization is **hierarchical / map-reduce**:

```
Transcript ──▶ chunk into windows (e.g. ~6–10k tokens, on speaker/topic boundaries)
            ──▶ MAP:    summarize each chunk with the user prompt        (parallel, budget-aware)
            ──▶ REDUCE: summarize the chunk-summaries into section rollups
            ──▶ SYNTH:  final pass → structured report (decisions, actions, themes, timeline)
```

- For 1h, this may be a single pass (skip map-reduce if transcript fits the model context).
- The depth of the reduce tree scales with length: 1h ≈ 1–2 levels, 100h ≈ 3–4 levels.
- Each map/reduce node is an independent **job** (see 4.9) so the whole thing is resumable and parallelism is throttled to respect the budget cap.
- Token/cost is estimated up front from transcript length and shown before running a cloud job.

### 4.8 LLM Adapter (pluggable) & TTS Adapter (optional, pluggable)

```
interface LLMProvider {
  capabilities() -> { contextWindow, costPer1kIn, costPer1kOut, local }
  complete(prompt, options) -> Completion
  stream(prompt) -> AsyncIterator<Token>
}
interface TTSProvider {
  capabilities() -> { voices, costPerChar, local }
  synthesize(text, voice) -> AudioFile
}
```

- **LLM local default:** Ollama / `llama.cpp` running a local model (e.g., an 8B-class instruct model) — free, private, good enough for minutes/action-items.
- **LLM cloud optional:** OpenAI / Anthropic / others — selected per-job for higher-quality long-context synthesis.
- **TTS** is optional (read the summary aloud): local (system TTS / Piper) or cloud (ElevenLabs/OpenAI). Off by default given the low-budget constraint.

### 4.9 Job Queue (resumable, crash-safe — NFR4)

- Persistent jobs table in SQLite. Each summarization spawns a DAG of map/reduce/synth jobs with explicit dependencies.
- On startup the engine **re-loads incomplete jobs and resumes** from the last completed node — a 100h job that crashed at 80% restarts near 80%, not 0%.
- Per-provider concurrency limits + a global budget guard that pauses cloud jobs when the spend cap is hit.

### 4.10 Provider Registry (the heart of "pluggable")

- A single config file (e.g., `providers.toml`/JSON) declares available providers, their type (stt/llm/tts), endpoint, model, credentials reference (key stored in OS keychain, **never** in the config file), and an optional monthly `$budget_cap`.
- The engine loads adapters by `kind` at runtime; adding a new vendor = implementing the interface + a registry entry. No core changes.
- A `capabilities()` call lets the UI show only valid options (e.g., hide streaming for a batch-only provider) and surface cost.

---

## 5. Data model (local SQLite + filesystem)

```
meetings
  id PK · title · started_at · ended_at · platform_guess · duration_sec
  status (recording|processing|done|failed) · participants_json

recording_segments
  id PK · meeting_id FK · seq · file_path · codec · start_ms · end_ms · bytes

transcript_segments            -- append-only
  id PK · meeting_id FK · seq · source (remote|local) · start_ms · end_ms
  text · confidence · provider · is_final
  -- FTS5 virtual table mirrors `text` for search

prompts
  id PK · name · body · variables_json · version · is_default · updated_at

summaries (reports)
  id PK · meeting_id FK · prompt_id FK · prompt_version · model · provider
  body_md · tokens_in · tokens_out · cost_estimate · created_at

jobs
  id PK · meeting_id FK · type (map|reduce|synth|transcribe)
  parent_job_id · status · input_ref · output_ref · attempts · error · cost

providers   -- mirror of registry for runtime state
  id PK · kind (stt|llm|tts) · name · model · endpoint · enabled
  budget_cap · spent_this_month
```

- **Audio** lives on the filesystem (segmented Opus); the DB holds only paths + offsets.
- **Encryption at rest (NFR1):** SQLite via SQLCipher; audio files encrypted with a key stored in the OS keychain (Keychain / DPAPI). App unlock gates decryption.

---

## 6. Capacity & sizing (single user, but long meetings)

**Audio storage** (mono speech):

| Format | Bitrate | 1h | 10h | 100h |
|---|---|---|---|---|
| Opus (default) | ~24 kbps | ~11 MB | ~108 MB | ~1.1 GB |
| FLAC ~ (16 kHz) | ~120 kbps | ~54 MB | ~540 MB | ~5.3 GB |
| WAV 16 kHz/16-bit | 256 kbps | ~115 MB | ~1.1 GB | ~11.5 GB |

→ Opus default keeps even a 100h archive ~1 GB. Lossless is opt-in.

**Transcript size:** ~150 words/min ≈ ~9k words/hour ≈ ~900k words for 100h ≈ a few MB of text — trivial for SQLite; FTS index a small multiple.

**Summarization compute/cost (the real constraint):**
- 100h ≈ ~1.2M tokens of transcript. A single-pass cloud summary is impossible (context) and, if it were, expensive — hence map-reduce.
- Map-reduce with a **local** LLM = $0 marginal, bounded only by time (throttled background job).
- If cloud is toggled, the budget guard estimates `tokens × price` before running and stops at the cap. This is why local-default + opt-in cloud directly serves the "low cloud budget" constraint.

**Live overhead (NFR3):** capture is sub-1% CPU; local streaming Whisper (small) runs comfortably under one core on a modern laptop. Heavier local models are reserved for the post-meeting batch pass, not the live path.

---

## 7. IPC / API contracts (UI shell ⇄ Core Engine)

Local JSON-RPC over a UNIX domain socket (macOS) / named pipe (Windows). Representative surface:

**Commands (request/response):**
```
devices.list()                          -> [Device]
capture.start({systemLoopback, mic})    -> {meetingId}
capture.stop({meetingId})               -> {meetingId, durationSec}
meetings.list({query, page})            -> [MeetingSummary]
meetings.get({meetingId})               -> Meeting + segments
report.generate({meetingId, promptId, llmProviderId}) -> {jobId}
report.get({meetingId})                 -> Summary
prompts.list() / prompts.save(prompt)   -> Prompt
providers.list() / providers.upsert()   -> Provider
budget.status()                         -> {capsByProvider, spentThisMonth}
```

**Events (engine → UI, push):**
```
capture.level     {meetingId, remoteDb, localDb}      // VU meters
transcript.partial{meetingId, text, source}
transcript.final  {meetingId, segment}
job.progress      {jobId, pct, stage}
job.done|failed   {jobId, result|error}
budget.warn       {provider, spent, cap}
```

This contract is the modularity seam: the engine could be driven by the RN UI, a CLI, automated tests, or a future mobile shell without change.

---

## 8. Reliability, failure handling & meeting-end detection

**Meeting-end trigger (FR4)** — any of:
- Loopback source reports silence + the known conferencing process (Teams/Meet browser tab/Webex) exits or releases the audio device, **or**
- A configurable silence timeout (e.g., 90 s of no remote audio), **or**
- The user presses Stop.

We don't rely on meeting-app APIs; we infer end from the audio device + process state, which works for any conferencing app (Meet runs in a browser tab — process heuristics + silence cover it).

**Crash safety:**
- Transcript and segments are flushed continuously (append-only + WAL), so an app/OS crash mid-meeting yields a valid recording + transcript up to the last flush; on restart the meeting is marked `recoverable` and the user can transcribe any un-transcribed tail from the saved audio.
- Summarization jobs are persisted and resumable (§4.9).

**Provider failures:** STT/LLM adapter calls have timeout + retry with backoff; on repeated cloud failure the engine can fall back to the local provider (configurable). Archive is never blocked by provider failure.

**Back-pressure:** the router protects real-time capture first (drop STT frames, never archive frames) — recording integrity > live transcript latency.

---

## 9. Privacy, security & consent

- **Local-only by default (NFR1):** no network calls unless a cloud provider is explicitly enabled; the UI clearly indicates when audio/text is leaving the machine.
- **Encryption at rest:** SQLCipher DB + encrypted audio; keys in OS keychain.
- **Secrets:** API keys in Keychain/DPAPI, referenced (not stored) by the config file.
- **Consent (legal):** recording laws vary (one-party vs all-party consent jurisdictions; corporate policy; platform ToS). The app **surfaces a consent reminder** and an optional in-meeting "recording" indicator, but enforcement/legality is the user's responsibility. We document this; we do not provide legal guarantees. *(This is information, not legal advice.)*
- **Data control:** per-meeting delete (audio + transcript + reports), and a global purge.

---

## 10. Key trade-offs (made explicit)

| Decision | Chosen | Alternative | Why |
|---|---|---|---|
| Audio capture | OS-native loopback / process tap | In-app "virtual speaker" (brief), or installed virtual driver | Loopback is passive, captures both sides, needs no install on modern OS, and can't break live playback. Virtual driver kept only as legacy fallback. |
| Core engine language | Rust headless engine | All-in-RN; or C++; or Tauri shell | RN can't capture audio or run local ML. Rust gives one reusable, safe, cross-platform core with mature `whisper.cpp`/`llama.cpp` bindings. |
| UI framework | React Native (per brief) | Tauri + web UI | Honors the brief and the deferred mobile path; the IPC seam means the shell is replaceable if RN-desktop maturity becomes a blocker. |
| Processing | Hybrid: local default, cloud opt-in | Cloud-only / local-only | Meets low-budget + privacy while allowing quality when wanted. |
| Live STT model | Small local Whisper streaming | Cloud streaming STT | $0, private, low-latency enough; cloud is a toggle for accuracy. |
| Long summaries | Map-reduce hierarchical | Single-pass long-context | 100h exceeds any context window; map-reduce is the only scalable, budget-controllable path. |
| Storage | Local SQLite + Opus files | Cloud / single big file | Matches "local-only"; Opus keeps 100h ~1 GB; segmented files = crash-safe + seekable. |
| Vendor coupling | Adapter interfaces + registry | Hard-coded SDK calls | The "pluggable/reusable" constraint — swap STT/LLM/TTS via config. |

**What I'd revisit as it grows:** speaker diarization (beyond me/others) once a good local model is viable; calendar-driven auto-record (v2); optional encrypted cloud sync if multi-device is ever needed; a plugin SDK so third parties can publish provider adapters; and re-evaluating RN-desktop vs Tauri after the MVP if RN-windows/macos friction is high.

---

## 11. Implementation plan (phased)

**Phase 0 — Foundations (engine skeleton + IPC)**
- Rust engine scaffold, JSON-RPC IPC, SQLite schema + migrations, config/registry loader, keychain integration.
- RN desktop shell (macos + windows) wired to the engine; basic settings screen.

**Phase 1 — Capture & archive (the riskiest piece — do it first)**
- macOS process-tap/ScreenCaptureKit capture + mic; Windows WASAPI loopback + mic.
- Audio router (tee), Opus archive writer, segmentation, VU-meter events.
- Prove the pass-through guarantee and "me vs. others" tagging.

**Phase 2 — Live transcription**
- STT adapter interface + local `whisper.cpp` streaming impl; transcript store (append-only + FTS); live partial/final transcript in UI.
- One cloud STT adapter (e.g., Deepgram) behind the toggle to validate the abstraction.

**Phase 3 — End detection + summarization**
- Meeting-end heuristics; prompt store + editor with versioning + seeded templates.
- Summarizer with map-reduce; job queue (resumable); local LLM (Ollama/llama.cpp) + one cloud LLM adapter; report store + markdown/PDF export.

**Phase 4 — Library, budget, polish**
- Meeting library, search, playback, regenerate-report; budget caps + cost estimates + warnings; optional TTS adapter; encryption hardening; consent UX.

**Phase 5 (deferred) — Mobile**
- Reuse engine under a mobile RN shell; replace capture layer with mobile-appropriate capture (platform-limited — likely mic-only / share-audio constraints).

**Sequencing rationale:** capture is the highest-risk, most OS-specific work and gates everything else, so it comes immediately after the engine skeleton. Each phase is independently demoable, and the adapter interfaces are introduced the first time a second provider appears (phases 2 and 3) to keep the abstraction honest.

---

## 12. Open questions for next iteration

1. Hardware floor — what's the minimum spec to target? It sets which local Whisper/LLM sizes are the default.
2. Report export — markdown only, or also PDF/DOCX out of the box?
3. Browser-based Meet — confirm process/tab heuristics are acceptable for end-detection, or do we want an optional browser-extension signal?
4. Should v1 ship the optional in-meeting "recording" indicator, or defer all consent UX to a banner?
