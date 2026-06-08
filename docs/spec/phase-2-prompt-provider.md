# Phase 2 — Prompt Store & Provider/Budget Layer

**Status:** Not started  
**Depends on:** Phase 1 (stable capture + DB migrations)  
**Blocks:** Phase 3 (summarizer consumes prompt store and provider registry)  
**Gaps addressed:** G1 (editable versioned prompt store), G4 (budget caps & cost estimation)

---

## Goal

Users can create, edit, and version summarization prompts. Providers are configured with monthly budget caps. Before any cloud LLM call, the system estimates cost and warns/pauses at the cap. Local jobs are never budget-blocked.

---

## Exit Criterion

**Both of the following must pass on macOS AND Windows before Phase 2 is done:**

1. User edits a seeded prompt in the UI, saves it, selects a provider (local Ollama), triggers summary generation, and sees a correctly formatted summary — the report's `prompt_version` in the DB matches the saved prompt version.
2. User configures a cloud provider with a `budget_cap = $0.10`. System generates a cost estimate before running. When simulated spend exceeds the cap, cloud jobs pause and the UI shows a warning. Local Ollama jobs proceed unblocked.

---

## Scope

**In:**
- G1: `prompts` table, CRUD API, variable interpolation, version retention, seeded templates, prompt editor UI.
- G4: `providers` table with `budget_cap` + `spent_this_month`; pre-run token/cost estimate; threshold warning at 80% of cap; hard pause at 100% for cloud jobs only.
- Extend `summaries` table with `prompt_id`, `prompt_version`, `model`, `provider`, `tokens_in`, `tokens_out`, `cost_estimate`.
- Tauri IPC commands: `prompts.list`, `prompts.get`, `prompts.save`, `providers.list`, `providers.upsert`, `budget.status`.
- Tauri events: `budget.warn { provider, spent, cap }`.

**Out:**
- No map-reduce summarization (Phase 3). Phase 2 uses Meetily's existing single-pass summarizer.
- No job queue (Phase 3). Summary is a synchronous Tauri command for now.
- No encryption (Phase 4).
- No meeting library/search (Phase 4).

---

## Data model changes (additive)

### Migration 003 — prompts table

```sql
CREATE TABLE IF NOT EXISTS prompts (
    id            INTEGER PRIMARY KEY,
    name          TEXT NOT NULL,
    body          TEXT NOT NULL,
    variables_json TEXT NOT NULL DEFAULT '[]',
    version       INTEGER NOT NULL DEFAULT 1,
    is_default    INTEGER NOT NULL DEFAULT 0,
    updated_at    TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%SZ','now'))
);

-- Ensure at most one default prompt
CREATE UNIQUE INDEX IF NOT EXISTS uq_prompts_default ON prompts(is_default) WHERE is_default = 1;
```

### Migration 004 — summaries extensions

```sql
ALTER TABLE summaries ADD COLUMN prompt_id    INTEGER REFERENCES prompts(id);
ALTER TABLE summaries ADD COLUMN prompt_version INTEGER;
ALTER TABLE summaries ADD COLUMN model        TEXT;
ALTER TABLE summaries ADD COLUMN provider     TEXT;
ALTER TABLE summaries ADD COLUMN tokens_in    INTEGER;
ALTER TABLE summaries ADD COLUMN tokens_out   INTEGER;
ALTER TABLE summaries ADD COLUMN cost_estimate REAL;
```

### Migration 005 — providers extensions

```sql
ALTER TABLE providers ADD COLUMN budget_cap        REAL;
ALTER TABLE providers ADD COLUMN spent_this_month  REAL NOT NULL DEFAULT 0.0;
ALTER TABLE providers ADD COLUMN reset_day         INTEGER NOT NULL DEFAULT 1;
```

`reset_day`: day of month when `spent_this_month` resets (default: 1st). The engine resets spend on the first request after the reset day each month.

---

## Prompt versioning model

- Editing a prompt creates a **new row** (new `id`, incremented `version`). Old rows are retained.
- A summary records `prompt_id` + `prompt_version` so it can always be reproduced from the exact prompt that generated it.
- "Edit" in the UI always creates a new version. There is no in-place update.
- `is_default = 1` points to the current default version for new summaries.

### Seeded templates (inserted on first run)

| name | variables |
|---|---|
| Action items & decisions | `{{meeting_title}}`, `{{date}}`, `{{participants}}` |
| Detailed minutes | `{{meeting_title}}`, `{{date}}`, `{{participants}}` |
| Executive summary | `{{meeting_title}}`, `{{date}}` |

Template bodies to be defined in `src-tauri/src/prompts/seeds.rs` as Rust string constants.

### Variable interpolation

Before passing `prompt.body` to the LLM, replace `{{var}}` tokens:

```rust
fn interpolate(body: &str, ctx: &PromptContext) -> String {
    body.replace("{{meeting_title}}", &ctx.title)
        .replace("{{date}}", &ctx.date)
        .replace("{{participants}}", &ctx.participants.join(", "))
}
```

Unknown variables left as-is (don't error; warn in logs).

---

## Budget logic

### Cost estimation

Before dispatching a cloud LLM call:

```rust
fn estimate_cost(provider: &Provider, tokens_in: u64, tokens_out_estimate: u64) -> f64 {
    let in_cost  = tokens_in as f64 / 1000.0 * provider.cost_per_1k_in;
    let out_cost = tokens_out_estimate as f64 / 1000.0 * provider.cost_per_1k_out;
    in_cost + out_cost
}
```

`tokens_out_estimate`: use 300 tokens as a default estimate if not known. The actual `tokens_out` is recorded after the call.

### Budget check sequence (per cloud job)

```
1. estimate = estimate_cost(provider, tokens_in, tokens_out_estimate)
2. if provider.spent_this_month + estimate > provider.budget_cap:
       emit budget.warn { spent: provider.spent_this_month, cap: provider.budget_cap }
       return Err(BudgetExceeded)
3. if provider.spent_this_month > 0.8 * provider.budget_cap:
       emit budget.warn (threshold warning, not a block)
4. dispatch LLM call
5. record actual tokens_in, tokens_out, cost = actual cost
6. UPDATE providers SET spent_this_month = spent_this_month + cost WHERE id = ?
```

**Local providers** (Ollama, llama.cpp, whisper.cpp local): skip steps 1–3 entirely. Never blocked.

### Monthly reset

On engine startup and before each cloud call, check:
```rust
if today.day() >= provider.reset_day && last_reset_date < this_month {
    UPDATE providers SET spent_this_month = 0.0, last_reset_date = today WHERE id = ?
}
```

---

## IPC layer (Tauri commands + events)

### Commands

```typescript
// prompts
invoke('prompts_list') -> Prompt[]
invoke('prompts_get', { id }) -> Prompt
invoke('prompts_save', { name, body, variables, isDefault }) -> Prompt  // always creates new version
invoke('prompts_delete', { id }) -> void  // only if no summaries reference it

// providers
invoke('providers_list') -> Provider[]
invoke('providers_upsert', { kind, name, model, endpoint, keychainRef, budgetCap }) -> Provider

// budget
invoke('budget_status') -> { providers: [{ id, name, spent, cap, resetDay }] }

// report generation (extends Meetily's existing command)
invoke('report_generate', { meetingId, promptId, llmProviderId }) -> { jobId }
```

### Events

```typescript
// budget warning (threshold or cap)
event 'budget.warn' -> { provider: string, spent: number, cap: number, blocked: boolean }
```

---

## Frontend (Next.js)

### New screens / components

1. **Prompt Editor** (`frontend/app/prompts/page.tsx`)
   - List of prompts (name, version, is_default badge).
   - Click to view/edit body in a textarea.
   - "Save as new version" button (never in-place edit).
   - "Set as default" button.
   - Variable token hints shown below the editor (list from `variables_json`).
   - Version history: dropdown showing past versions (read-only view).

2. **Provider & Budget Settings** (`frontend/app/settings/providers/page.tsx`)
   - List configured providers (kind, name, model, enabled toggle).
   - Per-provider: budget cap input ($/month), current spend progress bar, reset day.
   - "Add provider" form: kind dropdown, name, model, endpoint URL, API key field (write-only — key goes to keychain via `providers_upsert`, never stored in frontend state).
   - Warning banner when spend > 80% of cap (triggered by `budget.warn` event).

3. **Cost estimate in report generation flow**
   - When user clicks "Generate Report" and a cloud provider is selected: show estimated cost before dispatching. "Proceed" / "Cancel" buttons.
   - If local provider: no estimate dialog, go directly.

### UI constraints

- The API key input must be write-only: show a "key saved" confirmation after save, never redisplay the key value.
- Budget warning state is driven by the `budget.warn` Tauri event — don't poll; subscribe on mount.

---

## Implementation steps

1. Write migrations 003, 004, 005. Test each on a populated Meetily DB.
2. Implement `PromptStore` in Rust (`src-tauri/src/prompts/store.rs`): CRUD + versioning + interpolation + seeds insertion on first run.
3. Implement `BudgetGuard` in Rust (`src-tauri/src/budget/guard.rs`): estimate, check, record, reset.
4. Expose Tauri commands: `prompts_list/get/save/delete`, `providers_list/upsert`, `budget_status`, and extend `report_generate` to accept `promptId` and `llmProviderId`.
5. Extend `report_generate` handler: interpolate prompt variables, pass to Meetily's existing LLM call, record `tokens_in/out/cost` to `summaries`.
6. Wire `budget.warn` Tauri event emission.
7. Build Prompt Editor UI.
8. Build Provider & Budget Settings UI.
9. Add cost estimate dialog to report generation flow.
10. Run manual exit criterion test (both OSes).

---

## Test plan

### Automated

#### `test_prompt_versioning`

```rust
#[test]
fn test_prompt_versioning() {
    let db = test_db();
    let v1 = store.save("Test", "Body v1", vec![], true);
    let v2 = store.save("Test", "Body v2", vec![], true);
    assert_eq!(v2.version, v1.version + 1);
    // v1 still exists
    let all = store.list_all_versions("Test");
    assert!(all.iter().any(|p| p.id == v1.id));
    // default points to v2
    assert_eq!(store.default().id, v2.id);
}
```

#### `test_variable_interpolation`

```rust
#[test]
fn test_variable_interpolation() {
    let body = "Meeting: {{meeting_title}} on {{date}}. Unknown: {{unknown}}";
    let ctx = PromptContext { title: "Q3 Sync".into(), date: "2026-06-08".into(), participants: vec![] };
    let result = interpolate(body, &ctx);
    assert_eq!(result, "Meeting: Q3 Sync on 2026-06-08. Unknown: {{unknown}}");
}
```

#### `test_budget_guard_blocks_at_cap`

```rust
#[test]
fn test_budget_guard_blocks_at_cap() {
    let mut provider = Provider { budget_cap: 0.10, spent_this_month: 0.09, cost_per_1k_in: 0.01, cost_per_1k_out: 0.01, local: false, .. };
    let guard = BudgetGuard::new();
    // estimated cost = 0.02, total would be 0.11 > 0.10
    let result = guard.check(&provider, 1000, 1000);
    assert!(matches!(result, Err(BudgetExceeded)));
}
```

#### `test_budget_guard_does_not_block_local`

```rust
#[test]
fn test_budget_guard_does_not_block_local() {
    let provider = Provider { local: true, budget_cap: 0.00, spent_this_month: 99.99, .. };
    let guard = BudgetGuard::new();
    let result = guard.check(&provider, 1_000_000, 1_000_000);
    assert!(result.is_ok());
}
```

#### `test_migration_003_004_005`

```rust
#[test]
fn test_migrations_on_meetily_db() {
    let db = load_meetily_fixture_db();
    run_migration(&db, "003_prompts.sql");
    run_migration(&db, "004_summaries_ext.sql");
    run_migration(&db, "005_providers_ext.sql");
    // All existing rows intact
    assert!(db.query_one::<i64>("SELECT COUNT(*) FROM meetings").unwrap() > 0);
    // New tables/columns exist
    db.query_one::<i64>("SELECT COUNT(*) FROM prompts").unwrap();
}
```

#### `test_seeds_inserted_once`

```rust
#[test]
fn test_seeds_inserted_once() {
    let db = test_db();
    insert_seeds(&db);
    let count1 = db.query_one::<i64>("SELECT COUNT(*) FROM prompts").unwrap();
    insert_seeds(&db);  // idempotent
    let count2 = db.query_one::<i64>("SELECT COUNT(*) FROM prompts").unwrap();
    assert_eq!(count1, count2);
}
```

### Manual

1. **Prompt editor:** open app → Prompts → edit "Action items & decisions" → save → verify new version number in DB.
2. **Default prompt:** set "Executive summary" as default → generate report → verify `summaries.prompt_id` and `prompt_version` match.
3. **Budget cap (cloud):** configure a cloud provider with $0.10 cap. Set `spent_this_month = 0.09` directly in DB. Click Generate Report → verify cost estimate dialog appears → click Proceed → verify `budget.warn` event fires → verify job is blocked.
4. **Local unblocked:** with same provider state, switch to Ollama → Generate Report → verify no estimate dialog → report generates.

---

## Risks

| Risk | Mitigation |
|---|---|
| Meetily's existing summarizer doesn't expose token counts | Wrap the LLM call; count tokens from the response object or estimate from response length |
| Keychain write fails on CI | Keychain integration is manual-test only; CI tests mock the keychain |
| `cost_per_1k_in/out` not in current Meetily provider config | Add to `providers` table migration; populate from a hardcoded capabilities table in the registry |
