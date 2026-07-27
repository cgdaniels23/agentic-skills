# Deployment

Deployment guide for the Perplexity skill runner. Fill in server details when infrastructure is provisioned.

## Server

| Field | Value |
|-------|-------|
| **SERVER_IP** | |
| **DOMAIN** | |

## Prerequisites

- Node.js 20+
- `PERPLEXITY_API_KEY` set in server environment (not committed to git)
- Optional: Supabase project with `skill_runs` and `artifacts` tables

## Environment setup

```bash
cp .env.example .env.local
# Edit .env.local with real values on the server — never commit this file
```

## Build and run

```bash
# After Next.js runtime is scaffolded:
npm ci
npm run build
npm run start
```

## Deployment commands

```bash
# SERVER_IP=
# DOMAIN=

# Example placeholder workflow — replace with your host-specific commands
# ssh user@SERVER_IP
# cd /var/www/perplexity-skill-mcp
# git pull origin main
# npm ci
# npm run build
# pm2 restart perplexity-skill-mcp
```

## Health check

```bash
# curl https://DOMAIN/api/health
```

## Rollback

```bash
# git checkout <previous-tag-or-commit>
# npm ci && npm run build && pm2 restart perplexity-skill-mcp
```
