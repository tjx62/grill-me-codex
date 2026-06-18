---
name: grill-me-codex
description: Two-act plan hardening. ACT 1 (you ↔ Claude) — Claude interviews you relentlessly about a plan or design, one question at a time, recommending an answer for each and exploring the codebase when it can answer itself, until every branch of the decision tree is resolved. ACT 2 (Claude ↔ Codex) — Claude writes the locked plan to PLAN.md and OpenAI Codex adversarially reviews it in a read-only sandbox (VERDICT:APPROVED/REVISE), Claude revises and re-submits to the SAME Codex session until APPROVED or a MAX_ROUNDS cap, then you sign off before any code. Use when the user says "/grill-me-codex", "grill me then have codex review", "grill me and stress-test the plan", "interview me about this plan then get a second model on it", or is about to build something high-stakes (auth, schema, concurrency, migrations, payments) and wants both alignment AND a cross-model sanity check before implementation. Builds on Matt Pocock's grill-me (MIT). For the docs-aware variant use /grill-with-docs-codex; if you already have a plan and want only the Codex review use /codex-review. NOT for reviewing already-written code (use /codex:review) and NOT for trivial changes.
---

# Grill-Me-Codex — Get Grilled, Then Get Reviewed

Two acts, two different jobs:

- **Act 1 fixes the #1 failure mode: building the wrong thing.** Claude interrogates *you* until intent is locked — no guessing at ambiguity. (This act is Matt Pocock's `grill-me`, used under MIT — see `THIRD-PARTY-NOTICES.md`.)
- **Act 2 fixes the #2 failure mode: a plan that sounds right but breaks.** A *different model* (local ollama) adversarially attacks the locked plan. Cross-model = no echo chamber.

You enter at two points only: answering the grill, and signing off the converged plan. The ollama model only sees what's in the prompt and never touches a file.

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

## ACT 2 — REVIEW (Claude ↔ Ollama)

### Prerequisites (verify once, fast)
- Ollama is running locally: `curl -s ${OLLAMA_HOST:-http://localhost:11434}/api/tags` should return JSON. Start with `ollama serve` if not.
- Target model is pulled: `ollama list` should show `$OLLAMA_MODEL`. Pull with `ollama pull $OLLAMA_MODEL` if missing.
- `jq` is installed: `which jq`. Install via your package manager if missing.
- On any curl error or unexpected response, surface it — don't silently retry.

### Tunables (read from args, else default)
| Var | Default | Meaning |
|-----|---------|---------|
| `MAX_ROUNDS` | `5` | Hard cap on review rounds. The loop ALWAYS terminates here. |
| `PLAN_FILE` | `PLAN.md` | The plan Act 1 produced. |
| `LOG_FILE` | `PLAN-REVIEW-LOG.md` | Append-only argument transcript. The artifact. |
| `OLLAMA_MODEL` | `gemma4:26b` | Ollama model for adversarial review. Any pulled model works. |
| `OLLAMA_HOST` | `http://localhost:11434` | Ollama API base URL. |

If invoked with e.g. `rounds=3`, use that for `MAX_ROUNDS`. Echo resolved values before starting.

### The review prompt (sent each round)

The plan is inlined in the prompt — ollama cannot browse the repo directly.

### Round 1 — initialize session

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

jq -n --arg content "$USER_MSG" '[{"role":"user","content":$content}]' > "$SESSION_FILE"

curl -sf "$OLLAMA_HOST/api/chat" \
  -H 'Content-Type: application/json' \
  -d "$(jq -n --arg model "$OLLAMA_MODEL" --argjson msgs "$(cat $SESSION_FILE)" \
       '{model:$model,messages:$msgs,stream:false}')" \
  | jq -r '.message.content' > /tmp/ollama-verdict.txt

jq --arg reply "$(cat /tmp/ollama-verdict.txt)" \
   '. + [{"role":"assistant","content":$reply}]' "$SESSION_FILE" > /tmp/session-tmp.json \
  && mv /tmp/session-tmp.json "$SESSION_FILE"
```

Confirm success: `/tmp/ollama-verdict.txt` is non-empty and ends with a VERDICT line. On curl failure or missing `.message.content`, stop and tell the user.

### Rounds 2..MAX — resume SAME session (prior critiques live in the messages array)

```bash
PLAN_CONTENT=$(cat "$PLAN_FILE")
FOLLOW_UP="I revised the plan. Here it is again:

---
${PLAN_CONTENT}
---

Re-review it — check whether your prior findings are addressed and flag anything new. End with VERDICT: APPROVED or VERDICT: REVISE."

jq --arg msg "$FOLLOW_UP" \
   '. + [{"role":"user","content":$msg}]' "$SESSION_FILE" > /tmp/session-tmp.json \
  && mv /tmp/session-tmp.json "$SESSION_FILE"

curl -sf "$OLLAMA_HOST/api/chat" \
  -H 'Content-Type: application/json' \
  -d "$(jq -n --arg model "$OLLAMA_MODEL" --argjson msgs "$(cat $SESSION_FILE)" \
       '{model:$model,messages:$msgs,stream:false}')" \
  | jq -r '.message.content' > /tmp/ollama-verdict.txt

jq --arg reply "$(cat /tmp/ollama-verdict.txt)" \
   '. + [{"role":"assistant","content":$reply}]' "$SESSION_FILE" > /tmp/session-tmp.json \
  && mv /tmp/session-tmp.json "$SESSION_FILE"
```

### Each round, after the model returns
1. Read `/tmp/ollama-verdict.txt`; append to `LOG_FILE`: `## Round <n> — Ollama` + the full critique.
2. Grep the last line for the verdict:
   - `VERDICT: APPROVED` → break to Resolution (converged).
   - `VERDICT: REVISE` → Claude decides **what's actually worth acting on** (Claude is final arbiter — ollama advises, doesn't command). Revise `PLAN_FILE`. Append `### Claude's response` to `LOG_FILE`: what changed, what was rejected, why. Increment round.
3. If round > `MAX_ROUNDS` → break to Resolution (deadlock).

### Resolution (you sign off — final gate)
- **APPROVED:** present the final `PLAN_FILE`, a 3-bullet summary of what the two acts improved, and the round count. Ask: *"Grilled + survived N rounds of adversarial review. Implement it now?"* Code only on yes. **No code is written during either act.**
- **MAX_ROUNDS hit without APPROVED (deadlock):** do NOT fake convergence. List each unresolved point + Claude's counter-position; hand it to the user to break the tie. A flagged disagreement beats a false "approved."

---

## Hard rules
- Act 1 always precedes Act 2 — don't write `PLAN.md` until the grill has actually resolved the decision tree with the user.
- The ollama model never touches files — it only sees what's passed in the prompt. Never pipe file-write commands into the session.
- The loop ALWAYS terminates at `MAX_ROUNDS`.
- Claude is final arbiter on every REVISE — incorporate good critiques, reject bad ones *with a logged reason*. Don't cave to everything (defeats the cross-model check) and don't ignore it (defeats the point).
- Code only after the user's final sign-off.
- `LOG_FILE` is the deliverable — keep the whole argument.
- `SESSION_FILE` accumulates the full conversation; don't delete it mid-loop or history is lost.

## What NOT to do
- Don't review already-written code.
- Don't skip Act 1 — the grill is half the value.
- Don't pass secrets or credentials into the plan text — ollama may be on a shared host.
