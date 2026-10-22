# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

Teamdesk: TanStack Start on Cloudflare Workers + D1, Turborepo/Bun workspace (`apps/web`, `packages/db`, `packages/services`). No sign-in — the acting member is passed explicitly as `actorId`.

## Commands
- `bun run dev` — applies D1 migrations + seed data first, then starts the app. Don't run `wrangler dev` or a plain dev server directly — the DB would be empty/out of date.
- `bun test <path>` for a single file — tests use in-memory SQLite, no network, so no fixture/server setup needed beforehand.
- `bun run check` is lint+typecheck together; `bun run fix` is the only thing that should touch formatting.

## Layering — don't skip it
Route component → tRPC router (`apps/web/src/server/trpc/routers/`) → service (`@repo/services/<module>`) → query functions (`@repo/db/<module>`). A route calling a service or db function directly, or a router doing anything beyond input-schema + one service call, breaks the pattern every other module follows.

## `@repo/db` entry points are split on purpose
- `@repo/db` and `@repo/db/<table>` import drizzle — server only. Importing either from a route component or any browser-rendered file pulls drizzle into the client bundle.
- `@repo/db/shared` is the only one safe in React components (plain JSON types, no drizzle).
Reason: there's no build-time guard for this — it's a silent bundle-size/runtime issue, not a type error.

## Services
- Must throw `ServiceError` subclasses (`src/errors.ts`) rather than plain `Error` — `init.ts` maps these to tRPC/HTTP status codes (404/409/403/400). A plain throw becomes an opaque 500.
- Re-validate input with the exported zod schema inside the service, even though the tRPC router already validated it — the service has non-tRPC callers (scripts, tests) that skip that step.
- Take a `Database` param and never read `env`/bindings themselves — this is what lets tests swap in `bun:sqlite` for D1 unchanged.
- A new service needs a line added in `apps/web/src/server/trpc/context.ts`, or it's unreachable from the app.

## Schema changes
After editing `packages/db/src/schema.ts`, run `bun run generate` and commit the output in `migrations/`, including `migrations/meta/`. Migrations are applied by wrangler, not drizzle-kit — editing the schema alone does nothing, and skipping the commit of `meta/` makes the next `generate` try to recreate existing tables.

## Don't touch
- `apps/web/src/components/ui/**` and `use-mobile.ts` — shadcn-generated, excluded from lint. Running `ultracite fix`/Biome on these has previously corrupted `combobox.tsx` by duplicating a JSX attribute.
- `apps/web/src/components/bookings/**` — PascalCase filenames are a deliberate, scoped lint exception (legacy from the first prototype), not a convention to extend to new files.
