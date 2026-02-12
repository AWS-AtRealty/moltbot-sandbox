# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Cloudflare Worker that runs [OpenClaw](https://github.com/openclaw/openclaw) (formerly Moltbot/Clawdbot) personal AI assistant in a Cloudflare Sandbox container. The worker proxies HTTP/WebSocket requests to the OpenClaw gateway running inside the container, serves an admin UI for device management, and handles R2 backup/restore for persistence.

## Commands

```bash
npm run start           # Local dev (wrangler dev) - requires .dev.vars
npm run dev             # Vite dev server (admin UI only, hot reload)
npm run build           # Build admin UI (React → dist/client/)
npm run deploy          # Build + deploy to Cloudflare
npm test                # Run tests (vitest, single run)
npm run test:watch      # Run tests in watch mode
npm run test:coverage   # Run tests with v8 coverage
npm run typecheck       # TypeScript type checking (tsc --noEmit)
npm run lint            # Lint with oxlint
npm run lint:fix        # Auto-fix lint issues
npm run format          # Format with oxfmt
npm run format:check    # Check formatting
npx wrangler tail       # View live production logs
npx wrangler secret list # Check configured secrets
```

Run a single test file: `npx vitest run src/auth/jwt.test.ts`

## Architecture

```
Browser → Cloudflare Worker (Hono) → Cloudflare Sandbox Container (OpenClaw Gateway)
```

- **Worker** (`src/index.ts`): Hono app that manages sandbox lifecycle, authenticates via Cloudflare Access JWTs, proxies HTTP/WebSocket to the container on port 18789, intercepts WebSocket errors, and runs a cron job every 5 minutes for R2 backup sync.
- **Container** (`Dockerfile`, `start-openclaw.sh`): Based on `cloudflare/sandbox:0.7.0` with Node.js 22 + OpenClaw CLI. Startup: R2 restore → `openclaw onboard` → config patching → gateway launch.
- **Admin UI** (`src/client/`): React 19 SPA served at `/_admin/` for device pairing, gateway restart, and R2 backup management.

### Key Modules

| Directory | Purpose |
|-----------|---------|
| `src/auth/` | Cloudflare Access JWT verification, JWKS caching, auth middleware |
| `src/gateway/` | Container process lifecycle, env var building, R2 mounting/sync |
| `src/routes/` | API routes (`/api/*`), admin UI serving (`/_admin/*`), CDP shim (`/cdp/*`), debug endpoints |
| `src/client/` | React admin UI (Vite build) |
| `skills/` | Pre-installed skills copied into the container |

### Ephemeral Container Model

The Cloudflare Sandbox container is **ephemeral** — all filesystem state is lost on restart (sleep, redeploy, crash). Only what's baked into the Docker image (`Dockerfile`) survives. This means:

- **Anything needed at runtime must be either**: (1) installed in the `Dockerfile` (Node.js, OpenClaw CLI, pnpm, rsync, skills), or (2) restored from R2 backup at startup by `start-openclaw.sh`.
- **R2 backup/restore is the persistence layer**: Config and workspace are synced to R2 every 5 minutes via cron and restored on cold start. Without R2, everything is ephemeral. The R2 bucket (`moltbot-data`) has this structure:
  - `openclaw/` — contains `openclaw.json` (main config) and other config files, restored to `/root/.openclaw/`. A copy of this file is kept at `openclaw.json` in the repo root for reference.
  - `workspace/` — contains agent folders (IDENTITY.md, memory/, assets/, etc.), restored to `/root/clawd/`
  - `skills/` — custom skills, restored to `/root/clawd/skills/`
  - `gogcli/` — gogcli (GOG skill) config and encrypted keyring, restored to `/root/.config/gogcli/`
  - `.last-sync` — timestamp file used to determine if R2 backup is newer than local
- **The startup script runs on every cold start** (`start-openclaw.sh`): It restores from R2, runs `openclaw onboard` if no config exists, patches the config with env vars (channels, auth, proxies), and launches the gateway. This is the place to add any setup that needs to happen on each container boot.
- **Adding a new system dependency** (e.g., apt package, global npm package) requires adding it to the `Dockerfile` and bumping the cache bust comment to force a rebuild.
- **Adding runtime files** that should persist across restarts requires adding them to the R2 sync logic in `src/gateway/sync.ts` (worker-side) and the restore logic in `start-openclaw.sh` (container-side).

### Request Flow

1. Auth middleware validates Cloudflare Access JWT (skipped with `DEV_MODE=true`)
2. Sandbox Durable Object initialized
3. Gateway started if not running (find existing process → start if missing → wait for port 18789)
4. Request proxied to container (HTTP or WebSocket upgrade)

### Environment Variable Mapping

Worker env vars are mapped to container env vars in `src/gateway/env.ts`. Key mapping: `MOLTBOT_GATEWAY_TOKEN` → `OPENCLAW_GATEWAY_TOKEN`, `DEV_MODE` → `OPENCLAW_DEV_MODE`.

## Code Style

- **Linter**: oxlint (Rust-based, config in `.oxlintrc.json`)
- **Formatter**: oxfmt (Rust-based, config in `.oxfmtrc.json`)
- **Style**: Single quotes, semicolons, trailing commas, 2-space indent, 100 char line width, LF line endings
- **TypeScript**: Strict mode. Prefer explicit types for function signatures. All env bindings typed in `MoltbotEnv` interface (`src/types.ts`).
- **Naming**: Files kebab-case, components PascalCase, functions camelCase, constants UPPER_SNAKE_CASE
- **Routes**: Keep handlers thin — extract logic to `src/gateway/` or `src/utils/`. Use Hono's `c.json()`, `c.html()` for responses.

## Testing

Tests use Vitest with globals enabled. Test files are colocated with source (`*.test.ts`). Frontend tests (`src/client/`) are excluded.

Coverage areas: `auth/` (JWT, JWKS, middleware), `gateway/` (env, process, r2, sync).

## Common Tasks

### Adding a New API Endpoint

1. Add route handler in `src/routes/api.ts`
2. Add types in `src/types.ts` if needed
3. Update `src/client/api.ts` if frontend needs it
4. Add tests

### Adding a New Environment Variable

1. Add to `MoltbotEnv` in `src/types.ts`
2. If passed to container, add to `buildEnvVars()` in `src/gateway/env.ts`
3. Update `.dev.vars.example`

### Adding an External Skill Binary

When an OpenClaw bundled skill requires an external binary (check `requires.bins` in the skill's `SKILL.md`), follow this checklist. Use pre-built binaries from GitHub releases when available; avoid installing Homebrew in the container.

1. **Dockerfile** — Download the pre-built binary for linux amd64/arm64:
   ```dockerfile
   ENV <TOOL>_VERSION=x.y.z
   RUN ARCH="$(dpkg --print-architecture)" \
       && case "${ARCH}" in \
            amd64) TOOL_ARCH="amd64" ;; \
            arm64) TOOL_ARCH="arm64" ;; \
            *) echo "Unsupported architecture: ${ARCH}" >&2; exit 1 ;; \
          esac \
       && curl -fsSLk <RELEASE_URL> -o /tmp/tool.tar.gz \
       && tar -xzf /tmp/tool.tar.gz -C /usr/local/bin <binary-name> \
       && rm /tmp/tool.tar.gz \
       && <binary-name> --version
   ```
   Bump the `# Build cache bust:` comment to force a rebuild.

2. **Credential persistence** (if the tool stores auth tokens):
   a. Determine where the tool stores config/credentials (usually `~/.config/<tool>/`)
   b. If it uses an OS keyring, configure it for file-based storage instead (env var or config)
   c. Add a Worker secret for any required password/key:
      - `src/types.ts` → add to `MoltbotEnv`
      - `src/gateway/env.ts` → add to `buildEnvVars()`
   d. Add R2 backup/restore:
      - `src/gateway/sync.ts` → add rsync command for the config dir → `R2:/<tool>/`
      - `start-openclaw.sh` → add restore block (copy from R2 mount to config dir)
      - `start-openclaw.sh` → export any required env vars (e.g., keyring backend)

3. **Update documentation**:
   - Add new env vars to the deployed configuration table in this file
   - Add the tool's R2 path to the R2 bucket structure list above

## Key Gotchas

- **CLI commands**: Always include `--url ws://localhost:18789`. Commands take 10-15 seconds due to WebSocket overhead.
- **R2 rsync**: Use `rsync -r --no-times` (s3fs doesn't support timestamps). Never `rm -rf` the mount directory — it IS the R2 bucket.
- **WebSocket in local dev**: `wrangler dev` has issues with WebSocket proxying through sandbox. HTTP works; deploy for full WS functionality.
- **OpenClaw config**: `agents.defaults.model` must be `{ "primary": "model/name" }` not a string. `gateway.mode` must be `"local"`. No `webchat` channel.
- **Dockerfile cache busting**: When changing `start-openclaw.sh`, bump the `# Build cache bust:` comment version.
- **Windows line endings**: `start-openclaw.sh` must have LF endings or it fails with exit code 126 in the container.

## Deployed Configuration

The production Cloudflare Worker (`moltbot-sandbox`) has the following variables and secrets configured:

| Name | Type | Purpose |
|------|------|---------|
| `ANTHROPIC_API_KEY` | Secret | Anthropic API key for Claude |
| `OPENAI_API_KEY` | Secret | OpenAI API key |
| `MOLTBOT_GATEWAY_TOKEN` | Secret | Gateway access token |
| `CF_ACCESS_AUD` | Plaintext | Cloudflare Access application audience |
| `CF_ACCESS_TEAM_DOMAIN` | Plaintext | Cloudflare Access team domain |
| `CF_ACCOUNT_ID` | Secret | Cloudflare account ID (for R2) |
| `R2_ACCESS_KEY_ID` | Secret | R2 storage access key |
| `R2_SECRET_ACCESS_KEY` | Secret | R2 storage secret key |
| `CDP_SECRET` | Secret | Shared secret for browser automation CDP endpoints |
| `WORKER_URL` | Secret | Public worker URL (for CDP) |
| `ELEVENLABS_API_KEY` | Secret | ElevenLabs API key (TTS) |
| `GOG_KEYRING_PASSWORD` | Secret | Password for gogcli file-based keyring (GOG skill OAuth tokens) |

## AI Contributions Policy

All AI usage must be disclosed. PRs must reference accepted issues, be fully human-tested, and have human-in-the-loop review. See `CONTRIBUTING.md` for details.
