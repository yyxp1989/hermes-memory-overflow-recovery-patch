# Hermes memory overflow recovery patch

This repository contains a minimal patch for Hermes Agent that fixes a long-standing runtime problem:

When built-in long-term memory (`memory` / `user`) is full, Hermes used to only return an overflow error such as:

- `would exceed the limit`
- `Replace or remove existing entries first`

and then stop.

## What this patch changes

The patch updates `run_agent.py` so that when a built-in memory `add` overflows, Hermes will:

1. detect the bounded-memory overflow result,
2. run a **synchronous memory-only cleanup review**,
3. share the same built-in memory store during cleanup,
4. retry the original `memory add` exactly once,
5. notify external memory providers **only if the final built-in write succeeds**.

## Files

- `patches/hermes-memory-overflow-recovery-2026-05-11.patch`

## Apply

From the root of a clean Hermes source checkout:

```bash
git apply patches/hermes-memory-overflow-recovery-2026-05-11.patch
```

Or:

```bash
git apply /absolute/path/to/hermes-memory-overflow-recovery-2026-05-11.patch
```

## Verify before applying

```bash
git apply --check patches/hermes-memory-overflow-recovery-2026-05-11.patch
```

## Suggested tests

Use the project venv if present:

```bash
.venv/bin/python -m pytest tests/run_agent/test_memory_overflow_recovery.py -q
.venv/bin/python -m pytest tests/run_agent/test_run_agent.py -q -k 'memory_overflow or run_conversation_memory_overflow_cleanup_retry_then_final_answer'
.venv/bin/python -m pytest tests/run_agent/test_memory_overflow_recovery.py tests/agent/test_memory_provider.py tests/run_agent/test_background_review.py tests/run_agent/test_run_agent.py -q -k 'memory or background_review or switch_model or run_conversation_memory_overflow_cleanup_retry_then_final_answer'
```

Fallback:

```bash
python3 -m pytest ...
```

## Scope

This is intentionally a minimal public artifact repo:

- one patch
- one README
- no local environment files
- no unrelated Hermes worktree changes
