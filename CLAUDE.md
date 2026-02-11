# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Cloudflare Worker that runs [OpenClaw](https://github.com/openclaw/openclaw) in a Cloudflare Sandbox container. The worker proxies HTTP/WebSocket to the OpenClaw gateway running inside the container, handles Cloudflare Access authentication, manages container lifecycle, and syncs state to R2.

## Commands

```bash
npm test                # Run tests (vitest, single run)
npm run test:watch      # Tests in watch mode
npm run test:coverage   # Tests with coverage report
npm run lint            # Lint with oxlint
npm run lint:fix        # Lint and auto-fix
npm run format          # Format with oxfmt
npm run format:check    # Check formatting
npm run typecheck       # TypeScript type checking (tsc --noEmit)
npm run build           # Build worker + admin UI (vite)
npm run start           # Local worker dev (wrangler dev)
npm run deploy          # Build + deploy to Cloudflare
```

To run a single test file: `npx vitest run src/auth/jwt.test.ts`

## Architecture

```
Browser → Cloudflare Worker (src/index.ts) → Sandbox Container → OpenClaw Gateway (:18789)
```

- **Worker** (`src/index.ts`): Hono app that starts the sandbox, validates CF Access JWTs, mounts R2, and proxies all HTTP/WebSocket traffic to the container gateway.
- **Container**: Built from `Dockerfile` (cloudflare/sandbox base + Node 22 + openclaw). Startup via `start-openclaw.sh` restores R2 backups, onboards, patches config, launches gateway.
- **Admin UI** (`src/client/`): React SPA served at `/_admin/` for device management and gateway controls. Built with Vite, base path `/_admin/`.

### Source Layout

- `src/auth/` — CF Access JWT verification middleware (jwt.ts, jwks.ts, middleware.ts)
- `src/gateway/` — Container management: process lifecycle, env var mapping, R2 mount/sync
- `src/routes/` — Route handlers: public (health/status), api (devices/gateway), admin-ui, debug, cdp
- `src/client/` — React admin UI (excluded from tests)
- `src/types.ts` — All TypeScript interfaces (`MoltbotEnv`, `AppEnv`, etc.)
- `src/config.ts` — Constants: port 18789, 180s startup timeout, R2 mount path

### Key Patterns

- Public routes mount before auth middleware; protected routes require CF Access JWT
- Catch-all route proxies to container gateway with WebSocket relay and error message transformation
- `buildEnvVars()` in `src/gateway/env.ts` maps worker secrets to container env vars
- `DEV_MODE` bypasses all auth + device pairing; `E2E_TEST_MODE` skips auth but keeps pairing
- CLI calls must include `--url ws://localhost:18789` and take 10-15s due to WebSocket overhead
- R2 mounted at `/data/moltbot` via s3fs; use `rsync -r --no-times` (s3fs can't set timestamps)
- R2 prefix is `openclaw/` (legacy `clawdbot/` handled automatically)

## Code Style

- **Linter**: oxlint with plugins: react, typescript, unicorn, oxc, import, vitest
- **Formatter**: oxfmt — single quotes, 2-space indent, trailing commas, 100 char line width, LF endings
- Explicit types on function signatures; strict TypeScript mode
- Thin route handlers — extract business logic to gateway/ modules
- Use Hono context methods (`c.json()`, `c.html()`) for responses

## Testing

Vitest with colocated `*.test.ts` files. Test utilities in `src/test-utils.ts` provide `createMockEnv()`, `createMockSandbox()`, `suppressConsole()`, etc.

## Adding New Features

**New API endpoint**: Add handler in `src/routes/api.ts` → types in `src/types.ts` → client in `src/client/api.ts` → tests.

**New environment variable**: Add to `MoltbotEnv` in `src/types.ts` → if passed to container, add to `buildEnvVars()` in `src/gateway/env.ts` → update `.dev.vars.example`.

## Gotchas

- Container cold starts take 1-2 minutes; `keepAlive: true` avoids this
- WebSocket proxying doesn't work reliably in local `wrangler dev`; deploy for full functionality
- `sandbox.proc.status` may not update immediately — verify success by checking output/files instead
- R2 mount directory IS the R2 bucket; `rm -rf /data/moltbot/*` deletes backup data
- `start-openclaw.sh` must use LF line endings
- Dockerfile has a cache bust comment — bump it when changing `start-openclaw.sh`
