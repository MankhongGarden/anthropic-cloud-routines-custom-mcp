# Replacing Local Cron with Anthropic Cloud Routines
## A Custom MCP Server on Next.js + Vercel with OAuth 2.1 + PKCE So Claude Can Fetch My Vendor Data Autonomously

**TL;DR**
- Replaced 12 Windows Task Scheduler jobs with 12 Anthropic Cloud Routines that fire at the same cron times — except they run on Anthropic's infrastructure, not my laptop.
- Built a custom MCP server at `/api/mcp/ops` on the same Next.js + Vercel project I was already shipping. 10 read-only tools that wrap Supabase / Stripe / Sentry / Vercel / Resend, plus one whitelisted `send_summary_email`.
- OAuth 2.1 + PKCE with stateless HS256 JWT — no auth-server DB. Three token types: 60s code · 1h access · 90d refresh. ~250 lines of TypeScript total across the OAuth files.
- Routines run inside the Anthropic Max plan's daily routine quota — verified zero extra-usage charge at console.anthropic.com after a full week of 12 routines firing daily.
- Three non-obvious findings the public guides don't mention:
  1. **Custom connectors don't auto-flow into routines** — each routine needs explicit `mcp_connections`. The default Gmail connector that auto-attaches has no `send_email` (only `create_draft`), so without a custom MCP your routine can't actually deliver mail.
  2. **`vercel env add` from CLI v53+ stores values as Sensitive type** — which AI routines cannot read. You have to use the REST API to store them as `encrypted` instead.
  3. **The `connector_uuid` you need to attach a connector to a routine isn't exposed anywhere in the UI or API** — you read it from `claude.ai`'s IndexedDB react-query cache.

---

## Why this turned into a thing

I had 12 cron jobs on my laptop. Daily security advisor scans for Supabase. Weekly Stripe reconciliation totals. Sentry error triage email at 09:00. Vercel deployment health pings. Resend deliverability summaries. All scheduled via Windows Task Scheduler, all running `claude -p --dangerously-skip-permissions <prompt>` headless and emailing the result.

Three problems:

1. **They didn't run when the laptop was off.** Closing the lid Friday night meant no Monday morning report.
2. **The headless `claude -p` invocation was eating into my Max plan's interactive token budget.** A morning of routine runs would already have me hitting rate-limit warnings before I sat down to actually work.
3. **There was no observability.** A failed run logged to a `.log` file on my disk that I never checked. I once discovered the Supabase advisor cron had been silently failing for a week because the PAT expired.

I knew Anthropic had been rolling out a `/schedule` feature on `claude.ai` ("CCR routines") — cron jobs that run on their infrastructure. I tried it. Three blockers showed up immediately:

- **It only auto-attaches Gmail + Google Drive connectors.** Custom connectors I had added at `claude.ai/customize/connectors` did not flow into routines automatically.
- **The Gmail connector only has `create_draft`, not `send_email`.** A routine that "emails me the report" was actually creating a draft I had to manually open and send. Defeats the point.
- **A routine that needs to read live vendor data** — Supabase advisors, Stripe payments, Sentry issues — has no way to do that with stock connectors.

So I built a custom MCP server. Here's the pattern.

---

## The architecture in one picture

```
┌─────────────────────────────┐         ┌─────────────────────────────┐
│ Anthropic CCR routine       │   HTTPS │  /api/mcp/ops on Vercel     │
│ (claude.ai/code/routines)   │ ───────►│  ── OAuth 2.1 + PKCE auth   │
└─────────────────────────────┘         │  ── 10 read-only tools      │
                                        │  ── send_summary_email      │
                                        └──────────┬──────────────────┘
                                                   │
                              ┌────────────────────┼──────────────────┐
                              │                    │                  │
                              ▼                    ▼                  ▼
                         [Supabase]            [Stripe]          [Sentry]
                         [Vercel]              [Resend]
```

The OAuth dance happens once per connector setup. After that, `claude.ai` stores access + refresh tokens and routines just use them.

---

## Phase 1 — pick the tools

Resist the temptation to add write tools. The OAuth secret will leak eventually (chat history echo · debug logs · someone shoulder-surfs your screen) and read-only contains the blast radius. The only "write" tool in my server is `send_summary_email`, and even that is restricted to a hardcoded recipient.

What I shipped:

| Tool | Wraps | Why useful |
|---|---|---|
| `supabase_get_advisors(type)` | Supabase Management API `/v1/projects/{ref}/advisors/{type}` | Daily security + performance scan |
| `supabase_admin_query(sql)` | Supabase Management API `/database/query` | Arbitrary `SELECT` (validated · `SELECT/WITH` only · no semicolons · 5000 char cap) |
| `stripe_list_payment_intents(filter)` | Stripe SDK `paymentIntents.list` | Per-payment investigation |
| `stripe_payment_intents_summary(window)` | Paginated count + sum | Weekly reconciliation total |
| `sentry_search_issues(filter)` | Sentry API `/issues/` | Daily error triage |
| `vercel_recent_deployments(limit)` | Vercel API `/v6/deployments` | Cron health + deploy state |
| `resend_list_emails(limit)` | Resend API `/emails` | Weekly deliverability trend |
| `resend_get_email(id)` | Resend API `/emails/{id}` | Single message status |
| `health_check()` | All-vendor ping | Setup verification + alerting |
| `send_summary_email(subject, body)` | Resend send, founder-whitelisted | The only writable tool |

`send_summary_email` is the trick that makes CCR routines actually useful. The recipient is baked as a constant in the server, not a parameter:

```ts
const FOUNDER_RECIPIENT = "founder@example.com";
const SUMMARY_FROM = "Project Ops <noreply@yourdomain.com>";
// hardcoded · ignore any `to` param even if accidentally exposed
```

A leaked secret can spam exactly one inbox. That's a containable blast.

---

## Phase 2 — the MCP route

Next.js App Router. One file at `src/app/api/mcp/ops/route.ts`. JSON-RPC 2.0 over plain POST. No SSE needed for routine workflows — they're request/response.

Methods to handle:

- `initialize` → return server info + `capabilities: { tools: {} }`
- `notifications/initialized` → return 204
- `tools/list` → return your tool definitions
- `tools/call` → dispatch to the per-tool handler, return `{content: [{type:'text', text: JSON.stringify(result)}], isError: false}`
- `GET` on the same URL → return server name + tool list (browser smoke test)

### The auth gate (two paths in one function)

```ts
export async function authenticate(req: Request) {
  const authHeader = req.headers.get("authorization");
  if (!authHeader?.startsWith("Bearer ")) {
    return unauthorized();
  }
  const token = authHeader.slice("Bearer ".length);

  // Path 1: JWT access token (what claude.ai sends after OAuth)
  try {
    const { payload } = await jwtVerify(token, OAUTH_KEY, { issuer: ISSUER, audience: RESOURCE });
    return { ok: true, subject: payload.sub };
  } catch {}

  // Path 2: Static Bearer (legacy · curl/PowerShell smoke tests)
  if (token === process.env.MCP_OPS_SECRET) {
    return { ok: true, subject: "static" };
  }

  return unauthorized();
}

function unauthorized() {
  return new Response("unauthorized", {
    status: 401,
    headers: {
      "WWW-Authenticate": `Bearer resource_metadata="${ISSUER}/.well-known/oauth-protected-resource"`,
    },
  });
}
```

The `WWW-Authenticate` header is what tells `claude.ai` where to discover the OAuth authorization server (RFC 9728). Without it, the custom connector setup fails with a generic "auth required" error.

### Read-only enforcement on `supabase_admin_query`

```ts
const READONLY_SQL = /^\s*(SELECT|WITH)\s/i;
if (!READONLY_SQL.test(sql)) throw new Error("Only SELECT or WITH allowed");
if (sql.includes(";")) throw new Error("No semicolons (no multi-statement)");
if (sql.length > 5000) throw new Error("Query too long");
```

Multi-statement guard via `;` rejection is the cheapest mitigation against "AI thinks it's clever and tries `SELECT 1; DROP TABLE users;`". Belt-and-suspenders on top of the SELECT prefix check.

---

## Phase 3 — OAuth 2.1 + PKCE

Eight files. Stateless JWT design — no DB. One signing secret. ~250 lines of TypeScript.

```
src/lib/oauth.ts                                          — JWT sign/verify · PKCE check · allowlists
src/app/.well-known/oauth-authorization-server/route.ts  — RFC 8414 metadata
src/app/.well-known/oauth-protected-resource/route.ts    — RFC 9728 resource metadata
src/app/api/oauth/register/route.ts                       — RFC 7591 dynamic client registration
src/app/authorize/page.tsx                                — consent page (server component)
src/app/authorize/ApproveButtons.tsx                      — Approve/Deny
src/app/api/oauth/approve/route.ts                        — sign code · 302 to claude.ai callback
src/app/api/oauth/token/route.ts                          — authorization_code + refresh_token grants
```

### Token design

Three JWT types, all HS256-signed with the same secret:

| Token | TTL | Embeds |
|---|---|---|
| Code | 60 seconds | `code_challenge` + `redirect_uri` + `scope` + `sub` |
| Access | 1 hour | `client_id` + `scope` + `sub` |
| Refresh | 90 days | `family` + `gen` (incremented each refresh) |

Stateless means I don't need a database for OAuth state. The trade-off is that revocation requires rotating the signing secret (which invalidates every issued token at once). For a solo-founder use case that's fine — emergency rotation is rare and acceptable.

Use `jose` for sign/verify (likely already a transitive dep if you use `@supabase/ssr`). One env var: `OAUTH_SIGNING_SECRET` (64 bytes of hex from `crypto.getRandomValues`).

### Founder-only auth gate on `/authorize`

```tsx
export default async function AuthorizePage({ searchParams }) {
  const supabase = await createClient();
  const { data: { user } } = await supabase.auth.getUser();
  if (!user) {
    redirect(`/login?next=${encodeURIComponent(currentUrl)}`);
  }
  if (user.email !== FOUNDER_EMAIL) {
    return <ErrorPanel message="This connector is not for general use." />;
  }
  // ... consent form ...
}
```

This piggybacks on your existing app login. Anyone who reaches `/authorize` must be logged in AND be the founder email. Even if someone discovers the URL, they can't proceed past the second check.

### `redirect_uri` allowlist (single highest-leverage defense)

```ts
const ALLOWED_REDIRECT_URIS = new Set([
  "https://claude.ai/api/mcp/auth_callback",
  "https://claude.com/api/mcp/auth_callback",
  "https://api.claude.ai/api/mcp/auth_callback",
]);
```

Even if every other check is wrong, an attacker can't redirect the authorization code to their own server. Hardcoded set, no regex matching, no wildcard.

---

## Phase 4 — env vars (the Sensitive-type trap)

This is where the public guides get it wrong.

If you run `vercel env add OAUTH_SIGNING_SECRET production` from CLI v53+, the value gets stored as type `Sensitive`. **AI routines cannot read Sensitive env vars** — they're only exposed at build time to specific build steps.

You have to use the REST API to store them as type `encrypted` instead:

```bash
curl -X POST \
  "https://api.vercel.com/v10/projects/$PROJECT_ID/env?upsert=true&teamId=$TEAM_ID" \
  -H "Authorization: Bearer $VERCEL_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "key": "OAUTH_SIGNING_SECRET",
    "value": "...",
    "type": "encrypted",
    "target": ["production", "preview", "development"]
  }'
```

The required env vars:

| Var | Purpose |
|---|---|
| `MCP_OPS_SECRET` | Legacy Bearer for smoke tests (64-byte hex) |
| `OAUTH_SIGNING_SECRET` | HS256 JWT signing (64-byte hex) |
| `SUPABASE_ACCESS_TOKEN` | Supabase Management API PAT |
| `VERCEL_TOKEN` | Vercel REST API (read your own deployments) |
| `SENTRY_AUTH_TOKEN` | Scopes `event:read + project:read + org:read` |
| `RESEND_API_KEY` | likely already set in your project |
| `STRIPE_SECRET_KEY` | likely already set |
| `NEXT_PUBLIC_SUPABASE_URL` + `SUPABASE_SERVICE_ROLE_KEY` | likely already set |

Trigger a redeploy after adding (use `POST /v13/deployments` with the `deploymentId` of the last `READY` production deployment).

---

## Phase 5 — register the connector

User-side only. OAuth requires consent, so this can't be automated.

1. Open `https://claude.ai/customize/connectors`
2. Click "Add custom connector"
3. **Name:** short alphanumeric (becomes the prefix in tool names — `mcp__YourName__tool`). Dashes allowed; dots and spaces are not.
4. **URL:** `https://yourdomain.com/api/mcp/ops`
5. Click Save → redirects to your `/authorize` page
6. (Log in if not already) → click Approve
7. 302 back to claude.ai → token exchange → connector connected

To attach this connector to a routine, you need its `connector_uuid` — and **this is not exposed in any UI or public API**. You read it from claude.ai's IndexedDB:

```js
// Run in the claude.ai DevTools console:
// 1. Get org UUID
fetch('/api/organizations', { credentials: 'include' })
  .then(r => r.json())
  .then(o => console.log('org', o?.[0]?.uuid));

// 2. Then DevTools → Application → IndexedDB → keyval-store → keyval
// Search the cached react-query data for your connector name.
// connector_uuid is in the surrounding JSON.
```

---

## Phase 6 — attach to routines

Once you have the UUID, create or update a routine with `mcp_connections`:

```json
{
  "mcp_connections": [
    {
      "connector_uuid": "<your-uuid>",
      "name": "YourName",
      "url": "https://yourdomain.com/api/mcp/ops"
    }
  ],
  "job_config": {
    "ccr": {
      "session_context": {
        "sources": [],
        "model": "claude-sonnet-4-6"
      },
      "events": [
        {
          "data": {
            "message": {
              "content": "Use mcp__YourName__supabase_get_advisors('security') to check the project. If any findings have level=ERROR, call mcp__YourName__send_summary_email with the details."
            }
          }
        }
      ]
    }
  }
}
```

**Set `sources: []` if your project's GitHub repo is not connected.** Otherwise the routine fails on manual `run` with `github_repo_access_denied`. Empty sources is fine — routines that get all their data from MCP tools never need a repo checkout.

---

## The thirteen gotchas I hit

In order of how much time each one cost me:

1. **Custom connectors don't auto-flow into routines.** Each routine needs explicit `mcp_connections`. Probe a routine with an introspection prompt to see what's actually attached — only `mcp__Gmail__*` (12 tools) will show without explicit MCP attach.

2. **Gmail connector has no `send_email` — only `create_draft`.** Routines that "send" via Gmail produce drafts you have to open and send manually. Always include your own `send_summary_email` tool.

3. **`vercel env add` CLI stores Sensitive type.** Sensitive env vars are not exposed to routine runtime. Use the REST API with `type: "encrypted"`.

4. **`sources: []` for non-GitHub-connected projects.** Even one-shot manual `run` triggers a GitHub auth check. Empty sources skips it.

5. **Tool prefix is `mcp__{name from mcp_connections}__{tool_name}`.** If the connector is named `Project-Ops`, your tools become `mcp__Project-Ops__health_check`. Test the exact prefix in your routine prompt by referencing it explicitly.

6. **OAuth 2.1 + PKCE is required.** Static Bearer auth doesn't appear as an option in the cloud custom connector UI (as of mid-2025+). Don't skip the OAuth setup expecting a simpler auth to be accepted.

7. **Supabase advisors path is path-style.** `/v1/projects/{ref}/advisors/{security|performance}` — NOT `?type=security`. Wrong path returns 404 with error text echoing your malformed URL.

8. **Sentry `statsPeriod` for issues endpoint only accepts `''`, `24h`, `14d`.** `1h` (which seems reasonable) returns 400 Invalid stats_period. Pick `24h` as your default.

9. **Sentry org slug ≠ name.** Likely has a random suffix like `your-org-1w`. Region might be EU (`de.sentry.io`) not US. Probe via Sentry's `find_organizations` first.

10. **Sentry token scope must include `event:read`.** Source-maps tokens (`project:read + project:releases + org:read`) return 403 on the issues endpoint. Create a new user auth token with the right scopes.

11. **React 19 form submitter quirk on the Approve page.** Putting two `<button name="decision" value="approve|deny">` in one form can result in `decision` being `null` at the server. Workaround: split into TWO separate `<form>` elements, each with a hidden `<input type="hidden" name="decision" value="...">` and one submit button.

12. **Cross-origin fetch into CORS wall.** If your Approve handler returns 302 to claude.ai, calling it via `fetch()` from the consent page hits "Failed to fetch" because fetch auto-follows the redirect into a cross-origin response it can't read. Use a native HTML form POST so the browser navigates at document level.

13. **`WebSearch` + `WebFetch` are available inside routines for free.** They're Anthropic-built tools, not MCP. Useful for "AI scouts the web then takes action" patterns without any extra connector.

---

## Hard-cap discipline for write-action routines

If you ever expand beyond `send_summary_email` to other write actions (sending an outreach email · charging a card · creating a database row), build it in three tiers so the cap isn't a prompt rule that AI can rationalize past:

```
┌──── Tier 1: Library (src/lib/<feature>/<action>.ts) ────────┐
│ Hard caps embedded as constants. Functions throw or         │
│ return {ok:false, reason:...} when the caller violates.     │
│ No prompt can override — caps are code.                     │
└─────────────────────────────────────────────────────────────┘
                          ▲
                          │ thin wrapper
┌──── Tier 2: MCP tool wrapper (api/mcp/ops/route.ts) ────────┐
│ Validates input shape + calls Tier 1. Adds no logic.        │
│ Returns Tier 1's structured result to caller.               │
└─────────────────────────────────────────────────────────────┘
                          ▲
                          │ JSON-RPC tools/call
┌──── Tier 3: Routine (claude.ai/code/routines) ──────────────┐
│ AI decides WHICH items to act on. Calls Tier 2.             │
│ Even if prompt is manipulated, Tier 1 caps still hold.      │
└─────────────────────────────────────────────────────────────┘
```

Prompt-level caps are advisory — AI can "decide" to send 5 when the prompt says 3 if the situation feels right. Library-level caps are walls — the function refuses to do the work, and AI cannot redeploy code from a routine.

For per-day caps, a single-row table keyed on `campaign_tag = daily_YYYY_MM_DD` works well as the running tally. Re-running the routine reads the row, sees N actions already done today, computes the remaining budget. Crash mid-loop = next run picks up where it left off without duplicating.

Useful guard layers, compose as needed:

- Per-call cap (max N items per single MCP invocation)
- Per-day cap (sum across all invocations within a UTC day)
- Per-domain/per-subject throttle (1 email/domain · 1 charge/customer)
- Time-window gate (e.g. send only 09-12 ICT)
- Dedupe layers (against authoritative state · against opt-out registry · against active-user table)

The MCP tool advertises these limits in its `description` field so the routine's planning prompt knows what to expect — but the prompt can lie. The library is the enforcement.

---

## Cost reality

CCR routines run inside the daily routine quota of the Anthropic Max plan I was already paying for. After running 12 routines daily for a week, my `console.anthropic.com` extra-usage page showed **zero additional charge**.

The Max 5x plan in effect during that test included 15 daily routine runs in the base quota. I was using 12 (one routine fires 2-3 times per day depending on cadence), so I stayed under the cap.

I'd avoid quoting a specific monthly cost projection — the quota and pricing details change. The practical takeaway: if you're already on Max plan and stay under the daily routine cap, the autonomous-cron pattern adds ฿0 to your monthly bill compared to running headless `claude -p` from local Task Scheduler. Verify your own plan's cap at `console.anthropic.com/settings/limits` before committing to a routine count.

---

## What else I'd do differently

- **Bake the OAuth files into a private template repo from day one.** Reusing across projects is the obvious next step (Phase 7 in the skill that codifies this), but I shipped one project's-worth of files inline first. Pulling them out into a reusable shape took an extra hour.
- **Skip the legacy static Bearer path if you only ever use claude.ai's flow.** I kept it for curl/PowerShell smoke tests, which is convenient but adds an attack surface (an env var that, if leaked, bypasses OAuth entirely). If your project is solo and you trust your dev environment, this is fine. For teams, rip it out.
- **Set up a `health_check` routine on Day 1.** I waited two weeks before adding one and missed two days of silent Sentry-token expiry. A daily `mcp__YourName__health_check` that emails on any vendor failure costs almost nothing and catches dependency rot fast.

---

## Anti-patterns to skip

- **Putting write tools behind `description`-only caps.** The description is a hint to AI. AI does not enforce it. Cap-as-library is the only durable defense.
- **Using `vercel env add` for routine-readable secrets.** Sensitive type is invisible to runtime. Use REST API + `type: "encrypted"`.
- **Embedding `recipient` as a `send_email` parameter.** A leaked secret + a parameterized recipient is a spam cannon. Hardcode the recipient.
- **Allowlist wildcards for `redirect_uri`.** Hardcoded set, no regex. The cost of explicitly listing four URIs is zero. The cost of a regex bug is unbounded.
- **Skipping the IndexedDB step to grab `connector_uuid` and hoping the API will expose it later.** It might. It doesn't right now. Get the UUID once, save it in your project's env-or-config, move on.

---

## Lessons

1. **OAuth 2.1 + PKCE with stateless HS256 JWT is plenty for solo-founder use.** No DB. ~250 lines. Revocation = rotate signing secret. Production-grade for what most personal projects need.
2. **The hardest part isn't the OAuth — it's discovery.** Where's the `connector_uuid`? Why does my routine see Gmail tools but not mine? Why does `vercel env add` silently store unreadable values? The cron-replacement pattern works once you've answered each.
3. **Read-only blast-radius defense beats every other security choice you can make in this kind of project.** Skip write tools until you have a hard-cap discipline ready.
4. **The cost story matters.** "Routines that run when laptop is off" is the headline. "And add nothing to my monthly bill if I'm already on Max plan and stay under the daily cap" is what makes the decision easy.
5. **An identity-block at the top of the routine prompt — "you have access to mcp__YourName__* tools, here is what each does, here are the caps" — pays off as much as it does in CLAUDE.md.** Routines without that context guess what's available; routines with it call the right tool the first time.

---

## Disclaimer

- This pattern is verified on Next.js 16 + Vercel + Anthropic Max plan as of May 2026. Anthropic's CCR feature is actively evolving; specific endpoint paths and auth requirements may change.
- Quoted env-var behavior (`vercel env add` storing as Sensitive type) is based on Vercel CLI v53. Verify on your CLI version before depending on it.
- Read-only `supabase_admin_query` is **not a substitute for proper RLS**. If your service-role key leaks, the SELECT-only validator becomes the only thing standing between an attacker and your data. Treat it as defense-in-depth, not as primary control.

---

*If you've built something similar and hit a different set of gotchas — especially around custom connector discovery or Sensitive-vs-encrypted env var behavior on other deployment platforms — I'd love to hear about it.*
