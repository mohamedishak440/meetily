# Phase 4 — Meeting-End Auto-Detect, Library, Encryption & Polish

**Status:** Not started  
**Depends on:** Phase 3 (job queue + summarizer must be working)  
**Blocks:** Nothing (this is the v1 completion phase)  
**Gaps addressed:** G5 (auto meeting-end detect), G8 (encryption at rest), G9 (library + search + playback + regenerate), G10 (consent UX + markdown export)

---

## Goal

Complete the v1 product. A meeting ends → report auto-generates. Past meetings are searchable. All data is encrypted at rest. Consent is surfaced. Users can export reports as markdown and delete any meeting completely.

---

## Exit Criterion

**All of the following must pass on macOS AND Windows before Phase 4 (and v1) is done:**

1. **Auto-detect:** start a real Teams or Google Meet call. Stop it (close the app or leave the call). Within 90 seconds of the audio going silent / process releasing the device, the app auto-generates a report without user intervention. A manual "Stop" button always works as well.
2. **Library + search:** the meeting library lists at least 3 past meetings. Full-text search on transcript text returns the correct meetings for a query. Results are rendered in < 500 ms on a 100-meeting DB.
3. **Playback:** click a meeting in the library → audio plays (segmented Opus). Seeking by clicking a transcript line jumps to the correct audio timestamp (± 2 seconds).
4. **Regenerate:** select a past meeting, change the prompt or provider, click Regenerate → a new `summaries` row is created with the new `prompt_id`/`prompt_version`/`model`.
5. **Encryption:** all `transcript_segments`, `summaries`, and audio files are encrypted at rest (SQLCipher + encrypted Opus files). On a fresh install, the app unlocks the DB on first launch using a key from the OS keychain.
6. **Delete:** delete a meeting → its audio files, transcript rows, summary rows, and job rows are all removed. Verify with a filesystem + DB check.
7. **Consent banner:** on first launch, a consent reminder is shown explaining that the app records audio. User acknowledges. It does not appear again. An optional in-meeting "recording" indicator is shown during active capture.
8. **Markdown export:** from a meeting's report view, "Export as Markdown" downloads a `.md` file containing the report body. File opens correctly in a text editor.

---

## Scope

**In:**
- G5: auto meeting-end heuristics (silence timeout + conferencing process/device release). User stop always available.
- G8: SQLCipher wrapping of the main DB; audio file encryption (AES-256-GCM); key in OS keychain (Keychain on macOS, DPAPI on Windows); app-unlock on launch.
- G9: meeting library (list, filter, sort), FTS5 transcript search, audio playback with seek-by-transcript-line, and regenerate-report flow.
- G10: first-launch consent banner, optional in-meeting recording indicator, markdown export, per-meeting delete (audio + transcript + summaries + jobs), and a global "Delete all data" option in Settings.

**Out:**
- PDF/DOCX export (Phase 5 / deferred).
- TTS playback of summaries (Phase 5 / deferred).
- Mobile (never in v1 scope).
- Diarization beyond me/others (PRO feature — not available).
- Cloud sync, multi-user, calendar integration (all out of v1 scope).

---

## G5 — Meeting-end auto-detect

### Heuristics (evaluated in order)

1. **User stop** — always immediately stops capture and triggers report generation.
2. **Process exit** — on macOS/Windows, watch for known conferencing process names to exit:
   - `Microsoft Teams`, `Webex`, `zoom.us`, `Google Chrome` (if Chrome is the active loopback source).
   - Implemented via polling `sysinfo` crate every 5 seconds. Not relied on for browsers (Meet runs in a tab).
3. **Audio device release** — loopback source reports the audio device was closed or its process released it. Signals immediate end.
4. **Silence timeout** — 90 seconds of `remote` channel audio below −60 dBFS triggers auto-stop. Configurable in settings (minimum 30s). User is shown a 10-second countdown with a "Keep recording" button before the stop fires.

### Auto-trigger pipeline

```
meeting_end_detected()
  → capture.stop()
  → mark meeting status = 'processing'
  → enqueue report_generate job (using default prompt + default provider)
  → emit capture.stopped event to UI
  → UI shows "Meeting ended — generating report..."
```

If report generation fails, mark meeting `status = 'failed'` and show a retry button in the library.

### Implementation

```rust
pub struct EndDetector {
    silence_threshold_db: f32,   // default -60
    silence_timeout_sec: u64,    // default 90
    watch_processes: Vec<String>,
}

impl EndDetector {
    fn on_audio_frame(&mut self, frame: &AudioFrame) {
        let db = rms_to_db(frame.rms());
        if db < self.silence_threshold_db {
            self.silent_since = self.silent_since.or(Some(Instant::now()));
        } else {
            self.silent_since = None;
        }
        if let Some(t) = self.silent_since {
            if t.elapsed().as_secs() >= self.silence_timeout_sec {
                self.trigger_end();
            }
        }
    }
}
```

The detector runs in the Audio Router thread — it receives frames from the `remote` channel only (silence of the mic alone should not trigger end).

---

## G8 — Encryption at rest

### SQLCipher

Replace the SQLite connection string with SQLCipher:

```rust
// src-tauri/src/db/connection.rs
fn open_encrypted(path: &Path, key: &str) -> Connection {
    let conn = Connection::open(path)?;
    conn.execute_batch(&format!("PRAGMA key = '{}';", key))?;
    conn
}
```

Key is fetched from the OS keychain on startup. If no key exists (first run), generate a random 32-byte key, store in keychain, open DB with it.

```rust
fn get_or_create_db_key(keychain: &Keychain) -> String {
    match keychain.get("com.meetingassistant.dbkey") {
        Ok(key) => key,
        Err(_) => {
            let key = generate_random_key_hex(32);
            keychain.set("com.meetingassistant.dbkey", &key).unwrap();
            key
        }
    }
}
```

**Keychain abstractions:**
- macOS: `security-framework` crate (`SecKeychainItem`).
- Windows: `windows` crate DPAPI (`CryptProtectData`/`CryptUnprotectData`). Store a DPAPI-protected blob in `%APPDATA%/MeetingAssistant/keys/`.

### Audio file encryption

Each `.opus` segment file is encrypted with AES-256-GCM using a per-meeting key derived from the DB key:

```
per_meeting_key = HKDF(db_key, salt=meeting_id_bytes, info="audio")
```

The Archive Writer encrypts each segment before writing to disk. Decryption happens in the playback path.

Segment file format:
```
[12-byte nonce][N-byte ciphertext][16-byte GCM tag]
```

### Migration path for existing (unencrypted) Meetily data

On first launch with encryption enabled:
1. Read all existing audio files → encrypt in place → delete originals.
2. Attach existing SQLite DB → dump → re-import into SQLCipher DB → delete plaintext DB.

This migration runs once and is gated by a `migrations` table flag.

---

## G9 — Library, search, playback, regenerate

### Meeting library

`frontend/app/library/page.tsx`:
- List meetings sorted by `started_at DESC`.
- Each row: title (or "Untitled Meeting"), date, duration, platform_guess, status badge (done / processing / failed / recoverable).
- Filter: text search (calls `meetings.list({ query })` which triggers FTS5), date range picker, platform dropdown.
- Click → meeting detail page.

### FTS5 search

```sql
-- Migration 007 — FTS5 virtual table
CREATE VIRTUAL TABLE IF NOT EXISTS transcript_fts USING fts5(
    text,
    content='transcript_segments',
    content_rowid='id'
);

CREATE TRIGGER IF NOT EXISTS trig_ts_insert AFTER INSERT ON transcript_segments BEGIN
    INSERT INTO transcript_fts(rowid, text) VALUES (new.id, new.text);
END;
CREATE TRIGGER IF NOT EXISTS trig_ts_update AFTER UPDATE ON transcript_segments BEGIN
    INSERT INTO transcript_fts(transcript_fts, rowid, text) VALUES ('delete', old.id, old.text);
    INSERT INTO transcript_fts(rowid, text) VALUES (new.id, new.text);
END;
CREATE TRIGGER IF NOT EXISTS trig_ts_delete AFTER DELETE ON transcript_segments BEGIN
    INSERT INTO transcript_fts(transcript_fts, rowid, text) VALUES ('delete', old.id, old.text);
END;
```

Search command:
```rust
fn search_meetings(db: &Db, query: &str, page: u32) -> Vec<MeetingSearchResult> {
    db.query(
        "SELECT m.id, m.title, m.started_at, snippet(transcript_fts, 0, '<b>', '</b>', '...', 10) as snippet
         FROM transcript_fts
         JOIN transcript_segments ts ON transcript_fts.rowid = ts.id
         JOIN meetings m ON ts.meeting_id = m.id
         WHERE transcript_fts MATCH ?1
         GROUP BY m.id
         ORDER BY rank
         LIMIT 20 OFFSET ?2",
        [query, page * 20]
    )
}
```

### Audio playback with seek-by-transcript-line

Meeting detail page shows interleaved transcript lines and a waveform/progress bar.

Click a transcript line (`start_ms` known) → compute which segment contains that timestamp:
```
segment_index = start_ms / segment_duration_ms
offset_within_segment = start_ms % segment_duration_ms
```

Seek implementation: pass `(segment_file_path, offset_ms)` to the Rust playback command, which decrypts the segment and seeks to the byte offset.

Tauri command:
```typescript
invoke('playback_start', { meetingId, startMs }) -> { jobId }
invoke('playback_seek',  { meetingId, posMs })
invoke('playback_stop',  { meetingId })
```

### Regenerate report

Meeting detail page → "Regenerate" button → opens a dialog:
- Prompt selector (dropdown of existing prompts with version).
- Provider selector (local / cloud).
- Estimated cost (from Phase 2 BudgetGuard).

Submit → calls `report_generate` (Phase 3 job pipeline) → new `summaries` row. All past summaries for the meeting remain queryable (full history).

---

## G10 — Consent UX & markdown export

### Consent banner (first launch)

Show a modal on first launch:

> **Recording notice**  
> This app records your meeting audio and transcribes it locally on your device. No audio or transcript is sent to external servers unless you explicitly enable a cloud provider in Settings.  
> Recording laws vary by jurisdiction. You are responsible for obtaining consent from all participants where required.  
> [I understand — continue]

Store acknowledgement: `settings.consent_acknowledged = true` in a local `settings` table or preferences file. Do not show again.

### In-meeting recording indicator

While capture is active: show a small persistent banner in the UI (not a system notification):

```
● Recording  [Stop]
```

Position: top of the main window. Color: red dot, white text. Visible throughout the meeting. This is informational for the user, not a broadcast to participants.

This feature is on by default (G10 spec). No toggle in v1 (banner is always shown during active capture).

### Markdown export

From the report view, "Export as Markdown" button:

```typescript
invoke('export_markdown', { summaryId }) -> { filePath: string }
```

Rust handler:
```rust
fn export_markdown(summary_id: i64, save_dir: &Path) -> Result<PathBuf> {
    let summary = db.get_summary(summary_id)?;
    let meeting = db.get_meeting(summary.meeting_id)?;
    let filename = format!("{}-{}.md",
        sanitize_filename(&meeting.title),
        meeting.started_at.format("%Y%m%d"));
    let path = save_dir.join(filename);
    fs::write(&path, &summary.body_md)?;
    Ok(path)
}
```

The saved file path is opened in the system file manager (Tauri's `shell.open`).

### Per-meeting delete

```typescript
invoke('meeting_delete', { meetingId }) -> void
```

Rust handler (inside a transaction):
1. Delete audio files: `rm -rf audio/<meeting_id>/` (all encrypted segments).
2. DELETE FROM jobs WHERE meeting_id = ?
3. DELETE FROM summaries WHERE meeting_id = ?
4. DELETE FROM transcript_segments WHERE meeting_id = ?
5. DELETE FROM recording_segments WHERE meeting_id = ?
6. DELETE FROM meetings WHERE id = ?
7. Rebuild FTS5 index: `INSERT INTO transcript_fts(transcript_fts) VALUES('rebuild')`

Confirm dialog before deletion: "This will permanently delete the recording, transcript, and all reports for this meeting. This cannot be undone."

### Global purge

Settings → "Delete all data":
1. Same as per-meeting delete for each meeting.
2. Truncate `prompts`, `providers` (user can reconfigure), `jobs`.
3. Delete the SQLCipher DB file and recreate empty.
4. Remove DB key from keychain (optional — ask user).

---

## Migration 007 — FTS5 + settings table

```sql
-- FTS5 (defined above in G9)

CREATE TABLE IF NOT EXISTS app_settings (
    key   TEXT PRIMARY KEY,
    value TEXT NOT NULL
);
-- Consent flag will be: INSERT INTO app_settings VALUES ('consent_acknowledged', 'false')
```

---

## Implementation steps

1. Write migration 007 (FTS5 virtual table + triggers + app_settings). Test triggers fire correctly on insert/update/delete.
2. Implement `EndDetector` (silence timeout + process watch + device release signal). Wire into Audio Router's remote-channel frame loop.
3. Add silence countdown UI (10-second countdown + "Keep recording" button).
4. Implement SQLCipher key generation + keychain read/write (macOS + Windows). Test key round-trip.
5. Wrap `Connection::open` with `open_encrypted`. Run all existing DB tests against encrypted DB.
6. Implement audio segment encryption (AES-256-GCM) in Archive Writer and decrypt in playback path.
7. Implement one-time encryption migration for existing plaintext data.
8. Build meeting library frontend (list, filter, FTS search).
9. Build meeting detail page (interleaved transcript + waveform + playback + seek).
10. Build regenerate-report dialog (prompt + provider selectors, cost estimate).
11. Implement `export_markdown` Tauri command.
12. Implement `meeting_delete` and global purge.
13. Add first-launch consent banner.
14. Add in-meeting recording indicator.
15. Run all automated tests.
16. Run full manual exit criterion checklist on both OSes.

---

## Test plan

### Automated

#### `test_end_detector_silence_timeout`

```rust
#[test]
fn test_silence_timeout_triggers_end() {
    let mut detector = EndDetector { silence_timeout_sec: 5, silence_threshold_db: -60.0, .. };
    let silent_frame = AudioFrame::silent(AudioSource::Remote);
    for _ in 0..50 {  // 50 * 100ms = 5 seconds
        detector.on_audio_frame(&silent_frame);
    }
    assert!(detector.end_triggered());
}
```

#### `test_end_detector_local_silence_no_trigger`

```rust
#[test]
fn test_local_silence_does_not_trigger_end() {
    let mut detector = EndDetector { silence_timeout_sec: 5, .. };
    let local_silent = AudioFrame::silent(AudioSource::Local);
    let remote_loud = AudioFrame::with_rms(AudioSource::Remote, 0.5);
    for _ in 0..100 {
        detector.on_audio_frame(&local_silent);
        detector.on_audio_frame(&remote_loud);
    }
    assert!(!detector.end_triggered());
}
```

#### `test_encryption_round_trip`

```rust
#[test]
fn test_audio_segment_encrypt_decrypt() {
    let key = generate_random_key(32);
    let plaintext = vec![0u8; 1024];
    let encrypted = encrypt_segment(&key, &plaintext);
    let decrypted = decrypt_segment(&key, &encrypted).unwrap();
    assert_eq!(plaintext, decrypted);
}
```

#### `test_fts_search_returns_correct_meeting`

```rust
#[test]
fn test_fts_search() {
    let db = test_db_with_fts();
    insert_transcript_segment(&db, meeting_id=1, text="quarterly budget review");
    insert_transcript_segment(&db, meeting_id=2, text="product roadmap discussion");
    let results = search_meetings(&db, "budget", 0);
    assert_eq!(results.len(), 1);
    assert_eq!(results[0].meeting_id, 1);
}
```

#### `test_meeting_delete_removes_all`

```rust
#[test]
fn test_meeting_delete_removes_all() {
    let db = test_db();
    let tmp = tempdir();
    seed_meeting(&db, &tmp, meeting_id=1);  // creates audio files + all rows
    delete_meeting(&db, &tmp, 1);
    assert_eq!(count_rows(&db, "transcript_segments", "meeting_id=1"), 0);
    assert_eq!(count_rows(&db, "summaries", "meeting_id=1"), 0);
    assert_eq!(count_rows(&db, "jobs", "meeting_id=1"), 0);
    assert!(!tmp.path().join("audio/1").exists());
}
```

### Manual exit criterion

Run all 8 exit criteria in order on macOS, then on Windows:

| # | Test | Steps |
|---|---|---|
| 1 | Auto-detect | Join Teams call → leave call → wait → verify report auto-generates in ≤90s |
| 2 | Library + search | Open library → search "budget" → verify correct meeting returned in <500ms |
| 3 | Playback | Click a transcript line → verify audio seeks to ±2s of that timestamp |
| 4 | Regenerate | Select meeting → change prompt → Regenerate → verify new `summaries` row |
| 5 | Encryption | Close app → inspect DB file with hex editor → verify non-plaintext. Reopen app → verify decrypts normally |
| 6 | Delete | Delete a meeting → verify no audio files remain → verify DB rows gone |
| 7 | Consent | Fresh install → verify consent banner on first launch → not on second launch |
| 8 | Export | Export a report → verify `.md` file contains report text → opens in text editor |

---

## Risks

| Risk | Mitigation |
|---|---|
| Meet-in-browser end-detection unreliable | Silence timeout is the primary signal for browser meetings; document in open-questions.md that a browser extension signal is a v2 option |
| SQLCipher performance overhead on large FTS queries | Benchmark before shipping; SQLCipher overhead for read-heavy workloads is typically <10% |
| DPAPI-encrypted key blobs not portable across Windows users | Scope: single-user app; document that data is tied to the Windows user profile |
| Encryption migration of large audio files is slow | Run migration as a background job with progress UI; app is usable before it completes |
| FTS5 rebuild on large DB is slow | Rebuild only on delete (targeted delete, not full rebuild for most operations); benchmark on 100-meeting DB |
