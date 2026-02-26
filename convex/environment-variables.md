# Environment Variable Patterns for Convex

Managing environment variables in Convex has quirks that differ from traditional frameworks. This guide covers the patterns that work reliably for both self-hosted (local Docker) and cloud deployments.

## How Convex Env Vars Work

Convex environment variables are **not** set via `.env` files. They're stored in the Convex deployment itself and accessed at runtime via `process.env` in actions or via the Convex dashboard.

There are two ways to set them:
1. **CLI:** `npx convex env set KEY value`
2. **Dashboard:** Convex dashboard > Settings > Environment Variables

For automated workflows, the CLI is essential.

## Self-Hosted (Local Docker)

### Prerequisites

The CLI needs two environment variables to find your local deployment:

```bash
export CONVEX_SELF_HOSTED_URL=http://127.0.0.1:<admin-port>
export CONVEX_SELF_HOSTED_ADMIN_KEY=<instance-name>|<key>
```

The admin port is typically `3210` (the Convex backend's admin interface). The admin key is generated when the Docker container first starts.

### Getting the Admin Key

```bash
docker exec <container-name> ./generate_admin_key.sh
```

This outputs the admin key derived from the instance secret stored in the Docker volume. It's **deterministic** — same volume = same key. But if the container is **recreated** (not just restarted), the key may change.

### Setting Variables

```bash
# Simple values
npx convex env set SITE_URL "https://myapp.example.com"

# Values with special characters (use -- separator)
npx convex env set -- MY_KEY "value-with-dashes"

# Check current value
npx convex env get MY_KEY

# List all
npx convex env list
```

### Sync Script Pattern

For projects with many env vars, maintain a local file and sync it to Convex:

```bash
#!/bin/bash
# sync-env-to-convex.sh
set -e

ENV_FILE=".env.convex.local"
[ -f "$ENV_FILE" ] || { echo "Missing $ENV_FILE"; exit 1; }

# Ensure CLI can reach deployment
export CONVEX_SELF_HOSTED_URL="http://127.0.0.1:3210"
export CONVEX_SELF_HOSTED_ADMIN_KEY="$(docker exec my-backend ./generate_admin_key.sh 2>/dev/null | tail -1)"

while IFS= read -r line; do
    # Skip comments and empty lines
    [[ "$line" =~ ^[[:space:]]*# ]] && continue
    [[ -z "$line" ]] && continue

    key="${line%%=*}"
    value="${line#*=}"

    # Strip surrounding quotes
    value="${value%\"}"
    value="${value#\"}"

    echo "Setting $key..."
    npx convex env set -- "$key" "$value"
done < "$ENV_FILE"

echo "Sync complete"
```

**Key design decisions:**
- Uses `--` separator so values starting with dashes aren't parsed as CLI flags
- Strips surrounding quotes (env files often quote values, Convex doesn't want the quotes)
- Reads admin key fresh from container (handles key changes after recreation)
- Processes line-by-line to handle multiline-unfriendly values

### What NOT to Put in the Sync File

Some variables should be set separately, not in the sync file:

- **JWT keys** (`JWT_PRIVATE_KEY`, `JWKS`) — These are multiline/complex values that break line-by-line parsing. Set them with a dedicated script. See [jwt-setup.md](./jwt-setup.md).
- **One-time secrets** — API keys that shouldn't be overwritten on every sync

## Cloud (Hosted Deployments)

### Authentication

Cloud deployments use a deploy key instead of admin key + URL:

```bash
export CONVEX_DEPLOY_KEY="prod:<deployment-name>|<key>"
```

Get your deploy key from the Convex dashboard > Settings > Deploy Key.

### Setting Variables

```bash
# With deploy key exported
npx convex env set MY_KEY "my-value" --prod

# Or for preview/dev deployments
npx convex env set MY_KEY "my-value" --preview-name my-preview
```

### Piping Values

For values that might confuse the shell (PEM keys, JSON), pipe them:

```bash
echo "$MY_COMPLEX_VALUE" | npx convex env set MY_KEY --prod
```

## Common Pitfalls

### 1. Local vars override deploy keys

If you use direnv or have `CONVEX_SELF_HOSTED_URL` set in your shell, it takes precedence over `CONVEX_DEPLOY_KEY`. The CLI silently targets your local deployment instead of cloud.

```bash
# WRONG — local vars win
export CONVEX_SELF_HOSTED_URL="http://127.0.0.1:3210"
CONVEX_DEPLOY_KEY="prod:my-app|key" npx convex env set FOO bar --prod

# CORRECT — unset local vars first
unset CONVEX_SELF_HOSTED_URL CONVEX_SELF_HOSTED_ADMIN_KEY CONVEX_URL
export CONVEX_DEPLOY_KEY="prod:my-app|key"
npx convex env set FOO bar --prod
```

### 2. Admin key stale after container recreation

The admin key is derived from a secret in the Docker volume. If you recreate the container (e.g., `docker compose up --force-recreate`), the secret may regenerate.

**Symptom:** `InvalidHeaderFailure: Invalid authentication header`

**Fix:** Re-read the admin key:
```bash
docker exec <container> ./generate_admin_key.sh
```

### 3. Quotes in values

Env files often quote values: `MY_KEY="my-value"`. The Convex CLI stores the quotes as part of the value. Strip them in your sync script.

### 4. Pipe character in admin key

The admin key format is `instance-name|actual-key`. In bash, `|` is the pipe operator. Always quote the key:

```bash
# WRONG — shell interprets | as pipe
export CONVEX_SELF_HOSTED_ADMIN_KEY=my-instance|abc123

# CORRECT
export CONVEX_SELF_HOSTED_ADMIN_KEY="my-instance|abc123"
```

### 5. `npx convex env set` hangs

If the command hangs, it usually means the CLI can't reach the deployment. Check:
- Is the Docker container running?
- Is `CONVEX_SELF_HOSTED_URL` pointing to the right port?
- Is the admin key current?

Use `curl http://127.0.0.1:3210/version` to verify the backend is reachable.

## Environment Separation Pattern

For projects with multiple environments (dev, staging, prod), keep separate env files and target each deployment explicitly:

```
.env.convex.local          # Local dev values
.env.convex.staging        # Staging values (checked in, no secrets)
.env.convex.production     # Prod values (checked in, no secrets)
```

Secrets go in a vault or CI/CD system, not in files.

```bash
# Deploy to staging
unset CONVEX_SELF_HOSTED_URL CONVEX_SELF_HOSTED_ADMIN_KEY
export CONVEX_DEPLOY_KEY="$STAGING_DEPLOY_KEY"
./sync-env-to-convex.sh .env.convex.staging

# Deploy to production
export CONVEX_DEPLOY_KEY="$PROD_DEPLOY_KEY"
./sync-env-to-convex.sh .env.convex.production
```

## Further Reading

- [Convex Environment Variables docs](https://docs.convex.dev/production/environment-variables)
- [jwt-setup.md](./jwt-setup.md) — JWT key setup (the hardest env var to get right)
