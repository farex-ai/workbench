# Farex Workbench: plan

Decided by the owner 2026-11-08. Local-first research workbench for evaluating routing failures in multi-agent AI systems.
Sibling of `../Routing (Pilot)/` (the experiment tool). This plan is not a brainstorm; scope below is fixed unless the owner changes it.

## Question the product answers
"When the same task is sent to a different specialist agent, what changes in correctness, instruction compliance, latency, and token usage?"

V1 turns existing routing-pilot artifacts into an interactive, reviewable experiment report.

**Not in scope:** chatbot, general-purpose agent, agent builder, production monitoring, hosted service, accounts, uploading anything.

## User
The owner, as the researcher running the Farex pilot. Understandable at a glance, with the underlying evidence one click away.

## Core workflow
Import an existing run -> check validity and completeness -> review responses blind -> save judgments ->
explicitly reveal -> compare conditions and inspect disagreements -> export a reproducible review/report package.

## Screens (V1)

### 1. Project overview
- Plain-language statement of the research question.
- Current experiment stage.
- Imported runs with review status.
- Run kind shown for every run, with four distinct labels: **dry run**, **mocked run**, **partial run**, **real model experiment**. Derived from the manifest (`dry_run`, `stage`, `aborted`, provider/mock fields), never guessed.
- Next action, e.g. "Review 12 remaining responses."

### 2. Import and validate
- User picks one artifact bundle from the pilot (a file or folder chosen explicitly). No scanning of the machine, no uploads, no network.
- Importer is written only after re-inspecting the current schemas (`docs/pilot-artifacts.md` is the starting note, not the spec).
- Reports: missing files, `schema_version` / `scorer_version` mismatches, incomplete cells, aborted runs, hash mismatches against the manifest, available provenance (git, model, digest, prompt hashes).
- An import that succeeds is "imported", never "verified". "Verified" is reserved for named checks that passed, listed by name.
- Raw imported files are copied byte-for-byte into the workspace and never modified (store a SHA-256 for each).

### 3. Blinded review
- Shows task, response, and the requirement/reference needed to judge (the sheet's `task_prompt`, `what_counts_as_correct`, `response`).
- Records **correct** and **instruction-compliant** separately. Each: yes / no / unjudgeable, with a free-text note.
- Hides condition, destination, automated score, latency, tokens.
- The review payload is built only from blinded fields (allowlist, not a denylist). Condition-revealing fields are never sent to this screen and never written to review exports. Tests assert this.
- Autosave on every judgment; resume where left off; order is fixed per run and recorded.
- Banner: response style may reveal the condition (e.g. code returned for a non-coding task). Blinding is partial, not perfect. Do not claim otherwise anywhere.
- Records who marked (`reviewer` field, free text, plus `kind`: human or AI). Required so "human-reviewed" is never claimed for AI marks.

### 4. Results (locked until reveal)
Source text of the owner's spec for this screen ended after "Show reference versus fixed-wrong outcomes." The rest is derived from the core workflow and **needs owner confirmation**:
- Locked until the owner explicitly ends blinded review and confirms reveal. Reveal is recorded with a timestamp and cannot be silently undone; judgments made after reveal are flagged as post-reveal.
- Reference versus fixed-wrong outcomes: correctness, compliance, unjudgeable counts, latency, tokens (completion tokens separately from prompt tokens).
- Uncertainty shown (counts and intervals, small-n warnings). A null result is displayed as a result.

### 5. Compare and disagreements (from the workflow; confirm)
- Per task and per condition, side by side.
- Disagreements between the owner's/reviewer's marks and the automated scorer, each openable to the full evidence (response, requirement, scorer components and reason).
- Where several reviewers exist (e.g. AI and human), show their agreement.

### 6. Export (from the workflow; confirm)
- A reproducible package: manifest of inputs with hashes, judgments with reviewer and timestamps, reveal record, workbench version, and the generated report. Re-running the export on the same inputs gives the same bytes apart from stated timestamps.
- The blinded review export contains no condition-revealing fields.

## Honesty rules carried over
- Existing Stage 2, 3 and 4 marks were made by AI. The human-review gate is not met. UI and exports say "AI-marked" for them.
- Nothing from a real run leaves the machine without the owner's explicit OK. No result figures in the repo, fixtures, docs or screenshots; use dry-run or synthetic data.
- Stay blinded while building: develop against dry-run bundles only; do not open real run logs, keys or scores before review is joined.

## Proposed technical approach (a default; change if the owner wants)
- Python 3 backend using the standard library where possible, serving a local-only UI on 127.0.0.1. Same language as the pilot, so the importer can reuse its scoring and join code by reading, not by importing frozen files.
- Plain HTML, CSS and JS for the UI, with no build step to start. No telemetry, no CDN loads, no network calls.
- Local workspace folder (`workspace/`, gitignored) holding copied raw bundles, an SQLite review database, and exports.
- Tests with `unittest`, including a leak test that fails if any blinded payload or review export contains a condition-revealing key.

## Milestones (each ends in small commits)
1. **Importer core.** Re-inspect pilot schemas; manifest and jsonl readers; run-kind classifier; validation checks with named results. Tests on dry-run bundles.
2. **Workspace storage.** Raw copy with hashes, review database, run registry.
3. **Blinded review backend.** Allowlist payload, save and resume, reviewer identity, leak test.
4. **Review UI.** Screens 2 and 3 first, then 1.
5. **Reveal and results.** Reveal gate, join to key, reference versus fixed-wrong tables.
6. **Compare and disagreements.**
7. **Export package** with reproducibility test.
8. **Docs and clean export** for a public repo through a history-free copy and a leak scan, only on the owner's go.

## Open decisions for the owner
1. Screens 4 to 6 above were inferred; please confirm or send the rest of the spec (the message ended mid-screen 4).
2. Repo name and whether to create it under `farex-ai` (suggested: `farex-ai/workbench`), and whether it starts private.
3. Reviewer identity: is a single reviewer enough for V1?
4. Whether AI-marked sheets can be imported as "AI-marked review" for comparison with a later human review (proposed: yes).

## Status
2026-11-08: plan written; no code yet. Local git repo, no remote.
