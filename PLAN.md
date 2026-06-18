# Plan: Model-switching for grill-me-codex skill family
_Locked via grill — by Claude + taylor_

## Goal
Add a `model=` skill arg to all three skills (`grill-me-codex`, `grill-with-docs-codex`, `codex-review`) that lets the user swap the Act 2 adversarial reviewer to any model — local ollama, OpenAI, Anthropic, or any OpenAI-compatible endpoint — without touching the skill files. The default stays `gemma4:26b` (ollama) so the skill works zero-config out of the box.

## Approach
1. Add `model=` and `base_url=` to each skill's tunables table (alongside the existing `MAX_ROUNDS`, `PLAN_FILE`, `LOG_FILE`).
2. Replace the hardcoded ollama curl block in each skill's Act 2 with a provider-dispatch pattern:
   - Detect provider from model name prefix:
     - `claude-*` → Anthropic (`https://api.anthropic.com/v1/messages`, `ANTHROPIC_API_KEY`)
     - `gpt-*`, `o1-*`, `o3-*` → OpenAI (`https://api.openai.com/v1/chat/completions`, `OPENAI_API_KEY`)
     - `base_url=` arg present → OpenAI-compatible at that URL (`OPENAI_API_KEY` or unauthenticated)
     - anything else → ollama at `OLLAMA_HOST` (existing path)
3. Each provider path builds its own curl command and normalises the response to a plain text string saved to `/tmp/ollama-verdict-$$.txt` (PID-suffixed to avoid collision if two sessions run simultaneously).
4. **Safety before each call:** truncate the verdict file (`> /tmp/ollama-verdict-$$.txt`) so a failed call never silently reuses stale output from a prior round.
5. **Curl flags:** all provider paths use `curl -sf` (`--silent --fail`) so network errors and non-2xx responses produce a non-zero exit; the skill checks `$?` and surfaces the error rather than continuing with an empty verdict.
6. **Input sanitization:** `model=` and `base_url=` values are passed via `jq --arg` (not raw shell interpolation) wherever they appear in JSON payloads. For URL construction, values are validated to contain only safe URL characters before use; the skill aborts with a clear message if they don't.
7. Prerequisites section updated: check the relevant env var / service is available before starting, based on the resolved model.
8. Apply identically to all three SKILL.md files.

## Key decisions & tradeoffs
- **Skill arg over env var for model selection** — consistent with existing `rounds=` pattern; explicit at call site.
- **Infer provider from model name prefix, not explicit `provider=` arg** — less to type; the prefixes are stable (`claude-`, `gpt-`, `o1-`, `o3-`). Edge cases covered by `base_url=`.
- **`base_url=` for OpenAI-compatible endpoints** — covers Groq, Together, Fireworks, etc. with one code path; no need to enumerate providers.
- **API keys via env vars only** — standard, no history leak, no config file to manage.
- **Default stays `gemma4:26b`** — local-first; cloud is opt-in.
- **PID-suffixed verdict file** — low-cost guard against concurrent session collision.
- **No `system` role transformation for Anthropic** — the review prompt is a user turn, not a system prompt. Adding a system→top-level transform is scope creep for a hypothetical future feature; rejected.

## Risks / open questions
- Anthropic API shape differs from OpenAI (`messages` endpoint, different auth header `x-api-key`, response at `.content[0].text` not `.choices[0].message.content`). Must be handled as a distinct path.
- OpenAI-compatible endpoints may require `OPENAI_API_KEY` or may be unauthenticated — skill should attempt the call and surface auth errors rather than pre-checking.
- Session history format: ollama and OpenAI both use `{"role","content"}` messages array; Anthropic uses the same shape for non-system turns. Since the review prompt is a user turn, the messages array is compatible across all three — no session format change needed.

## Out of scope
- Act 1 model switching (the grill is always Claude — that's the point).
- Streaming responses (all providers called with `stream:false` / no streaming).
- Storing preferred model persistently (use env var or always pass the arg).
- Token/cost tracking.
- Anthropic `system` prompt transformation (not used in this skill).
