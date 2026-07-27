# Perplexity Skill Runner

API-first, reusable skill system built on versioned JSON manifests and server-side Perplexity completions. Designed for a solo operator: fast MVP, strong safety boundaries, and future MCP portability.

## Directory structure

```
perplexity-skill-runner/
├── skills/                          # Portable, versioned skill manifests
│   ├── research_brief_v1.json
│   └── proposal_generator_v1.json
├── schemas/
│   └── skill-manifest.schema.json   # JSON Schema draft 2020-12 for manifests
├── connectors/
│   └── README.md                    # Future outbound integrations (v1: none)
├── docs/
│   └── architecture.md              # System design and lifecycle
└── README.md
```

| Path | Purpose |
|------|---------|
| `skills/` | Immutable skill definitions: input/output schemas, prompts, model settings, safety rules |
| `schemas/` | Shared JSON Schema for validating manifest files at build or deploy time |
| `connectors/` | Placeholder for future explicit, operator-approved external actions |
| `docs/` | Architecture and design decisions |

## Runtime contract

### `POST /api/run-skill`

**Request**

```json
{
  "skill_id": "research_brief_v1",
  "input": {
    "question": "Should we adopt tool X for our team?",
    "audience": "Engineering leadership",
    "decision_needed": "Go/no-go on a 90-day pilot"
  }
}
```

**Success response** (`200`)

```json
{
  "skill_id": "research_brief_v1",
  "version": "1.0.0",
  "run_id": "uuid",
  "output": { }
}
```

**Error response** (`4xx` / `5xx`)

```json
{
  "error": "validation_failed",
  "message": "input.question must be at least 10 characters",
  "run_id": "uuid"
}
```

### Execution steps

1. Validate request input against the skill's `input_schema`
2. Load the skill manifest by `skill_id`
3. Render prompt blocks with validated input (data only, never as instructions)
4. Call Perplexity API server-side only
5. Parse and validate JSON against `output_schema`
6. Persist a run record (skill id, version, request, rendered prompt, response, duration, status, errors)
7. Return validated output

## Environment variables

| Variable | Required | Description |
|----------|----------|-------------|
| `PERPLEXITY_API_KEY` | Yes | Perplexity API key. **Server-side only.** Never expose to the browser or commit to git. |

Optional (when Supabase persistence is added):

| Variable | Description |
|----------|-------------|
| `SUPABASE_URL` | Supabase project URL |
| `SUPABASE_SERVICE_ROLE_KEY` | Service role key for server-side inserts |

## Skill versioning rules

1. **Immutable manifests** — Once deployed, a skill file is never edited in place for production use.
2. **New version = new file** — Prompt, schema, or model changes require a new file (e.g. `research_brief_v2.json`) with updated `id` and `version`.
3. **Semver** — Use `MAJOR.MINOR.PATCH`. Breaking input/output schema changes bump MAJOR; prompt refinements bump MINOR; typo fixes bump PATCH.
4. **Reproducibility** — Run records store `skill_id` + `version` so historical outputs can be traced to exact manifest content.

## Safety rules

- **No browser API key** — All Perplexity calls happen in server route handlers or edge functions.
- **User content is data** — Validated input is injected into prompt templates; it cannot override system prompts or safety rules.
- **No external writes in v1** — `allow_external_writes: false` on all v1 skills. No email, CRM, or webhook side effects.
- **Denylists** — Each skill declares prohibited behaviors (fabricated citations, invented pricing, etc.) in `safety_rules.denylist`.
- **Citations required** — Research skills set `require_citations: true` and enforce source arrays in output schema.
- **JSON-only output** — Responses must parse as JSON matching `output_schema`; malformed output is a failed run.

## v1 boundary

| In scope | Out of scope |
|----------|--------------|
| Two skills: `research_brief_v1`, `proposal_generator_v1` | MCP server or tools |
| One provider: Perplexity (`sonar-pro`) | Multi-provider routing |
| Server-side `/api/run-skill` endpoint | Client-side model calls |
| Run persistence (Supabase) | Autonomous external actions |
| JSON Schema validation (in/out) | Skill authoring UI |
| Manifest-driven prompt rendering | Connectors (email, CRM, etc.) |

## Available skills

### `research_brief_v1`

Decision-ready, cited research brief. Requires `question`, `audience`, `decision_needed`. Optional: `context`, `geography`, `time_horizon`, `must_include`.

### `proposal_generator_v1`

Concise B2B proposal draft. Requires `client_name`, `client_problem`, `desired_outcome`, `offer`, `scope`. Optional: `timeline`, `investment`, `proof_points`, `tone`. Never invents case studies, pricing, or legal terms.

## Local validation

Validate manifests against the schema before deploy:

```bash
# Example using ajv-cli (after npm install -g ajv-cli)
ajv validate -s schemas/skill-manifest.schema.json -d "skills/*.json"
```

See [docs/architecture.md](./docs/architecture.md) for the full system design.

## GitHub and Deployment Workflow

1. **Clone** — `git clone git@github.com:<your-user>/perplexity-skill-mcp.git`
2. **Configure** — Copy `.env.example` to `.env.local` and set `PERPLEXITY_API_KEY` locally. Never commit `.env` or `.env.local`.
3. **Validate manifests** — Run the local validation command below before every deploy.
4. **Commit** — Use descriptive messages; bump skill version files instead of editing deployed manifests in place.
5. **Deploy** — Follow [DEPLOYMENT.md](./DEPLOYMENT.md) once `SERVER_IP` and `DOMAIN` are set.

Secrets (`PERPLEXITY_API_KEY`, Supabase service role key) live only in server environment variables or a local `.env.local` file excluded by `.gitignore`.
