# UPSTREAM.md — Divergence Log

Fork of [Zackriya-Solutions/meetily](https://github.com/Zackriya-Solutions/meetily).  
**Pinned upstream commit:** `0281737d87d26352fb0adc78c8c0975f691b23d1` (tag `v0.4.0`, 2026-06-05)  
**Our branch:** `our-baseline` (diverges from `v0.4.0`)

## How to read this file

Each section below tracks one module or area where our fork diverges from upstream. Format:

```
## <area>
- Upstream state at pin: <what upstream does>
- Our divergence: <what we changed and why>
- Merge risk: low | medium | high
- Last synced: <date>
```

When upstream releases a new version, diff each listed area against the new tag and decide whether to cherry-pick or re-diverge.

---

## Phase 0 divergences

### ffmpeg.rs clippy fixes

- **Upstream state at pin:** `frontend/src-tauri/build/ffmpeg.rs` had 2 clippy warnings: unused `std::io::Read` import; `Iterator::last` on a `DoubleEndedIterator`.
- **Our divergence:** Removed unused import; changed `.last()` → `.next_back()`. Upstream formatting preserved.
- **Merge risk:** low (build script only)
- **Last synced:** 2026-06-08

---

## Known conflicts to track (not yet diverged — flagged for Phase 1)

### macOS system audio: BlackHole vs ScreenCaptureKit process-tap

- **Upstream state at pin:** `audio/devices/platform/macos.rs` + `audio/capture/core_audio.rs` use ScreenCaptureKit. Upstream CLAUDE.md mentions BlackHole but the code uses SCK. To be verified during Phase 0 smoke test.
- **Our divergence (planned, Phase 1):** Replace any reliance on virtual audio drivers with passive ScreenCaptureKit process-tap (macOS 13+). No BlackHole install required. See design §2.1 and G6/G7.
- **Merge risk:** high (capture core is central)
- **Last synced:** 2026-06-08

### CLAUDE.md

- **Upstream state at pin:** Meetily's own 17KB CLAUDE.md with upstream dev conventions.
- **Our divergence:** Replaced with our project-specific CLAUDE.md. Useful upstream content extracted to `docs/meetily-reference.md`.
- **Merge risk:** low (documentation only)
- **Last synced:** 2026-06-08

---

## Build notes (gaps vs `docs/BUILDING.md`)

Fill in during Phase 0 build — any `brew install`, dependency, or step missing from the upstream build doc:

| Step | OS | Gap | Fix applied |
|---|---|---|---|
| (fill during Phase 0 smoke test) | | | |
