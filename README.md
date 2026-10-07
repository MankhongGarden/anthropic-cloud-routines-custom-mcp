# Replacing Local Cron with Anthropic Cloud Routines
## A Custom MCP Server on Next.js + Vercel with OAuth 2.1 + PKCE So Claude Can Fetch My Vendor Data Autonomously

**TL;DR**
- Replaced 12 Windows Task Scheduler jobs with 12 Anthropic Cloud Routines that fire at the same cron times — except they run on Anthropic's infrastructure, not my laptop.
- Built a custom MCP server at `/api/mcp/ops` on the same Next.js + Vercel project I was already shipping. 10 read-only tools that wrap Supabase / Stripe / Sentry / Vercel / Resend, plus one whitelisted `send_summary_email`.
- OAuth 2.1 + PKCE with stateless HS256 JWT — no auth-server DB. Three token types: 60s code · 1h access · 90d refresh. ~250 lines of TypeScript total across the OAuth files.
- Routines run inside the Anthropic Max plan's daily routine quota — verified zero extra-usage charge at console.anthropic.com after a full week of 12 routines firing daily.
- Three non-obvious findings the public guides don't mention:
  1. **Which connectors a routine gets depends on how you create it** — set `mcp_connections` explicitly on every routine. The Gmail connector has no tool that sends mail, so without a custom MCP your routine can't actually deliver a report.
  2. ~~`vercel env add` stores values as Sensitive type, which AI routines cannot read.~~ **Corrected 2026-10:** the routine never reads your Vercel env — your MCP server does, at runtime. Sensitive only stops *you* reading the value back later. See Phase 4.
  3. **The `connector_uuid` you need to attach a connector to a routine isn't shown in the connectors UI** — the easiest place to get it is the response of a routine create call (see Phase 5).
- **Updated 2026-10:** the patterns held, but the failures that cost me most came *after* launch — routines deleted while docs still called them live, a pre-filled buffer hiding a dead producer, and a date-key bug where every run reported success while data went missing. See [What broke after launch](#what-broke-after-launch-updated-2026-10).

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

## Phase 4 — env vars

> **Correction (2026-10).** The first version of this section said routines can't read Sensitive env vars, so you must store everything as `encrypted` via the REST API. That was the wrong diagnosis. The routine never touches your Vercel env — it calls your MCP server, and the server reads its env at runtime. What Sensitive type actually blocks is reading the value back afterwards (Vercel API or `vercel env pull`).

The standard `vercel env add NAME production` is fine for secrets. Add a value with `--no-sensitive` only if something will need to read it back later. The REST API is an alternative if CLI auth is awkward on your machine:

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

To attach this connector to a routine, you need its `connector_uuid`, which the connectors page doesn't show.

**Updated 2026-10.** The original version of this section dug the UUID out of claude.ai's IndexedDB cache. By September that search returned nothing, so don't rely on it. The easier route: a routine created through the API *without* `mcp_connections` comes back with every connector on the account attached, UUIDs included (see gotcha 1). Copy the one you need from that response, then update the routine down to just that connector.

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

1. **Set `mcp_connections` explicitly on every routine.** In May, a routine set up without it saw only the Gmail tools. **Updated 2026-10:** a routine created through the API without `mcp_connections` gets *every* connector on the account attached, including ones it has no business touching. Either way, list exactly the connectors each routine needs. If you pass the list explicitly and the routine also needs Gmail, Gmail has to be in that list too.

2. **The Gmail connector can't send mail.** **Updated 2026-10:** it is now Google's own Gmail MCP. Its tools include `search_threads`, `get_message`, `get_thread`, `create_draft`, `forward` and label management, but nothing that sends a new message to your inbox. Keep your own `send_summary_email`. For a routine that only reads mail, narrow Gmail's entry in `mcp_connections` with `"permitted_tools": ["search_threads", "get_message", "get_thread"]`.

3. ~~`vercel env add` CLI stores Sensitive type, not exposed to routine runtime.~~ **Corrected 2026-10:** this was wrong. Sensitive vars are still read by your server at runtime. Sensitive only blocks reading the value back, so use `--no-sensitive` for the ones you'll need to read later. See Phase 4.

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

Two things I learned later:

- **The daily cap is per account, not per project.** Routines from every project on the same account share it. The first sign you've hit it is an email saying your routines are paused for the day, while some routines quietly skip.
- **Billing for automated usage has been in flux.** Anthropic announced a move of Agent SDK and `claude -p` usage into a separate credit pool, then paused it ([support article](https://support.claude.com/en/articles/15036540) still said paused as of 2026-10-07). Re-check that page and your limits before you quote a cost.

---

## What else I'd do differently

- **Bake the OAuth files into a private template repo from day one.** Reusing across projects is the obvious next step (Phase 7 in the skill that codifies this), but I shipped one project's-worth of files inline first. Pulling them out into a reusable shape took an extra hour.
- **Skip the legacy static Bearer path if you only ever use claude.ai's flow.** I kept it for curl/PowerShell smoke tests, which is convenient but adds an attack surface (an env var that, if leaked, bypasses OAuth entirely). If your project is solo and you trust your dev environment, this is fine. For teams, rip it out.
- **Set up a `health_check` routine on Day 1.** I waited two weeks before adding one and missed two days of silent Sentry-token expiry. A daily `mcp__YourName__health_check` that emails on any vendor failure costs almost nothing and catches dependency rot fast.

---

## Anti-patterns to skip

- **Putting write tools behind `description`-only caps.** The description is a hint to AI. AI does not enforce it. Cap-as-library is the only durable defense.
- **Embedding `recipient` as a `send_email` parameter.** A leaked secret + a parameterized recipient is a spam cannon. Hardcode the recipient.
- **Allowlist wildcards for `redirect_uri`.** Hardcoded set, no regex. The cost of explicitly listing four URIs is zero. The cost of a regex bug is unbounded.
- **Not recording `connector_uuid` once you have it.** Save it in your project's env or config so you don't have to go looking for it again.
- **Trusting a doc that says a routine is running.** List the live routines instead. See below.

---

## Lessons

1. **OAuth 2.1 + PKCE with stateless HS256 JWT is plenty for solo-founder use.** No DB. ~250 lines. Revocation = rotate signing secret. Production-grade for what most personal projects need.
2. **The hardest part isn't the OAuth — it's discovery.** Where's the `connector_uuid`? Why does my routine see Gmail tools but not mine? The cron-replacement pattern works once you've answered each. (After launch, the hardest part turned out to be noticing when a routine has quietly stopped doing its job. See the next section.)
3. **Read-only blast-radius defense beats every other security choice you can make in this kind of project.** Skip write tools until you have a hard-cap discipline ready.
4. **The cost story matters.** "Routines that run when laptop is off" is the headline. "And add nothing to my monthly bill if I'm already on Max plan and stay under the daily cap" is what makes the decision easy.
5. **An identity-block at the top of the routine prompt — "you have access to mcp__YourName__* tools, here is what each does, here are the caps" — pays off as much as it does in CLAUDE.md.** Routines without that context guess what's available; routines with it call the right tool the first time.

---

## What broke after launch (updated 2026-10)

The setup above held up. These are the problems that showed up in the months afterwards. None of them raised an error.

1. **Routines that are gone while your docs say they're live.** In August I listed a project's routines and got `[]`. Both had been deleted three weeks earlier to free up quota, but a handoff doc still described one as running, and nothing had produced content since. The live routine list is the source of truth, and memory or docs lag behind it. The opposite happens too: forgotten "ghost" routines keep firing and use up quota. When quota runs out unexpectedly, compare the live list with your notes. Treat deleting a producer routine as a system change, not a config tweak. With no scheduled producer, the work quietly becomes a human job, so put it wherever work gets planned, with the date the data runs out.

2. **A pre-filled buffer hides a dead producer.** For a daily horoscope app, a routine fills about 30 days of content ahead. A check of "is tomorrow filled?" kept saying *yes* for weeks after the routine stopped. The failure only showed on the day users missed content. Part of the app had already served 20 days of empty sections, because the page using the data skips missing rows without any error. Monitor **runway**: how many consecutive days ahead are filled. Set the alert threshold to roughly how long a human needs to refill it, plus one cycle.

3. **Don't re-generate the whole buffer by age.** "Re-queue any row older than 24h" looks harmless. For content that depends only on its inputs (a reading for a fixed future date), it regenerated the entire buffer every day: several times the work it needed. Runs blew past their time budget, and the wording changed from one day to the next for the same reading. Generate each row once instead. The "what's missing" tool returns only rows that don't exist yet, and the save tool leaves an existing row alone. Refresh only when you explicitly mark a row invalid (prompt change, input change).

4. **Date-key bug: success everywhere, data missing.** The routine sandbox clock is UTC. A routine scheduled before 07:00 in UTC+7 runs while the sandbox still reads *yesterday*. A prompt saying "use today's date" wrote a morning digest under the previous day's key, a slot already consumed. That dropped a section from the digest two days running. Every run reported success, and the save tool returned `saved: true` every time. Fix: make the first tool call `TZ=<your zone> date +%F`, use that string as-is, and say "do not add or subtract a day". Don't explain the offset in the prompt instead: it's right at the scheduled hour, but on a manual run at any other hour the agent "corrects" a date that was already right. End each run's final message with the key it wrote under, so the runs list shows it at a glance. If a run shows success but today's data is missing, check the date before you touch auth.

5. **Cron is UTC too.** 06:17 Monday in UTC+7 is 23:17 *Sunday* UTC (`17 23 * * 0`). Spread weekly routines across days instead of piling them onto Monday, which is the day that hits the cap first.

6. **A manual run doesn't replace the scheduled one.** Triggering a run doesn't move `next_run_at`, so the same work takes two slots of the daily cap. To fill data *now*, do the same tool calls from an interactive session instead. Save manual runs for disabled routines, testing a prompt change, or recovering from a missed fire.

7. **Don't test side-effecting routines by running the real one.** Create a throwaway one-shot routine a few minutes out, with the prompt forced into dry-run mode, then read its run log.

8. **Prompt details for cloud runs.** Deferred MCP tools didn't come up when the routine searched its tools for `mcp__`. Name them explicitly in the routine prompt (`select:mcp__YourName__tool1,...`). Also add "never use PushNotification": a probe run sent a phone notification without being asked.

9. **Disabling is the reversible delete.** The routine API had no delete action when I checked. `{"enabled": false}` stops a routine from firing or using quota and keeps its config. True deletion is UI-only. For routines that should stop on a date, a cron month filter (e.g. `0 9 * 5,6 *` fires in May and June only) does it without a calendar reminder.

---

## Disclaimer

- This pattern is verified on Next.js 16 + Vercel + Anthropic Max plan as of May 2026. Anthropic's CCR feature is actively evolving; specific endpoint paths and auth requirements may change.
- The env-var claim in the original May version (Sensitive vars unreadable by routines) was wrong and is corrected above. Check Sensitive vs. readable-back behavior on your own Vercel CLI version.
- Read-only `supabase_admin_query` is **not a substitute for proper RLS**. If your service-role key leaks, the SELECT-only validator becomes the only thing standing between an attacker and your data. Treat it as defense-in-depth, not as primary control.

---

*If you've built something similar and hit a different set of gotchas — especially around custom connector discovery or Sensitive-vs-encrypted env var behavior on other deployment platforms — I'd love to hear about it.*
