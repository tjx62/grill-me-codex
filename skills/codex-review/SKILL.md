---
name: codex-review
description: A standalone adversarial PLAN-review loop where Claude Code (builder) and a configurable reviewer model (ollama, OpenAI, Anthropic, or any OpenAI-compatible endpoint) tag-team an implementation plan before any code is written. Use this when you ALREADY have a plan or a clear idea and just want the cross-model stress-test — no requirements interview first. Claude drafts/loads the plan into PLAN.md, the reviewer returns VERDICT:APPROVED or VERDICT:REVISE, Claude revises and re-submits to the SAME session (context preserved) until APPROVED or a configurable MAX_ROUNDS cap is hit. Human approves the converged plan before code. Use when the user says "/codex-review", "codex review my plan", "adversarial plan review", "make Claude and another model argue/fight over the plan", or is about to build something high-stakes (auth, schema, concurrency, migrations, payments) and wants a second-model sanity check on the PLAN before implementation. For a guided requirements interview BEFORE the review, use /grill-me-codex instead. NOT for reviewing already-written CODE and NOT for trivial changes.
---

# Codex-Review — Adversarial Plan-Review Loop

Two models, one plan, a bounded argument. **Claude is the builder and orchestrator. A configurable reviewer model is the read-only critic** — it sees only what's passed in the prompt and cannot touch any file. They communicate through `PLAN.md` + a session JSON file that accumulates conversation history across rounds. The human enters at exactly two points: kickoff and final sign-off.

The reviewer is swappable via `model=` — pass `model=claude-sonnet-4-6`, `model=gpt-4o`, `model=llama3.2:3b`, or any OpenAI-compatible endpoint via `base_url=`. Default is `gemma4:26b` (local ollama, zero-config).

This is a **deliberate, high-stakes tool** — reach for it on auth, data models, concurrency, migrations, payments, anything expensive to get wrong. Skip it for obvious/cheap work.

## Prerequisites (verify once, fast)

- `jq` is installed: `which jq`. Install via your package manager if missing.
- **If `REVIEW_MODEL` starts with `claude-`:** `ANTHROPIC_API_KEY` must be set in the environment.
- **If `REVIEW_MODEL` starts with `gpt-`, `o1-`, or `o3-`, or `REVIEW_BASE_URL` is set:** `OPENAI_API_KEY` should be set (some compatible endpoints are unauthenticated — attempt the call and surface any auth error).
- **Otherwise (default ollama):** Ollama is running: `curl -s ${OLLAMA_HOST:-http://localhost:11434}/api/tags` returns JSON. Model is pulled: `ollama list` shows `$REVIEW_MODEL`. Start with `ollama serve` / `ollama pull $REVIEW_MODEL` if needed.
- On any curl error or empty response, surface it — do not silently retry.

## Tunable variables (read from skill args, else default)

| Var | Default | Meaning |
|-----|---------|---------|
| `MAX_ROUNDS` | `5` | Hard cap on review rounds. The loop ALWAYS terminates at this. |
| `PLAN_FILE` | `PLAN.md` | Where the evolving plan lives (repo root). |
| `LOG_FILE` | `PLAN-REVIEW-LOG.md` | Append-only transcript of the argument (every round's critique + what changed). The artifact. |
| `REVIEW_MODEL` | `gemma4:26b` | Model for adversarial review. Prefix determines provider: `claude-*` → Anthropic, `gpt-*/o1-*/o3-*` → OpenAI, anything else → ollama. |
| `OLLAMA_HOST` | `http://localhost:11434` | Ollama base URL (ollama path only). |
| `REVIEW_BASE_URL` | _(none)_ | Override API base URL for OpenAI-compatible endpoints (Groq, Together, Fireworks, etc.). When set, forces OpenAI-compatible path regardless of model name. |

If the user invoked the skill with an argument like `rounds=3` or `model=gpt-4o`, use those values. Echo the resolved values back before starting.

## Flow

### Step 0 — Kickoff (human gate #1)

The invocation itself is the kickoff. Confirm scope in one line: what is being planned. If the user gave no task, ask for it (one question). Then proceed — do NOT ask for approval round-by-round; that comes at the end.

### Step 1 — Claude plans

Do real planning: read the relevant code, think through the approach, surface decisions and tradeoffs. Then write the plan to `PLAN_FILE` in this structure:

```markdown
# Plan: <task>
_Round 0 — initial draft by Claude_

## Goal
<one paragraph>

## Approach
<numbered steps, concrete>

## Key decisions & tradeoffs
<the contestable choices — name them explicitly so the reviewer has something to bite>

## Risks / open questions
<what you're unsure about>

## Out of scope
<bounds>
```

Initialize `LOG_FILE`:
```markdown
# Plan Review Log: <task>
Started <stamp the user's local time if known, else "session start">. MAX_ROUNDS=<n>.
```

Show the user the plan inline and say you're sending it to the reviewer for adversarial review.

### Step 2 — The loop

Maintain `ROUND` (start 1). `SESSION_FILE` is initialized in round 1 and accumulates history each round.

**The review prompt** sent to the model each round. The plan is inlined — the reviewer cannot browse the repo directly.

**Round 1** (initializes the session file):

```bash
REVIEW_MODEL="${REVIEW_MODEL:-gemma4:26b}"
REVIEW_BASE_URL="${REVIEW_BASE_URL:-}"
OLLAMA_HOST="${OLLAMA_HOST:-http://localhost:11434}"
SESSION_FILE="/tmp/review-session.json"
VERDICT_FILE="/tmp/review-verdict.txt"

PLAN_CONTENT=$(cat "$PLAN_FILE")
USER_MSG="You are an adversarial reviewer for an implementation plan. Be skeptical and specific — your job is to find what breaks, not to be agreeable.

Here is the plan:

---
${PLAN_CONTENT}
---

Identify concrete flaws: security holes, race conditions, missing edge cases, schema conflicts, wrong assumptions, observability gaps, simpler alternatives. For each, give a one-line fix. End your reply with EXACTLY one line: \`VERDICT: APPROVED\` if the plan is sound enough to implement, or \`VERDICT: REVISE\` if it still has material problems."

jq -n --arg content "$USER_MSG" '[{"role":"user","content":$content}]' > "$SESSION_FILE"

# Truncate verdict file before call so a failed call never reuses stale output
> "$VERDICT_FILE"

# Provider dispatch
if [[ "$REVIEW_MODEL" == claude-* ]]; then
  [ -z "$ANTHROPIC_API_KEY" ] && { echo "ERROR: ANTHROPIC_API_KEY is not set"; exit 1; }
  curl -sf "https://api.anthropic.com/v1/messages" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "Content-Type: application/json" \
    -d "$(jq -n --arg model "$REVIEW_MODEL" --argjson msgs "$(cat $SESSION_FILE)" \
         '{model:$model,max_tokens:4096,messages:$msgs}')" \
    | jq -r '.content[0].text' > "$VERDICT_FILE"
elif [[ "$REVIEW_MODEL" == gpt-* || "$REVIEW_MODEL" == o1-* || "$REVIEW_MODEL" == o3-* || -n "$REVIEW_BASE_URL" ]]; then
  BASE_URL="${REVIEW_BASE_URL:-https://api.openai.com/v1}"
  AUTH_ARGS=()
  [ -n "$OPENAI_API_KEY" ] && AUTH_ARGS=(-H "Authorization: Bearer $OPENAI_API_KEY")
  curl -sf "$BASE_URL/chat/completions" \
    -H "Content-Type: application/json" \
    "${AUTH_ARGS[@]}" \
    -d "$(jq -n --arg model "$REVIEW_MODEL" --argjson msgs "$(cat $SESSION_FILE)" \
         '{model:$model,messages:$msgs}')" \
    | jq -r '.choices[0].message.content' > "$VERDICT_FILE"
else
  curl -sf "$OLLAMA_HOST/api/chat" \
    -H "Content-Type: application/json" \
    -d "$(jq -n --arg model "$REVIEW_MODEL" --argjson msgs "$(cat $SESSION_FILE)" \
         '{model:$model,messages:$msgs,stream:false}')" \
    | jq -r '.message.content' > "$VERDICT_FILE"
fi

[ $? -ne 0 ] || [ ! -s "$VERDICT_FILE" ] && { echo "ERROR: reviewer call failed or returned empty — check API key / service"; exit 1; }

jq --arg reply "$(cat "$VERDICT_FILE")" \
   '. + [{"role":"assistant","content":$reply}]' "$SESSION_FILE" > /tmp/session-tmp.json \
  && mv /tmp/session-tmp.json "$SESSION_FILE"
```

Confirm success: `$VERDICT_FILE` is non-empty and contains a VERDICT line. On failure, stop and tell the user.

**Rounds 2..MAX** (resume the SAME session — prior critiques are in the accumulated messages array):

```bash
REVIEW_MODEL="${REVIEW_MODEL:-gemma4:26b}"
REVIEW_BASE_URL="${REVIEW_BASE_URL:-}"
OLLAMA_HOST="${OLLAMA_HOST:-http://localhost:11434}"
SESSION_FILE="/tmp/review-session.json"
VERDICT_FILE="/tmp/review-verdict.txt"

PLAN_CONTENT=$(cat "$PLAN_FILE")
FOLLOW_UP="I revised the plan. Here it is again:

---
${PLAN_CONTENT}
---

Re-review it — check whether your prior findings are addressed and flag anything new. End with VERDICT: APPROVED or VERDICT: REVISE."

jq --arg msg "$FOLLOW_UP" \
   '. + [{"role":"user","content":$msg}]' "$SESSION_FILE" > /tmp/session-tmp.json \
  && mv /tmp/session-tmp.json "$SESSION_FILE"

# Truncate verdict file before call
> "$VERDICT_FILE"

# Provider dispatch (same routing as round 1)
if [[ "$REVIEW_MODEL" == claude-* ]]; then
  curl -sf "https://api.anthropic.com/v1/messages" \
    -H "x-api-key: $ANTHROPIC_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "Content-Type: application/json" \
    -d "$(jq -n --arg model "$REVIEW_MODEL" --argjson msgs "$(cat $SESSION_FILE)" \
         '{model:$model,max_tokens:4096,messages:$msgs}')" \
    | jq -r '.content[0].text' > "$VERDICT_FILE"
elif [[ "$REVIEW_MODEL" == gpt-* || "$REVIEW_MODEL" == o1-* || "$REVIEW_MODEL" == o3-* || -n "$REVIEW_BASE_URL" ]]; then
  BASE_URL="${REVIEW_BASE_URL:-https://api.openai.com/v1}"
  AUTH_ARGS=()
  [ -n "$OPENAI_API_KEY" ] && AUTH_ARGS=(-H "Authorization: Bearer $OPENAI_API_KEY")
  curl -sf "$BASE_URL/chat/completions" \
    -H "Content-Type: application/json" \
    "${AUTH_ARGS[@]}" \
    -d "$(jq -n --arg model "$REVIEW_MODEL" --argjson msgs "$(cat $SESSION_FILE)" \
         '{model:$model,messages:$msgs}')" \
    | jq -r '.choices[0].message.content' > "$VERDICT_FILE"
else
  curl -sf "$OLLAMA_HOST/api/chat" \
    -H "Content-Type: application/json" \
    -d "$(jq -n --arg model "$REVIEW_MODEL" --argjson msgs "$(cat $SESSION_FILE)" \
         '{model:$model,messages:$msgs,stream:false}')" \
    | jq -r '.message.content' > "$VERDICT_FILE"
fi

[ $? -ne 0 ] || [ ! -s "$VERDICT_FILE" ] && { echo "ERROR: reviewer call failed or returned empty — check API key / service"; exit 1; }

jq --arg reply "$(cat "$VERDICT_FILE")" \
   '. + [{"role":"assistant","content":$reply}]' "$SESSION_FILE" > /tmp/session-tmp.json \
  && mv /tmp/session-tmp.json "$SESSION_FILE"
```

**Each round, after the model returns:**
1. Read `$VERDICT_FILE`. Append to `LOG_FILE`: `## Round <n> — <REVIEW_MODEL>` + the full critique.
2. Grep the last line for the verdict token.
   - `VERDICT: APPROVED` → break the loop, go to Step 3 (converged).
   - `VERDICT: REVISE` → Claude reads the critique, decides **what's actually worth acting on** (Claude has final say — the reviewer advises, it does not command). Revise `PLAN_FILE`. Append to `LOG_FILE`: `### Claude's response` + what you changed and what you rejected and why. Increment `ROUND`.
3. If `ROUND > MAX_ROUNDS` → break to Step 3 (deadlock).

### Step 3 — Resolution (human gate #2)

**If APPROVED:** Present to the user — the final `PLAN_FILE`, a 3-bullet summary of what the argument improved, and the round count. Ask: *"Plan survived N rounds of adversarial review. Implement it now?"* Only on a yes does Claude write code. **No code is written during the loop.**

**If MAX_ROUNDS hit without APPROVED (deadlock):** Do NOT pretend it converged. Surface the unresolved disagreements explicitly: list each point the model still flags and Claude's counter-position. Hand it to the human to break the tie. This is a legitimate, useful outcome — a flagged disagreement beats a false "approved."

## Hard rules

- The reviewer model never touches files — it only sees what's passed in the prompt. Never pipe file-write commands into the session.
- The loop ALWAYS terminates at `MAX_ROUNDS`. No unbounded recursion.
- Claude is the final arbiter on every REVISE — incorporate good critiques, reject bad ones *with a reason logged*. Don't cave on everything (defeats the cross-model check) and don't ignore it (defeats the point).
- Code only after human gate #2.
- `LOG_FILE` is the deliverable — it tells the whole story of the argument. Keep it complete.
- `SESSION_FILE` accumulates the full conversation; don't delete it mid-loop or history is lost.

## What NOT to do

- Don't use this to review existing code.
- Don't skip the log — the argument transcript is the most valuable artifact.
- Don't pass secrets or credentials into the plan text — the reviewer model may be on a shared or remote host.
