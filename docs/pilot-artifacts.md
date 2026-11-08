# Pilot artifact formats (as observed 2026-11-08)

Source: `Routing (Pilot)/artifacts/`. Observed from dry-run files and file headers only. No real run log, key or score
was opened for this note. The importer must re-inspect the code in `farex_pilot/` and `farex_stage4/` before it is
written, because these formats may change at a re-freeze.

## Files per run
| File | Where | Contains condition or score? | Notes |
|---|---|---|---|
| `generic_v2.<RUN>.manifest.json` | `artifacts/runs/` | no per-cell data | `run_id, suite, stage, dry_run, started, ended, aborted, abort, git, hashes (23 files), python, os, scorer_version, execution_order, sandbox, note`. Stage 4 adds `stage4, model, provider, options, seed_supported, parent_run_id, workers, calls_sent, cells_completed, spent_usd_micro`. |
| `generic_v2.<RUN>.jsonl` | `artifacts/runs/` | YES | One row per call. Keys: `run_id, schema_version, row_id, row_type, task_id, task_type, condition, reference_destination, selected_destination, system_prompt_file, system_prompt_sha256, task_sha256, response, status, score_status, scored, pass, components, reason, scorer_version, scorer_sha256, latency_ms, prompt_tokens, completion_tokens, token_source, model, model_digest, attempt, is_warmup, error, error_class, retry_eligible, dry_run, timestamp`. Stage 4 adds `http_status, reasoning_content, reasoning_tokens, returned_model, spend_usd_micro, spend_source`. |
| `review_sheet.<RUN>.csv` | `artifacts/review/` | no | Columns: `review_id, task_prompt, what_counts_as_correct, response, human_correct, human_compliant, human_notes`. This is the only blinded input. |
| `review_key.<RUN>.json` | `artifacts/private/` | YES | Maps `review_id` to the cell (condition, destination). Must not be read until reveal. |
| `*_reviewed.csv` | `artifacts/review/` | no | Sheet with marks filled in. In existing runs the marks came from an AI. |
| `*.report.md`, `*.human_review.md` | `artifacts/runs/` | YES | Generated after join. Gitignored upstream. |
| `DRYRUN-*` | `artifacts/dry_run/` | n/a | Synthetic canned answers. Not results. |

## Consequences for the workbench
- The review screen can be built from the sheet CSV alone. The jsonl and key are loaded only after reveal.
- The review sheet has `human_*` column names even when an AI filled them. The workbench must record who marked.
- Run kind must be derived from the manifest (`dry_run`, `stage`, `aborted`, mock flags) and shown, never assumed.
- A review sheet has no latency, tokens or condition, which matches the blinding requirement. Response text can still hint at the condition.
