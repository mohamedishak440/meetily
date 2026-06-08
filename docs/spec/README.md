# Phase Specs — AI Meeting Assistant

Each spec is independently implementable and testable. Phases are ordered by dependency; start Phase N only when Phase N-1's exit criterion passes on both macOS and Windows.

| Phase | Spec | Gaps | Status |
|---|---|---|---|
| 0 — Fork & baseline | [phase-0-fork-baseline.md](phase-0-fork-baseline.md) | — | Not started |
| 1 — Capture & harden | [phase-1-capture-harden.md](phase-1-capture-harden.md) | G6, G7 | Not started |
| 2 — Prompt & provider | [phase-2-prompt-provider.md](phase-2-prompt-provider.md) | G1, G4 | Not started |
| 3 — Summarization & jobs | [phase-3-summarization-jobs.md](phase-3-summarization-jobs.md) | G2, G3 | Not started |
| 4 — Polish (v1 complete) | [phase-4-polish.md](phase-4-polish.md) | G5, G8, G9, G10 | Not started |
| 5 — Deferred | [phase-5-deferred.md](phase-5-deferred.md) | TTS, mobile, export, calendar | Deferred |

## DB migration sequence

| Migration | Phase | Content |
|---|---|---|
| 001 | 1 | Source tag + WAL + meetings.status on transcript_segments |
| 002 | 1 | recording_segments table |
| 003 | 2 | prompts table |
| 004 | 2 | summaries extensions (prompt_id, tokens, cost) |
| 005 | 2 | providers extensions (budget_cap, spent_this_month) |
| 006 | 3 | jobs table |
| 007 | 4 | FTS5 virtual table + triggers + app_settings |

All migrations are additive — never alter or drop Meetily's existing columns.

## Gap index

| Gap | Description | Phase |
|---|---|---|
| G1 | Editable versioned prompt store | 2 |
| G2 | Map-reduce summarization (1h→100h) | 3 |
| G3 | Resumable job queue | 3 |
| G4 | Budget caps & cost estimation | 2 |
| G5 | Meeting-end auto-detect | 4 |
| G6 | "Me vs. others" source tagging | 1 |
| G7 | Crash-safe transcript + archive | 1 |
| G8 | Encryption at rest (SQLCipher + audio + keychain) | 4 |
| G9 | Library, search, playback, regenerate | 4 |
| G10 | Consent UX + markdown export + delete | 4 |
