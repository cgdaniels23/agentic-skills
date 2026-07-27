# Connectors

Future home for outbound integrations (email, CRM, Slack, webhooks, etc.).

## v1 boundary

**No connectors are implemented in v1.** The skill runner performs read-only Perplexity API completions and persists run records. It does not send email, update CRMs, or trigger external workflows autonomously.

## Design intent

Connectors will be optional, explicit, operator-initiated side effects — never implicit model-driven writes. Each connector should:

1. Declare required scopes and credentials separately from skill manifests.
2. Be invoked only after a successful skill run and explicit operator approval.
3. Log every external action with a reference to the originating `skill_run` record.

## MCP portability

When MCP support is added, connectors may be exposed as MCP tools. Skill manifests themselves remain provider-agnostic JSON; connectors are runtime adapters, not prompt instructions.
