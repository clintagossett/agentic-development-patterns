# JWT Key Setup for Convex Auth

Convex Auth (`@convex-dev/auth`) uses RS256 for signing JWTs. This requires two environment variables:

- **`JWT_PRIVATE_KEY`** — RSA private key in PEM format (signs tokens)
- **`JWKS`** — JSON Web Key Set with the public key (verifies tokens)

Getting these set correctly is the #1 source of auth issues agents hit. Here's how to do it right.

## Generate Keys

```bash
# Generate RSA-2048 key pair
temp_dir=$(mktemp -d)
openssl genrsa -out "$temp_dir/private.pem" 2048 2>/dev/null
openssl rsa -in "$temp_dir/private.pem" -pubout -out "$temp_dir/public.pem" 2>/dev/null

# Read private key
JWT_PRIVATE_KEY=$(cat "$temp_dir/private.pem")

# Extract modulus as base64url (NOT standard base64)
n=$(openssl rsa -in "$temp_dir/private.pem" -pubout -outform DER 2>/dev/null | \
    openssl rsa -pubin -inform DER -text -noout 2>/dev/null | \
    grep -A 100 "Modulus:" | tail -n +2 | tr -d ' :\n' | \
    xxd -r -p | base64 | tr '+/' '-_' | tr -d '=\n')

# Build JWKS
JWKS=$(jq -n --arg n "$n" --arg e "AQAB" \
    '{"keys":[{"kty":"RSA","n":$n,"e":$e,"use":"sig","alg":"RS256","kid":"convex-auth-key"}]}')

rm -rf "$temp_dir"
```

## Set Keys in Convex

### Self-Hosted (Local Docker)

```bash
# Use -- separator so PEM dashes aren't parsed as CLI flags
npx convex env set -- JWT_PRIVATE_KEY "$JWT_PRIVATE_KEY"
npx convex env set -- JWKS "$JWKS"
```

The CLI needs to find your deployment. Ensure these are in your `.env.local` or environment:

```bash
CONVEX_SELF_HOSTED_URL=http://127.0.0.1:<admin-port>
CONVEX_SELF_HOSTED_ADMIN_KEY=<instance-name>|<key>
```

### Cloud (Hosted Deployments)

```bash
# Pipe the value to avoid shell interpretation issues
export CONVEX_DEPLOY_KEY="prod:<deployment-name>|<key>"
echo "$JWT_PRIVATE_KEY" | npx convex env set JWT_PRIVATE_KEY --prod
echo "$JWKS" | npx convex env set JWKS --prod
```

## Common Pitfalls

### 1. PEM value parsed as CLI flag

The private key starts with `-----BEGIN PRIVATE KEY-----`. Without the `--` separator, the CLI interprets the dashes as option flags and fails silently or with a confusing error.

```bash
# WRONG
npx convex env set JWT_PRIVATE_KEY "-----BEGIN PRIVATE KEY-----..."

# CORRECT
npx convex env set -- JWT_PRIVATE_KEY "$JWT_PRIVATE_KEY"
```

### 2. Standard base64 instead of base64url in JWKS

The JWKS `n` (modulus) field must use base64url encoding:
- Replace `+` with `-`
- Replace `/` with `_`
- Strip all `=` padding

Standard base64 produces a key that looks valid but silently fails signature verification. The `tr '+/' '-_' | tr -d '='` in the generation script handles this.

### 3. Admin key becomes stale

The self-hosted Convex Docker container generates its admin key on first start and persists it in a Docker volume. If you **recreate** the container (not just restart), the key regenerates and your local `.env` files have a stale key.

**Symptom:** `InvalidHeaderFailure: Invalid authentication header`

**Fix:** Retrieve the new admin key:

```bash
docker exec <container-name> ./generate_admin_key.sh
```

Then update your `.env.local` or `.env.nextjs.local` with the new key.

### 4. direnv vars conflict with deploy keys

If you use direnv to set `CONVEX_SELF_HOSTED_URL` for local dev, those vars will override `CONVEX_DEPLOY_KEY` when you try to target a cloud deployment.

```bash
# WRONG — local vars take precedence
CONVEX_DEPLOY_KEY="prod:my-app|key" npx convex env set FOO bar --prod

# CORRECT — unset local vars first
unset CONVEX_SELF_HOSTED_URL CONVEX_SELF_HOSTED_ADMIN_KEY CONVEX_URL
CONVEX_DEPLOY_KEY="prod:my-app|key" npx convex env set FOO bar --prod
```

### 5. JWT keys in your env sync file

If you put `JWT_PRIVATE_KEY` or `JWKS` in the file you sync to Convex (e.g., `.env.convex.local`), they'll get overwritten on every sync. Keep JWT keys managed separately — generate once, set once, don't touch unless rotating.

### 6. Docker volume destruction

The Docker volume holds the Convex data **and** the instance secret (used to derive the admin key). Destroying the volume (`docker compose down -v`, `docker volume rm`) means:
- Admin key changes
- All data is lost
- JWT keys set via `npx convex env set` are gone
- All user sessions are invalidated

Never run `docker compose down -v` or `docker system prune` in a Convex development environment.

## Automation Script Pattern

Here's the pattern for a reusable setup script that handles all of this:

```bash
#!/bin/bash
set -e

# 1. Check container is running
CONTAINER="my-app-backend"
docker ps --format '{{.Names}}' | grep -q "^${CONTAINER}$" || {
    echo "Container not running"; exit 1
}

# 2. Get admin key
ADMIN_KEY=$(docker exec "$CONTAINER" ./generate_admin_key.sh | grep -v "^Admin key:" | tail -1)

# 3. Check if JWT keys already exist
export CONVEX_SELF_HOSTED_URL="http://127.0.0.1:3210"
export CONVEX_SELF_HOSTED_ADMIN_KEY="$ADMIN_KEY"

existing=$(npx convex env get JWT_PRIVATE_KEY 2>/dev/null || echo "")
if [ -n "$existing" ]; then
    echo "JWT keys already set — skipping generation"
    exit 0
fi

# 4. Generate and set keys (using the commands from "Generate Keys" above)
# ... generate JWT_PRIVATE_KEY and JWKS ...

npx convex env set -- JWT_PRIVATE_KEY "$JWT_PRIVATE_KEY"
npx convex env set -- JWKS "$JWKS"

echo "JWT keys configured"
```

Key design decisions:
- **Idempotent:** Checks if keys exist before generating new ones
- **Non-destructive:** Never overwrites existing keys (use a `--force` flag if needed)
- **Admin key refresh:** Always retrieves fresh admin key from container

## Verification

```bash
# Check keys are set
npx convex env get JWT_PRIVATE_KEY | head -1
# Should output: -----BEGIN PRIVATE KEY-----

npx convex env get JWKS | jq .keys[0].kty
# Should output: "RSA"
```

## Further Reading

- [Convex Auth docs](https://labs.convex.dev/auth)
- [@convex-dev/auth package](https://www.npmjs.com/package/@convex-dev/auth)
