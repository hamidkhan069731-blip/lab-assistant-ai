# Roadmap — Lab Assistant AI

Sequenced so that safety infrastructure exists before any AI feature ships.
Each phase has an exit test stated in clinical workflow terms.

---

## Phase 0 — Safety foundation

- Schema, migrations, auth, RBAC, audit log
- Report state machine with immutable `signed` records
- Reference-range tables (age, sex, pregnancy where applicable)
- Seed data: synthetic patients only — **never real patient data in the repo**
- CI: typecheck, tests, migration check

**Exit test:** a technician cannot sign a report; a pathologist can; a signed
report cannot be edited in SQL.

---

## Phase 1 — Manual pipeline, end to end

- Manual result entry with validation, units, delta checks
- Report template per panel with deterministic number injection
- Reviewer UI, sign, release-to-portal (manual send)
- Basic turnaround-time reporting

**Exit test:** a real-style order goes order → result → validated → drafted →
signed → released, with a complete audit trail.

---

## Phase 2 — Analyzer intake

- HL7 v2 MLLP listener; field mapping per analyzer model
- CSV drop-folder watcher with schema detection
- Quarantine queue for malformed payloads
- Unit normalisation with conversion provenance

**Exit test:** an analyser result reaches the correct order without retyping,
and a deliberately malformed message lands in quarantine rather than in a report.

---

## Phase 3 — Critical values

- Panic-value rules per test/age/sex
- Escalation chains, timers, call log
- Read-back comparison
- Incident report for missed read-backs

**Exit test:** a synthetic panic value pages the on-call user, escalates when
ignored, and closes only after a matching read-back; the incident report shows
any miss.

---

## Phase 4 — AI report drafting (under review)

- Structured output schema for narratives
- Numeric-token guard rejecting invented numbers
- Side-by-side reviewer screen with source values and model version
- Optional offline/self-hosted inference path

**Exit test:** 100 consecutive drafts pass schema and numeric guard; reviewer
edits are versioned; no draft can be signed without a human action.

---

## Phase 5 — Requisition OCR

- Scan/PDF/WhatsApp intake with confidence scoring
- Catalogue matching with alias map and human queue
- Duplicate-order detection
- Correction capture as labelled examples (de-identified)

**Exit test:** a photographed requisition becomes a matched test list; anything
below threshold is in the queue, and nothing low-confidence was auto-accepted.

---

## Phase 6 — Operational Q&A

- Read-only replica and allow-listed views
- Text-to-SQL with parser guard, statement timeout, row cap
- SQL shown with every answer
- Role-scoped access to anything identifying

**Exit test:** the ten most common staff questions are answered correctly, the
SQL is shown, and a request for patient-identifying data is refused for a
non-privileged role.

---

## Phase 7 — QC and reagent look-back

- Levey-Jennings charts and Westgard rules
- QC lock-out blocking release on failure
- Reagent lot and expiry tracking linked to affected results
- Look-back query after a failed control or recalled lot

**Exit test:** a failed control blocks the affected test and produces a complete
list of results to review, with the reagent lot recorded.

---

## Not yet (deliberate)

- Autonomous reporting — the regulatory and ethical position is not close
- Diagnostic suggestions or treatment advice — out of scope permanently
- Patient-facing chatbot — requires consent, disclosure and a much higher bar
- Multi-lab federation — a different product; one lab at a time
- Model fine-tuning on real results — requires governance that does not exist here

---

## Definition of done

1. Safety guard tested, including the failure path (invented number rejected).
2. Audit entries for every view, edit, sign and release.
3. Migration reversible; restore rehearsed on a copy.
4. No PHI in logs, fixtures or the repo.
5. Documentation updated with the change, including the clinical workflow it touches.
