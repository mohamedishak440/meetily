# Meetily Upstream Reference

Detailed Meetily CE internals for reference during fork development. Pinned to v0.4.0 (commit `0281737`). Check `UPSTREAM.md` for divergences.

## Audio module layout

```
frontend/src-tauri/src/audio/
├── devices/
│   ├── discovery.rs           # list_audio_devices, trigger_audio_permission
│   ├── microphone.rs          # default_input_device
│   ├── speakers.rs            # default_output_device
│   ├── configuration.rs       # AudioDevice types
│   └── platform/
│       ├── windows.rs         # WASAPI (~200 lines)
│       ├── macos.rs           # ScreenCaptureKit
│       └── linux.rs           # ALSA/PulseAudio
├── capture/
│   ├── microphone.rs
│   ├── system.rs
│   └── core_audio.rs          # macOS ScreenCaptureKit integration
├── pipeline.rs                # AudioMixerRingBuffer, ProfessionalAudioMixer, AudioPipelineManager
├── recording_manager.rs
├── recording_commands.rs      # Tauri command interface
└── recording_saver.rs
```

## Audio pipeline internals

- `AudioMixerRingBuffer`: accumulates mic + system samples asynchronously; aligns into 50ms windows
- `ProfessionalAudioMixer`: RMS-based ducking prevents system audio from drowning mic
- `AudioPipelineManager`: orchestrates VAD, mixing, distribution to recording and transcription paths
- `AudioMetricsBatcher`: batches metrics to reduce emit overhead
- `AudioBufferPool`: pre-allocated buffers for the recording path

## Whisper model paths

| Context | Path |
|---|---|
| Development | `frontend/models/` |
| macOS production | `~/Library/Application Support/Meetily/models/` |
| Windows production | `%APPDATA%\Meetily\models\` |

## GPU acceleration

| Platform | Backend | Enable |
|---|---|---|
| macOS | Metal + CoreML | automatic |
| Windows/Linux NVIDIA | CUDA | `--features cuda` |
| Windows/Linux AMD/Intel | Vulkan | `--features vulkan` |
| Any | CPU fallback | `--features cpu` (or no GPU feature) |

See `docs/GPU_ACCELERATION.md` for full setup guide.

## Frontend state management

`SidebarProvider` (`frontend/src/components/Sidebar/SidebarProvider.tsx`) holds global state:
- meetings list, current meeting, recording status, transcript updates, summary state
- Communicates with Rust core via Tauri commands (invoke) and events (listen)
- Pattern: Tauri command → Rust state update → emit event → frontend listener → React context update

## LLM providers in upstream

Upstream supports: Ollama (local), Claude, Groq, OpenRouter. Provider config in `frontend/src-tauri/src/` (exact path varies by version).

## Build notes (gaps vs BUILDING.md)

Document any missing steps found during Phase 0 build here so future sessions don't repeat the investigation.

| Step | OS | Gap | Fix |
|---|---|---|---|
| (fill in during Phase 0 build) | | | |
