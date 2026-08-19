---
name: vercel-project-checkup
description: Hygiene checkup for a Next.js project hosted on Vercel — dependency freshness, pnpm audit, Tailwind v4, pnpm toolchain and workspace config, Web Analytics, Fluid Compute, and the search-engine / AI-crawler and firewall policy. Use when asked to "check", "audit", or "run a checkup on" a project, or to verify a project is fully updated and configured.
---

# Vercel project checkup

Read-only by default. Gather every finding first, report as one table, then ask
before changing anything. Never edit files or publish firewall changes as a side
effect of "checking".

Steps 1–2 read the repo, 3–5 read the platform. Details live in `references/`;
this file is the decision logic.

## Tool ladder for Vercel settings

Use in this order. Do not skip down a rung until the current one fails.

1. **`vercel api` via the CLI** — the primary path. An authenticated passthrough
   to the REST API, it exposes everything the dashboard shows. Run as
   `pnpx vercel api …`. Recipes in `references/vercel-api.md`.
2. **Vercel MCP** — only for `search_vercel_documentation` and `list_projects` /
   `list_teams`. `get_project` returns a **trimmed** object with no analytics,
   no resource config and no firewall data — never read configuration state
   from it.
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

Then confirm auth with `pnpx vercel whoami`.

## Step 1 — Dependencies

Three passes, reported separately. Full mechanics and the hold table are in
`references/dependency-holds.md`.

| Pass | Command | Reports |
| --- | --- | --- |
| 1 | `pnpm outdated --compatible` | in-range minors/patches — safe to take |
| 2 | `pnpm outdated` | majors, minus held packages |
| 3 | `latest_in_major <pkg> <major>` | new minors inside a held major |

Held packages (node/`@types/node` 24, `typescript` 6, `lucide-react` 0) are
deliberate, not stale — suppress them in pass 2. Pass 3 exists because they
would otherwise fall through the gap between passes 1 and 2 and stop being
checked at all.

Read a declared range as intent, not as a style error: `~` on `typescript` is
correct. Also run `pnpm audit` and report anything other than "No known
vulnerabilities found".

## Step 2 — Repo config

Details for each in `references/workspace-config.md`.

- **Tailwind v4 and the ESLint config** — one coupled job, per
  `references/tailwind-and-eslint.md`. Tailwind at `^4` via
  `@tailwindcss/postcss`; `eslint.config.mjs` using `defineConfig`, with
  `settings.react.version` as the bare major `"19"` (eslint v10 only) and
  `settings.tailwindcss.cssConfigPath` once on `@obusk/eslint-config-next@16.3`.
  If the project has no Tailwind, skip every Tailwind item — not a finding.
- **pnpm toolchain** — `packageManager` against `npm view pnpm version`, and the
  `@obusk/pnpm-plugin-defaults` pin in `pnpm-workspace.yaml` `configDependencies`
  (*not* `devDependencies`) against `npm view @obusk/pnpm-plugin-defaults version`.
  A missing key is the finding.
- **`@types/react` / `@types/react-dom`** — pinned in both `package.json` and
  `overrides`; the two must match and must move together.
- **Stale workspace entries** — dead audit overrides, and `trustPolicyExclude` /
  `allowBuilds` entries naming versions no longer in the tree. A dead override
  is proven by re-resolving without it, never by eyeballing versions.
- **`next-env.d.ts`** — must be **both** gitignored and untracked.
  Ignored-but-committed is the common broken state, since `.gitignore` does not
  apply retroactively to tracked files.
- **`tsconfig.json`** — `pnpm build` lets Next patch what it manages; compare the
  rest against `references/tsconfig.reference.json`. Expect `paths` to differ.
- **CI** — `pnpm/setup` (not `pnpm/action-setup`) with `cache: true`, and no
  leftover `run_install` / `standalone`. Write action refs in **tag** form and
  let Dependabot re-pin the SHA; never hand-resolve a commit hash.
- **Analytics wiring** — `@vercel/analytics` in `dependencies` *and*
  `<Analytics />` rendered in the root layout. The package alone reports nothing.

## Step 3 — Platform settings

One `vercel api` call to the project endpoint covers these:

- **Web Analytics** — `webAnalytics.enabledAt` present means enabled. An object
  with only an `id` and no `enabledAt` means **not** enabled. Cross-check
  against Step 2: package installed but analytics off (or the reverse) is the
  common broken state.
- **Fluid Compute** — `defaultResourceConfig.fluid`. `false` is the finding.
- Report when off-nominal: `speedInsights` (same `enabledAt` rule), `nodeVersion`
  vs `engines.node`, and `ssoProtection` on a site meant to be public.

## Step 4 — Crawling policy

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
`references/crawling-and-firewall.md` — most people who say "block AI" mean
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

Where AI crawling is already disallowed, check **how** the agent list is
sourced. A hardcoded array in `robots.ts` is a `fix`, not an `ok` — it was
correct when written and rots from there. Replace it with
`@geosuite/ai-crawler-bots`, but diff the two lists first and keep anything the
package lacks; it is not a superset.

Apply per `references/crawling-and-firewall.md`: robots for the polite signal,
the firewall for enforcement, the two kept in agreement.

## Step 5 — Firewall policy

Read the firewall config once (`references/vercel-api.md`). **Budget first**:
Hobby allows 3 custom rules and 1 rate-limit rule per project. Count existing
rules before proposing new ones, and say what a proposal would consume.

### 5a. Bot Protection

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

### 5b. Deny `.php` probes

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
schema does not catch — conditions in `references/crawling-and-firewall.md`.

### 5c. Rate limiting

**Do not offer by default, and never add a blanket per-IP limit on `/`.** As a
crawler defence it is weak — per-region counters, CDN-cached pages, shared-IP
false positives, and on Hobby it burns the only rate-limit slot. The full
argument is in `references/crawling-and-firewall.md`.

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

## Step 6 — Report

One table: check, status (`ok` / `fix` / `ask` / `unknown`), detail. Put the
Step 4 questions and any Step 5 hesitation below the table as explicit asks.
Then stop and let the user choose what to apply.

## Applying fixes

Only after the user chooses.

- **Dependencies** — follow the ordered steps in
  `references/dependency-holds.md`. Each has its own command and they are not
  interchangeable; the config dependency goes first and the fresh lockfile last.
- **Tailwind v4 → ESLint config** — a fixed sequence
  (`references/tailwind-and-eslint.md`): `pnpx @tailwindcss/upgrade`, bump
  `tailwind-merge`, review `globals.css`, then `@obusk/eslint-config-next@16.3`,
  then `cssConfigPath`. Lint **fails between the first and fourth steps and that
  is accepted** — carry on rather than fixing or reverting. If work stops in
  that window, say so in the report.
- **Other repo edits** (robots, `<Analytics />`, `vercel.json`, workflows) —
  one concern per commit.
- **Platform changes** — calls in `references/vercel-api.md`. Firewall changes
  land in a **draft** and are not live until published; say so, and confirm
  before publishing.

Verify each change with `pnpm lint` (which chains prettier) and `pnpm build`,
and **finish the checkup by running `pnpm lint` once more**. A green final lint
is the exit condition; the Tailwind purgatory is the only place a red one is
acceptable.
