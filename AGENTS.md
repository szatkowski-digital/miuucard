# AGENTS.md — MiuuCard Monorepo

## 1. Project Overview & Architecture
This is a TypeScript monorepo (`npm workspaces` + `turbo`) containing two Expo mobile applications (initialized with `blank-typescript`) and shared domain/infrastructure packages:

- `apps/client` — Consumer mobile app **MiuuCard** (`package` / `bundleIdentifier`: `app.miuucard.client`).
- `apps/business` — Merchant/partner mobile app **MiuuCard Business** (`package` / `bundleIdentifier`: `app.miuucard.business`).
- `packages/ui` — Shared React Native UI components, theme tokens, typography, and design primitives.
- `packages/supabase` — Shared Supabase client, generated `Database` types, `expo-secure-store` session adapter, auth hooks, and query layer.
- `packages/r2` — Shared Cloudflare R2 helpers (uploads and downloads strictly via short-lived backend presigned URLs).
- `packages/shared` — Shared `zod` validation schemas, domain types, constants, and utility functions.

### Code Organization Rules
- Never duplicate business logic, Supabase types, validation schemas, or API queries between `apps/client` and `apps/business` — extract them into the appropriate package under `packages/*`.
- Keep `apps/*` focused strictly on screens, navigation, app-specific state, and native configuration (`app.json`, permissions, icons).

---

## 2. Tech Stack & Conventions
- **Language:** TypeScript (`strict: true`; avoid `any` unless explicitly justified).
- **Validation:** Use `zod` for all external inputs, API payloads, and environment variable validation.
- **Data & State:** Use `@tanstack/react-query` for server state and caching; use `zustand` strictly for lightweight local UI state.
- **Auth & Session Storage:** Use `@supabase/supabase-js` backed by `expo-secure-store` for session persistence (never store auth tokens in unencrypted `AsyncStorage`).
- **Environment Compatibility:** Developers work on Windows (PowerShell) and automated runners execute on Linux. Keep scripts and CLI commands cross-platform.

---

## 3. Security Rules (Critical)
This repository is continuously audited by automated security tests covering role isolation (`unauthenticated`, `client`, `business`), Supabase Row Level Security (RLS), and Cloudflare R2 access controls. Strictly enforce the following:

1. **Zero Secrets in Mobile Clients:**
   - Only public identifiers (`EXPO_PUBLIC_SUPABASE_URL`, `EXPO_PUBLIC_SUPABASE_ANON_KEY`, public API base URLs) may exist in client code or `EXPO_PUBLIC_*` variables.
   - Never commit or reference `SUPABASE_SERVICE_ROLE_KEY`, Cloudflare R2 secret keys (`R2_SECRET_ACCESS_KEY`), or admin tokens inside mobile apps or shared client packages.

2. **Authorization & Supabase RLS:**
   - Assume every Supabase table, view, RPC, and API endpoint can be queried directly from outside the mobile app using the public `anon` key.
   - Role boundaries (`unauthenticated` vs `client` vs `business`) must be enforced at the database layer (RLS policies / `security definer` RPCs validating `auth.uid()`) and backend API layer — never solely by hiding screens in the UI.
   - A `client` user must never access `business` mutations or data, and User A must never read or modify resources belonging to User B (strict IDOR/BOLA prevention).

3. **Cloudflare R2 Storage:**
   - Private assets in R2 must never be exposed via a public bucket without authorization checks.
   - Uploading and reading private files must happen exclusively via short-lived **presigned URLs** minted on the backend after verifying the caller's ownership or role permissions for that specific resource.
   - Object keys in R2 must enforce tenant/user path isolation and validate file MIME type and size prior to generating a presigned URL.

4. **Environment Separation (`dev` / `test` / `prod`):**
   - Never hardcode URLs, project IDs, or bucket names in source code.
   - All environment configuration must load via environment variables (`.env.local` / `.env.test` ignored by Git) so automated security runs can target an isolated test Supabase project and test R2 bucket.

---

## 4. Workspace Commands
- Start client app: `npm run start --prefix apps/client`
- Start business app: `npm run start --prefix apps/business`
- Typecheck all workspaces: `npx turbo run typecheck`
- Lint all workspaces: `npx turbo run lint`

---

## 5. Agent Working Guidelines
- Before installing a new dependency, check whether it already exists in the root `package.json` or another workspace package.
- Whenever modifying auth flows, Supabase queries, user roles, or R2 file handling:
  1. Ensure no sensitive columns or cross-user records can leak in responses.
  2. Ensure unauthorized requests fail explicitly with `401` or `403`.
  3. Keep shared types synchronized across `packages/supabase`, `packages/shared`, `apps/client`, and `apps/business`.
- Do not change app identifiers (`app.miuucard.client`, `app.miuucard.business`) unless explicitly instructed.