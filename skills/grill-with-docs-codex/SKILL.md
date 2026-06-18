---
name: grill-with-docs-codex
description: Two-act plan hardening with living documentation. ACT 1 (you ↔ Claude) — Claude interviews you relentlessly about a plan, one question at a time, challenging it against your project's existing domain model and glossary (CONTEXT.md), sharpening fuzzy terms, stress-testing with concrete scenarios, cross-referencing code, and updating CONTEXT.md + ADRs inline as decisions crystallise. ACT 2 (Claude ↔ reviewer model) — Claude writes the locked plan to PLAN.md and a configurable reviewer model (ollama, OpenAI, Anthropic, or any OpenAI-compatible endpoint) adversarially reviews it (VERDICT:APPROVED/REVISE), Claude revises and re-submits to the SAME session until APPROVED or a MAX_ROUNDS cap, then you sign off before any code. Use when the user says "/grill-with-docs-codex", "grill me against the docs then have codex review", "stress-test this against our domain model then get a second model on it", or is about to build something high-stakes in a project with established terminology/ADRs and wants alignment, documentation, AND a cross-model sanity check. Builds on Matt Pocock's grill-with-docs (MIT). NOT for reviewing already-written code (use /codex:review) and NOT for trivial changes.
---

# Grill-with-Docs-Codex — Grill Against Your Domain, Then Get Reviewed

Two acts. Act 1 aligns intent *and* keeps your living docs honest; Act 2 has a different model attack the result.

- **Act 1** is Matt Pocock's `grill-with-docs`, used under MIT (see `THIRD-PARTY-NOTICES.md`). It interrogates you, challenges your plan against `CONTEXT.md`/ADRs, and updates them inline.
- **Act 2** is the adversarial review loop — a configurable reviewer model (ollama by default, or OpenAI/Anthropic/any OpenAI-compatible endpoint via `model=`), cross-model, read-only (prompt-only), bounded.

You enter at two points: answering the grill, and signing off the converged plan.

---

## ACT 1 — GRILL WITH DOCS (you ↔ Claude)

<what-to-do>

Interview me relentlessly about every aspect of this plan until we reach a shared understanding. Walk down each branch of the design tree, resolving dependencies between decisions one-by-one. For each question, provide your recommended answer.

Ask the questions one at a time, waiting for feedback on each question before continuing.

If a question can be answered by exploring the codebase, explore the codebase instead.

</what-to-do>

<supporting-info>

## Domain awareness

During codebase exploration, also look for existing documentation:

### File structure

Most repos have a single context:

```
/
├── CONTEXT.md
├── docs/
│   └── adr/
│       ├── 0001-event-sourced-orders.md
│       └── 0002-postgres-for-write-model.md
└── src/
```

If a `CONTEXT-MAP.md` exists at the root, the repo has multiple contexts. The map points to where each one lives:

```
/
├── CONTEXT-MAP.md
├── docs/
│   └── adr/                          ← system-wide decisions
├── src/
│   ├── ordering/
│   │   ├── CONTEXT.md
│   │   └── docs/adr/                 ← context-specific decisions
│   └── billing/
│       ├── CONTEXT.md
│       └── docs/adr/
```

Create files lazily — only when you have something to write. If no `CONTEXT.md` exists, create one when the first term is resolved. If no `docs/adr/` exists, create it when the first ADR is needed.

## During the session

### Challenge against the glossary

When the user uses a term that conflicts with the existing language in `CONTEXT.md`, call it out immediately. "Your glossary defines 'cancellation' as X, but you seem to mean Y — which is it?"

### Sharpen fuzzy language

When the user uses vague or overloaded terms, propose a precise canonical term. "You're saying 'account' — do you mean the Customer or the User? Those are different things."

### Discuss concrete scenarios

When domain relationships are being discussed, stress-test them with specific scenarios. Invent scenarios that probe edge cases and force the user to be precise about the boundaries between concepts.

### Cross-reference with code

When the user states how something works, check whether the code agrees. If you find a contradiction, surface it: "Your code cancels entire Orders, but you just said partial cancellation is possible — which is right?"

### Update CONTEXT.md inline

When a term is resolved, update `CONTEXT.md` right there. Don't batch these up — capture them as they happen. Use the format in [CONTEXT-FORMAT.md](./CONTEXT-FORMAT.md).

`CONTEXT.md` should be totally devoid of implementation details. Do not treat `CONTEXT.md` as a spec, a scratch pad, or a repository for implementation decisions. It is a glossary and nothing else.

### Offer ADRs sparingly

Only offer to create an ADR when all three are true:

1. **Hard to reverse** — the cost of changing your mind later is meaningful
2. **Surprising without context** — a future reader will wonder "why did they do it this way?"
3. **The result of a real trade-off** — there were genuine alternatives and you picked one for specific reasons

If any of the three is missing, skip the ADR. Use the format in [ADR-FORMAT.md](./ADR-FORMAT.md).

</supporting-info>

### Handoff to Act 2

When the decision tree is resolved, the glossary/ADRs are updated, and we're aligned, **write the agreed plan to `PLAN.md`** (use the canonical terms from `CONTEXT.md`), then run Act 2:

```markdown
# Plan: <task>
_Locked via grill-with-docs — by Claude + <user>. Terms per CONTEXT.md._

## Goal
<one paragraph, in the project's ubiquitous language>

## Approach
<numbered, concrete steps>

## Key decisions & tradeoffs
<the contestable choices the grill resolved — link any ADRs created>

## Risks / open questions
<anything still open>

## Out of scope
<bounds>
```

Initialize `PLAN-REVIEW-LOG.md`:
```markdown
# Plan Review Log: <task>
Act 1 (grill-with-docs) complete — plan locked, CONTEXT.md/ADRs updated. MAX_ROUNDS=<n>.
```

---

## ACT 2 — REVIEW (Claude ↔ reviewer model)

### Prerequisites
- `jq` is installed: `which jq`. Install via your package manager if missing.
- **If `REVIEW_MODEL` starts with `claude-`:** `ANTHROPIC_API_KEY` must be set in the environment.
- **If `REVIEW_MODEL` starts with `gpt-`, `o1-`, or `o3-`, or `REVIEW_BASE_URL` is set:** `OPENAI_API_KEY` should be set (some compatible endpoints are unauthenticated — attempt the call and surface any auth error).
- **Otherwise (default ollama):** Ollama is running: `curl -s ${OLLAMA_HOST:-http://localhost:11434}/api/tags` returns JSON. Model is pulled: `ollama list` shows `$REVIEW_MODEL`. Start with `ollama serve` / `ollama pull $REVIEW_MODEL` if needed.
- On any curl error or unexpected response, surface it — don't silently retry.

### Tunables (args, else default)
| Var | Default | Meaning |
|-----|---------|---------|
| `MAX_ROUNDS` | `5` | Hard cap. Loop ALWAYS terminates here. |
| `PLAN_FILE` | `PLAN.md` | The plan from Act 1. |
| `LOG_FILE` | `PLAN-REVIEW-LOG.md` | Append-only argument transcript. |
| `REVIEW_MODEL` | `gemma4:26b` | Model for adversarial review. Prefix determines provider: `claude-*` → Anthropic, `gpt-*/o1-*/o3-*` → OpenAI, anything else → ollama. |
| `OLLAMA_HOST` | `http://localhost:11434` | Ollama base URL (ollama path only). |
| `REVIEW_BASE_URL` | _(none)_ | Override API base URL for OpenAI-compatible endpoints (Groq, Together, Fireworks, etc.). When set, forces OpenAI-compatible path regardless of model name. |

If invoked with e.g. `rounds=3` or `model=gpt-4o` → use those values. Echo resolved values first.

### Review prompt (each round)

The plan (and CONTEXT.md if it exists) is inlined in the prompt — the reviewer cannot browse the repo directly.

### Round 1 — initialize session

```bash
REVIEW_MODEL="${REVIEW_MODEL:-gemma4:26b}"
REVIEW_BASE_URL="${REVIEW_BASE_URL:-}"
OLLAMA_HOST="${OLLAMA_HOST:-http://localhost:11434}"
SESSION_FILE="/tmp/review-session.json"
VERDICT_FILE="/tmp/review-verdict.txt"

PLAN_CONTENT=$(cat "$PLAN_FILE")
CONTEXT_SECTION=""
[ -f CONTEXT.md ] && CONTEXT_SECTION="

Domain glossary (CONTEXT.md):
---
$(cat CONTEXT.md)
---"

USER_MSG="You are an adversarial reviewer for an implementation plan. Be skeptical and specific — your job is to find what breaks, not to be agreeable.${CONTEXT_SECTION}

Here is the plan:

---
${PLAN_CONTENT}
---

Identify concrete flaws: security holes, race conditions, missing edge cases, schema conflicts, domain-language mismatches, wrong assumptions, observability gaps, simpler alternatives. For each, give a one-line fix. End with EXACTLY one line: \`VERDICT: APPROVED\` or \`VERDICT: REVISE\`."

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

### Each round
1. Read `$VERDICT_FILE`; append `## Round <n> — <REVIEW_MODEL>` + critique to `LOG_FILE`.
2. Last line verdict: `APPROVED` → Resolution (converged); `REVISE` → Claude decides what's worth acting on (final arbiter), revise `PLAN_FILE`, append `### Claude's response` (what changed/rejected + why), increment.
3. round > `MAX_ROUNDS` → Resolution (deadlock).

### Resolution (you sign off)
- **APPROVED:** present final plan + 3-bullet summary of what the two acts improved + round count. Ask before implementing. No code during either act.
- **Deadlock (cap hit, no APPROVED):** list unresolved points + Claude's counter-position; hand to user. Don't fake convergence.

---

## Hard rules
- Act 1 precedes Act 2. `CONTEXT.md` stays a glossary only — no implementation details.
- The reviewer model never touches files — it only sees what's passed in the prompt.
- Loop ALWAYS terminates at `MAX_ROUNDS`. Claude is final arbiter on REVISE (reject with logged reason). Code only after sign-off. `LOG_FILE` is the deliverable.
- `SESSION_FILE` accumulates the full conversation; don't delete it mid-loop or history is lost.

## What NOT to do
- Don't review already-written code. Don't skip Act 1.
- Don't pass secrets or credentials into the plan text — the reviewer model may be on a shared or remote host.
