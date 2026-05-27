# Token Accounting

## Claude-Style Usage Fields

When Claude Code prints JSON, usage may contain Claude-style fields:

```json
{
  "usage": {
    "inputTokens": 1000,
    "cacheReadInputTokens": 800,
    "cacheCreationInputTokens": 200,
    "outputTokens": 120
  }
}
```

Recommended derived fields:

```text
input_total = inputTokens + cacheReadInputTokens + cacheCreationInputTokens
cached_input = cacheReadInputTokens
cache_creation_input = cacheCreationInputTokens
uncached_input = inputTokens + cacheCreationInputTokens
cache_hit_rate = cached_input / input_total
```

Use the same formulas for every run in a comparison.

## MiMo OpenAI-Compatible Usage Fields

Direct MiMo OpenAI-compatible API responses may expose:

```json
{
  "usage": {
    "prompt_tokens": 1000,
    "completion_tokens": 120,
    "total_tokens": 1120,
    "prompt_tokens_details": {
      "cached_tokens": 800
    },
    "completion_tokens_details": {
      "reasoning_tokens": 30
    }
  }
}
```

Recommended derived fields:

```text
input_total = usage.prompt_tokens
cached_input = usage.prompt_tokens_details.cached_tokens
uncached_input = input_total - cached_input
output_total = usage.completion_tokens
reasoning_output = usage.completion_tokens_details.reasoning_tokens
cache_hit_rate = cached_input / input_total
```

## Experiment Log Schema

Record one JSONL row per run:

```json
{
  "timestamp": "2026-05-09T00:00:00+08:00",
  "suite": "real-coding-v1",
  "task_id": "review-small-diff",
  "variant": "baseline",
  "phase": "measured",
  "agent": "claude-code",
  "model": "mimo-v2.5-pro",
  "flags": [],
  "input_total": 0,
  "cached_input": 0,
  "uncached_input": 0,
  "cache_creation_input": 0,
  "output_total": 0,
  "latency_ms": 0,
  "success": true,
  "notes": ""
}
```

For the candidate variant, include:

```json
"flags": ["--exclude-dynamic-system-prompt-sections"]
```

## Interpretation Rules

- Higher cache-hit rate is useful only if task correctness is preserved.
- Lower uncached input is the cleaner signal than lower total tokens.
- Output tokens can vary because the model may answer differently; report them separately.
- A cold candidate run can be more expensive than a warm baseline. Treat warm-up and measured phases separately.
- If cached-token fields are absent, report "cache usage not observable" instead of inventing a hit rate.
