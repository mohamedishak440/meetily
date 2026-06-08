# Implementation Plan — AI Meeting Assistant on Tauri (forking Meetily)

**Version:** 1.0 · **Date:** 2026-06-07
**Companions:** `ai-meeting-assistant-design.md` (the design), `reuse-research.md` (the reuse survey)
**Decision locked:** Build on **Tauri**, forking/adapting **[Meetily](https://github.com/Zackriya-Solutions/meetily)** (MIT) as the base.

---

## 1. Strategy

Meetily already implements ~90% of our design on the exact stack we'd otherwise assemble by hand: a single Tauri app with a **Rust core** (Audio Engine, Transcription Engine, SQLite, Summary Engine) and a **Next.js frontend**. Rather than greenfield, we **fork Meetily Community Edition** and bend it to our design spec — keeping its capture, transcription, and inference plumbing, and rebuilding the pieces our design treats as first-class that Meetily either does differently or gates behind PRO.

Consequences of choosing Tauri over the brief's React Native preference:
- We gain the fastest route to a working product and a battle-tested capture path.
- We lose the brief's RN mobile-later path. Mobile becomes a separate effort (Tauri 2 mobile or a future RN shell over the same Rust engine). The Rust core stays reusable either way, so this is deferred, not lost.

**Working principle:** keep Meetily's engine modules; replace its product surfaces (prompt handling, provider/budget config, end-of-meeting flow, long-transcript summarization, encryption) with our design's versions, behind clean interfaces so the fork stays mergeable with upstream where practical.

---

## 2. Component mapping — our design ⇄ Meetily

| Our design (engine) | Meetily module | Status |
|---|---|---|
| Capture layer (loopback + mic) | **Audio Engine** (Rust) | Reuse as-is. Captures mic + system audio with mixing/ducking. |
| Audio Router (tee/fan-out) | inside Audio Engine | Reuse; verify archive-vs-STT back-pressure priority. |
| Archive Writer (Opus, segmented) | Audio Engine recording | Adapt — confirm/segment + Opus encode + crash-safe checkpoints. |
| STT Adapter | **Transcription Engine** (whisper.cpp / Parakeet) | Reuse; ensure provider seam matches our `STTProvider`. |
| Transcript Store (append-only + FTS) | **Database** (SQLite) | Adapt schema — add append-only segments, FTS5, source tag. |
| Prompt Store (editable + versioned) | Summary prompt handling | **Build** — versioned, editable, templated (our FR6). |
| Summarizer (map-reduce 1h→100h) | **Summary Engine** | **Extend** — add hierarchical map-reduce for long transcripts. |
| LLM Adapter (pluggable) | Summary Engine provider layer | Reuse + extend (Ollama/Claude/Groq/OpenRouter/OpenAI present). |
| TTS Adapter (optional) | — | **Build** (optional, Piper) — deferred. |
| Provider Registry + budget caps | Settings/provider config | **Extend** — add `$budget_cap`, spend tracking, cost estimates. |
| Job Queue (resumable) | — | **Build** — resumable map/reduce/synth jobs. |
| Meeting-end trigger | manual stop | **Build** — auto-detect (silence + process/device release). |
| Encryption at rest | local plaintext | **Build** — SQLCipher + encrypted audio + keychain. |
| UI shell | **Next.js frontend** | Reuse + add prompt editor, budget UI, library/search. |
| IPC seam | Tauri commands/events | Reuse. |

**Reuse wholesale:** Audio Engine, Transcription Engine, Tauri shell, GPU acceleration, base SQLite/Next.js scaffolding.
**Adapt:** schema, archive writer, provider/settings layer, summary engine.
**Build new:** versioned prompt store, map-reduce summarizer, job queue, budget caps, meeting-end auto-detect, encryption, library search, optional TTS.

---

## 3. Gap analysis (what we add to reach our design)

These are the deltas between Meetily-as-shipped and our design. Each becomes a workstream.

**G1 — Editable, versioned prompt store (FR6).** Meetily has summary generation; custom templates are a PRO feature. We build a community-side `prompts` table (id, name, body, variables, version, is_default), a prompt editor in the UI, variable interpolation (`{{title}}`, `{{participants}}`, `{{date}}`), and version retention so a report can be reproduced from its exact prompt version.

**G2 — Long-transcript summarization (FR5, 1h→100h).** Meetily summarizes typical meetings; 100h exceeds any context window. We add hierarchical **map-reduce** in the Summary Engine: chunk on speaker/topic boundaries → map-summarize → reduce rollups → final synth. Depth scales with length. Each node is a job (G3).

**G3 — Resumable job queue (NFR4).** New `jobs` table + scheduler in the Rust core. A summarization spawns a DAG of map/reduce/synth jobs; on restart, incomplete jobs resume from the last completed node. Per-provider concurrency + global budget guard.

**G4 — Budget caps & cost estimation (NFR2, "low cloud budget").** Extend the provider config with `budget_cap` + `spent_this_month`; estimate `tokens × price` before any cloud job; show the estimate, warn at threshold, and pause cloud jobs at the cap (local jobs never blocked).

**G5 — Meeting-end auto-detect (FR4).** Today: manual stop. Add heuristics: remote-audio silence timeout + conferencing process/device release → auto-stop and trigger report generation. User stop and a configurable timeout remain.

**G6 — "Me vs. others" attribution.** Tag transcript segments `local` (mic) vs `remote` (loopback) at capture, persisted on each segment. (Full diarization stays out of scope / PRO.)

**G7 — Crash-safe transcript + archive.** Append-only transcript segments with WAL; segmented Opus with per-segment fsync; on restart, mark recoverable meetings and offer to transcribe any un-transcribed audio tail.

**G8 — Encryption at rest (NFR1).** SQLCipher for the DB; encrypt audio files; key in OS keychain (Keychain/DPAPI); app-unlock gates decryption.

**G9 — Library, search, regenerate.** Meeting list, FTS5 search over transcripts, audio playback with seek, and "regenerate report with a different prompt/model."

**G10 — Consent UX & export.** Consent reminder + optional recording indicator; markdown export in v1 (PDF/DOCX later; note Meetily gates advanced export behind PRO).

---

## 4. Data model changes (on Meetily's SQLite)

Add/extend tables to match the design's model:

```
meetings            (extend)  + status(recording|processing|done|failed|recoverable),
                              + participants_json, platform_guess, duration_sec
recording_segments  (new)     id, meeting_id, seq, file_path, codec, start_ms, end_ms, bytes
transcript_segments (new/adapt) append-only; + source(local|remote), is_final, provider, confidence
                              + FTS5 mirror over text
prompts             (new)     id, name, body, variables_json, version, is_default, updated_at
summaries           (adapt)   + prompt_id, prompt_version, model, provider,
                              + tokens_in, tokens_out, cost_estimate
jobs                (new)     id, meeting_id, type(map|reduce|synth|transcribe),
                              parent_job_id, status, input_ref, output_ref, attempts, error, cost
providers           (adapt)   + budget_cap, spent_this_month
```

Migrations are additive where possible to ease upstream merges. Wrap the DB in SQLCipher (G8).

---

## 5. Phased milestones

Each phase is independently demoable. Phases 0–1 are fork/setup and validation; the new-build work front-loads the riskiest gaps.

### Phase 0 — Fork & build baseline (set up the ground)
- Fork Meetily; build from source on macOS (Apple Silicon) and Windows x64 per the repo's BUILDING guide (Rust + Node/pnpm).
- Get a clean local build running end-to-end (capture → transcribe → summarize) on both OSes.
- Stand up our fork's CI (build + lint + test on mac + win), branch strategy, and an `UPSTREAM.md` tracking which modules we diverge from.
- **Exit:** our fork builds and records a real meeting on both platforms.

### Phase 1 — Validate & harden capture (de-risk first)
- Verify mic + system-audio capture and the pass-through guarantee (user keeps hearing the meeting, unmodified) on Teams, Google Meet (browser), Webex.
- Implement G6 (local/remote source tagging) and G7 (segmented Opus + append-only transcript + WAL/fsync, crash recovery).
- **Exit:** kill the app mid-meeting → valid recording + transcript survive; restart offers to finish the tail.

### Phase 2 — Prompt store & provider/budget layer
- Build G1 (versioned editable prompt store + editor UI + variable interpolation + seeded templates).
- Extend G4 (provider registry with budget caps, spend tracking, pre-run cost estimate, threshold warnings, cap-pause for cloud).
- **Exit:** user edits a prompt, picks a provider, sees a cost estimate, generates a summary; cloud pauses at the cap.

### Phase 3 — Long-transcript summarization & jobs
- Build G3 (resumable job queue) and G2 (map-reduce summarizer) on top of the Summary Engine.
- Validate on 1h, 10h, and a synthetic 100h transcript; confirm resume-after-crash near the failure point.
- **Exit:** a 100h transcript summarizes to a structured report locally (Ollama), resumable, within budget if cloud is used.

### Phase 4 — Meeting-end auto-detect, library, encryption, polish
- Build G5 (auto-stop + auto-trigger report), G9 (library + FTS search + playback + regenerate), G8 (encryption at rest), G10 (consent UX + markdown export).
- **Exit:** meeting ends → report auto-generates; library searchable; data encrypted at rest.

### Phase 5 — Optional / deferred
- G10 advanced exports (PDF/DOCX), optional TTS (Piper), and the mobile path (separate effort — Tauri 2 mobile or RN shell over the same Rust core).

**Sequencing rationale:** keep Meetily's working engine, validate the one irreversible guarantee (passive pass-through capture) immediately, then layer the product gaps in dependency order — prompt/provider before summarization, jobs before long summaries, end-detect/library/encryption last as they don't gate earlier demos.

---

## 6. Work breakdown by repo area

- **`frontend/` (Next.js):** prompt editor + version history; provider/budget settings with cost estimates; meeting library + FTS search + playback; regenerate-report flow; consent banner/indicator.
- **Rust core — Audio Engine:** source tagging, Opus segmentation, fsync checkpoints, meeting-end heuristics.
- **Rust core — Transcription Engine:** confirm `STTProvider` seam; persist append-only finals with source/confidence.
- **Rust core — Summary Engine:** map-reduce orchestration; prompt-version binding; cost accounting.
- **Rust core — Database:** schema migrations; FTS5; SQLCipher wrapping; jobs table + scheduler.
- **Rust core — new Job Queue module:** DAG scheduler, resume-on-startup, concurrency + budget guard.
- **Config/registry:** add `budget_cap`/spend; keys in OS keychain (not config files).
- **Build/CI:** dual-OS build, signing/notarization (mac) and installer (win), test matrix.

---

## 7. Testing strategy

- **Capture integration:** automated loopback test asserting bit-identical pass-through and non-empty dual-channel capture; manual matrix across Teams/Meet/Webex on mac + win.
- **Crash recovery:** fault-injection killing the app mid-meeting and mid-summarization; assert transcript/archive validity and job resume.
- **Summarizer scale:** unit tests on chunk-boundary logic; end-to-end on 1h/10h/100h synthetic transcripts; assert bounded memory and budget adherence.
- **Provider abstraction:** contract tests against the `STTProvider`/`LLMProvider` interfaces with a fake adapter; one local + one cloud adapter each.
- **Budget guard:** assert cloud jobs pause exactly at cap and local jobs never block.
- **Migration tests:** forward migrations on a populated Meetily DB; round-trip with SQLCipher.
- **Verification gate per phase:** no phase is "done" until its Exit criterion passes on both macOS and Windows.

---

## 8. Risks & mitigations

| Risk | Mitigation |
|---|---|
| **Upstream drift** — Meetily evolves fast (v0.4.0 just shipped), our fork diverges | Keep engine modules close to upstream; isolate new modules; track deltas in `UPSTREAM.md`; periodic rebase. |
| **PRO/community line** — diarization, auto-join, advanced export are PRO | Our v1 scope avoids them (me/others tagging + markdown only); revisit later as build-or-license. |
| **Browser-based Meet end-detection** — no process to watch for a tab | Use audio-silence timeout + optional browser-signal later; manual stop always available. |
| **100h compute time locally** — long background jobs | Throttled resumable jobs + progress UI; recommend cloud toggle for speed within budget. |
| **macOS permissions/notarization** — screen/audio capture entitlements | Handle permission prompts in onboarding; budget time for signing/notarization in Phase 0. |
| **Tauri mobile gap** vs brief's RN preference | Accept desktop-first; keep Rust core shell-agnostic so a future RN/Tauri-mobile shell reuses it. |
| **License hygiene** — MIT fork + bundled models | Track every dependency/model license (whisper.cpp MIT, llama.cpp MIT, Parakeet model license — verify); preserve attributions Meetily already lists. |

---

## 9. Definition of done (v1)

A forked Tauri app that, on macOS and Windows: passively captures a Teams/Meet/Webex meeting (user hears it unmodified), transcribes locally in real time with me/others attribution, stores audio + transcript locally and encrypted, auto-detects meeting end, and generates an editable, prompt-driven, version-reproducible report — using a local model by default and an optional cloud provider within a configurable budget cap — with a searchable library to revisit and regenerate past reports.

---

## 10. Immediate next steps

1. Fork Meetily and complete a clean dual-OS source build (Phase 0).
2. Confirm the four open design questions still pending: hardware floor (sets default model sizes), export formats for v1, Meet/browser end-detection approach, and consent-UX scope.
3. Start Phase 1 capture validation — it's the one guarantee we can't compromise and the cheapest place to fail fast.

---

## Sources

- [Meetily repo](https://github.com/Zackriya-Solutions/meetily) · [Meetily architecture doc](https://github.com/Zackriya-Solutions/meetily/blob/main/docs/architecture.md) · [Building from source](https://github.com/Zackriya-Solutions/meetily/blob/main/docs/BUILDING.md)
- [Tauri](https://tauri.app/) · [whisper.cpp](https://github.com/ggerganov/whisper.cpp) · [Ollama](https://github.com/ollama/ollama) · [Screenpipe](https://github.com/mediar-ai/screenpipe)
