# Architecture

Perplexity Skill Runner is an API-first, manifest-driven system for running versioned AI skills server-side. It is designed for a solo operator, fast MVP delivery, strong safety boundaries, and future MCP portability.

## Principles

| Principle | Implementation |
|-----------|----------------|
| Portable skills | Versioned JSON manifests in `skills/` |
| Server-side only | Perplexity API key never exposed to the browser |
| Data, not instructions | User input is validated and injected as structured data |
| Immutable versions | Prompt changes require a new skill version file |
| Observable runs | Every invocation persisted with full audit trail |
| Minimal v1 scope | Two skills, one provider, no MCP, no external writes |

## Components

```
┌─────────────┐     POST /api/run-skill      ┌──────────────────┐
│   Client    │ ───────────────────────────► │  Next.js API     │
│  (browser)  │                              │  Route Handler   │
└─────────────┘                              └────────┬─────────┘
                                                      │
                    ┌─────────────────────────────────┼─────────────────────────────────┐
                    │                                 │                                 │
                    ▼                                 ▼                                 ▼
           ┌────────────────┐              ┌──────────────────┐              ┌─────────────────┐
           │ Skill Registry │              │ Prompt Renderer  │              │ Run Persistence │
           │ (JSON files)   │              │ (template bind)  │              │ (Supabase)      │
           └────────────────┘              └────────┬─────────┘              └─────────────────┘
                                                    │
                                                    ▼
                                           ┌──────────────────┐
                                           │ Perplexity API   │
                                           │ (server-side)    │
                                           └──────────────────┘
```

## Request lifecycle

1. **Authenticate** — Verify caller (session, API key, or internal-only in v1).
2. **Validate input** — Parse request body; validate against the skill's `input_schema`.
3. **Load manifest** — Resolve `skill_id` to a JSON file in `skills/`. Reject unknown or deprecated IDs.
4. **Render prompt** — Bind validated input into `prompt_template.instructions` placeholders. Wrap user values in clear data delimiters. Never concatenate raw user text into the system prompt.
5. **Call Perplexity** — Server-side POST to Perplexity chat completions using `model_settings`. Attach `output_contract` to the user message.
6. **Parse response** — Extract JSON from model output (strip accidental markdown fences if present).
7. **Validate output** — Validate parsed JSON against `output_schema`.
8. **Persist run** — Write a `skill_runs` record: skill id, version, request, rendered prompt, raw response, validated output, duration, status, errors.
9. **Return** — Return validated output JSON to the caller.

## Prompt rendering rules

- `prompt_template.system` is fixed per skill version and never modified at runtime.
- User fields replace `{{field_name}}` tokens in `instructions` only.
- Missing optional fields render as `(not provided)`.
- Arrays render as bullet lists.
- All user strings are treated as untrusted data; the renderer must not interpret embedded "ignore previous instructions" text as commands.

## Safety model

- `safety_rules.allow_external_writes: false` — Runtime must not perform side effects beyond the Perplexity call and run persistence.
- `safety_rules.require_citations: true` — Enforced at prompt level for research skills; future runtime may add post-validation.
- `safety_rules.denylist` — Documented prohibitions included in system prompt and available for future automated checks.

## Versioning

- Skill IDs include a version suffix: `research_brief_v1`, `proposal_generator_v1`.
- The `version` field uses semver (`1.0.0`).
- Production prompt or schema changes require a new file (e.g. `research_brief_v2.json`) with an incremented ID and version.
- Old versions remain loadable for audit reproducibility.

## v1 non-goals

- MCP server or tool exposure
- Autonomous external writes (email, CRM, webhooks)
- Multi-provider model routing
- Client-side Perplexity calls
- Skill authoring UI

## Future MCP portability

Skill manifests are provider-agnostic JSON. An MCP adapter can:

1. Expose each skill as a tool with `input_schema` as the tool input schema.
2. Delegate execution to the same `/api/run-skill` handler.
3. Return validated `output_schema` results as structured tool output.

No manifest changes required for MCP adoption.
