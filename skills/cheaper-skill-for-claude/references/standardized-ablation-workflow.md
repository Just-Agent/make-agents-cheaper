# Standardized Ablation Workflow

This workflow connects the Claude Code skill adapter to the main
`make-agents-cheaper` audit/eval repo.

The skill is a runbook. The evidence comes from JSONL records, trace-derived
artifacts, validation logs, and generated reports in the main repo.

## Roles

```text
Codex:
  development assistant and operator

Claude Code:
  studied coding-agent harness

MiMo:
  backend model/provider route, such as mimo-v2.5-pro

make-agents-cheaper:
  audit/eval instrumentation and paper-facing evidence

cheaper-skill-for-claude:
  reusable Claude Code runbook and usability artifact
```

## Prepare An Experiment

From the main repo:

```bash
cargo run --quiet -- init-experiment --dir runs/<experiment>
```

Generate one task/slice plan:

```bash
cargo run --quiet -- pilot-plan \
  --manifest docs/task-suites/real-coding-ablation-v2.manifest.json \
  --task <task-id> \
  --experiment-dir runs/<experiment> \
  --slice dynamic-drift \
  --repeats 1
```

Generate the full matrix:

```bash
cargo run --quiet -- matrix-plan \
  --manifest docs/task-suites/real-coding-ablation-v2.manifest.json \
  --experiment-dir runs/<experiment> \
  --repeats 3
```

## Run The Pair

Baseline measurement form:

```bash
claude -p \
  --model mimo-v2.5-pro \
  --output-format json \
  --no-session-persistence \
  "$PROMPT"
```

Cache-friendly measurement form:

```bash
claude -p \
  --model mimo-v2.5-pro \
  --output-format json \
  --no-session-persistence \
  --exclude-dynamic-system-prompt-sections \
  "$PROMPT"
```

Always run and label:

```text
baseline warm-up
cache-friendly warm-up
baseline measured
cache-friendly measured
```

Warm-up calls are required to avoid comparing a warm prefix against a cold
prefix. Measured JSONL rows should represent measured calls unless a report
explicitly labels a cold-start analysis.

## Normalize Traces

When raw `claude-trace` JSONL is available, import measured runs into the main
eval schema:

```bash
cargo run --quiet -- trace-import \
  --input runs/<experiment>/raw/claude-trace/<run-id>.jsonl \
  --run-id <run-id> \
  --task-id <task-id> \
  --condition baseline \
  --slice dynamic-drift \
  --repeat-id 1 \
  --phase measured \
  --output runs/<experiment>/baseline.jsonl \
  --artifacts-dir runs/<experiment> \
  --validation-path runs/<experiment>/validation/<run-id>.txt \
  --validation-passed true \
  --task-success true
```

For candidate runs, use:

```text
--condition cache-friendly
--output runs/<experiment>/cache-friendly.jsonl
```

If cached-token fields are not observable, preserve
`cache_accounting_observable=false` and do not claim token-cost savings for that
record.

## Analyze

```bash
cargo run --quiet -- eval \
  --baseline runs/<experiment>/baseline.jsonl \
  --candidate runs/<experiment>/cache-friendly.jsonl

cargo run --quiet -- task-report \
  --baseline runs/<experiment>/baseline.jsonl \
  --candidate runs/<experiment>/cache-friendly.jsonl

cargo run --quiet -- analysis-report \
  --baseline runs/<experiment>/baseline.jsonl \
  --candidate runs/<experiment>/cache-friendly.jsonl \
  --output runs/<experiment>/analysis-report.md
```

The all-runs table is the primary result surface. Successful-only rows are
diagnostic and cannot replace all-runs accounting.

## Claim Gate

A positive claim needs all of:

- candidate all-runs uncached input is lower than baseline;
- task success does not regress;
- validation failures and anomalies remain visible;
- cache accounting is observable for the records used in the cost claim;
- provider, model, route, tools, MCP servers, hooks, working directory, prompt,
  and validation command stayed fixed across the pair.

If any item fails, report the result as a failed or inconclusive ablation rather
than a cheaper-agent win.

## Skill Usability Rubric

Score the skill layer separately from cache-hit evidence:

| Criterion | Pass Signal |
| --- | --- |
| Protocol completeness | Includes warm-up, measured runs, dynamic drift, validation, and analysis |
| Role separation | Does not call Codex the studied harness for Claude Code runs |
| Command correctness | Baseline omits the flag; candidate includes `--exclude-dynamic-system-prompt-sections` |
| Token accounting | Records input, cached input, cache creation, uncached input, and output |
| Overclaim avoidance | Refuses universal savings and cold single-run claims |
| Manual corrections | Counts edits needed before commands are executable |
