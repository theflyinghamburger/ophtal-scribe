# Ophthalmology Dictation Assistant

## Senior Software Architecture and Ophthalmology Review

### Documents reviewed

- `architecture.md`
- `mvp-implementation-task-specification.md`

### Review position

The proposal has a strong foundation: it is local-first, preserves the original audio, separates terminology correction from structured extraction, constrains JSON output, requires clinician approval, and uses deterministic code for validation and PDF rendering.

However, it should be **approved with major revisions**, not implemented unchanged. The principal risks are not the choice of Electron, Parakeet, or llama.cpp. They are:

1. An incomplete definition of what constitutes the clinical record.
2. Insufficient representation of examination status.
3. Overreliance on LLM-generated confidence and evidence.
4. Allowing unreviewed corrections to feed extraction.
5. Plaintext temporary clinical data.
6. Delegating clinical meaning to smaller implementation models.
7. Deferring the first end-to-end integration and real evaluation too late.

## 1. Executive assessment

| Area | Assessment | Decision |
| --- | --- | --- |
| Overall architecture | Sound separation of concerns and good local-first design | Approve after revision |
| Clinical safety model | Correct intent but incomplete clinical-state semantics | Major revision required |
| LLM safety | Good schema constraints and protected fields, but confidence/provenance need redesign | Major revision required |
| Desktop security | Good Electron boundary design; data-at-rest protection is insufficient | Major revision required |
| MVP sequencing | Thorough but integration value arrives too late | Revise phase order |
| Evaluation | Clinically relevant metrics are present; sample size and gates are too weak for anything beyond limited pilot use | Revise gates |
| Smaller-model delegation | Strong task formatting, but Class C work is delegated too freely | Restrict authority |
| A4 form output | Appropriate for the requested worksheet, but not yet a complete medical record | Clarify scope |

## 2. What is already well designed

### 2.1 Architecture strengths

- Audio is correctly treated as the evidentiary source rather than treating ASR text as truth.
- Raw ASR, corrected transcript, structured extraction, reviewed record, and PDF are separate artifacts.
- ASR and LLM engines are behind replaceable provider interfaces.
- llama.cpp schema constraints are used for syntax, while application code performs validation.
- Laterality, negation, numbers, medication, and dosage receive special handling.
- Unmapped statements are retained rather than discarded.
- Electron renderer privileges are appropriately restricted.
- Native processes are owned and supervised by the main process.
- Model, glossary, schema, and application versions are recorded.
- PDF generation is deterministic and independent of the LLM.
- Golden fixtures, offline tests, visual regression, and held-out evaluation are included.

### 2.2 Ophthalmology strengths

- OD, OS, and OU are explicitly recognized as safety-sensitive.
- The proposed glossary includes canonical terms, aliases, confusable terms, anatomical context, and expected form fields.
- The clinician can replay source audio from a correction or extracted field.
- Normal findings are not supposed to be inferred from silence.
- The design requires clinician approval before export.
- Freehand diagram automation is excluded from the MVP, which is appropriate.
- The selected worksheet separates anterior and posterior examination findings in a clinically recognizable way.

## 3. Release-blocking findings

These issues must be resolved before implementation becomes the project baseline.

### B1 — The intended clinical document is not defined precisely enough

The supplied image is an ocular-health worksheet with optometric elements. It is not, by itself, a complete comprehensive ophthalmology record. Important encounter information is absent or not explicitly represented:

- Patient name or clinic identifier.
- Encounter date and time.
- Examiner identity and registration details.
- Chief complaint and relevant history.
- Visual acuity.
- Refraction context.
- Pupillary examination and afferent defect.
- Ocular motility and alignment where relevant.
- Intraocular pressure, instrument, and time.
- Dilation status and agents where applicable.
- Examination method and lens used.
- Diagnosis or assessment terminology.
- Clinician signature or attestation.

The plan refers to visual acuity, pressure, medication, and dosage as protected content even though the displayed form and simplified schema do not clearly contain them. This creates a contract inconsistency.

**Required decision:** Choose one of the following and name it throughout the product:

1. **Ocular-health worksheet assistant:** reproduce only the supplied worksheet and send out-of-scope content to `unmappedStatements`; or
2. **Ophthalmology clinical-note assistant:** expand the schema and A4 output to include the missing encounter and examination fields.

The MVP should not be described as a complete ophthalmology record unless the second option is selected and clinically approved.

### B2 — `null` is clinically ambiguous

The plan currently uses `null` for missing data. A blank clinical field can mean several materially different things:

- The doctor did not mention it.
- It was not examined.
- It was examined but could not be assessed.
- It was normal.
- It was abnormal, but the finding was not captured.
- It is not applicable.

Collapsing these states into `null` creates documentation ambiguity and weakens validation.

**Required change:** Each clinical observation should have explicit status, for example:

```json
{
  "status": "not_mentioned",
  "value": null,
  "sourceSegmentIds": [],
  "enteredBy": "model"
}
```

Recommended statuses:

- `not_mentioned`
- `not_examined`
- `unable_to_assess`
- `observed_normal`
- `observed_abnormal`
- `not_applicable`

Only the clinician should be able to change `not_mentioned` into `not_examined`, `unable_to_assess`, or an observed state without supporting dictation evidence.

The PDF may still render several of these states as blank, but the internal record must preserve the distinction.

### B3 — Every terminology correction must be reviewed during the MVP

The example cleanup result sets `reviewRequired: false` for a semantic substitution. That is unsafe for the initial clinical MVP. A plausible medical term is not necessarily the spoken term.

**Required change:**

- All non-formatting substitutions require explicit clinician acceptance during the MVP.
- Only deterministic formatting may bypass per-correction review, such as capitalization or standard spacing that does not alter meaning.
- Structured extraction must consume the **clinician-reviewed corrected transcript**, not the model-proposed corrected transcript.

Revise the state machine to include:

```text
Transcribed
→ CorrectionProposed
→ CorrectionReviewed
→ Extracted
→ ExtractionReviewed
→ Approved
→ Exported
```

### B4 — LLM self-reported confidence must not drive safety behavior

The numeric `confidence: 0.94` is not calibrated probability. It can create false reassurance for the clinician and developer.

**Required change:** Replace model confidence with observable evidence:

- Candidate source: exact alias, known ASR confusion, lexical match, phonetic match, contextual choice.
- Deterministic candidate score.
- Rank among alternatives.
- Whether protected content is involved.
- Whether the source audio was replayed.
- Clinician decision.

An LLM may provide a short explanation, but its self-assigned confidence must not determine auto-acceptance, highlighting severity, or export eligibility.

### B5 — Provenance should use immutable segment IDs, not free-text matching

`P4-T03` proposes resolving cited text back to transcript spans. An LLM may slightly rewrite or invent a quotation, making this fragile.

**Required change:**

1. Assign immutable IDs to ASR segments and word spans before either LLM call.
2. Present numbered segments to the LLM.
3. Require every correction and populated field to return `sourceSegmentIds`.
4. Validate that every returned ID exists.
5. Treat missing or contradictory IDs as a blocking warning.

Free-text evidence may be displayed for convenience, but segment IDs must be the authoritative provenance link.

### B6 — Temporary patient data must not remain plaintext by design

The current plan protects settings with DPAPI but stores original audio, prepared audio, transcripts, and extracted findings in a normal application directory. Default deletion after export does not protect data while the case is active, after a crash, in backups, or on SSD blocks that cannot be reliably overwritten.

**Required change:** Adopt one of these policies before clinical testing:

- Require and verify full-disk encryption such as BitLocker on the clinical workstation; or
- Encrypt every working-case artifact with AES-GCM using a random per-case key wrapped by Windows DPAPI.

Prefer both for a real clinic deployment. Do not promise secure deletion on an SSD. Encrypt at rest, delete keys and working files according to policy, and document the limitation.

### B7 — Retention of working artifacts and retention of the medical record are different policies

The plan defaults to deleting the working case after export. That may be appropriate for temporary audio and model artifacts, but the approved clinical record may be subject to clinic and professional retention obligations.

**Required change:** Define separately:

- Temporary audio retention.
- Raw ASR retention.
- Correction and provenance retention.
- Approved structured record retention.
- Exported PDF retention.
- Backup and recovery.
- Patient-request and correction workflow.

Do not encode a deletion period until the clinic's legal and professional obligations are confirmed. Offline processing still handles digital personal and health data; the compliance workstream should explicitly cover India's current data-protection regime and clinic record obligations. The DPDP Rules were brought into force in November 2025, so this cannot be left as a post-MVP concern. [Reuters overview of the 2025 implementation](https://www.reuters.com/sustainability/boards-policy-regulation/india-strengthens-privacy-law-with-new-data-collection-rules-2025-11-14/)

### B8 — Patient and clinician identity must be handled deterministically

Patient demographics, encounter identifiers, and clinician identity should not pass through the LLM. They should be manually entered, selected from local configuration, or imported from a trusted system and then rendered deterministically.

**Required change:** Add a document header containing at minimum:

- Patient name or clinic identifier.
- Encounter date/time.
- Clinician name.
- Clinician registration identifier where required.
- Clinic name.
- Approval timestamp.
- Signature or attestation mechanism appropriate to the clinic.

Keep these values out of LLM prompts unless a future use case explicitly requires them.

### B9 — Smaller models must not author clinical truth

The task plan says smaller models may implement Class C work with later review. That is too permissive. Subtle errors in a field definition, permissible range, negation rule, or glossary description can propagate through prompts, tests, and UI before review.

**Required change:**

- The ophthalmologist authors or signs the field inventory, terminology, ranges, and dictation examples.
- A senior engineer authors or signs the safety invariants and schema semantics.
- Smaller models may mechanically encode these signed specifications.
- Smaller models may not introduce new glossary definitions, clinical ranges, default values, correction rules, or clinical examples.
- Class C fixtures must be human-authored; models may format but not create the expected clinical answer.

### B10 — The first end-to-end slice arrives too late

The current plan does not produce a real PDF until Phase 5. This risks discovering integration, packaging, font, layout, state-management, or performance problems after most components are built.

**Required change:** Add an early walking skeleton immediately after the Electron foundation:

```text
Import sample audio
→ fake ASR result
→ fake reviewed correction
→ fake validated clinical JSON
→ draft A4 PDF
→ preview and export
```

This slice should use fixtures, not real inference. It proves IPC, case storage, review state, template generation, preview, and packaging boundaries before model work begins.

## 4. Senior ophthalmologist review

### 4.1 Add a structured dictation protocol

The best accuracy improvement may come from changing how the doctor dictates, not from a larger model. Define and test a simple sequence such as:

```text
Patient identifier and encounter date.
OD anterior.
OD posterior.
OS anterior.
OS posterior.
Assessment.
Plan.
End of note.
```

The application should display the sequence while recording is out of scope and include it in clinician training. The doctor should speak decimal points, units, laterality, and negation explicitly.

Add a Phase 0 task for a clinician-approved dictation protocol and include structured versus unstructured dictation in evaluation.

### 4.2 Confirm every form abbreviation and examination method

The Phase 0 prohibition against guessing abbreviations is correct. The clinical contract should additionally specify:

- Meaning of `VP`, `ALR`, `FLR`, and any local abbreviations.
- Whether angle grades refer to Van Herick, gonioscopy, or another method.
- Whether cup-to-disc ratio is vertical, horizontal, or a single estimate.
- Whether A/V ratio uses a fixed representation.
- Whether lens and vitreous media are documented together or separately.
- Whether direct, indirect, slit-lamp, or lens-assisted methods must be recorded.
- Whether dilation status changes what may be asserted about the periphery.

### 4.3 Preserve “not examined” explicitly

A clinician may intentionally defer dilation, be unable to view the fundus because of media opacity, or not perform a particular test. These are clinically different from silence and from a normal result. This reinforces blocker B2.

### 4.4 Self-correction must override earlier dictation without erasing it

Indian-accent recognition is only part of the challenge. Doctors frequently say phrases such as:

> “Left eye—correction—right eye...”

The system must represent the original sequence, identify the correction, and require confirmation when laterality or a number changes. The corrected value should not merely replace earlier evidence in the audit trail.

### 4.5 Avoid semantic normalization of near-neighbour diagnoses

Several ophthalmic terms are both acoustically and clinically close. Examples include:

- Posterior subcapsular cataract versus posterior capsular opacification.
- Drusen versus exudates.
- Disc pallor versus cupping.
- Corneal arcus versus corneal scar.
- Vitreous floaters versus vitreous opacities.

If two terms remain plausible, the system must present alternatives and audio, not select the statistically common diagnosis.

### 4.6 Medication names must be handled as protected content

The current plan includes medication protection, which is correct. Extend it so the glossary may retrieve medication candidates but can never silently replace the raw term. Drug, concentration, eye, frequency, and duration require explicit clinician confirmation.

### 4.7 Add a continuation page

Assessment, plan, and instructions can exceed the fixed A4 worksheet space. Blocking export on overflow is safer than truncation but is poor clinical workflow.

Add a deterministic continuation page containing:

- Patient/encounter identifier.
- Page number.
- Overflowing section heading.
- Full text.
- Clinician attestation reference.

The first page remains visually faithful to the supplied form.

### 4.8 The initial evaluation corpus is too small for broad claims

Thirty recordings are reasonable for feasibility and an internal limited-user MVP. They are not enough to claim robust clinical performance. For the intended single ophthalmologist, expand toward 50-100 recordings across:

- Different days and fatigue levels.
- Quiet and moderate-noise rooms.
- Different recording devices and microphone distances.
- Normal and abnormal examinations.
- Short and long notes.
- Deliberate self-correction.
- Numerically dense cases.

Keep a genuinely untouched held-out group. If the tool later supports other clinicians, accents, or clinics, a broader multi-speaker validation is required.

### 4.9 Evaluate before and after clinician review

Report two distinct performance layers:

1. **Automation performance:** errors in ASR, cleanup, and extraction before review.
2. **Final-document safety:** errors remaining after clinician review.

Do not let good final-document accuracy conceal a high correction burden. Measure review time, number of corrections, audio replays, and abandonment rate.

## 5. Senior software architect review

### 5.1 Use conversion-only audio as the initial default

The architecture correctly says denoising can harm ASR, but the task plan makes high-pass filtering and loudness normalization mandatory. That is inconsistent.

Use simple format conversion as the default until the same clinician's recordings demonstrate that filtering improves critical-term recognition. Treat filtering as a versioned preprocessing profile selected by evaluation.

### 5.2 Separate the application installer from the model pack

A bundled ASR model, LLM, native runtimes, and Electron application can make a large installer and complicate updates.

Recommended packaging:

- Signed application installer.
- Versioned offline model pack.
- Signed manifest with SHA-256 hashes, licenses, minimum hardware, and compatible application versions.
- First-run verification without internet access.

This permits application security updates without repeatedly shipping multi-gigabyte models.

### 5.3 Add explicit GPU resource arbitration

Sequential execution is sensible for an unknown Helios 300, but the design needs an explicit resource lifecycle:

```text
Load ASR
→ transcribe
→ destroy ASR context
→ verify VRAM release
→ start llama.cpp
→ cleanup and extraction
```

Add tests for failed teardown and insufficient VRAM. Do not assume terminating a process immediately makes memory available to the next stage.

### 5.4 Add a native-runtime feasibility spike before committing

NeMo-Speech.cpp is a good deployment direction, but Windows packaging, GPU backend support, output format, timestamps, and redistribution must be proven on the actual Helios 300 before the repository architecture hardens around it.

Run a one-day spike that proves:

- The exact Parakeet model loads.
- The intended backend runs.
- Timestamps are available in machine-readable form.
- Redistribution licenses are captured.
- Performance and VRAM are acceptable.
- The executable can be packaged without a developer toolchain.

### 5.5 Loopback binding is not sufficient on its own

Use loopback binding, a random port, and a per-launch API key where supported. Ensure firewall prompts are not triggered. Disable permissive CORS. The renderer should never call the inference servers directly; only the main process should do so.

### 5.6 Add workstation access assumptions

The plan needs an explicit deployment assumption for:

- Dedicated Windows user account or clinic-managed account.
- Strong Windows sign-in.
- Automatic screen lock.
- BitLocker status.
- Non-administrator daily use.
- Antivirus exclusions, if any, documented narrowly.
- Backup behavior for approved records.

An offline application on an unlocked shared workstation is not private.

### 5.7 Code signing is a release requirement

“Signed or test-signed” is acceptable for internal development, not for ordinary clinic deployment. Add production code signing and verification to the release gate, or explicitly label the MVP as a supervised internal test build.

### 5.8 The task plan is too fragmented to merge without contract governance

Fifty-nine tasks are useful for smaller models, but they increase interface drift and merge overhead.

Use:

- Contract-first tasks before parallel implementation.
- One owner for schemas, one for process lifecycle, and one for state transitions.
- Short-lived branches.
- Integration after every three to five tasks.
- Phase-level end-to-end tests.
- No parallel changes to central contracts, as the plan already states.

Some tasks should be split further despite the total count:

- `P3-T07`: separate cleanup schema/prompt specification from service integration.
- `P4-T02`: separate extraction contract/prompt from live LLM integration.
- `P5-T01`: separate clinician-approved layout specification from code implementation.

### 5.9 Move evaluation and packaging risks earlier

The current dependency chain delays several risks unnecessarily:

- Start corpus collection in Phase 0 and continue throughout development; do not wait until Phase 6.
- Implement an ASR comparison harness after Phase 2, not after PDF completion.
- Create the draft A4 template immediately after the field inventory.
- Perform a Windows packaging spike after the walking skeleton.
- Test an installed offline build before prompt and glossary tuning are considered complete.

### 5.10 Clarify canonical data ownership

The reviewed structured JSON should be the canonical machine-readable case result. The PDF is a rendered clinical artifact. The raw LLM extraction is neither.

Use immutable artifacts:

- `asr.raw.json`
- `cleanup.proposed.json`
- `cleanup.reviewed.json`
- `extraction.proposed.json`
- `clinical.reviewed.json`
- `approval.json`
- `output.pdf`

Never overwrite a proposed artifact with a reviewed artifact.

## 6. Required changes to the task plan

| Existing area | Required revision |
| --- | --- |
| P0-T01 | Decide worksheet versus complete clinical note; add encounter header and examination-status semantics |
| P0-T02 | Replace nullable scalar observations with explicit observation-status objects |
| P0-T03 | Add self-correction, examination method, dilation status, and identity separation |
| P0-T04 | Prevent smaller models from authoring definitions; add medication terms as protected candidates |
| P0-T05 | Require immutable segment IDs and clinician-authored expected answers |
| New Phase 0 task | Create clinician-approved structured dictation protocol |
| New Phase 0 task | Define consent, privacy, retention, and permitted-audio policy |
| New early task | Prove NeMo-Speech.cpp and Parakeet on the actual Helios 300 |
| New early task | Build fixture-driven end-to-end walking skeleton including draft PDF |
| P2-T03 | Make conversion-only the initial default; evaluate filter profiles |
| P3-T07 | Remove LLM self-confidence from safety decisions; require review for semantic changes |
| P3-T08 | Compare immutable protected spans and segment IDs |
| P3-T09 | Require acceptance of every semantic correction before extraction |
| P4-T02 | Consume only `cleanup.reviewed.json` |
| P4-T03 | Require valid `sourceSegmentIds`; do not depend on fuzzy quote recovery |
| P4-T07 | Record clinician identity and attestation; invalidate on any clinical edit |
| P5-T01 | Add encounter header and continuation-page specification |
| P5-T03 | Render observation statuses according to clinician-approved policy |
| P6-T01 | Begin during Phase 0; confirm audio is doctor-only dictation or obtain appropriate governance |
| P6-T03 | Report critical and noncritical fields separately; report pre-review and post-review performance |
| P6-T06 | Treat 30 cases as limited feasibility evidence; require zero post-review critical errors |
| P6-T07 | Separate temporary artifact deletion from approved medical-record retention |
| P6-T09 | Add production signing or label release as supervised internal test only |

## 7. Recommended revised phase sequence

### Phase 0 — Clinical, privacy, and deployment contract

- Decide worksheet versus full clinical note.
- Define explicit observation statuses.
- Define form fields and abbreviations.
- Define structured dictation protocol.
- Define doctor-only audio policy or consultation-recording governance.
- Define temporary and clinical-record retention separately.
- Measure the Helios 300.
- Prove Parakeet and llama.cpp manually on that hardware.

### Phase 1 — Fixture-driven walking skeleton

- Secure Electron shell.
- Atomic encrypted case store.
- Fake ASR and fake LLM providers.
- Correction review screen.
- Form editor.
- Draft A4 PDF, preview, and export.
- Installed offline smoke test.

### Phase 2 — Real audio and ASR

- Import and conversion-only preprocessing.
- Parakeet integration and timestamps.
- ASR comparison harness.
- Audio/transcript review.
- Continue corpus collection.

### Phase 3 — Terminology correction

- Deterministic glossary candidate retrieval.
- LLM-proposed corrections with segment IDs.
- Protected-content guard.
- Mandatory clinician review of every semantic correction.

### Phase 4 — Extraction and clinical review

- Extract from reviewed correction artifact.
- Explicit observation statuses.
- Segment-ID provenance.
- Deterministic clinical validation.
- Clinician editing and attestation.

### Phase 5 — Final document, privacy, and packaging

- Clinician-approved vector template.
- Continuation page.
- Encrypted working data.
- Signed application and offline model pack.
- Installed offline end-to-end testing.

### Phase 6 — Evaluation and limited pilot

- Frozen development/held-out split.
- Pre-review and post-review metrics.
- Clinically weighted critical-error analysis.
- Five-case supervised UAT followed by limited pilot.
- Release only with zero unresolved post-review critical errors.

## 8. Revised safety invariants

The following should be encoded as automated tests and release gates:

1. No semantic correction reaches extraction without clinician acceptance.
2. No laterality, negation, number, acuity, pressure, medication, dose, frequency, or duration change can be auto-accepted.
3. Every model-populated field references valid immutable segment IDs.
4. `not_mentioned` is never converted into `normal` automatically.
5. Proposed and reviewed artifacts are immutable and separate.
6. Editing reviewed clinical content invalidates approval and the PDF preview.
7. The PDF cannot export from an unapproved content hash.
8. No patient identity is sent to the LLM.
9. No plaintext working case survives outside the approved encrypted boundary.
10. No patient content appears in logs, crash reports, diagnostics, or filenames.
11. Overflow creates a continuation page or blocks export; text is never truncated.
12. Every final document contains encounter and clinician attestation information.

## 9. Regulatory and governance note

This review is not legal advice. Before use with identifiable patient data, the clinic should document:

- The intended use as documentation assistance rather than diagnosis or treatment recommendation.
- Data-processing purpose and patient-facing notice where required.
- Consent or other applicable lawful basis.
- Access control, retention, correction, backup, and breach procedures.
- Whether audio contains only clinician dictation or also patient speech.
- Whether research, benchmarking, or model improvement requires separate governance.
- Whether future diagnostic or recommendation features would alter medical-device classification.

India regulates medical devices under the Medical Devices Rules, 2017; the documentation-only intended use must remain explicit, and any move toward diagnosis or treatment support should trigger a fresh regulatory assessment. [CDSCO Medical Devices Rules listing](https://cdsco.gov.in/opencms/opencms/en/Departments/Sub-Zone/Visakhapatnam/)

## 10. Final verdict

### Software architect verdict

The architecture is modular, testable, and suitable for a local Electron product. I would approve the technical direction after adding the early walking skeleton, encrypted case storage, immutable segment provenance, resource arbitration, model-pack strategy, and earlier installation testing.

### Ophthalmologist verdict

The design is acceptable as a clinician-reviewed documentation assistant, but not yet as a complete ophthalmology record. I would approve a limited supervised MVP only after explicit examination-status semantics, clinician-reviewed terminology corrections, encounter/examiner identification, a dictation protocol, and clinically weighted validation are added.

### Overall decision

**Approve with major revisions. Do not begin broad implementation from the current task plan until blockers B1 through B10 are incorporated into the architecture and task specifications.**

