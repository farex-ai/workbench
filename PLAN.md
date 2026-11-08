# Farex Workbench: plan (V1)

Direction decided by the owner 2026-11-08: an artifact-driven routing-evaluation workbench, local-first.
Spec revised 2026-11-08 (full screens 1-6, boundaries, acceptance). This plan stops before implementation.

## 1. Question and scope
"When the same task is sent to a different specialist agent, what changes in correctness, instruction compliance,
latency, and token usage?"

V1 turns existing routing-pilot artifacts into an interactive, reviewable experiment report. It consumes runs; it does
not execute experiments, call models or APIs, run generated code, or add a sandbox.

Not building: chatbot, general assistant, agent builder, "another Remy", leaderboard, production observability,
clinical tool, or a marketing page instead of the workbench. No auth, billing, teams, cloud database or telemetry.
Notion is a project-summary layer only, never the source of truth.

User: the owner, as the researcher running the pilot. Brand: the approved Farex "Trace" kit v0.2 exists
(`../branding/kit/tokens.css`, dark default, light theme, status colors, tabular figures). Use it; vendor a copy of the
tokens and fonts into the app (no CDN).

## 2. Inspected input schemas
Read-only inspection of `../Routing (Pilot)/` on 2026-11-08, from dry-run files, file headers, the task suite and
`docs/scoring.md`. No real run log, key or score was opened. The importer must re-check against
`farex_pilot/` and `farex_stage4/` code and each bundle's own `schema_version` when it is built.

| Artifact | Path (pilot) | Content | Reveals condition or score? |
|---|---|---|---|
| Manifest | `artifacts/runs/generic_v2.<RUN>.manifest.json` (dry runs: `artifacts/dry_run/`) | `run_id, suite, stage, dry_run, started, ended, aborted, abort, git, hashes (23 files), python, os, scorer_version, execution_order, sandbox, note`. Stage 4 adds `stage4, model, provider, options, seed_supported, parent_run_id, workers, calls_sent, cells_completed, spent_usd_micro`. | No per-cell data |
| Run log | `...<RUN>.jsonl`, schema `pilot-row-2.0.0` | One row per attempt. `row_type` is `warmup`, `worker` or `router`. Keys: `run_id, row_id, call_id, attempt, is_warmup, task_id, task_type, condition, reference_destination, selected_destination, system_prompt_file, system_prompt_sha256, task_sha256, response, response_source, status, scored, score_status, pass, components, reason, scorer_version, scorer_sha256, latency_ms, prompt_tokens, completion_tokens, token_source, model, model_digest, options, error, error_class, retry_eligible, router, router_outcome, dry_run, timestamp`, plus Stage 4: `http_status, reasoning_content, reasoning_tokens, returned_model, spend_usd_micro, spend_source`, and Ollama timing fields locally. | Yes (condition, destination, score) |
| Review sheet | `artifacts/review/review_sheet.<RUN>.csv` | `review_id, task_prompt, what_counts_as_correct, response, human_correct, human_compliant, human_notes` | No. The only blinded input. |
| Review key | `artifacts/private/review_key.<RUN>.json` | `{run_id, results_file, sheet, mapping}`; `mapping[review_id] = {row_id, call_id, task_id, condition}` | Yes |
| Reviewed sheet | `..._reviewed.csv` | Same columns with marks filled. Existing ones were marked by AI although the columns say `human_*`. | No |
| Task suite | `tasks/generic_v2.json` | `{suite, status, source, tasks[40]}`; each task: `id, type, prompt, prompt_original, requirement, expected, expression` | Task definitions (needed for reference after reveal) |
| Freeze files | `freeze/*.freeze.json` | Hashes of frozen files | Provenance |
| Reports | `*.report.md`, `*.human_review.md` | Generated after join | Yes. Not imported; the workbench computes its own. |

Facts observed in dry-run logs that the design depends on:
- Conditions seen: `reference`, `cross:<type>` (fixed wrong), `router`, and a `warmup` row. Spec comparisons use `reference` versus `cross:*`; `router` rows are shown separately (router vs worker measurements stay apart).
- Component scores live in `components` (`correct`, `compliant`, plus task-type extras). `pass` is the scorer's overall rule; `score_status` can be non-`scored` (review-needed etc.); `error_class` marks transport or sandbox problems.
- Tokens carry `token_source`; some values can be missing. Show missing explicitly.
- `attempt` and `retry_eligible` exist, so a logical call (`call_id`) can have several attempt rows. Rule for the final outcome (assumed: highest `attempt` row per `call_id`) must be verified against the runner code during milestone 1 and written into the docs.
- Task's declared overall pass rule: the scorer's `pass` per task type (`docs/scoring.md`); the workbench shows `pass` as recorded and never recomputes it.

Run kind classification (manifest-driven, shown on every run): **dry run** (`dry_run: true` or `DRYRUN-` id),
**mocked run** (mock provider or synthetic `response_source`; confirm the exact markers in `farex_stage4/mock.py`),
**partial run** (`aborted`, or `cells_completed` below expected, or missing cells), **real model experiment** (none of the
above, with provider, model and digest recorded). Unknown stays "unclassified", never defaulted to real.

## 3. Stack and storage
- **Backend:** Python 3 (the pilot runs 3.14), standard library only: `http.server`, `sqlite3`, `csv`, `json`, `hashlib`. Single local process.
- **Frontend:** plain HTML, CSS, JS modules, no build step, no CDN, no framework. Brand tokens vendored from the Trace kit.
- **Why:** fewest moving parts, matches the pilot language, nothing to install, easy to audit that nothing leaves the machine.
- **Network posture:** bind 127.0.0.1 on a random port; check Host and Origin; per-session token on write requests; no outbound requests; strict CSP (`default-src 'self'`, no inline scripts).
- **Untrusted content:** model responses, reasoning text and notes are rendered with `textContent` inside `<pre>`; never as HTML, never evaluated, never linked. Nothing is executed.

**Workspace (gitignored, local):**
```
workspace/
  farex.db                      SQLite: runs, reviewers, judgments, reveals, exports
  runs/<run_id>/<bundle_sha256>/raw/   byte-for-byte copies, mode 0444, SHA-256 recorded
  runs/<run_id>/<bundle_sha256>/sealed/ review key copy; opened only by the reveal module
  backups/                      timestamped copies of farex.db
  exports/                      generated packages
```
- **Association:** every judgment row stores `run_id`, `bundle_sha256` (hash of the imported manifest+log+sheet set), `review_id`, `reviewer_id`, and the sheet hash. Re-importing a changed bundle creates a new bundle record; judgments never attach to a different hash.
- **Judgments are append-only:** each save inserts a row; the latest per (reviewer, review_id) wins; history stays. Fields: `correct` (yes/no/unjudgeable), `compliant` (yes/no/unjudgeable), `note`, `saved_at`, `post_reveal` flag.
- **Reviewer identity:** `reviewer_id`, name, `kind` (human or ai). "Human-reviewed" shown only when a human reviewer completed the sheet. Existing AI-marked sheets import as AI marks.
- **Backup:** the app copies `farex.db` to `backups/` on app start, before reveal, and on every export (keep the last 20). Judgment CSV export is the portable backup. Documented in the README.
- **Raw files are never modified.** Revised analyses and corrections are stored as separate records (for example a note that `coding_h6` was ruled a slip), shown beside the original, never replacing it.

## 4. Screen map and data flow
```
Overview ──> Import & validate ──> Blinded review ──> [Reveal gate] ──> Results ──> Task drill-down ──> Export
   ^              |                     |                  |              |             |
   └─ next action ┘            sheet CSV only        key + log unsealed   log + key     source records
```
- **Overview** (`GET /api/projects`): question text, stage, run list with kind badge and review status, one visible next action ("Review 12 remaining responses", "Run has 3 incomplete cells", "Reveal when ready").
- **Import & validate:** user selects a bundle (file/folder picker or typed path). Checks, each named with pass/fail/not run: required files present, JSON/CSV parse, `schema_version` known, manifest hashes match, `run_id` consistent across files, sheet rows equal key mapping, every mapped `row_id` exists in the log, incomplete cells, aborted, scorer version. Shows provenance (git, model, digest, prompt hashes, freeze). Result is "imported" plus a list of checks. The word "verified" appears only next to a specific named check that passed.
- **Blinded review:** payload comes only from the sheet via an allowlist (`review_id, task_prompt, what_counts_as_correct, response`) and is served from a module that cannot read the log or key. Controls: correct, compliant (yes / no / unjudgeable), note. Autosave, resume at first unjudged, fixed order stored per run. Banner: response style can hint at the condition; blinding is partial.
- **Reveal gate:** owner presses "End review", sees counts judged/unjudgeable/missing, confirms. Writes a reveal record, opens the sealed key, loads the log. Later edits are marked `post_reveal`. Reveal is not undone silently; an explicit "reopen" is recorded.
- **Results:** reference vs fixed-wrong tables with correctness, compliance and the task's `pass` kept separate; counts with denominators (as "n of N") and percentages only beside them; automated vs human vs disagreements as separate columns; unscored, review-needed, transport-error and sandbox-error kept as visible rows; latency and provider tokens with "missing" explicit; router and worker separated. No dollar cost, no significance tests, no generalization text.
- **Task drill-down:** per task, the reference and wrong-route responses side by side; selected destinations, prompt file and hash, attempts list versus final outcome, errors, scorer components and reason, human marks and notes; each figure links to its source record (file, line, `row_id`).
- **Export:** zip with `judgments.csv`, `judgments.json`, `report.md` (methods, outcomes, limitations, artifact references with hashes), `manifest.json` (inputs, versions, reveal record, reviewers). Partial runs and revised analyses are labeled in the report. The blinded export (sheet plus judgments before reveal) contains only allowlisted fields. Deterministic output apart from stated timestamps.

## 5. Implementation sequence (small commits each)
1. **Importer core + fixtures.** Readers for manifest, jsonl, sheet, key, tasks; run-kind classifier; named validation checks. Synthetic fixtures copied/derived from dry-run bundles only.
2. **Workspace + storage.** Raw copy with hashes and read-only mode; SQLite schema; backups.
3. **Blinded review service.** Allowlist payload, append-only judgments, resume, reviewer identity. Leak test lands here.
4. **UI shell + Import screen + Review screen** (brand tokens), then Overview.
5. **Reveal gate + Results.** Metrics module with counts and denominators; attempts vs final outcome.
6. **Drill-down + disagreements + source links.**
7. **Export package + report generator**, determinism test.
8. **Docs** (README: storage, backup, blinding limits) and a history-free public refresh when the owner says.

## 6. Acceptance tests (map to the owner's 7 criteria)
1. **Import without editing:** hash of each source file before and after import is identical; workspace copy hash equals source; copies are read-only.
2. **Run kind:** fixtures for dry, mocked, partial (aborted and incomplete) and real-shaped manifests classify correctly; unknown manifests are "unclassified"; a successful import never produces a "verified" label.
3. **Blind review and resume:** kill and restart the app mid-review; judgments persist and the next unjudged item opens; correct and compliant save independently; unjudgeable plus note works.
4. **Reveal:** Results endpoints return 403 before reveal; the review payload and blinded export contain none of: `condition`, `selected_destination`, `reference_destination`, `pass`, `components`, `score_status`, latency, token fields, `system_prompt_file` (allowlist test plus a deliberate-leak test that must fail loudly).
5. **Traceability:** every metric in Results has a drill-down listing its row ids; each row id resolves to a line in the raw log; totals equal denominators.
6. **Disagreements:** a fixture with known scorer-versus-human mismatches lists exactly those, each with scorer components and the human note.
7. **Export:** package contains judgments, report, manifest; re-export on the same inputs is byte-identical except stated timestamps; raw logs untouched; partial runs carry a "partial" label.
Also: responses containing `<script>`, HTML and markdown render as inert text; no outbound network calls (test with a socket guard); no write outside `workspace/`.

## 7. Honesty rules carried over
- Stage 2, 3 and 4 marks were made by AI; the human-review gate is not met. UI and exports say "AI-marked".
- No real result figures in the repo, fixtures, docs or screenshots. Build against dry-run and synthetic data only.
- The workbench is procedural blinding in one local process, not security against its own user; document that.

## 8. Blocking questions
None block milestones 1-3. These matter before the first real-data review:
1. **Blinding is already spent for the existing real runs.** You have seen the Stage 3 results (and Stage 4 was joined). A review by you of those sheets is not blind. Options: (a) a second reviewer who has not seen results marks them; (b) review them anyway and label the review "reviewer had seen results"; (c) wait for a new run. Which?
2. Should V1 support more than one reviewer per run (needed to compare the AI marks with a human)? Plan assumes yes (reviewer table exists; UI shows one at a time).

## Status
2026-11-08: plan complete for review; no application code. Public repo `farex-ai/workbench` (MIT), plan and docs only.
