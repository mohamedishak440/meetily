# Reuse Research — Open-Source Apps & Libraries for the AI Meeting Assistant

**Date:** 2026-06-07 · Companion to `ai-meeting-assistant-design.md`
**Question 1:** Is there a reusable open-source application?
**Question 2:** Are there frameworks/libraries to reuse for quick development?

---

## TL;DR

Yes to both — and strongly.

1. **There is an open-source app that already implements ~90% of this design: [Meetily](https://github.com/Zackriya-Solutions/meetily)** (MIT, 12.6k★, macOS/Windows/Linux). It does local mic + system-audio capture, real-time Whisper/Parakeet transcription, local Ollama summarization with pluggable cloud providers, editable summaries, and local-only storage. It validates every major decision in our design — and can be either forked or used as a reference implementation.

2. **The library stack is mature and mostly MIT/Apache-licensed.** The hardest part of our design (cross-platform system-audio loopback) is solved by `cpal` + the `screencapturekit` crate; transcription by `whisper.cpp`; summarization by `llama.cpp`/Ollama; the desktop shell by Tauri (or React Native if we hold to the brief). You can assemble an MVP from these without writing capture, STT, or LLM inference from scratch.

The one caveat worth flagging up front: the strongest reusable app (Meetily) is built on **Tauri**, not React Native — which trades against the brief's RN preference. See §4.

---

## 1. Question 1 — Reusable open-source applications

### Primary match: Meetily (Zackriya-Solutions/meetily)

This is essentially our design, already built and shipping (v0.4.0, Jun 2026).

| Our requirement | Meetily |
|---|---|
| Passive local capture, mic + system audio | Yes — "professional audio mixing," captures mic + system audio simultaneously with ducking/clipping prevention |
| macOS + Windows desktop | Yes (also Linux) |
| Real-time transcription, local | Yes — Whisper.cpp + NVIDIA Parakeet, on-device |
| Hybrid AI (local default, cloud opt-in) | Yes — Ollama local default; Claude, Groq, OpenRouter, OpenAI, or any OpenAI-compatible endpoint |
| Editable / prompt-driven summaries | Yes (community); custom summary templates are a PRO feature |
| Local-only storage | Yes — recordings, transcripts, models all stored locally |
| Pluggable vendors | Yes — provider abstraction over STT and LLM |
| GPU acceleration | Yes — Metal/CoreML (mac), CUDA/Vulkan (Win/Linux) |
| Modular core | Yes — Rust backend + Next.js frontend over Tauri |
| License | **MIT** — free to fork and modify |

**Architecture:** single self-contained Tauri app; Rust backend for core logic, Next.js frontend for UI. It openly reuses code from `whisper.cpp`, [Screenpipe](https://github.com/mediar-ai/screenpipe), and the `transcribe-rs` crate.

**Licensing nuance to plan around:** the Community Edition is MIT and "free forever," but several capabilities we may want are **PRO (paid, separate codebase)**: speaker diarization, auto-detect/join meetings, advanced PDF/DOCX exports, calendar integration. Our design only needs "me vs. others" attribution and markdown export, so the community edition covers the v1 scope — but don't assume diarization/auto-join come for free.

### Secondary / reference options

- **[Screenpipe](https://github.com/mediar-ai/screenpipe)** — Rust, continuous local screen+audio capture with a plugin pipeline. Meetily borrows from it. Useful as a library-grade capture+pipeline reference even if not adopted wholesale.
- **[Ghostmeet](https://github.com/Higangssh/ghostmeet)** — open-source, captures any browser-tab audio (Meet/Zoom/Teams) and transcribes with Whisper. It's a **Chrome-extension** model, not a desktop app — a different capture approach (tab audio vs. system loopback), useful only if browser-tab capture is ever preferred.
- **[WhisperDesk](https://pvas-development.github.io/whisperdesk/)** — local Whisper transcription desktop app (file/media oriented). Good reference for the transcription+review UI, weaker on live-meeting capture.
- **[Whispering](https://github.com/) (Epicenter)** — local-first transcription app, bring-your-own-key to local (Speaches) or cloud (Whisper/Groq/ElevenLabs). Good reference for the pluggable-provider UX.
- **[AI-Powered-Meeting-Summarizer](https://github.com/AlexisBalayre/AI-Powered-Meeting-Summarizer)** — minimal Gradio app: whisper.cpp → Ollama summary. Tiny, readable reference for the transcribe→summarize loop.

**Recommendation for Q1:** treat **Meetily as either the fork base or the reference architecture.** Forking gets you to a working product fastest; using it as a reference (and assembling from libraries) gives full control over the RN shell and the prompt/vendor abstractions we specified. Decision hinges on the RN-vs-Tauri trade-off (§4).

---

## 2. Question 2 — Reusable frameworks & libraries (mapped to our design)

Mapped to the component layers from `ai-meeting-assistant-design.md`.

### 2.1 Capture layer (the hard part — solved)

| Library | What it gives us | License | Notes |
|---|---|---|---|
| **[cpal](https://github.com/RustAudio/cpal)** | Cross-platform audio I/O in Rust; **WASAPI loopback** on Windows, CoreAudio on macOS | Apache-2.0 / MIT | The backbone of our `AudioSource` interface. Loopback supported on Windows; macOS loopback via CoreAudio on 14.6+ and ScreenCaptureKit. |
| **[screencapturekit (crate)](https://crates.io/crates/screencapturekit)** | Safe Rust bindings to Apple ScreenCaptureKit; **system audio + mic capture** on macOS 13+ | MIT | Our macOS system-mix path. Pairs with cpal for mic. |
| **[huxinhai/audio-capture](https://github.com/huxinhai/audio-capture)** | Ready-made cross-platform loopback recorder (WASAPI + ScreenCaptureKit) | OSS | Drop-in reference / starting point for the capture engine. |
| **Screenpipe capture modules** | Production-tested capture + pipeline in Rust | MIT | Reuse patterns for tee/fan-out and back-pressure. |

This directly de-risks the highest-risk part of our build plan (Phase 1). We do **not** need to write WASAPI/CoreAudio FFI by hand.

### 2.2 Transcription (STT)

| Library | Role | License |
|---|---|---|
| **[whisper.cpp](https://github.com/ggerganov/whisper.cpp)** | Local streaming + batch STT; Metal/CoreML/CUDA/Vulkan accel | MIT |
| **[transcribe-rs](https://crates.io/crates/transcribe-rs)** | Rust wrapper used by Meetily; clean STT-provider seam | OSS |
| **NVIDIA Parakeet (ONNX)** | Faster alternative ASR model (Meetily ships it) | Model license — check |
| **Deepgram / AssemblyAI SDKs** | Cloud STT adapters behind the toggle | Commercial API |

Maps 1:1 onto our `STTProvider` interface: whisper.cpp = local default, cloud SDKs = opt-in adapters.

### 2.3 Summarization (LLM)

| Library | Role | License |
|---|---|---|
| **[llama.cpp](https://github.com/ggerganov/llama.cpp)** | Local LLM inference engine | MIT |
| **[Ollama](https://github.com/ollama/ollama)** | Local model server/manager (Gemma, Llama, Mistral) — easiest local-LLM integration | MIT |
| **OpenAI / Anthropic / Groq / OpenRouter SDKs** | Cloud LLM adapters; OpenAI-compatible endpoint = one adapter covers many | Commercial API |

Maps onto our `LLMProvider` interface. Ollama is the fastest route to a working local summarizer; our map-reduce logic sits on top of it.

### 2.4 TTS (optional)

| Library | Role | License |
|---|---|---|
| **[Piper](https://github.com/rhasspy/piper)** | Fast local neural TTS | MIT |
| **OS native TTS** | Zero-dependency fallback | — |
| **ElevenLabs / OpenAI TTS** | Cloud adapters (off by default per budget) | Commercial |

### 2.5 Desktop shell & core

| Option | Fit | Trade-off |
|---|---|---|
| **React Native** (`react-native-windows` + `react-native-macos`) | Matches the brief; reuses RN skills; clean path to deferred mobile | Native audio/ML must live in native modules or a sidecar; less battle-tested for this exact use case than Tauri |
| **Tauri** (Rust core + web UI) | What Meetily uses; smallest gap to a working app; Rust core already = our engine | Diverges from the RN preference; mobile story (Tauri 2 mobile) less mature than RN |

Either way, the **Rust headless core** we specified is reusable across both shells — which is exactly why the design put the engine behind an IPC seam.

### 2.6 Storage & infra

| Library | Role | License |
|---|---|---|
| **SQLite + FTS5** | Transcript/metadata store + search | Public domain |
| **SQLCipher** | Encryption at rest | BSD |
| **Opus (libopus) / Symphonia** | Compact speech archive + decode | BSD / MPL-2.0 |
| OS keychain (Keychain / DPAPI) | Secret storage | — |

---

## 3. Build-vs-reuse recommendation

Three viable paths, fastest to most-control:

**Path A — Fork Meetily (fastest).** Start from a working MIT app; adapt the prompt store to our editable/versioned spec, confirm the vendor registry matches our `$budget_cap` requirement, and accept Tauri instead of RN. Best if time-to-MVP dominates. Watch the PRO/community line for diarization/auto-join.

**Path B — Assemble from libraries with our own RN shell (balanced).** Use cpal + screencapturekit + whisper.cpp + Ollama behind the Rust core we designed, with a React Native shell. Honors the brief and gives full control over the pluggable abstractions, while still reusing every hard component. This is the path our design document already lays out — the research confirms each layer has a mature library.

**Path C — Reference-only (most control, slowest).** Read Meetily/Screenpipe for capture patterns, build clean. Only worth it if licensing or architecture constraints rule out reuse.

**Suggested:** **Path B**, with Meetily/Screenpipe as live references for the capture layer specifically — that's where reading working code saves the most time. If the RN preference is soft, **Path A** is the quickest route to something usable.

---

## 4. Key caveat — RN vs Tauri

Our design and the brief prefer **React Native**; the strongest reusable app (Meetily) and most of the reference code are **Tauri/Rust + web**. The good news is the expensive, reusable parts (capture, STT, LLM) are shell-agnostic Rust/C++ libraries — they plug into either shell. So the RN preference costs us the ability to *fork Meetily wholesale*, but **not** the ability to reuse the library stack. If mobile-later (the deferred iOS/Android target) is a real priority, RN remains the stronger long-term shell; if desktop-now speed dominates, Tauri/Meetily wins.

---

## Sources

- [Meetily — github.com/Zackriya-Solutions/meetily](https://github.com/Zackriya-Solutions/meetily)
- [Self-Hosted Meeting Transcription: 10 Open Source Tools Compared (2026) — meetily.ai](https://meetily.ai/blog/best-self-hosted-meeting-transcription-tools-2026)
- [Local Meeting Notes with Whisper + Ollama — dev.to/zackriya](https://dev.to/zackriya/local-meeting-notes-with-whisper-transcription-ollama-summaries-gemma3n-llama-mistral--2i3n)
- [Ghostmeet — github.com/Higangssh/ghostmeet](https://github.com/Higangssh/ghostmeet)
- [WhisperDesk — pvas-development.github.io/whisperdesk](https://pvas-development.github.io/whisperdesk/)
- [Whispering (open-source local-first) — slator.com](https://slator.com/whispering-open%E2%80%91source-local%E2%80%91first-transcription-app/)
- [AI-Powered-Meeting-Summarizer — github.com/AlexisBalayre](https://github.com/AlexisBalayre/AI-Powered-Meeting-Summarizer)
- [cpal — github.com/RustAudio/cpal](https://github.com/RustAudio/cpal) · [ScreenCaptureKit loopback issue #876](https://github.com/RustAudio/cpal/issues/876)
- [screencapturekit crate — crates.io](https://crates.io/crates/screencapturekit)
- [audio-capture — github.com/huxinhai/audio-capture](https://github.com/huxinhai/audio-capture)
- [Screenpipe — github.com/mediar-ai/screenpipe](https://github.com/mediar-ai/screenpipe)
