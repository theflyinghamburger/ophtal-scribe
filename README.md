# Ophthalmology Dictation Assistant (`ophtal-scribe`)

A fully local, offline Electron desktop app for 64-bit Windows that converts an ophthalmologist's
prerecorded dictation into one specific A4 ocular-health worksheet PDF. No cloud, no remote
inference, no microphone capture in the MVP.

It is a **worksheet-filling assistant** — not a general clinical-note system, not an EMR, and not a
diagnostic or treatment-recommendation system.

> **Status: Phase 0 — clinical contract.** This repository currently holds design documents only.
> No implementation has begun; there is no `package.json` and no source tree yet.

## Pipeline

```
import audio → canonical 16kHz mono WAV (FFmpeg) → Parakeet ASR →
deterministic glossary candidate generation → LLM cleanup → LLM extraction →
deterministic validation → single editable final worksheet → approve → PDF → purge
```

## The four constraints that shape everything

1. **No PII, at all.** The app accepts no patient name, DOB, address, phone, clinic or encounter
   identifier, or clinician identity. The privacy strategy is *not collecting* identifiers rather
   than encrypting them.
2. **Sessions are ephemeral.** No database, no case history. Everything lives in a randomly-named
   temporary session directory that is purged after successful export.
3. **One human review gate.** The clinician reviews the completed worksheet once, immediately before
   export. Individual LLM corrections are never separately accepted or rejected.
4. **The LLM proposes; deterministic code and the clinician decide.** LLMs do exactly two jobs —
   terminology cleanup and structured extraction — both at `temperature: 0` under a JSON schema
   constraint. Validation, protected-content checks, form mapping, and PDF rendering are
   deterministic TypeScript.

## Documents

| Document | What it covers |
| --- | --- |
| [`docs/architecture.md`](docs/architecture.md) | The system definition: worksheet boundary, observation statuses, protected content, provenance, process and IPC boundaries, hardware profiles, failure policy. |
| [`docs/mvp-implementation-task-specification.md`](docs/mvp-implementation-task-specification.md) | 60 tasks across phases P0–P6, each with inputs, requirements, outputs, validation, and acceptance criteria. |
| [`CLAUDE.md`](CLAUDE.md) | Working guidance for coding agents in this repo. |
| [`docs/agents/`](docs/agents/) | Issue-tracker, triage-label, and domain-doc conventions. |
| [`docs/archive/`](docs/archive/) | Superseded architecture and clinical review notes. |

## Work tracking

Every task in the task specification has a GitHub issue titled `<task id> — <task title>`, carrying:

- a `phase:<n>` label and a `Phase <n>` milestone;
- a `class:S|M|C` capability label, which gates **review**, not authorship —
  **S** normal review, **M** senior engineering review, **C** ophthalmologist *and* senior
  engineering review;
- native issue dependencies mirroring the spec's `Depends on` list, so an issue with
  `blocked_by > 0` is not yet startable.

Phase exit gates are issues labelled `exit-gate`, blocked by every task in their phase.

```bash
# tasks that are ready to start right now
gh issue list --state open --json number,title,labels,issueDependenciesSummary \
  --jq '.[] | select(.issueDependenciesSummary.blockedBy == 0) | "\(.number) \(.title)"'
```

## Clinical content is authored, not generated

Field definitions, glossary terms, permissible ranges, correction rules, and golden-fixture expected
answers come from the ophthalmologist. Encoding an approved specification is fine; inventing one is
not. Form-specific abbreviations (`VP`, `ALR`, `FLR`, the angle grading method, C/D notation) stay
`requires_clinician_definition` until confirmed.

No patient audio, transcripts, worksheet values, or model weights are ever committed to this
repository.
