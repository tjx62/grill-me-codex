# Plan Review Log: Model-switching for grill-me-codex skill family
Act 1 (grill) complete — plan locked with the user. MAX_ROUNDS=5.

## Round 1 — Ollama
**Security Hole: Command Injection**
The plan proposes building `curl` commands using `model=` and `base_url=` arguments; if these strings are not strictly sanitized, an attacker can inject arbitrary shell commands into the execution environment.
*Fix: Sanitize all input arguments to strip characters like `;`, `&`, `|`, and backticks before interpolation.*

**Race Condition: Shared File Collision**
Using a hardcoded `/tmp/ollama-verdict.txt` means that if two instances of a skill run concurrently (e.g., in a CI pipeline or parallel testing), they will overwrite each other's results, leading to non-deterministic "hallucinated" reviews.
*Fix: Use `mktemp` or append the process ID (`$$`) to the filename to ensure execution isolation.*

**Reliability Gap: Stale Data Persistence**
If the API call fails (e.g., a 500 error or network timeout) and the script does not explicitly clear the verdict file, the downstream Act 3 logic will consume the result from the *previous* successful run.
*Fix: Explicitly delete or truncate the verdict file at the immediate start of the Act 2 provider-dispatch block.*

**Logic Error: Anthropic System Prompt Mismatch**
The assumption that "no session format change is needed" because there is no system prompt is brittle; if a user eventually adds a system instruction to the skill, the current plan will produce malformed Anthropic API requests.
*Fix: Implement a specific transformation step that maps `system` role content to the top-level `system` parameter for the Anthropic provider path.*

**Observability Gap: Silent Failure on Network Errors**
The plan focuses on surfacing "auth errors" but does not address how `curl` handles connection timeouts or DNS failures, which may result in a successful script exit with an empty or incomplete verdict file.
*Fix: Add `--fail` to the `curl` command and implement an explicit check of `$?` to abort the skill if the provider call fails.*

VERDICT: REVISE

### Claude's response
**Accepted:**
- Command injection: accepted — `model=`/`base_url=` passed via `jq --arg` (not raw interpolation); URL values validated before use.
- Stale verdict file: accepted — truncate verdict file before each call.
- Concurrent collision: accepted lightly — PID-suffixed temp file (`$$`), low cost.
- Silent curl failure: accepted — `curl -sf` already in current skill; plan now explicitly requires it on all provider paths.

**Rejected:**
- Anthropic system prompt transformation: rejected — the review prompt is a user turn, not a system prompt. A `system`→top-level transform for a hypothetical future feature is out of scope. Explicitly noted in "Out of scope".
