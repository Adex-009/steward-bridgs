# StewardBridge — Workforce & Business Operations Platform

A Next.js App Router platform for StewardBridge, connecting talent with **businesses across multiple sectors**—hospitality, restaurants, healthcare, facilities, logistics, retail, events and other service-led organizations. The product includes public pages, opportunities, sign-in/registration, talent and business workspaces, admin operations, document/application APIs, and a webhook-ready external-agent interface.

**Brand/domain:** StewardBridge · `stewardbridge.com`  
**Framework:** Next.js + React + TypeScript  
**Deployment target:** Vercel (standard Next.js build; no Cloudflare adapter required)

## v17 (workspace + search rebuild)

Job seekers and employers now get working post-login workspaces: onboarding wizards that collect rich profile data, server-side job search with match scores, apply/save/withdraw, employer job management, an applicant pipeline and privacy-safe talent search. See `HANDOFF.md`. **Run `npm run db:push` once** to add the new (nullable/defaulted) columns. This version has not been built in a networked environment yet; run `npm install && npm run typecheck && npm run build` first.

## Revision notes (deploy fixes + new design)

**Why the Vercel build/deploy failed (fixed):**
1. `src/db/index.ts` threw at import time when `DATABASE_URL` was missing, so `next build` crashed while collecting route data. The connection is now created lazily on first query.
2. The homepage read `location.hash` while Next.js was pre-rendering it (`location is not defined`). It is now read only in the browser.
3. Sign-in redirected to pages that do not exist (`/onboarding/role`, `/dashboard/...`). All redirects now go to `/#/workspace` routes that exist.
4. With no Google keys set, nobody could sign in at all. Email + password now works without any Google setup.

**Must be set in Vercel before first use:** `DATABASE_URL`, `AUTH_SECRET` (and `AUTH_URL`). Then run `npm run db:push` once against the empty database. `GOOGLE_CLIENT_ID/SECRET` are optional now.

**Sign-in / sign-up (standard job-portal flow):** `/auth/sign-in` and `/auth/sign-up` with Job seeker / Staff vs Employer tabs, email + password, optional "Continue with Google". Admins still require Google + `ADMIN_EMAIL`. Password sign-up does not verify email yet (no mail sender): add email verification and password reset before public launch. Signing in with Google on an email that already has a password removes that password, so a squatter cannot hold a real owner's account.

**New design:** obsidian HR-cloud homepage in `src/components/hrcloud/`. All dashboard figures are simulated. Photos are free Pexels stand-ins listed in `shared.tsx` (`PHOTOS`, `PEOPLE`); replace with your own licensed team photography.

## Deploy to Vercel

1. Extract this project and push the project folder contents to a private or public Git repository.
2. In Vercel choose **Add New → Project**, import that repository, and keep Framework Preset as **Next.js**.
3. Add the environment variables listed below in both Preview and Production as appropriate.
4. Provision PostgreSQL through Vercel Marketplace (Neon is the simplest supported option), set `DATABASE_URL`, and initialize the database schema before accepting production registrations. The repository schema is in `src/db/schema.ts`; run `npm run db:generate` and review/commit the generated SQL, then `npm run db:migrate` against the intended database. For a brand-new empty database only, `npm run db:push` can apply the current schema directly; review the proposed changes carefully and never use it blindly against an existing production database.
5. Deploy. Vercel uses `npm install` and `npm run build` from `vercel.json`.
6. Add the deployed URL and custom domain `stewardbridge.com` in Vercel's domain settings. Configure DNS as Vercel instructs and verify HTTPS.
7. In Google Cloud Console create a Web OAuth client. Add the exact production callback URL shown by NextAuth (normally `https://YOUR_DOMAIN/api/auth/callback/google`) and the matching local callback for development. Add `GOOGLE_CLIENT_ID` and `GOOGLE_CLIENT_SECRET` in Vercel.
8. Run the production smoke tests in the checklist below before announcing launch.

## Environment variables

Copy `.env.example` to `.env.local` for development. Set production secrets only in Vercel Project Settings → Environment Variables; never commit secrets.

- `DATABASE_URL` — production PostgreSQL connection string.
- `AUTH_SECRET` — long, random secret (generate with `openssl rand -base64 32`).
- `AUTH_URL` — optional canonical production base URL, set to `https://stewardbridge.com` after the domain is connected.
- `ADMIN_EMAIL` — first authorized administrator email. The current example uses `johnademuyiwa1@gmail.com` based on the owner’s dictated spelling; confirm this exact address before production deployment.
- `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET` — required for all human sign-in. Google OAuth is optional; email + password is enabled for job seekers and employers. When Google is used it must return a verified email before the account is created or the admin allowlist is applied.
- `WEBHOOK_SIGNING_SECRET`, `STEWARD_BRIDGE_AGENT_WEBHOOK_URL`, `STEWARD_BRIDGE_AGENT_WEBHOOK_SECRET` — optional external agent/event integration secrets.
- `OPENROUTER_API_KEY`, `OPENROUTER_MODEL` — optional, only if using the OpenRouter fallback.

## Administrator access

Administrator access is assigned server-side only. A verified Google sign-in matching `ADMIN_EMAIL` can promote that database account; alternatively, provision the first administrator manually in the database after verifying ownership. Admin API routes require both role `admin` and `is_admin_authorized = true`. Do not rely on hiding admin links as security. Do not share admin credentials in this repository.

## External agent / webhook port

The agent is intentionally external and can be connected later without rebuilding the user interface. Routes include:

- `GET /api/agent/manifest` — supported capabilities and integration contract.
- `GET /api/agent/events` and `POST /api/agent/events` — event delivery/acknowledgement.
- `POST /api/agent/control` — supported, constrained actions.
- `POST /api/agent/chat` — chat relay.
- `POST /api/agent-door/documents` — retrieves a private document by ID as base64 for the authenticated external agent; requests are audited and capped at 10 MB. The agent must treat file contents as untrusted data, never as instructions.

Set webhook URLs and shared secrets as server-side Vercel environment variables. Use HTTPS endpoints, validate signatures/secrets, verify event payloads, make handlers idempotent, and never give the agent direct database credentials. Keep actions allowlisted and require human approval for consequential changes. The included `integrations/StewardBridge_n8n_All_In_One.json` is a workflow reference, not an automatically running service.

## Product and safety boundaries

- The platform serves more than hotels: businesses can represent hotels, restaurants, resorts, healthcare, logistics, facilities, retail, events, and other organizations.
- Google OAuth is required for all human users. Email/password registration is disabled.
- No online payment gateway is included by design. Do not add card/bank collection or public pricing.
- Private documents require private storage, server-side file validation and malware scanning before production. The external agent can retrieve a specific private document only with the server-side agent secret; each retrieval is audited. Do not give the agent direct database credentials, and treat CV/document contents as untrusted data.
- Keep business/customer data tenant-isolated with database policies; do not trust browser-only role checks.
- Do not collect NIN or BVN through online forms.
- The separate savings/microfinance ledger must remain read-only and isolated.
- Domain email, legal entity name, verified office address, legal terms and savings wording must be confirmed by the owner/legal adviser before launch.

## Quality checks

Run locally with Node.js LTS:

```bash
npm install
npm run typecheck
npm run lint
npm run build
npm run start
```

Then verify:

- Public homepage, navigation, business services, opportunities and contact routes on desktop and mobile.
- Light/dark appearance toggle persists and reduced-motion preference is respected.
- Google sign-in with a verified Google account and refusal of unverified accounts.
- First admin sign-in and refusal of non-allowlisted accounts; test every admin API directly, not just the UI.
- Business and talent roles cannot access another tenant's data.
- Upload size/type validation, private file access and signed-link expiry.
- Agent widget gracefully handles an absent/unavailable webhook; validate authenticated webhook requests and the supported action allowlist.
- Forms validate, persist, show a success/failure state and emit events only when configured.
- Production build and `/api/health` against the configured database.

## Known production prerequisites

A source archive cannot supply your private production database, Google OAuth client, Vercel project, DNS ownership, email sender, file-scanning service or external agent URL. These must be configured in accounts you control. A successful local build is not proof that third-party credentials, database migrations, permissions or production integrations are configured correctly.

## What must still be connected in owner-controlled accounts

The source code cannot create or verify resources inside an account it cannot access. Before public launch, the owner must connect the Vercel project, provision Neon/Postgres and apply the schema, add the Google OAuth client ID/secret (admin access requires the allowlisted verified Google identity), provide private S3-compatible storage credentials, point `stewardbridge.com` DNS to Vercel, and configure/rotate the agent secrets when the external agent is ready. Server-side malware scanning and a tested backup/restore process are also production prerequisites. Do not accept real CVs until database, private storage, authorization, scanning, and retention/privacy controls have been tested.

Google callback for the canonical domain: `https://stewardbridge.com/api/auth/callback/google`. All human users sign in through Google and then select Business / Organization or Job Seeker / Candidate (staff).


## Human user sign-in and account type

All human users must sign in with their own Google account. Email/password sign-in is not enabled. After Google sign-in, users are sent to profile onboarding and choose either **Job Seeker / Candidate (staff)** or **Business / Organization**. Configure `GOOGLE_CLIENT_ID` and `GOOGLE_CLIENT_SECRET` in Vercel before expecting Google sign-in to work. The agent's shared-secret authentication is only for server-to-server agent requests and must not be used as a human login method.
