# Farex Workbench

A local-first research workbench for evaluating routing failures in multi-agent AI systems.

It helps answer one question: when the same task is sent to a different specialist agent, what changes in correctness,
instruction compliance, latency and token usage?

Version 1 will import artifacts from the [Farex routing pilot](https://github.com/farex-ai/farex), let you review responses
blind, save your judgments, reveal the conditions only when you choose, compare outcomes, and export a reproducible
report package. Everything stays on your machine.

It is not a chatbot, an agent builder or a production monitoring tool.

**Status:** planning. There is no code yet. See [PLAN.md](PLAN.md).

Blinding is partial: a response's style can hint at which agent produced it. Imported runs are never labelled
"verified" just because they loaded.
