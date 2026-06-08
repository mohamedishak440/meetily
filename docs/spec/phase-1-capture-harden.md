# Phase 1 — Validate & Harden Capture

**Status:** Not started  
**Depends on:** Phase 0 (clean fork build on both OSes)  
**Blocks:** Phase 2 (transcription depends on reliable capture)  
**Gaps addressed:** G6 (local/remote source tagging), G7 (crash-safe transcript + archive)

---

## Goal

Prove that the capture path is correct, crash-safe, and non-intrusive. The user must hear the meeting bit-identically throughout. A crashed app mid-meeting must leave a valid transcript + audio archive. This is the one irreversible guarantee — validate it before any product work builds on top.

---

## Exit Criterion

**All of the following must pass on macOS AND Windows before Phase 1 is done:**

1. **Pass-through:** start capture, join a real Teams/Meet/Webex call, play audio — the user hears it through their normal speakers, unmodified, throughout recording. Verified manually on each platform.
2. **Dual-channel capture:** recording contains two non-empty audio channels: `remote` (system loopback) and `local` (mic). Verified by automated test.
3. **Source tagging:** every `transcript_segment` row persisted to SQLite carries `source = 'local'` or `source = 'remote'`. No null sources.
4. **Crash safety:** kill the app process mid-meeting (SIGKILL / Task Manager). On restart: the meeting is marked `recoverable`; saved audio segments are valid Opus files; transcript rows exist up to the last flush. App offers to transcribe any un-transcribed audio tail.
5. **Matrix:** manual capture test passes on Teams, Google Meet (Chrome), and Webex on both macOS and Windows.

---

## Scope

**In:**
- G6: source-tag all transcript segments (`local` vs `remote`) at the point of capture. Propagate through Audio Router → Transcription Engine → DB.
- G7: segmented Opus archive with per-segment fsync; append-only transcript segments with WAL; crash-recovery detection on startup; UI offer to finish un-transcribed tail.
- Confirm and document Meetily's existing loopback mechanism (Core Audio process tap / ScreenCaptureKit on macOS; WASAPI loopback on Windows). Do not replace it unless broken.
- Automated test: assert dual-channel, non-empty capture output.
- Manual test matrix: three conferencing apps × two OSes.

**Out:**
- No summarization, no prompt store, no job queue (all later phases).
- No UI changes beyond the crash-recovery offer prompt.
- No encryption (Phase 4).
- Do not alter Meetily's capture code beyond adding source tags and crash-safe write logic.

---

## Architecture: what changes

### Audio capture source tags (G6)

Meetily's Audio Engine already captures mic and system audio. We add a `source` field to every audio frame struct flowing through the pipeline.

```rust
// src-tauri/src/audio/types.rs (new or adapt)
pub enum AudioSource {
    Local,   // mic — "me"
    Remote,  // system loopback — "others"
}

pub struct AudioFrame {
    pub samples: Vec<f32>,
    pub timestamp_ms: u64,
    pub source: AudioSource,
    // existing fields stay
}
```

The Audio Router (tee/fan-out) preserves the `source` tag on frames sent to both the Archive branch and the STT branch. The Transcription Engine attaches the source to each `TranscriptEvent` it emits.

### Transcript segment schema (G7 + G6)

Add columns to Meetily's existing transcript table (additive migration — do not drop or alter existing columns):

```sql
-- Migration 001_add_source_and_crash_safety.sql
ALTER TABLE transcript_segments ADD COLUMN source TEXT CHECK(source IN ('local','remote')) NOT NULL DEFAULT 'remote';
ALTER TABLE transcript_segments ADD COLUMN is_final INTEGER NOT NULL DEFAULT 0;
ALTER TABLE transcript_segments ADD COLUMN confidence REAL;
ALTER TABLE transcript_segments ADD COLUMN provider TEXT;

-- Enable WAL mode (idempotent)
PRAGMA journal_mode=WAL;

ALTER TABLE meetings ADD COLUMN status TEXT NOT NULL DEFAULT 'done'
    CHECK(status IN ('recording','processing','done','failed','recoverable'));
```

> Migration must be additive. Test it runs clean on a populated Meetily database.

### Recording segments table (new)

```sql
-- Migration 002_recording_segments.sql
CREATE TABLE IF NOT EXISTS recording_segments (
    id          INTEGER PRIMARY KEY,
    meeting_id  INTEGER NOT NULL REFERENCES meetings(id),
    seq         INTEGER NOT NULL,
    file_path   TEXT NOT NULL,
    codec       TEXT NOT NULL DEFAULT 'opus',
    start_ms    INTEGER NOT NULL,
    end_ms      INTEGER,
    bytes       INTEGER,
    UNIQUE(meeting_id, seq)
);
```

### Opus segmentation (G7)

Archive Writer splits output into ~10-minute Opus segments. After each segment is fully written and fsynced, it inserts a row into `recording_segments`. Partial (current) segment tracked in memory; on crash it is not inserted — the DB shows a gap, and on recovery the app scans the audio directory for orphan files.

```
audio/
  <meeting-id>/
    seg_0001.opus    # 10 min, complete, has DB row
    seg_0002.opus    # current, may be partial on crash — no DB row yet
```

Fsync sequence per segment:
1. Write all encoded frames to file.
2. `fsync(fd)`.
3. Insert `recording_segments` row inside a transaction.
4. Commit transaction.
5. Open new segment file.

### Crash recovery (G7)

On startup:

```rust
fn recover_incomplete_meetings(db: &Db, audio_dir: &Path) {
    // 1. Find meetings with status = 'recording'
    // 2. For each: scan audio_dir/<meeting_id>/ for .opus files not in recording_segments
    // 3. Mark meeting status = 'recoverable'
    // 4. Emit event to UI: meetings.recoverable { meetingId, untranscribedSecs }
}
```

UI (minimal addition): when `meetings.recoverable` event received, show a banner:
> "Meeting [title] was interrupted. Transcribe the remaining audio?"  [Transcribe] [Dismiss]

Pressing Transcribe enqueues a `transcribe` job (job type used in Phase 3; for now just invoke the STT path directly on the recovered segment files).

---

## Implementation steps

1. **Read** Meetily's Audio Engine source (`src-tauri/src/audio/`) and identify exactly where mic frames and loopback frames originate. Confirm the tee/fan-out exists.
2. **Add `AudioSource` enum** to the frame type. Patch the macOS capture path to tag loopback frames `Remote` and mic frames `Local`. Patch the Windows capture path the same way.
3. **Propagate source tag** through Audio Router → STT branch → `TranscriptEvent` → `transcript_segments` column.
4. **Write migration 001** (source, is_final, confidence, provider on transcript_segments; WAL; meetings.status). Run it against a copy of a real Meetily DB to verify it's clean.
5. **Write migration 002** (recording_segments table).
6. **Adapt Archive Writer** to segment output into ~10-min Opus files. Implement fsync-then-insert sequence.
7. **Implement `recover_incomplete_meetings`** called at engine startup.
8. **Add UI banner** for recoverable meetings (minimal — one banner, two buttons, no new screens).
9. **Write automated capture test** (see test plan).
10. **Manual matrix test** (see test plan) on both OSes.
11. **Update `UPSTREAM.md`** with every modified Meetily module.

---

## Test plan

### Automated tests

#### `test_dual_channel_capture` (Rust integration test)

```rust
#[test]
fn test_dual_channel_capture() {
    // Start the audio engine with a fake audio source emitting:
    //   - Local frames: sine wave at 440 Hz
    //   - Remote frames: sine wave at 880 Hz
    // Capture for 5 seconds.
    // Assert:
    //   - At least one frame with source = Local received
    //   - At least one frame with source = Remote received
    //   - No frame with source = None
}
```

#### `test_source_tag_persisted` (DB integration test)

```rust
#[test]
fn test_source_tag_persisted() {
    // Replay a canned 10-second dual-source capture into the transcription pipeline.
    // Assert all transcript_segment rows have source IN ('local', 'remote').
    // Assert no source IS NULL.
}
```

#### `test_segment_fsync_and_recovery` (integration test)

```rust
#[test]
fn test_segment_fsync_and_recovery() {
    // Start a meeting, write 2 complete segments (20 min simulated, fast clock).
    // Simulate crash: drop the engine without calling stop().
    // Restart engine.
    // Assert meetings.status = 'recoverable' for the interrupted meeting.
    // Assert recording_segments has rows for segments 1 and 2 (complete ones).
    // Assert seg_0001.opus and seg_0002.opus are valid Opus files (can be decoded).
}
```

#### `test_migration_on_populated_db` (migration test)

```rust
#[test]
fn test_migration_001_on_meetily_db() {
    // Load a snapshot of a real Meetily SQLite DB (check in a test fixture).
    // Run migration 001.
    // Assert all existing rows still present.
    // Assert new columns exist with correct defaults.
    // Assert journal_mode = WAL.
}
```

### Manual test matrix

| Platform | App | Pass-through | Dual-channel non-empty | Source tags correct |
|---|---|---|---|---|
| macOS (Apple Silicon) | Teams | | | |
| macOS (Apple Silicon) | Google Meet (Chrome) | | | |
| macOS (Apple Silicon) | Webex | | | |
| Windows x64 | Teams | | | |
| Windows x64 | Google Meet (Chrome) | | | |
| Windows x64 | Webex | | | |

**Pass-through verification:** join a call, play audio, confirm you hear it normally throughout the entire recording session. No echo, no interruption, no volume change.

### Crash recovery test (manual, both OSes)

1. Start recording a meeting.
2. Wait until at least one complete 10-min segment is on disk.
3. Kill the process: `kill -9 <pid>` (macOS) / Task Manager > End Process (Windows).
4. Restart the app.
5. Verify: banner appears offering to transcribe the interrupted meeting.
6. Verify: `meetings.status = 'recoverable'` in SQLite (use DB browser).
7. Verify: `seg_0001.opus` decodes without error (`ffprobe` or `opusdec`).
8. Click Transcribe. Verify transcript appears.

---

## Risks

| Risk | Mitigation |
|---|---|
| Core Audio process tap requires macOS 14.2+ | Detect OS version; fall back to ScreenCaptureKit (macOS 13+) or skip system audio on ≤12 with a UI warning |
| WASAPI loopback not available on Windows Server or older Windows 10 builds | Document minimum: Windows 10 2004+ (build 19041) |
| Meetily's frame types are not easily extended | If the struct is not extensible, wrap it in a newtype with source tag instead of modifying |
| Per-segment fsync causes noticeable write latency | Write on a background thread; ring buffer absorbs bursts; benchmark on a slow HDD |
| `recover_incomplete_meetings` is slow on many orphan segments | Limit scan to meetings in 'recording' state; run async at startup |
