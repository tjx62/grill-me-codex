---
name: grill-me-codex
description: Two-act plan hardening. ACT 1 (you ↔ Claude) — Claude interviews you relentlessly about a plan or design, one question at a time, recommending an answer for each and exploring the codebase when it can answer itself, until every branch of the decision tree is resolved. ACT 2 (Claude ↔ reviewer model) — Claude writes the locked plan to PLAN.md and a configurable reviewer model (ollama, OpenAI, Anthropic, or any OpenAI-compatible endpoint) adversarially reviews it (VERDICT:APPROVED/REVISE), Claude revises and re-submits to the SAME session until APPROVED or a MAX_ROUNDS cap, then you sign off before any code. Use when the user says "/grill-me-codex", "grill me then have codex review", "grill me and stress-test the plan", "interview me about this plan then get a second model on it", or is about to build something high-stakes (auth, schema, concurrency, migrations, payments) and wants both alignment AND a cross-model sanity check before implementation. Builds on Matt Pocock's grill-me (MIT). For the docs-aware variant use /grill-with-docs-codex; if you already have a plan and want only the review use /codex-review. NOT for reviewing already-written code (use /codex:review) and NOT for trivial changes.
---

# Grill-Me-Codex — Get Grilled, Then Get Reviewed

Two acts, two different jobs:

- **Act 1 fixes the #1 failure mode: building the wrong thing.** Claude interrogates *you* until intent is locked — no guessing at ambiguity. (This act is Matt Pocock's `grill-me`, used under MIT — see `THIRD-PARTY-NOTICES.md`.)
- **Act 2 fixes the #2 failure mode: a plan that sounds right but breaks.** A *different model* adversarially attacks the locked plan. Cross-model = no echo chamber. The reviewer model is configurable — pass `model=claude-sonnet-4-6`, `model=gpt-4o`, `model=llama3.2:3b`, or any OpenAI-compatible endpoint via `base_url=`.

You enter at two points only: answering the grill, and signing off the converged plan. The reviewer model only sees what's in the prompt and never touches a file.

---

## ACT 1 — GRILL (you ↔ Claude)

> Interview me relentlessly about every aspect of this plan until we reach a shared understanding. Walk down each branch of the design tree, resolving dependencies between decisions one-by-one. For each question, provide your recommended answer.
>
> Ask the questions one at a time, waiting for my answer before continuing.
>
> If a question can be answered by exploring the codebase, explore the codebase instead.

When the decision tree is resolved and we're aligned, **write the agreed plan to `PLAN.md`** in this structure, then move to Act 2:

```markdown
# Plan: <task>
_Locked via grill — by Claude + <user>_

## Goal
<one paragraph — reflects what the grilling actually settled>

## Approach
<numbered, concrete steps>

## Key decisions & tradeoffs
<the contestable choices the grill resolved — name them so the adversarial reviewer has something to bite>

## Risks / open questions
<anything still genuinely open>

## Out of scope
<bounds the grill established>
```

Initialize `PLAN-REVIEW-LOG.md`:
```markdown
# Plan Review Log: <task>
Act 1 (grill) complete — plan locked with the user. MAX_ROUNDS=<n>.
```

---

## ACT 2 — REVIEW (Claude ↔ reviewer model)

### Prerequisites (verify once, fast)
- `jq` is installed: `which jq`. Install via your package manager if missing.
- **If `REVIEW_MODEL` starts with `claude-`:** `ANTHROPIC_API_KEY` must be set in the environment.
- **If `REVIEW_MODEL` starts with `gpt-`, `o1-`, or `o3-`, or `REVIEW_BASE_URL` is set:** `OPENAI_API_KEY` should be set (some compatible endpoints are unauthenticated — attempt the call and surface any auth error).
- **Otherwise (default ollama):** Ollama is running: `curl -s ${OLLAMA_HOST:-http://localhost:11434}/api/tags` returns JSON. Model is pulled: `ollama list` shows `$REVIEW_MODEL`. Start with `ollama serve` / `ollama pull $REVIEW_MODEL` if needed.
- On any curl error or empty response, surface it — don't silently retry.

### Tunables (read from args, else default)
| Var | Default | Meaning |
|-----|---------|---------|
| `MAX_ROUNDS` | `5` | Hard cap on review rounds. The loop ALWAYS terminates here. |
| `PLAN_FILE` | `PLAN.md` | The plan Act 1 produced. |
| `LOG_FILE` | `PLAN-REVIEW-LOG.md` | Append-only argument transcript. The artifact. |
| `REVIEW_MODEL` | `gemma4:26b` | Model for adversarial review. Prefix determines provider: `claude-*` → Anthropic, `gpt-*/o1-*/o3-*` → OpenAI, anything else → ollama. |
| `OLLAMA_HOST` | `http://localhost:11434` | Ollama base URL (ollama path only). |
| `REVIEW_BASE_URL` | _(none)_ | Override API base URL for OpenAI-compatible endpoints (Groq, Together, Fireworks, etc.). When set, forces OpenAI-compatible path regardless of model name. |

If invoked with e.g. `rounds=3` or `model=claude-sonnet-4-6`, use those values. Echo resolved values before starting.

### The review prompt (sent each round)

The plan is inlined in the prompt — the reviewer model cannot browse the repo directly.

### Round 1 — initialize session

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

Confirm success: `$VERDICT_FILE` is non-empty and ends with a VERDICT line. On failure, stop and tell the user.

### Rounds 2..MAX — resume SAME session (prior critiques live in the messages array)

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

### Each round, after the model returns
1. Read `$VERDICT_FILE`; append to `LOG_FILE`: `## Round <n> — <REVIEW_MODEL>` + the full critique.
2. Grep the last line for the verdict:
   - `VERDICT: APPROVED` → break to Resolution (converged).
   - `VERDICT: REVISE` → Claude decides **what's actually worth acting on** (Claude is final arbiter — the reviewer advises, doesn't command). Revise `PLAN_FILE`. Append `### Claude's response` to `LOG_FILE`: what changed, what was rejected, why. Increment round.
3. If round > `MAX_ROUNDS` → break to Resolution (deadlock).

### Resolution (you sign off — final gate)
- **APPROVED:** present the final `PLAN_FILE`, a 3-bullet summary of what the two acts improved, and the round count. Ask: *"Grilled + survived N rounds of adversarial review. Implement it now?"* Code only on yes. **No code is written during either act.**
- **MAX_ROUNDS hit without APPROVED (deadlock):** do NOT fake convergence. List each unresolved point + Claude's counter-position; hand it to the user to break the tie. A flagged disagreement beats a false "approved."

---

## Hard rules
- Act 1 always precedes Act 2 — don't write `PLAN.md` until the grill has actually resolved the decision tree with the user.
- The reviewer model never touches files — it only sees what's passed in the prompt. Never pipe file-write commands into the session.
- The loop ALWAYS terminates at `MAX_ROUNDS`.
- Claude is final arbiter on every REVISE — incorporate good critiques, reject bad ones *with a logged reason*. Don't cave to everything (defeats the cross-model check) and don't ignore it (defeats the point).
- Code only after the user's final sign-off.
- `LOG_FILE` is the deliverable — keep the whole argument.
- `SESSION_FILE` accumulates the full conversation; don't delete it mid-loop or history is lost.

## What NOT to do
- Don't review already-written code.
- Don't skip Act 1 — the grill is half the value.
- Don't pass secrets or credentials into the plan text — the reviewer model may be on a shared or remote host.
