---
name: codex-review
description: A standalone adversarial PLAN-review loop where Claude Code (builder) and OpenAI Codex (read-only critic) tag-team an implementation plan before any code is written. Use this when you ALREADY have a plan or a clear idea and just want the cross-model stress-test — no requirements interview first. Claude drafts/loads the plan into PLAN.md, Codex reviews it in a read-only sandbox and returns VERDICT:APPROVED or VERDICT:REVISE, Claude revises and re-submits to the SAME Codex session (context preserved) until APPROVED or a configurable MAX_ROUNDS cap is hit. Human approves the converged plan before code. Use when the user says "/codex-review", "codex review my plan", "have Codex review my plan", "argue this plan with Codex", "adversarial plan review", "make Claude and Codex argue/fight over the plan", or is about to build something high-stakes (auth, schema, concurrency, migrations, payments) and wants a second-model sanity check on the PLAN before implementation. For a guided requirements interview BEFORE the review, use /grill-me-codex instead. NOT for reviewing already-written CODE (that is the Codex plugin's /codex:review) and NOT for trivial changes.
---

# Codex-Review — Adversarial Plan-Review Loop

Two models, one plan, a bounded argument. **Claude is the builder and orchestrator. A local ollama model is the read-only critic** — it sees only what's passed in the prompt and cannot touch any file. They communicate through `PLAN.md` + a session JSON file that accumulates conversation history across rounds. The human enters at exactly two points: kickoff and final sign-off.

This is a **deliberate, high-stakes tool** — reach for it on auth, data models, concurrency, migrations, payments, anything expensive to get wrong. Skip it for obvious/cheap work.

## Prerequisites (verify once, fast)

- Ollama is running locally: `curl -s ${OLLAMA_HOST:-http://localhost:11434}/api/tags` should return JSON. Start with `ollama serve` if not running.
- Target model is pulled: `ollama list` should show `$OLLAMA_MODEL`. Pull with `ollama pull $OLLAMA_MODEL` if missing.
- `jq` is installed (used to build JSON payloads): `which jq`. Install via your package manager if missing.
- On any curl error or unexpected JSON shape, surface it — do not silently retry.

## Tunable variables (read from skill args, else default)

| Var | Default | Meaning |
|-----|---------|---------|
| `MAX_ROUNDS` | `5` | Hard cap on review rounds. The loop ALWAYS terminates at this. |
| `PLAN_FILE` | `PLAN.md` | Where the evolving plan lives (repo root). |
| `LOG_FILE` | `PLAN-REVIEW-LOG.md` | Append-only transcript of the argument (every round's critique + what changed). The artifact. |
| `OLLAMA_MODEL` | `gemma4:26b` | Ollama model to use for adversarial review. Any pulled model works; larger models give richer critiques. |
| `OLLAMA_HOST` | `http://localhost:11434` | Ollama API base URL. Override if running on a different host/port. |

If the user invoked the skill with an argument like `rounds=3`, use that for `MAX_ROUNDS`. Echo the resolved values back before starting.

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
<the contestable choices — name them explicitly so Codex has something to bite>

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

Show the user the plan inline and say you're sending it to Codex for adversarial review.

### Step 2 — The loop

Maintain `ROUND` (start 1). `SESSION_FILE` is initialized in round 1 and accumulates history each round.

**The review prompt** sent to the model each round. The plan is inlined — ollama cannot browse the repo directly.

**Round 1** (initializes the session file):

```bash
OLLAMA_HOST="${OLLAMA_HOST:-http://localhost:11434}"
OLLAMA_MODEL="${OLLAMA_MODEL:-gemma4:26b}"
SESSION_FILE="/tmp/ollama-review-session.json"

PLAN_CONTENT=$(cat "$PLAN_FILE")
USER_MSG="You are an adversarial reviewer for an implementation plan. Be skeptical and specific — your job is to find what breaks, not to be agreeable.

Here is the plan:

---
${PLAN_CONTENT}
---

Identify concrete flaws: security holes, race conditions, missing edge cases, schema conflicts, wrong assumptions, observability gaps, simpler alternatives. For each, give a one-line fix. End your reply with EXACTLY one line: \`VERDICT: APPROVED\` if the plan is sound enough to implement, or \`VERDICT: REVISE\` if it still has material problems."

# Initialize session
jq -n --arg content "$USER_MSG" '[{"role":"user","content":$content}]' > "$SESSION_FILE"

# Call ollama
curl -sf "$OLLAMA_HOST/api/chat" \
  -H 'Content-Type: application/json' \
  -d "$(jq -n --arg model "$OLLAMA_MODEL" --argjson msgs "$(cat $SESSION_FILE)" \
       '{model:$model,messages:$msgs,stream:false}')" \
  | jq -r '.message.content' > /tmp/ollama-verdict.txt

# Append assistant reply to session history
jq --arg reply "$(cat /tmp/ollama-verdict.txt)" \
   '. + [{"role":"assistant","content":$reply}]' "$SESSION_FILE" > /tmp/session-tmp.json \
  && mv /tmp/session-tmp.json "$SESSION_FILE"
```

Confirm success: `/tmp/ollama-verdict.txt` is non-empty and contains a VERDICT line. If the curl fails or `.message.content` is missing, stop and tell the user (likely ollama not running or model not pulled).

**Rounds 2..MAX** (resume the SAME session — prior critiques are in the accumulated messages array):

```bash
PLAN_CONTENT=$(cat "$PLAN_FILE")
FOLLOW_UP="I revised the plan. Here it is again:

---
${PLAN_CONTENT}
---

Re-review it — check whether your prior findings are addressed and flag anything new. End with VERDICT: APPROVED or VERDICT: REVISE."

# Append new user turn to history
jq --arg msg "$FOLLOW_UP" \
   '. + [{"role":"user","content":$msg}]' "$SESSION_FILE" > /tmp/session-tmp.json \
  && mv /tmp/session-tmp.json "$SESSION_FILE"

# Call ollama with full history
curl -sf "$OLLAMA_HOST/api/chat" \
  -H 'Content-Type: application/json' \
  -d "$(jq -n --arg model "$OLLAMA_MODEL" --argjson msgs "$(cat $SESSION_FILE)" \
       '{model:$model,messages:$msgs,stream:false}')" \
  | jq -r '.message.content' > /tmp/ollama-verdict.txt

# Append assistant reply
jq --arg reply "$(cat /tmp/ollama-verdict.txt)" \
   '. + [{"role":"assistant","content":$reply}]' "$SESSION_FILE" > /tmp/session-tmp.json \
  && mv /tmp/session-tmp.json "$SESSION_FILE"
```

**Each round, after the model returns:**
1. Read `/tmp/ollama-verdict.txt`. Append to `LOG_FILE`: `## Round <n> — Ollama` + the full critique.
2. Grep the last line for the verdict token.
   - `VERDICT: APPROVED` → break the loop, go to Step 3 (converged).
   - `VERDICT: REVISE` → Claude reads the critique, decides **what's actually worth acting on** (Claude has final say — ollama advises, it does not command). Revise `PLAN_FILE`. Append to `LOG_FILE`: `### Claude's response` + what you changed and what you rejected and why. Increment `ROUND`.
3. If `ROUND > MAX_ROUNDS` → break to Step 3 (deadlock).

### Step 3 — Resolution (human gate #2)

**If APPROVED:** Present to the user — the final `PLAN_FILE`, a 3-bullet summary of what the argument improved, and the round count. Ask: *"Plan survived N rounds of adversarial review. Implement it now?"* Only on a yes does Claude write code. **No code is written during the loop.**

**If MAX_ROUNDS hit without APPROVED (deadlock):** Do NOT pretend it converged. Surface the unresolved disagreements explicitly: list each point the model still flags and Claude's counter-position. Hand it to the human to break the tie. This is a legitimate, useful outcome — a flagged disagreement beats a false "approved."

## Hard rules

- The ollama model never touches files — it only sees what's passed in the prompt. Never pipe file-write commands into the session.
- The loop ALWAYS terminates at `MAX_ROUNDS`. No unbounded recursion.
- Claude is the final arbiter on every REVISE — incorporate good critiques, reject bad ones *with a reason logged*. Don't cave on everything (defeats the cross-model check) and don't ignore it (defeats the point).
- Code only after human gate #2.
- `LOG_FILE` is the deliverable — it tells the whole story of the argument. Keep it complete.
- `SESSION_FILE` accumulates the full conversation; don't delete it mid-loop or history is lost.

## What NOT to do

- Don't use this to review existing code.
- Don't skip the log — the argument transcript is the most valuable artifact.
- Don't pass secrets or credentials into the plan text — ollama may be running on a shared host.
