# Farex Workbench

Local-first research workbench for evaluating routing failures in multi-agent AI systems. Turns routing-pilot artifacts
(`../Routing (Pilot)/artifacts/`) into a blinded-review, reveal, compare and export workflow. Read `PLAN.md` first, then
root `../CLAUDE.md` session rules (they apply here).

## Decisions
- Direction fixed by the owner 2026-11-08: single-user, local-first, no network, no uploads, not a chatbot or agent builder.
- Import never means "verified". Raw imported files stay unchanged.
- Blinded payloads use an allowlist of fields. Condition, destination, scores, latency and tokens never reach the review screen or its export.
- Existing Stage 2, 3 and 4 marks are AI marks; say so in the UI and docs.

## Rules for this folder
- Build and test on dry-run or synthetic bundles only. Do not open real run logs, keys or scores before review is joined.
- No real result figures in code, fixtures, docs or screenshots.
- Small real commits; push only when the owner says so, and public copies come from a clean export.

## Status (2026-11-08)
Plan written (`PLAN.md`), artifact formats noted (`docs/pilot-artifacts.md`). No code. Public at github.com/farex-ai/workbench (MIT), pushed 2026-11-08 with the owner's OK.
