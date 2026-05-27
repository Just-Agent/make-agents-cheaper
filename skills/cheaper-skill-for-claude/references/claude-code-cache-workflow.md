# Claude Code Cache-Friendly Workflow

## What This Adapter Does

This adapter uses Claude Code's own CLI surface to make the early prompt prefix more stable:

```bash
--exclude-dynamic-system-prompt-sections
```

It does not patch Claude Code internals. It asks Claude Code to place dynamic machine-local sections later in the request while preserving their information.

## Why It Can Help

Prompt caching generally rewards stable prefixes. In coding-agent runs, the early request often contains a large repeated prefix:

- system and developer instructions
- tool schemas
- repo rules
- stable project context
- conversation/session framing

If volatile fields appear early, the prefix shape can change before the provider reaches the cacheable region. Moving those volatile fields later can improve the chance that the stable prefix matches a previously cached prefix.

## Basic Command Forms

Interactive:

```bash
claude --model mimo-v2.5-pro --exclude-dynamic-system-prompt-sections
```

JSON measurement:

```bash
claude -p \
  --model mimo-v2.5-pro \
  --output-format json \
  --no-session-persistence \
  --exclude-dynamic-system-prompt-sections \
  "Summarize the repository structure."
```

Baseline for comparison:

```bash
claude -p \
  --model mimo-v2.5-pro \
  --output-format json \
  --no-session-persistence \
  "Summarize the repository structure."
```

## Fair A/B Protocol

Use paired warm-up before interpreting measured results.

1. Choose one provider, one model, one working directory, one tool/MCP/hook setup, and one prompt.
2. Run baseline once as warm-up.
3. Run candidate once as warm-up.
4. Run baseline again and save JSON.
5. Run candidate again and save JSON.
6. Repeat for at least 3 to 5 prompts in the same task suite.
7. Add a dynamic drift suite where cwd/git/env-like state changes between tasks.
8. Compare cached input tokens, uncached input tokens, output tokens, latency, and task correctness.

The key fairness rule: do not compare a warm baseline against a cold candidate. A changed prefix shape often pays a one-time cache creation cost before later runs can benefit.

## Dynamic Drift Suite Ideas

Use tasks that keep the high-level workflow stable but change volatile local state:

- edit a small file, then ask for a targeted review
- create an untracked fixture, then run a repo summary
- switch between sibling task folders
- vary a task-specific environment variable
- update git status while keeping the same model and command shape

This suite tests whether the candidate is robust when local dynamic sections drift.

## Implementation Surface

This is a plugin/skill-layer workflow, not a model-layer change:

- model-side work makes tokens cheaper through training, inference kernels, KV cache, or serving infrastructure
- harness-side work makes repeated input cheaper by shaping prompt/cache behavior around the existing model/provider API
- this adapter sits at the harness layer and uses Claude Code's supported CLI flag

## What Not To Do

- Do not remove context required for correctness.
- Do not reorder tool execution traces after the fact.
- Do not change tools/MCP/hook configuration during an A/B run.
- Do not switch models mid-experiment.
- Do not report provider price savings when only token usage was measured.
