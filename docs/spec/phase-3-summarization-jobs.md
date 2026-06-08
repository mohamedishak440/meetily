# Phase 3 — Long-Transcript Summarization & Job Queue

**Status:** Not started  
**Depends on:** Phase 2 (prompt store, provider registry, budget guard)  
**Blocks:** Phase 4 (library uses job status; auto-detect triggers jobs)  
**Gaps addressed:** G2 (map-reduce summarization), G3 (resumable job queue)

---

## Goal

Summarize transcripts of any length — 1h, 10h, 100h — into a structured report. Each summarization spawns a DAG of independent jobs persisted in SQLite. On restart, incomplete jobs resume from the last completed node. Budget guard (Phase 2) gates each cloud node. Local jobs (Ollama) are never blocked.

---

## Exit Criterion

**All of the following must pass on macOS AND Windows before Phase 3 is done:**

1. **1h transcript:** a ~9,000-word synthetic transcript summarizes to a structured report in a single pass (no map-reduce needed). Local Ollama provider. Report stored in `summaries`.
2. **10h transcript:** a ~90,000-word synthetic transcript summarizes correctly using map-reduce (at least 2 levels). Report is coherent. Budget records populated.
3. **100h transcript:** a ~900,000-word synthetic transcript summarizes using ≥3 levels of map-reduce. Memory usage stays bounded (< 500 MB RSS throughout). Completes without error (local Ollama, throttled).
4. **Crash resume:** kill the engine mid-100h summarization (after ≥1 reduce node completes). Restart. Engine resumes from the last completed node — not from the start. Final report is identical to a non-crashed run on the same input.
5. **Budget pause and resume:** on a cloud provider, set budget cap such that it's hit mid-job. Verify cloud jobs pause. Increase cap via settings. Verify jobs resume. Local jobs in a parallel run are unaffected throughout.

---

## Scope

**In:**
- G3: `jobs` table + DAG scheduler in Rust core. Resume-on-startup. Per-provider concurrency limits. Global budget guard integration (from Phase 2).
- G2: hierarchical map-reduce summarizer on top of the Job Queue. Chunk transcript on speaker/topic boundaries. Depth scales with length (1h ≈ 1 pass, 100h ≈ 3–4 levels).
- Replace Meetily's single-pass `report_generate` with the job-based pipeline. The IPC command `report_generate` now returns a `jobId`; progress flows via `job.progress` events.
- Frontend: job progress UI (phase label + percentage), replace synchronous summary wait with async event subscription.

**Out:**
- No encryption (Phase 4).
- No meeting library/search (Phase 4).
- No auto meeting-end detection (Phase 4).
- TTS deferred (Phase 5).

---

## Data model changes (additive)

### Migration 006 — jobs table

```sql
CREATE TABLE IF NOT EXISTS jobs (
    id             INTEGER PRIMARY KEY,
    meeting_id     INTEGER NOT NULL REFERENCES meetings(id),
    type           TEXT NOT NULL CHECK(type IN ('map','reduce','synth','transcribe')),
    parent_job_id  INTEGER REFERENCES jobs(id),
    status         TEXT NOT NULL DEFAULT 'pending'
                       CHECK(status IN ('pending','running','done','failed','paused')),
    input_ref      TEXT,   -- JSON: {source: 'transcript'|'job_output', id: ...}
    output_ref     TEXT,   -- JSON: {kind: 'text'|'summary_id', value: ...}
    attempts       INTEGER NOT NULL DEFAULT 0,
    error          TEXT,
    cost           REAL,
    created_at     TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%SZ','now')),
    updated_at     TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%SZ','now'))
);

CREATE INDEX IF NOT EXISTS idx_jobs_meeting ON jobs(meeting_id);
CREATE INDEX IF NOT EXISTS idx_jobs_status  ON jobs(status);
```

---

## Architecture

### Job types

| Type | Input | Output |
|---|---|---|
| `map` | Transcript chunk (start_ms, end_ms) | Chunk summary text |
| `reduce` | Array of job IDs (whose outputs are chunk summaries) | Rollup summary text |
| `synth` | Root reduce job ID | Final structured report (markdown) |
| `transcribe` | `recording_segment` ID | Transcript rows (recovery path from Phase 1) |

### DAG construction

When `report_generate` is called:

```
1. Count transcript tokens for the meeting.
2. If tokens <= model_context_window * 0.7:
       create one 'synth' job (single-pass, no map/reduce)
3. Else:
       chunk transcript → N chunks of ~6k tokens on speaker/topic boundaries
       create N 'map' jobs
       group map job IDs into reduce groups of ≤ 10
       create reduce jobs (recursively if > 10 groups)
       create one 'synth' job depending on the root reduce job
4. Persist all jobs to DB in a single transaction (all 'pending').
5. Return the synth job ID to the caller.
```

### Scheduler (Rust)

```rust
pub struct JobScheduler {
    db: Arc<Db>,
    budget_guard: Arc<BudgetGuard>,
    concurrency: HashMap<ProviderId, usize>,  // max parallel per provider
}

impl JobScheduler {
    pub fn start(&self) {
        // On startup: load all jobs with status='running' → reset to 'pending' (they died)
        // Load 'pending' jobs whose dependencies are all 'done' → dispatch
        // On job done: update DB, emit job.done, check if dependents are now ready
        // On budget pause: set status='paused'; resume when budget.status changes
    }
}
```

**Concurrency limits:**
- Local (Ollama): max 1 concurrent job (single model instance; configurable).
- Cloud: max 3 concurrent jobs per provider (configurable per provider).

**Resume-on-startup:**
1. Set all `status='running'` → `status='pending'` (engine crashed while running them).
2. Load `pending` jobs where all `parent_job_id` dependencies are `done`.
3. Dispatch.

### Chunking (map phase)

```rust
fn chunk_transcript(segments: &[TranscriptSegment], max_tokens: usize) -> Vec<Chunk> {
    // Prefer splitting on speaker-change boundaries.
    // If no speaker change within max_tokens, split on sentence boundary (period + newline).
    // Each chunk carries: start_ms, end_ms, source distribution, text.
}
```

Token count: use a tiktoken-compatible byte-pair estimate or `whisper.cpp`'s tokenizer. Exact count not required; stay conservatively under the model's context window.

### Map prompt

```
You are summarizing a segment of a meeting transcript.
Transcript segment ({{start_time}} – {{end_time}}):
{{chunk_text}}

Provide a concise summary of the key points, decisions, and action items in this segment.
```

### Reduce prompt

```
You are combining multiple meeting segment summaries into a cohesive rollup.
Segment summaries:
{{summaries}}

Combine these into a single coherent summary, preserving all decisions and action items.
```

### Synth prompt (uses the user's selected prompt from Phase 2)

Variable `{{transcript_summary}}` added to the context, containing the root reduce output (or the direct transcript for single-pass). User's prompt body is interpolated with all variables including `{{transcript_summary}}`.

---

## IPC changes

### Command update

```typescript
// report_generate now always returns a jobId (was synchronous before)
invoke('report_generate', { meetingId, promptId, llmProviderId }) -> { jobId: number, synth_job_id: number }
```

### Events

```typescript
event 'job.progress' -> { jobId: number, pct: number, stage: 'map' | 'reduce' | 'synth', completedNodes: number, totalNodes: number }
event 'job.done'     -> { jobId: number, summaryId: number }
event 'job.failed'   -> { jobId: number, error: string, attempts: number }
event 'job.paused'   -> { jobId: number, reason: 'budget_exceeded' }
```

---

## Frontend changes

Replace the synchronous "Generating summary..." spinner with an async job progress view:

```
[Summarizing: map phase — 12 / 24 chunks complete — 50%]  [Cancel]
[████████░░░░░░░░░░░░] 50%
```

- Subscribe to `job.progress` on mount with the job's `synth_job_id`.
- On `job.done`: fetch and display the summary.
- On `job.failed`: show error with retry button.
- On `job.paused`: show "Paused — budget cap reached. Increase cap in Settings to resume." with a Settings link.

---

## Implementation steps

1. Write migration 006 (`jobs` table). Test on populated DB.
2. Implement `chunk_transcript` function with speaker-boundary splitting. Unit-test boundary cases.
3. Implement DAG constructor: `build_summarization_dag(meeting_id, tokens, model_context) -> Vec<Job>`.
4. Implement `JobScheduler`: startup recovery, dependency resolution, dispatch loop, concurrency limits, budget guard integration, event emission.
5. Implement `map` job handler: fetch chunk, interpolate map prompt, call LLM, store output in `jobs.output_ref`.
6. Implement `reduce` job handler: collect parent outputs, interpolate reduce prompt, call LLM, store output.
7. Implement `synth` job handler: interpolate user's selected prompt (Phase 2) with `transcript_summary`, call LLM, write `summaries` row.
8. Update `report_generate` Tauri command to use the new DAG + scheduler pipeline.
9. Update frontend: async progress view, event subscriptions.
10. Run all automated tests (see below).
11. Run manual exit criterion tests (1h, 10h, 100h, crash resume, budget pause).

---

## Test plan

### Automated

#### `test_chunk_on_speaker_boundary`

```rust
#[test]
fn test_chunk_on_speaker_boundary() {
    let segments = vec![
        TranscriptSegment { source: Local, text: "Hello...".repeat(500), .. },
        TranscriptSegment { source: Remote, text: "Response...".repeat(500), .. },
    ];
    let chunks = chunk_transcript(&segments, 6_000);
    // Speaker boundary was respected — second chunk starts at a Remote segment
    assert!(chunks[1].segments[0].source == Remote);
}
```

#### `test_dag_single_pass_for_1h`

```rust
#[test]
fn test_dag_single_pass() {
    // 9k tokens fits in a 12k context window
    let dag = build_summarization_dag(1, 9_000, 12_000);
    assert_eq!(dag.len(), 1);
    assert_eq!(dag[0].job_type, JobType::Synth);
}
```

#### `test_dag_map_reduce_for_100h`

```rust
#[test]
fn test_dag_map_reduce_for_100h() {
    // 1.2M tokens, 12k context window
    let dag = build_summarization_dag(1, 1_200_000, 12_000);
    let map_count = dag.iter().filter(|j| j.job_type == JobType::Map).count();
    let reduce_count = dag.iter().filter(|j| j.job_type == JobType::Reduce).count();
    let synth_count = dag.iter().filter(|j| j.job_type == JobType::Synth).count();
    assert!(map_count >= 100);
    assert!(reduce_count >= 10);
    assert_eq!(synth_count, 1);
    // Only one synth, it depends on a reduce, not directly on maps
    let synth = dag.iter().find(|j| j.job_type == JobType::Synth).unwrap();
    let parent = dag.iter().find(|j| Some(j.id) == synth.parent_job_id).unwrap();
    assert_eq!(parent.job_type, JobType::Reduce);
}
```

#### `test_scheduler_resumes_after_crash`

```rust
#[test]
fn test_scheduler_resumes_after_crash() {
    let db = test_db();
    // Simulate: 5 map jobs done, 2 reduce jobs done, 1 synth pending
    seed_jobs(&db, 5, JobStatus::Done, 2, JobStatus::Done, JobStatus::Pending);
    // Simulate crashed job: 1 reduce 'running' (engine died)
    db.execute("INSERT INTO jobs (..., status) VALUES (..., 'running')");
    
    let scheduler = JobScheduler::new(db.clone());
    scheduler.recover();
    
    // 'running' job reset to 'pending'
    let pending = db.query_all::<Job>("SELECT * FROM jobs WHERE status='pending'");
    assert!(pending.iter().any(|j| j.job_type == JobType::Reduce));
}
```

#### `test_memory_bounded_100h`

Integration test using a real local model (marked `#[ignore]` for CI, run in manual gate):
```rust
#[test]
#[ignore = "requires Ollama + 30min runtime"]
fn test_memory_bounded_100h() {
    // Generate a 900k-word synthetic transcript
    // Run the full summarization pipeline
    // Poll RSS every 10 seconds
    // Assert max RSS < 500 MB
}
```

#### `test_budget_pauses_cloud_not_local`

```rust
#[test]
fn test_budget_pauses_cloud_not_local() {
    let mut cloud = Provider { local: false, budget_cap: 0.01, spent_this_month: 0.01, .. };
    let local = Provider { local: true, budget_cap: 0.0, .. };
    
    let guard = BudgetGuard::new();
    assert!(guard.check(&cloud, 1000, 300).is_err());   // cloud blocked
    assert!(guard.check(&local, 1_000_000, 300).is_ok()); // local always ok
}
```

### Manual exit criterion tests

| Test | Steps | Pass |
|---|---|---|
| 1h single-pass | Generate report on ~9k-word transcript, local Ollama | Report appears, `jobs` has 1 row (synth), `summaries.prompt_version` set |
| 10h map-reduce | Generate on ~90k-word transcript | Report coherent, `jobs` has map+reduce+synth rows, all done |
| 100h map-reduce | Generate on ~900k-word synthetic transcript | Completes, memory < 500 MB, report generated |
| Crash resume | Start 100h job, kill mid-reduce, restart | Scheduler resumes from last completed node; final report identical |
| Budget pause | Configure cloud provider, set cap to hit mid-job | Cloud jobs `status='paused'`, local jobs unaffected, UI shows paused state |
| Budget resume | Increase cap in settings | Paused jobs transition to `pending` then `running` |

---

## Risks

| Risk | Mitigation |
|---|---|
| Local Ollama OOM on 100h (100 map calls sequential) | Throttle to 1 concurrent local job; each job loads/unloads context independently |
| Chunk token counting inaccurate | Use conservative estimate (assume ~1.3 tokens per word); stay at 70% of context window max |
| Job dependency graph corrupted on partial write | Wrap DAG construction in single DB transaction; if it fails, no jobs created |
| `reduce` job outputs too large for the next reduce call | Cap reduce input size; if sum of parent outputs > max_tokens, do an extra reduce level |
| 100h synthetic test is slow for CI | Mark as `#[ignore]`; run in dedicated "scale gate" workflow, not per-PR |
