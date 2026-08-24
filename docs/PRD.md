# PRD — Web App Starter Kit

**Status:** Draft v0.2
**Date:** 2026-08-24
**Owner:** Hayden Crain

---

## 1. Problem

Every new web app project restarts the same three-day slog: wire auth, get sessions working correctly across SSR, configure a database with sane authorization, and get a deploy pipeline that isn't a Friday-afternoon surprise. That work is undifferentiated, easy to get subtly wrong (especially auth), and gets copy-pasted forward with its bugs intact.

## 2. Goal

A cloneable repository that gives you a **deployed, authenticated, database-backed React web app in under 15 minutes**, with auth configured correctly rather than merely present — and which then gets out of the way.

### Non-goals

- Not a SaaS-in-a-box. No billing, no orgs/teams, no admin dashboard in v1.
- Not a component library or design system. Consumes Cloudflare's Kumo rather than growing its own.
- Not a CMS, not a mobile app, not a monorepo template for multiple apps.
- Not multi-cloud. Cloudflare is the deploy target; portability is a nice-to-have, not a constraint.

## 3. Success criteria

The starter is done when a competent developer, starting from `git clone`, can:

| # | Criterion | Measure |
|---|---|---|
| S1 | Run locally against a real database | `pnpm install && pnpm db:start && pnpm dev` — working app, no manual config beyond copying `.env.example` |
| S2 | Sign up, verify email, log in, reset password, log out | All flows work locally against the Supabase local stack, no cloud account required |
| S3 | Deploy to production | One documented command + a documented list of secrets to set. Target < 15 min from clone to live URL |
| S4 | Understand what to delete | Every "example" file is marked, and a `docs/CUSTOMIZING.md` says what to strip |
| S5 | Trust the auth | Session handling, token refresh, MFA, and route protection are covered by tests and documented threat-model notes |
| S6 | Verify the security property | A test asserts no Supabase token is reachable from `document.cookie` or the client bundle, and lint fails on a browser-client import |

If S5 fails, the project has no reason to exist — the whole value proposition is "auth you don't have to second-guess".

## 4. Stack decisions

Versions verified on npm 2026-08-24.

| Layer | Choice | Version | Why |
|---|---|---|---|
| Framework | React Router v8 (framework mode) | `8.3.0` | SSR with loaders/actions, first-party Cloudflare adapter, small runtime. Stable and boring. |
| Build | Vite | `8.x` | Native to the RR v8 dev/build pipeline |
| Runtime/host | Cloudflare Workers | `wrangler 4.x` | Target platform, per requirement |
| Database | Supabase Postgres | — | Target platform, per requirement |
| Auth | Supabase Auth via `@supabase/ssr` | `0.12.x` | Cookie-based sessions that work correctly in SSR; same identity system as the DB |
| Data access | `@supabase/supabase-js` + Postgres RLS | `2.112.x` | Authorization lives in the database, so it can't be bypassed by a forgotten check in a loader |
| Migrations | Supabase CLI | — | SQL-first, versioned in-repo, applies to local and hosted identically |
| UI kit | `@cloudflare/kumo` (Base UI under the hood) | `2.12.0` | Accessible components with keyboard/focus/ARIA already handled. Same house as the deploy target. |
| Styling | Kumo semantic tokens; Tailwind v4 only where Kumo doesn't reach | `4.1.x` | Minimise hand-written utility classes — see 4.2 |

### 4.1 Data access

Every table is RLS-enabled with deny-by-default. Server-side reads use a client bound to the **request's user JWT**, not the service role key. The service role key is used only in narrowly-scoped, explicitly-named server modules (e.g. webhook handlers), never in a route loader.

### 4.2 Styling posture: Kumo first, Tailwind last

Kumo ships prebuilt styles (`@cloudflare/kumo/styles`) and a semantic token system, so the starter should lean on **component props and semantic tokens**, not utility soup:

- Surfaces use the hierarchy `bg-kumo-base` → `bg-kumo-elevated` → `bg-kumo-recessed`, never raw palette classes like `bg-blue-500`.
- Dark mode is automatic via `light-dark()` in CSS custom properties, driven by `data-mode="light|dark"` on a parent element. **No `dark:` variants anywhere.**
- Tailwind is permitted for layout only (flex/grid/spacing) where no Kumo component exists. Colour, typography, and elevation always come from tokens.
- Adopt Kumo's own lint stance: raw Tailwind colour classes should fail lint in this repo too.

Peer dependencies are à la carte — `@phosphor-icons/react` and `zod` are needed broadly; `echarts` only if chart components get used, so it stays out of v1.

### 4.3 The load-bearing decision: RLS as the authorization layer

**Why:** it makes the dangerous thing the awkward thing. A developer who forgets an ownership check gets an empty result set instead of a data leak.

**Cost:** policies are SQL, need their own tests, and can be hard to debug. Mitigation is a policy test suite (see 6.4) and a `docs/RLS.md` explaining the patterns used.

## 5. Architecture

```
Browser  ── holds NO Supabase token. Ever.
  │  httpOnly + Secure + SameSite cookie, opaque to page JS
  ▼
Cloudflare Worker  ──  React Router v8 SSR   ← the only Supabase caller
  │   loaders/actions build a per-request Supabase client
  │   from the request cookies; tokens never cross to the client bundle
  ▼
Supabase (Postgres + GoTrue Auth + Storage)
      RLS policies enforce authorization per row
```

Key structural rules:

- **The browser never gets a Supabase token.** No `createBrowserClient`, anywhere. This is enforced by lint (see §6.1a), not by memory.
- **One Supabase client per request.** Created in a `getServerClient(request)` helper that reads/writes cookies via `@supabase/ssr`. Never a module-level singleton — that's a cross-request session-leak bug waiting to happen in a Worker.
- **Auth check in one place.** A `requireUser(request)` helper used by protected loaders/actions; it redirects to `/login?next=…` on failure. Route protection is not spread across components.
- **Env access is typed.** Cloudflare bindings and secrets are surfaced through a single validated config module that fails loudly at startup rather than producing `undefined` at runtime.

## 6. Scope — v1

### 6.1 Authentication (the core deliverable)

- Email + password sign-up with email verification
- OAuth: GitHub and Google, both behind config flags so an unconfigured provider hides its button rather than erroring
- Magic link / OTP sign-in
- Password reset flow (request → emailed link → set new password)
- Session refresh handled transparently server-side; rotated tokens written back to cookies on the same response
- **MFA / TOTP**: enrolment with QR code, verification at sign-in, recovery codes issued once and stored hashed, and a "disable MFA" flow that re-authenticates first
- Logout, including server-side session invalidation
- Protected route pattern + a public route pattern, both demonstrated
- Post-auth redirect that preserves the originally requested URL and rejects open-redirect targets

**Explicitly configured correctly, not just present:** httpOnly + Secure + SameSite cookies (achievable because of §6.1a), PKCE flow for OAuth, redirect URL allowlist documented for both local and prod, and email templates that point at the right domain in each environment.

> **Why httpOnly is possible here.** `@supabase/ssr@0.12.4` defaults to `httpOnly: false` (verified in `dist/main/utils/constants.js`) because a browser-side `supabase-js` client must be able to read the token. The starter never creates one — see §6.1a — so it sets `httpOnly: true` explicitly.

### 6.1a Session architecture — DECIDED: server-only Supabase (BFF)

**Decision:** the browser never holds a Supabase token. All database and auth calls originate in the Worker. Cookies are `httpOnly`, so page JavaScript — including any injected by an XSS — cannot read them.

This removes the credential-exfiltration class outright. An XSS on this app can still act as the user *while the user is on the page*, but it cannot steal a durable credential and replay it later from elsewhere. That is the difference between an incident and a breach.

Alternatives considered and rejected for v1: an opaque session ID backed by a server-side store (**B**), replacing Supabase Auth with Better Auth (**C**), and an external IdP bridged through Supabase's third-party JWT support (**D**). B is the natural next step and the design must not foreclose it; C breaks `auth.uid()` and therefore the RLS model; D adds vendor cost without solving the browser-token problem on its own.

#### What this obligates

| Obligation | Detail |
|---|---|
| No browser client | `createBrowserClient` is never called. Enforced with a lint rule banning the import outside `app/server/**`, so a future contributor gets a build failure rather than a silent security regression |
| Cookies set explicitly | `httpOnly: true`, `secure: true`, `sameSite: "lax"`, host-only, with the `__Host-` prefix where the path allows |
| Storage uploads proxy through the Worker | Avatar upload (§6.2) posts to an action, which streams to Supabase Storage using the request-scoped client. Size and MIME limits enforced server-side |
| Mutations are Origin-checked | `Origin` / `Sec-Fetch-Site` validated on every non-GET. SameSite=Lax is not CSRF protection on its own |
| Strict CSP with nonces | XSS is the root cause; httpOnly limits blast radius but the CSP is what prevents the injection. Nonce generated per request in the Worker |
| Keep B one module away | All session read/write goes through a single `app/server/session.ts`. Swapping the cookie payload for an opaque ID backed by KV or a Durable Object must not touch any route |

#### The cost, stated plainly

**Realtime is the casualty.** Supabase Realtime subscribes directly from the browser and needs a token to do it. Under this design it does not work out of the box. Options if a consuming app needs it, documented in `docs/REALTIME.md` rather than solved in v1:

- Proxy the Realtime socket through a Worker (Durable Object fan-out) — most work, keeps the security property
- Mint a separate short-lived, narrowly-scoped token for Realtime only — weakens the property in a bounded way
- Accept a browser client *only* for Realtime, and treat it as an explicit, documented exception

Direct-from-browser Storage uploads are also gone, but proxying those is cheap and already covered above.

#### Hardening shipped alongside

- Refresh-token rotation with reuse detection — confirm Supabase's is enabled, and assert it in a test
- Short access-token TTL
- Rate limiting + bot management on auth routes at the Cloudflare edge
- Redirect allowlist on `?next=` (open-redirect)

### 6.2 Accounts

- `profiles` table keyed to `auth.users.id`, populated by a trigger on user creation
- Settings page: display name, avatar upload (proxied through a Worker action to Supabase Storage, RLS'd), email change, password change, MFA enrolment/disable, recovery-code regeneration
- Account deletion that actually cascades

### 6.3 App shell

- Marketing/landing route, auth routes, and an authenticated app area with nav and user menu
- Loading and error boundaries wired at the route level
- Dark mode via Kumo's `data-mode` attribute, set by an inline pre-hydration script so there is no flash of wrong theme on SSR
- One example CRUD resource ("notes" or similar) demonstrating the full pattern: schema → RLS policy → loader → action → optimistic UI. Clearly labelled as deletable.

### 6.4 Testing

- Unit/integration via Vitest
- RLS policy tests: for each table, assert that user A cannot read or write user B's rows. Run against the local Supabase stack in CI.
- E2E auth flows via Playwright: sign-up, verify, login, reset, logout, protected-route redirect, MFA enrolment + challenge + recovery-code use
- **Security-property tests:** assert `document.cookie` exposes no `sb-*` token, assert the built client bundle contains no Supabase key or `createBrowserClient` call, assert a cross-origin POST is rejected
- CI on GitHub Actions: typecheck, lint, test, build

### 6.5 Developer experience

- `.env.example` with every variable documented inline
- `pnpm db:start` / `db:reset` / `db:migrate` / `db:types` scripts — generated TypeScript types from the schema, checked in
- `docs/`: SETUP, DEPLOY, RLS, CUSTOMIZING, and a short ARCHITECTURE
- `CLAUDE.md` so AI agents working in a cloned repo understand the conventions

### 6.6 Deploy

- `wrangler.toml` with dev/prod environments
- Documented secret list and the exact `wrangler secret put` commands
- GitHub Actions deploy workflow, manual-approval gated
- Documented Supabase project setup: which redirect URLs, which auth settings, which providers

## 7. Open questions

1. **Email delivery.** Supabase's built-in SMTP is rate-limited and unsuitable for production. Do we ship a Resend integration OOTB, or document the swap and leave it? *Leaning: document it, with a `docs/EMAIL.md` and templates ready to paste.*
2. **Cloudflare Access to Postgres.** Direct connections from Workers to Supabase need attention (connection pooling / Hyperdrive). Using supabase-js over HTTP sidesteps this entirely — confirm that holds for all v1 needs.
3. **Rate limiting on auth endpoints.** Supabase provides some. Is that sufficient, or do we add a Cloudflare rate-limiting rule for `/login` and `/signup`? *Leaning: add it — it's cheap and it's exactly the kind of thing people forget.*
4. **Does the avatar-upload proxy need a size guard beyond the Worker request limit?** Cloudflare caps request body size by plan; large uploads may need a signed-URL exception to the no-browser-token rule. *Leaning: cap avatars small enough that it never comes up in v1.*
5. **Do we ship anonymous/guest auth?** Useful for try-before-signup products, extra RLS complexity. *Leaning: no in v1, note as v2.*
6. **How do we keep this from rotting?** A starter kit with 18-month-old dependencies is worse than no starter kit. Renovate bot + a quarterly manual pass? Needs an owner.
7. **Template mechanism.** GitHub template repo, `degit`, or a `create-` CLI that prompts for project name and strips examples? *Leaning: template repo for v1, CLI if it gets traction.*

## 8. Risks

| Risk | Impact | Mitigation |
|---|---|---|
| Supabase SSR cookie handling is subtly wrong under Workers' streaming responses | High — silent session loss or leakage | E2E tests that assert cookie rotation; test against deployed preview, not just local |
| A contributor reintroduces a browser Supabase client, silently undoing §6.1a | High — reopens the exfiltration class | Lint rule bans the import outside `app/server/**`; S6 bundle test fails the build |
| Realtime turns out to be needed sooner than expected | Medium — the BFF choice becomes friction | `docs/REALTIME.md` documents three escape hatches up front, so it is a known trade rather than a surprise |
| RLS policies become the hard part and slow feature work | Medium | Ship 3–4 reusable policy patterns, documented, so the common cases are copy-paste |
| React Router v8 or Cloudflare adapter churn breaks the template | Medium | Pin exact versions, CI runs the full E2E suite on dependency PRs |
| Scope creep into a SaaS starter (orgs, billing) | Medium — dilutes the "boring and reliable" value | Non-goals in section 2 are the contract. Orgs and billing are separate v2 layers, not v1 additions |

## 9. Phasing

- **M1 — Skeleton.** RR v8 + Vite + Workers deploying a hello-world. Local Supabase running. CI green.
- **M2 — Auth.** All of 6.1 and 6.1a, including MFA, plus the E2E and security-property suites. This is the milestone that matters.
- **M3 — Accounts + shell.** 6.2 and 6.3, including the example CRUD resource and RLS tests.
- **M4 — Polish.** Docs, `.env.example`, deploy workflow, template-repo setup, `CLAUDE.md`.

M2 is the point at which the repo is already useful to you personally. M4 is the point at which it's useful to someone else.

## 10. v2 candidates

Option B session store (opaque ID in KV/Durable Object, instant revocation) · Realtime via Worker proxy · Organizations/teams with role-based RLS · Stripe billing · Anonymous auth · Passkeys/WebAuthn · Audit log · Feature flags · i18n · Cloudflare R2 for larger file storage · A `create-web-app` CLI
