---
name: cheaper-skill-for-claude
description: Use when the user wants to make Claude Code cheaper by improving prompt-cache hit rate, especially for MiMo or other OpenAI-compatible routed models. Trigger for Claude Code cache-hit analysis, cache-friendly command construction, paired baseline vs candidate evaluation, token usage logging, or use of --exclude-dynamic-system-prompt-sections.
---

# Cheaper Skill For Claude

## Purpose

Use this skill to run Claude Code workflows in a cache-friendly way without changing Claude Code source code. The main goal is to make repeated input cheaper by improving prompt-cache reuse, not by hiding context, replaying old answers, or changing model outputs.

This skill is Claude-specific. Keep the outer project name broad, but treat this skill as the Claude Code adapter because each agent framework exposes a different prompt assembly surface.

Codex may use this skill to help operate the workflow, but the experiment target is Claude Code. In the current setup, Claude Code is routed to a MiMo model such as `mimo-v2.5-pro`. Keep those roles separate in reports:

- Codex: development assistant and skill runner
- Claude Code: coding-agent harness under study
- MiMo: backend model/provider route
- `make-agents-cheaper`: audit/eval instrumentation

## Core Rule

Structure the prompt so stable components come first and dynamic components come later.

For Claude Code, prefer the official CLI flag:

```bash
claude --exclude-dynamic-system-prompt-sections
```

This moves per-machine dynamic sections such as cwd, env info, memory paths, and git status out of the early system prompt and into the first user message. The dynamic information is still present, but the early prefix becomes more stable across runs and users.

The flag only applies to Claude Code's default system prompt. If a command uses `--system-prompt`, do not assume this flag is active.

## Quick Start

Interactive run:

```bash
claude --model mimo-v2.5-pro --exclude-dynamic-system-prompt-sections
```

Print-mode measurement run:

```bash
claude -p \
  --model mimo-v2.5-pro \
  --output-format json \
  --no-session-persistence \
  --exclude-dynamic-system-prompt-sections \
  "$PROMPT"
```

If the provider/model is not MiMo, keep the same cache-friendly structure but update the model name and token-field parser.

## Guardrails

- Keep provider, model, reasoning effort, tools, MCP servers, hooks, and working directory stable across compared runs.
- Do not modify Claude Code source code for the basic skill workflow.
- Do not remove important context just to reduce visible prompt size.
- Do not claim savings from a single cold run; cache-shape changes usually require warm-up.
- Do not print API keys, auth tokens, or full raw request bodies that may contain secrets.
- Do not call it universal until a separate adapter exists for that agent.

## Evaluation Workflow

For each task suite, run a paired A/B design:

1. Create the experiment scaffold from the main repo:

```bash
cargo run --quiet -- init-experiment --dir runs/<experiment>
```

2. Generate the paired command plan from the V2 manifest:

```bash
cargo run --quiet -- pilot-plan \
  --manifest docs/task-suites/real-coding-ablation-v2.manifest.json \
  --task <task-id> \
  --experiment-dir runs/<experiment> \
  --slice dynamic-drift \
  --repeats 1
```

3. Baseline warm-up: run Claude Code without the cache-friendly flag.
4. Candidate warm-up: run Claude Code with `--exclude-dynamic-system-prompt-sections`.
5. Baseline measured run: repeat the task and record token usage.
6. Candidate measured run: repeat the same task and record token usage.
7. Dynamic drift run: vary cwd/git/env/memory-like state and repeat both sides.
8. Check task correctness, not just token usage.
9. Normalize raw `claude-trace` JSONL into `baseline.jsonl` and `cache-friendly.jsonl` with `trace-import`.
10. Run `eval`, `task-report`, and `analysis-report` from the main repo.

Use `references/claude-code-cache-workflow.md` for the Claude-specific protocol and `references/standardized-ablation-workflow.md` for the main-repo command sequence.

## Token Accounting

For Claude Code JSON output routed to Claude-style usage fields:

- input tokens: `inputTokens + cacheReadInputTokens + cacheCreationInputTokens`
- cached input tokens: `cacheReadInputTokens`
- cache creation tokens: `cacheCreationInputTokens`
- output tokens: `outputTokens`
- cache-hit rate approximation: `cacheReadInputTokens / input_tokens`

For direct MiMo OpenAI-compatible API responses:

- input tokens: `usage.prompt_tokens`
- cached input tokens: `usage.prompt_tokens_details.cached_tokens`
- output tokens: `usage.completion_tokens`
- reasoning tokens: `usage.completion_tokens_details.reasoning_tokens`

Use references/token-accounting.md before comparing cost or cache-hit rate.

The main repo's `trace-import` command handles common Anthropic/Claude Code and OpenAI-compatible usage shapes. If token fields are missing, set or preserve `cache_accounting_observable=false` and do not claim token-cost savings for those records.

## Usability Ablation Output

When this skill is tested as an artifact, report it as skill usability, not cache-hit proof. The expected output is:

- the generated baseline and cache-friendly command plan;
- whether warm-up and measured runs are separated;
- whether token fields match the documented accounting rules;
- whether Codex, Claude Code, MiMo, `make-agents-cheaper`, and this skill are kept separate;
- how many manual corrections were needed.

## When To Stop

Stop and report uncertainty when token fields are missing, a router hides cache usage, model/provider changes between A/B runs, or the task output quality changes. In those cases, the result can still be useful as an engineering trace, but it is not strong evidence for a cheaper-agent claim.
