# Phase 0 — Fork & Build Baseline

**Status:** Not started  
**Depends on:** Nothing (entry phase)  
**Blocks:** All subsequent phases

---

## Goal

Stand up the forked Meetily repo as our working codebase. Get a clean end-to-end build (capture → transcribe → summarize) running on both macOS (Apple Silicon) and Windows x64. Establish CI, branch strategy, and divergence tracking before any product code is written.

---

## Exit Criterion

**Both of the following must pass before Phase 0 is done:**

1. `make build` (or equivalent repo script) completes without error on macOS (Apple Silicon) and Windows x64.
2. A real meeting is recorded end-to-end: app captures mic + system audio → Whisper transcribes live → Ollama local model generates a summary. No cloud provider required.

---

## Scope

**In:**
- Fork Meetily CE from `github.com/Zackriya-Solutions/meetily` at a pinned commit (tag the pin).
- Verify build from source per Meetily's `docs/BUILDING.md`.
- Set up CI (GitHub Actions) for mac-arm64 and windows-x64: build + lint.
- Create `UPSTREAM.md` to track every module we diverge from.
- Document open design questions in `docs/open-questions.md`.

**Out:**
- No feature code. No schema changes. No new modules.
- Do not modify Meetily's Rust engine, Audio Engine, Transcription Engine, or frontend. Only build-system and fork scaffolding.

---

## Deliverables

### Repository structure after Phase 0

```
(repo root)
├── CLAUDE.md                  # already exists
├── UPSTREAM.md                # new — divergence log
├── docs/
│   ├── open-questions.md      # new — four open questions from design §12
│   ├── spec/                  # this directory
│   └── (existing docs)
├── .github/
│   └── workflows/
│       ├── build-mac.yml      # new
│       └── build-windows.yml  # new
└── (Meetily source tree — unchanged from upstream)
```

### `UPSTREAM.md`

Template entry per divergence:
```
## <module-name>
- Upstream commit: <sha>
- Our divergence: <what and why>
- Merge risk: low | medium | high
- Last synced: <date>
```

Initial state: empty (no divergences yet).

### `docs/open-questions.md`

Document the four open questions from design §12 that must be answered before Phase 1 ends:

1. **Hardware floor** — minimum CPU/RAM/GPU to target. Sets default Whisper model size (tiny/base/small/medium).
2. **Export formats** — markdown-only for v1, or also PDF/DOCX?
3. **Meet/browser end-detection** — silence + process heuristics acceptable, or do we want optional browser-extension signal?
4. **Consent UX scope** — in-meeting recording indicator in v1, or banner-only?

### CI workflows

`build-mac.yml`:
```yaml
on: [push, pull_request]
jobs:
  build-mac:
    runs-on: macos-14          # Apple Silicon
    steps:
      - uses: actions/checkout@v4
      - name: Install Rust
        uses: dtolnay/rust-toolchain@stable
      - name: Install Node/pnpm
        uses: pnpm/action-setup@v3
        with: { version: 9 }
      - name: Build
        run: <build command from BUILDING.md>
      - name: Lint (rustfmt + clippy)
        run: cargo fmt --check && cargo clippy -- -D warnings
      - name: Lint (eslint)
        run: cd frontend && pnpm lint
```

`build-windows.yml`: same structure, `runs-on: windows-latest`, add any Windows-specific deps from BUILDING.md.

---

## Implementation steps

1. **Fork** Meetily on GitHub. Pin to the current latest release tag. Record the pinned commit SHA in `UPSTREAM.md` header.
2. **Clone** the fork locally. Confirm the directory matches Meetily's documented layout.
3. **macOS build:** follow `docs/BUILDING.md` exactly. Capture every `brew install`, `cargo`, `pnpm` command that was needed and not in the doc — add them to `UPSTREAM.md` as a "build notes" section.
4. **Windows build:** same — follow BUILDING.md on Windows x64. Document gaps.
5. **Smoke test (manual):** open the app, start recording (mic only or system audio), speak for 30 seconds, stop, verify transcript appears, verify summary generates with a local Ollama model.
6. **Create `UPSTREAM.md`** with the pinned commit SHA and empty divergence table.
7. **Create `docs/open-questions.md`** with the four questions.
8. **Add CI workflows.** Push to a branch, verify both workflows go green.
9. **Tag** the baseline commit as `v0-baseline` in the fork.

---

## Test plan

| Test | How | Pass condition |
|---|---|---|
| macOS build | `<build script>` on Apple Silicon runner | Zero errors, binary produced |
| Windows build | `<build script>` on windows-latest runner | Zero errors, binary produced |
| macOS smoke | Manual: record 30s, stop, check transcript + summary | Transcript non-empty, summary non-empty, no crash |
| Windows smoke | Same manual test on Windows x64 | Same |
| Lint | CI on every push | rustfmt + clippy + eslint all green |

---

## Risks

| Risk | Mitigation |
|---|---|
| Meetily's BUILDING.md is incomplete | Document every gap; add to UPSTREAM.md build-notes |
| Missing macOS permissions (mic, screen recording) | Grant during smoke test; note in build-notes for CI runners |
| Ollama not present on CI runner | CI builds only (no smoke); smoke test is manual |
| Windows build requires Visual Studio version mismatch | Pin VS version in workflow; document in build-notes |
