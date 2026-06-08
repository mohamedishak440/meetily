# Open Questions

Questions that must be answered before Phase 1 exits. Answers go here; if an answer changes a spec, update the spec too.

---

## Q1 — Hardware floor

**Question:** What is the minimum CPU/RAM/GPU we must support?

**Why it matters:** Sets the default Whisper model size (tiny/base/small/medium). Smaller = works on more machines. Larger = better accuracy. If we can assume Apple M1 / recent Intel i5 with 8 GB RAM, `base` or `small` works. If we need to support older hardware, `tiny` only.

**Answer:** _pending_

**Decision needed by:** end of Phase 1 (before transcription engine is hardened).

---

## Q2 — v1 export formats

**Question:** Is markdown-only sufficient for v1, or do users need PDF/DOCX on launch?

**Why it matters:** PDF/DOCX (D1 in Phase 5) requires bundling pandoc or typst — adds ~30 MB to the binary and build complexity. Markdown export is 10 lines of Rust. Meetily PRO gates PDF/DOCX; we'd be building it ourselves.

**Answer:** _pending_ (design says markdown-only; confirm this is acceptable to stakeholders)

**Decision needed by:** before Phase 4 (G10 — export UX).

---

## Q3 — Meet-in-browser end-detection

**Question:** For Google Meet running in Chrome, are silence + process-exit heuristics good enough to detect meeting end, or do we need an optional browser-extension signal?

**Why it matters:** Phase 4 (G5) end-detector uses 4 heuristics (user stop, process exit, device release, silence timeout). Browser-based Meet doesn't have a process exit signal — silence timeout (90s default) is the fallback. If users run long pauses during Meet calls this will false-positive.

**Answer:** _pending_

**Options:**
- A) Silence timeout only (simplest; may false-positive on long pauses)
- B) Silence timeout + optional browser extension that sends a `meeting.ended` signal to the app via localhost

**Decision needed by:** before Phase 4 (G5 implementation).

---

## Q4 — Consent UX scope for v1

**Question:** In v1, is a one-time consent banner on first launch sufficient, or do we need an always-visible in-meeting recording indicator?

**Why it matters:** Design §9 requires both. The always-on red dot banner during capture is in Phase 4 spec. Some jurisdictions require visible recording notice. If we must ship the always-on indicator in v1, it affects Phase 4 complexity.

**Answer:** _pending_ (current plan: always-on red dot banner in Phase 4, consent modal on first launch; confirm this is acceptable)

**Decision needed by:** before Phase 4 (G10 — consent UX).
