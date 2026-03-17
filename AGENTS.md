# AGENTS.md

See `CLAUDE.md` for project overview, architecture, and conventions.

## Cursor Cloud specific instructions

### Quick reference

| Action | Command |
|---|---|
| Install deps | `pnpm install` |
| Lint | `pnpm lint` (Biome) |
| Test | `pnpm test` (requires turso dev running) |
| Dev dashboard | `pnpm dev:dashboard` |
| Dev status-page | `pnpm dev:status-page` |
| Dev web | `pnpm dev:web` |
| DB migrate + seed | `pnpm dx` (requires turso dev running) |

### Required system tools (pre-installed in snapshot)

- **Node.js** >= 20 (via nvm)
- **pnpm** 10.26.0 (declared as `packageManager`)
- **Bun** — used by `apps/server`, `apps/workflows`, and `packages/db` scripts
- **Turso CLI** — provides the local libSQL database via `turso dev`

### Starting the local database

The database must be running before tests, migrations, or any dev app:

```sh
turso dev --db-file openstatus-dev.db
```

This starts libSQL on port 8080. Note that `pnpm dev:dashboard` (and similar dev commands) automatically start `turso dev` via the `@openstatus/db` `dev` script, so you only need to run it separately for `pnpm dx` or `pnpm test`.

### Running tests

Tests require the turso dev database to be running. Start it in a separate process, then:

```sh
pnpm dx        # env + migrate + seed (only needed on first run or schema changes)
pnpm test      # runs all test suites across the monorepo
```

Server tests (`apps/server`) use Bun's test runner.

### Authentication in dev mode

The dashboard uses NextAuth with a magic link provider in development. When `NODE_ENV=development` or `SELF_HOST="true"`, the Resend provider is available and prints the magic link URL to the console (stdout of the Next.js process) instead of sending an email. Look for `>>> Magic Link:` in the terminal output. The seeded user email is `ping@openstatus.dev`.

### Environment files

`pnpm dx` copies `.env.example` → `.env` for each package that has an `env` script. For the dashboard, also ensure `SELF_HOST="true"` is set in `apps/dashboard/.env` to enable magic link login.

### Gotchas

- `pnpm install` may warn about ignored build scripts. The `pnpm.onlyBuiltDependencies` allowlist in root `package.json` controls which postinstall scripts run. Critical binaries (biome, esbuild, tailwindcss/oxide) work without their postinstall on this platform.
- Port 8080 must be free before starting `turso dev` or any dev app that includes the db filter.
- The `pnpm dev:dashboard` command starts both the Next.js dev server (port 3000) and turso dev (port 8080) via turborepo.
