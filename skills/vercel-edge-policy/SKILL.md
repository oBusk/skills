---
name: vercel-edge-policy
description: Crawling and firewall policy for a project hosted on Vercel — search-engine and AI-crawler rules, robots.txt and Content-Signal, Bot Protection, the AI Bots ruleset, scanner denies and rate limiting. Use when asked about robots, crawlers, AI bots, SEO blocking, the WAF, or the Vercel firewall.
---

# Vercel edge policy

Read-only by default. Gather every finding first, report as one table, then ask
before changing anything. Firewall writes land in a **draft** and are not live
until published — never publish as a side effect of "checking".

This skill owns what happens at the edge. Repo hygiene — dependencies, Tailwind,
tsconfig, Web Analytics, Fluid Compute — belongs to `nextjs-project-checkup`,
which is a separate skill and is **not** invoked from here. Run it yourself if
you want both; a skill quietly skipping half a checklist is worse than two
commands.

## Tool ladder for Vercel settings

Use in this order. Do not skip down a rung until the current one fails.

1. **`vercel api` via the CLI** — the primary path. An authenticated passthrough
   to the REST API, it reads and writes the whole firewall config. Run as
   `pnpx vercel api …`. Recipes in `references/vercel-api.md`.
2. **Vercel MCP** — only for `search_vercel_documentation` and `list_projects` /
   `list_teams`. `get_project` returns a **trimmed** object with **no firewall
   data at all** — never read policy state from it.
3. **`vercel firewall` subcommands** — avoid. `vercel firewall overview` fails
   outright on plans without IP Bypass (`IP Bypass is unavailable for this plan
   (404)`), taking the whole readout with it. `vercel api` has no plan gate.
4. **claude-in-chrome** — last resort, with the user's go-ahead, when the CLI is
   unauthenticated and they would rather not log in.

Tools are whatever the session happens to have, so probe rather than assume: if
`pnpx vercel whoami` fails, report every platform check as `unknown — CLI not
authenticated` and tell the user to run `pnpx vercel login`. Do not guess.

## Step 0 — Identify the project

Find the linked project and team id, in order:

- `.vercel/project.json` → `projectId`, `orgId`
- `.vercel/repo.json` — repo-level links have **no** `project.json`; check this
  before concluding the repo is unlinked. `projects` is an **array**, one entry
  per linked directory, each with its own `directory`, `id` and `orgId`. Pick
  the entry whose `directory` is the deepest match for the directory you are
  checking; never take `projects[0]` on faith. If no entry matches uniquely,
  stop and ask — the wrong pick silently inspects, and later mutates, a
  different project.
- otherwise `list_projects` over MCP, or ask for the dashboard URL

Then confirm auth with `pnpx vercel whoami`. Read the firewall config once
up front (`references/vercel-api.md`) — every step below reads from it.

## Step 1 — Crawling policy

Two independent questions:

**A. Should this project allow search-engine crawling?**
**B. Should this project allow AI-crawler traffic?**

Infer intent before asking. Signals, strongest first:

| Signal | Reads as |
| --- | --- |
| `ai_bots` **`active: true`** and `action: deny` | AI crawling unwanted |
| `ai_bots` absent, or `active: false` | AI crawling allowed (default) |
| robots disallowing `/` for `*` | search crawling unwanted |
| robots naming AI user-agents under `disallow` | AI crawling unwanted |
| No robots file, no firewall rules | **ambiguous — ask** |

Read `active` before `action`. A disabled ruleset keeps its last `action`, so an
inactive `ai_bots` can still read `deny` — treating that as intent inverts the
answer. Inactive is inactive, whatever the action says. Both rulesets ship
inactive; the dashboard labels that state **Allow** for AI Bots and **Off** for
Bot Protection, so "Allow" is the default rather than a decision someone made.

Ask only what the signals leave genuinely open, as two separate questions. If
the user hesitates on B, offer the three-way split in
`references/crawling.md` — most people who say "block AI" mean
training bots only, not AI search or a fetch a human just asked for.

When signals **disagree**, do not ask — the inconsistency *is* the finding:

- `ai_bots: deny` + robots silent on AI → enforcement without the polite signal;
  add the robots rules.
- robots disallows AI + `ai_bots` off → a request bots may ignore, with nothing
  enforcing it. Check *which* agents robots names before proposing anything: if
  it disallows every AI agent, offer `ai_bots: deny`. If it names only training
  bots — the common case — say so and **ask**, because the ruleset is
  all-or-nothing and enabling it would also block AI search and user-requested
  fetches that robots deliberately left alone.
- robots disallows `*` + SEO clearly wanted → surface it.

**C. `Content-Signal` is expected on every project.** It states policy by
purpose rather than by agent name, so it covers crawlers no list knows about yet
— the one non-rotting layer. Nothing enforces it. Set values matching the agent
list already in the file; a missing `Content-Signal` is a `fix`, not an `ask`.
Emit it from `robots.ts` via `rules[].other`, and **without** the Cloudflare
policy preamble.

Where AI crawling is already disallowed, diff the project's agent list against
`references/ai-crawlers.txt`, which is the canonical copy this skill owns.
Entries missing from the project are a `fix`; entries the project has and the
canonical list lacks are a prompt to update the canonical list, not to delete
from the project. The list is deliberately static and committed — no package
dependency — and this checkup is its refresh mechanism.

Apply per `references/crawling.md`: robots for the polite signal,
`references/firewall.md` for enforcement, the two kept in agreement. A static
`public/robots.txt` and a `robots.ts` are both fine, but a project carrying
**both** is serving one and silently ignoring the other — report that.

## Step 2 — Firewall policy

**Budget first**: Hobby allows 3 custom rules and 1 rate-limit rule per
project, Pro 40 and Enterprise 1000. Count existing
rules before proposing new ones, and say what a proposal would consume.

### 2a. Bot Protection

Default recommendation is **Challenge**. Verified bots (Googlebot, Bingbot) are
excluded automatically, so Challenge does **not** cost SEO — that is never a
reason to hold back. Three things are:

1. **A reverse proxy in front of Vercel** (Cloudflare, Azure, another CDN).
   Vercel states Bot Protection does not work here: the proxy masks the signals
   it relies on, real users get challenged, and rotating exit IPs force repeated
   re-challenges. Hard blocker — recommend Off or Log.
2. **Non-browser clients.** Challenge hits anything not browser-like, and only
   *verified* bots are exempt. Grep first: `app/**/route.ts`, webhook receivers,
   feed routes, mobile or server-to-server consumers, CI smoke tests, uptime
   monitors. These need bypass rules **first** — managed rulesets have no path
   scoping, and custom rules run before them.
3. **No traffic baseline yet.** Vercel's guidance is Log first, then Challenge.

For a static marketing or portfolio site with no API routes and no proxy — the
common case here — recommend Challenge without hesitation. Otherwise recommend
Log and name the endpoint that made you hesitate.

### 2b. Let crawlers reach `robots.txt`

`ai_bots: deny` 403s `/robots.txt` itself, so the crawler cannot read the policy
written for it. A project with `ai_bots` active and no bypass rule for
`/robots.txt` is a `fix`. Why, and the recipe, in `references/firewall.md`.

### 2c. Deny `.php` probes

Every project here is Next.js, so a `.php` request is a vulnerability scanner.
Check for a rule denying them; propose one if absent.

Be accurate about why: Next already 404s these, so it is **not** closing a
vulnerability. It denies at the edge so the request never invokes a function,
cutting scanner noise out of logs and analytics. Say that rather than calling it
a security fix.

Add it as a **WAF custom rule** (`rules.insert` in `references/vercel-api.md`),
so the whole edge policy lives in one place and it applies without a deploy. It
spends one of the plan's custom rules, so count the existing ones first.
`vercel.json` `routes[].mitigate` is the fallback: it spends a deployment route
(2048 available) instead of a firewall rule, but `routes` conflicts with
`rewrites` / `redirects` / `headers` / `cleanUrls` / `trailingSlash` in a way the
schema does not catch — conditions in `references/firewall.md`.

### 2d. Rate limiting

**Do not offer by default, and never add a blanket per-IP limit on `/`.** As a
crawler defence it is weak — per-region counters, CDN-cached pages, shared-IP
false positives, and on Hobby it burns the only rate-limit slot. The full
argument is in `references/firewall.md`.

Propose it only when the repo has an endpoint where a burst is expensive or
harmful, scoped to that path:

| Endpoint kind | Detect by | Suggested shape |
| --- | --- | --- |
| Login / auth | `signin`, `login`, `auth` route handlers | 10/min per IP, deny 15m |
| Contact / form POST | server actions, form POST handlers | 5/min per IP, challenge |
| LLM or paid API proxy | `ai`, `openai`, `@ai-sdk`, `resend`, `stripe` | tight, deny |
| Search / expensive query | DB or search client in a route handler | 30/min per IP, challenge |

Otherwise report it as "not recommended" with the reason. Prefer `challenge`
over `deny` on user-facing paths so a false positive is recoverable.

## Step 3 — Report

One table: check, status (`ok` / `fix` / `ask` / `unknown`), detail. Put the
Step 1 questions and any Step 2 hesitation below the table as explicit asks.
Then stop and let the user choose what to apply.

## Applying fixes

Only after the user chooses.

- **Repo edits** (robots output, `vercel.json`) — one concern per commit, and
  verify with `pnpm lint` and `pnpm build`.
- **Firewall changes** — calls in `references/vercel-api.md`. They land in a
  **draft** and are not live until published; say so, and confirm before
  publishing.
- **Verify from outside.** A firewall change is not done until a request proves
  it. `curl -A "GPTBot/1.1"` against the affected path, after publishing.
