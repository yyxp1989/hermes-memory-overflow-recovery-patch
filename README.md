# Hermes memory overflow recovery patch

这个仓库提供一个 **Hermes Agent 最小补丁**，修复 built-in 长期记忆满了以后，agent 只返回报错、但**不会主动清理并重试**的问题。

## 中文说明

### 这个问题是什么

当 Hermes 内建长期记忆（`memory` / `user`）达到字符上限时，原先常见表现是：

- memory tool 返回 `success: false`
- 报错里包含：
  - `would exceed the limit`
  - `Replace or remove existing entries first`
- 然后 agent 停在这里，只告诉你 memory 满了

也就是说，**存储层知道它满了，但 runtime 没有继续做“清理 → 重试”这一步**。

### 这个补丁修了什么

这个补丁修改的是 `run_agent.py` 的运行时行为，而不是去改变 `tools/memory_tool.py` 的 bounded store 设计。

补丁生效后，当 built-in memory 的 `add` 因 overflow 失败时，Hermes 会：

1. 识别这是 memory overflow
2. 启动一个 **只使用 memory toolset** 的 cleanup review
3. 与原 built-in memory **共享同一个 memory store**
4. 同步执行清理
5. 对原始 `memory add` **重试一次**
6. 只有最终 built-in 写入成功时，才 bridge 到外部 memory provider

### 为什么这样修

这个问题的根因不在 `MemoryStore.add()`。

`MemoryStore.add()` 作为底层存储，直接拒绝超限写入本身是合理的；真正缺的是上层 `run_agent.py` 在看到 overflow 后，没有主动发起一次 memory-only cleanup 并重试原写入。

所以这份 patch 的原则是：

- **不改存储层语义**
- **只补 runtime 恢复逻辑**
- **保持行为保守：只 cleanup 一次，只 retry 一次**

### 补丁包含什么

- `run_agent.py` 中的 overflow cleanup + retry 修复
- 回归测试：
  - helper 层
  - `_invoke_tool(...)` 入口
  - sequential tool-call 入口
  - 完整 `run_conversation()` 对话流
  - failed built-in write 不得 bridge 外部 memory provider

---

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
