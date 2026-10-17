# Architecture — Lab Assistant AI

Companion to the [README](../README.md). The decisions here are the ones that
matter for safety and auditability: where the model sits, how numbers are
protected, and how a report becomes immutable.

---

## 1. The trust boundary

```
        ┌──────────────── trusted ────────────────┐
LIS data │  numbers, ranges, flags, identity       │
─────────┼─────────────────────────────────────────┤
 model   │  prose only, structured output,         │
 boundary│  schema-validated, labelled as draft    │
         └─────────────────────────────────────────┘
```

The model never produces a number that reaches a report unless that number was
extracted from a document with a confidence score and a human confirmed it. In
the drafting path, numbers are injected from the validated result set into the
template; the model sees them as text and writes around them, and the output
schema requires the numeric fields to be byte-identical to the input.

Implementation:

1. Validation engine produces `ValidatedResult[]` (value, unit, range, flags).
2. Template renderer fills the structured skeleton — numbers and ranges.
3. Model call receives the skeleton plus style guidance; returns
   `{ narrative, observations, follow_up_note }` validated by JSON schema.
4. A guard compares every numeric token in the model output against the source
   set; any unexplained number rejects the draft and re-queues it.

---

## 2. Report state machine

```
draft ──submit──► under_review ──sign──► signed ──amend──► amended (v2)
   │                   │
   └──discard──►        └──return──► draft (with reviewer comments)
```

- Only `signed` may be released to the patient or the ordering doctor.
- `signed` rows are immutable at the database level (trigger blocks UPDATE on
  content columns; amendments insert a new version).
- Each transition records actor, role, timestamp, IP, and the model version used
  for the draft.
- The PDF is rendered server-side, hashed, and the hash stored with the signature
  record so tampering is detectable.

---

## 3. Critical-value engine

Configuration table:

```
critical_rule
  test_id, age_min, age_max, sex, low_threshold, high_threshold,
  severity, escalation_chain_id, readback_required
```

Flow:

1. Validated result crosses a rule → `critical_event` created, state `open`.
2. Dispatch to the on-call person per the schedule (SMS + phone + in-app).
3. Timer starts (configurable, e.g. 10 minutes). No acknowledgement → next in
   chain, then lab manager.
4. Receiving clinician must **read the value back**; the operator types what they
   heard and the system compares. Mismatch → repeat and log.
5. Close only with a read-back match or a documented override with a reason.
6. Any event closed without read-back is an `incident` and appears on the
   daily quality report.

The engine is rule-based, not model-based, on purpose. A model may *draft* the
call script; it may not decide whether a value is critical.

---

## 4. Intake pipeline

```
scan/pdf ─► ingest ─► classify (requisition? report? other)
                    └► extract (vision or text parse) ─► fields + confidence
                        └► catalogue match (synonyms, fuzzy, Roman Urdu)
                            └► confidence ≥ threshold ? auto-accept
                                                     : human queue
```

- Extraction results are stored raw alongside the parsed structure.
- Catalogue matching is deterministic first (exact + alias map), fuzzy second,
  model third — only for the residual set.
- Duplicate detection: same patient identifier + test + window, or same sample
  barcode already accepted.
- Every human correction in the queue becomes a labelled example for improving
  the alias map (no automatic training on PHI).

---

## 5. Operational Q&A (text-to-SQL)

- Read-only database role; a replica connection; statement timeout 10 s; row cap
  enforced server-side.
- Allow-listed schema exposed to the model: views with PHI minimal, e.g.
  `v_pending_tests`, `v_tat_by_station`, `v_qc_status` — not raw patient tables.
- The model outputs SQL; a parser rejects DML/DDL and any table outside the
  allow-list.
- The response always includes the SQL and the execution time so the user can
  judge it. If confidence in the mapping is low, the assistant asks a
  clarifying question instead of guessing.
- Non-privileged roles cannot query patient-identifying views at all.

---

## 6. Data model (core)

`patient` · `order` (requisition) · `specimen` · `test_catalog` ·
`result` (raw + validated) · `reference_range` · `report` (+ versions) ·
`critical_rule` · `critical_event` · `qc_run` · `qc_rule_result` ·
`reagent_lot` · `audit_event`

Results keep `raw_value`, `raw_unit`, `validated_value`, `validated_unit`,
`conversion_note` — never overwriting the analyser output.

---

## 7. PHI handling

- Model calls: patient identifiers replaced by internal pseudonyms; only the
  clinical context needed for the task is sent.
- Logs: no PHI in application logs; prompts stored encrypted with retention limit.
- Optional local inference path for labs that prohibit external processing.
- Access to a patient record generates an `audit_event` with actor and reason.

---

## 8. Failure modes

| Failure | Answer |
|---|---|
| Analyser sends garbled results | Quarantine queue; never silently accept |
| Model output contains an invented number | Guard rejects the draft; not shown to reviewer |
| QC control fails | Release blocked for affected tests; look-back query listed |
| Critical call not acknowledged | Automatic escalation up the chain, incident logged |
| LIS unavailable | Intake queues locally; nothing is lost, nothing is guessed |
| Reported unit mismatch | Explicit conversion record; ambiguous conversion → human decision |
