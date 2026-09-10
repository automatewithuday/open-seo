# About this fork

This is the Martechs fork of [every-app/open-seo](https://github.com/every-app/open-seo). It runs OpenSEO self-hosted on Cloudflare for [martechs.io](https://martechs.io) and lets an unattended automation call the OpenSEO MCP with a Cloudflare Access service token. It is not a general-purpose distribution; treat it as one operator's deployment with a small, documented delta over upstream.

Maintainer: Martechs (automatewithuday). Upstream tracks `every-app/open-seo` `main`.

## What differs from upstream

One functional change, plus operator docs and scripts.

### 1. A service token may call the MCP (`src/middleware/ensure-user/cloudflareAccess.ts`)

Upstream's `AUTH_MODE=cloudflare_access` resolves the user from the Access JWT's `email` claim and rejects requests without one. Cloudflare Access **service tokens** (Service Auth policies) carry no email; the JWT has only a `common_name` equal to the token's Client ID. Upstream therefore cannot be called by a machine.

The patch adds one branch: if the JWT has no email but its `common_name` equals `MCP_SERVICE_TOKEN_CLIENT_ID`, the request is resolved as a user inside the shared workspace under `MCP_SERVICE_TOKEN_EMAIL`. Exactly one token is honoured. Everything else in the auth path is unchanged, and the branch is a no-op unless both variables are set.

New optional environment variables (see `.env.selfhost.example`):

| Variable | Meaning |
|---|---|
| `MCP_SERVICE_TOKEN_CLIENT_ID` | Client ID of the one Access service token allowed in. The value sent as `CF-Access-Client-Id`, 32 hex characters plus `.access`. |
| `MCP_SERVICE_TOKEN_EMAIL` | A **dedicated** address for the service identity, such as `automation@yourdomain.com`. User emails are unique in OpenSEO, so reusing a real person's address blocks that person's own login. |

Tests: `src/middleware/ensure-user/cloudflareAccess.test.ts`. The change was offered upstream as [every-app/open-seo#317](https://github.com/every-app/open-seo/pull/317).

### 2. Operator scripts (repo root)

None of these contain secrets. They read the Client ID and Workers subdomain from the ignored `.env.selfhost`, and the secret from `~/.config/openseo/`, which only the operator's user can read.

| Script | Purpose |
|---|---|
| `store-secret.sh` | One-time: store the Access client secret in `~/.config/openseo/cf-access-client-secret` (mode 600). |
| `verify-mcp.sh` | Prove the service token reaches `/api/health` and `/mcp`. Prints HTTP status and redirect target only. |
| `connect-claude-mcp.sh` | Register the self-hosted MCP in the operator's own Claude Code with the service token headers. |
| `store-api-token.sh` | One-time: store a read-only Cloudflare API token for debugging Access apps and policies. |

## How it is deployed

- `pnpm deploy:selfhost`: Vite build in `selfhost` mode, typecheck, then Alchemy deploys the Worker, D1, KV, and R2 on the Cloudflare free plan (a card on file is still required for R2).
- `AUTH_MODE=cloudflare_access` with a **bring-your-own** Access application: `TEAM_DOMAIN` and `POLICY_AUD` are set, so the deploy script does not manage Access policies.
- The Access application carries two policies: an Allow policy for the operator's email, and a Service Auth policy for the one service token.
- Google OAuth (Search Console and GA4) uses an Internal-type client on the operator's Google Workspace so refresh tokens do not expire like External Testing apps. Callback paths are `/api/gsc/oauth/callback` and `/api/ga4/oauth/callback`; the GA4 integration also needs the Analytics Admin API enabled.
- No DataForSEO key by default. Search Console, GA4, and the built-in crawler only, until a deposit is added on purpose.

## Who calls it

A Trigger.dev project in the private martechs.io site repository runs headless Claude Code on a schedule (daily monitor, weekly action run, monthly audit). Each run writes an MCP config pointing at this Worker's `/mcp` with the `CF-Access-Client-Id` and `CF-Access-Client-Secret` headers, reads and writes the OpenSEO project context and research log, and turns any file changes into pull requests that a human merges. Nothing on the website changes without review.

## Lessons that cost time

- **Attach the policies to the application.** Reusable policies exist independently of applications. Two correct policies with `app_count=0` still yield a 302 to the login page for every request.
- **Leave Managed OAuth off** on the Access application. With it on, the `/mcp` protected-resource metadata advertises only browser OAuth and the service token is never consulted.
- **Service token secrets** are now 54 characters starting `cfast_` (format changed 2026-08-26). Client IDs are 39 characters ending `.access`. A 38-character Client ID means the suffix was lost in a paste.
- **The service email must be dedicated.** Setting it to a real user's address collided with that user's record and made projects disappear from the dashboard for them.
- Access answers a rejected token with **HTTP 302 and an HTML login page**, not 401. Any client that follows redirects will "see a webpage instead of data".

## Keeping up with upstream

```bash
git fetch upstream
git merge upstream/main
pnpm test
pnpm deploy:selfhost
```

`deploy:selfhost` builds and typechecks, then runs Alchemy. In a non-interactive shell Alchemy prints the plan and stops, asking for `--yes`; finish with `pnpm alchemy deploy --env-file .env.selfhost --stage selfhost --yes`. Confirm with `./verify-mcp.sh` afterwards.

Conflicts, if any, are confined to `src/middleware/ensure-user/cloudflareAccess.ts`, its test, `.env.selfhost.example`, `alchemy.run.ts`, and `src/env.d.ts`. `README.md` carries only a short pointer to this file so it merges cleanly.

## What is never committed

`.env.selfhost` (gitignored), the Access client secret, the Cloudflare API token, Google OAuth client secret, `BETTER_AUTH_SECRET`, and the Alchemy state directory. If you fork this fork, rotate everything in Cloudflare Zero Trust and Google Cloud first.
