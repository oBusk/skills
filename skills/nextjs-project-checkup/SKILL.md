---
name: nextjs-project-checkup
description: Hygiene checkup for a Next.js project hosted on Vercel — dependency freshness, pnpm audit, Tailwind v4, ESLint config, pnpm toolchain and workspace config, tsconfig, CI, Web Analytics and Fluid Compute. Use when asked to "check", "audit", or "run a checkup on" a project, or to verify a project is fully updated and configured. Crawling and firewall policy live in the separate vercel-bot-policy skill.
---

# Next.js project checkup

Read-only by default. Gather every finding first, report as one table, then ask
before changing anything. Never edit files as a side effect of "checking".

Steps 1–2 read the repo, Step 3 reads the platform. Details live in
`references/`; this file is the decision logic.

## Tool ladder for Vercel settings

Use in this order. Do not skip down a rung until the current one fails.

1. **`vercel api` via the CLI** — the primary path. An authenticated passthrough
   to the REST API, it exposes everything the dashboard shows. Run as
   `pnpx vercel api …`. Recipes in `references/vercel-api.md`.
2. **Vercel MCP** — only for `search_vercel_documentation` and `list_projects` /
   `list_teams`. `get_project` returns a **trimmed** object with no analytics,
   no resource config and no firewall data — never read configuration state
   from it.
3. **claude-in-chrome** — last resort, with the user's go-ahead, when the CLI is
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
correct. Moving `typescript` to 6 needs an explicit `types` array in
`tsconfig.json` in the same commit, or global test types stop resolving. Also run `pnpm audit` and report anything other than "No known
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
- **`AGENTS.md` / `CLAUDE.md`** — must be **committed**. `next dev` writes a
  managed block into them from Next 16.3; untracked or gitignored is the
  finding, and a regenerated block mid-task belongs in the commit, not a report.
- **`next-env.d.ts`** — must be **both** gitignored and untracked.
  Ignored-but-committed is the common broken state, since `.gitignore` does not
  apply retroactively to tracked files.
- **`tsconfig.json`** — `pnpm build` lets Next patch what it manages; compare the
  rest against `references/tsconfig.reference.json`. Expect `paths` to differ.
- **CI** — `pnpm/setup` (not `pnpm/action-setup`) with `cache: true`, and no
  leftover `run_install` / `standalone`. Write action refs in **tag** form and
  let `pnpm update` resolve and pin the SHA; never hand-resolve a commit hash.
- **`.github/dependabot.yml`** — should not exist. `pnpm update` covers action
  pins, so a `github-actions` ecosystem entry is duplicate work; delete the file
  if that is all it declares. Security alerts are repo settings and are not
  affected.
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

## Step 4 — Report

One table: check, status (`ok` / `fix` / `ask` / `unknown`), detail. Put anything
needing a decision below the table as an explicit ask, then stop and let the
user choose what to apply.

Crawling and firewall policy are **not** in scope here — they belong to the
`vercel-bot-policy` skill. Do not invoke it; say in the report that bot and firewall
policy were not checked, so a clean report is never mistaken for a full one.

## Applying fixes

Only after the user chooses.

- **Dependencies** — follow the ordered steps in
  `references/dependency-holds.md`; the config dependency goes first and the
  fresh lockfile last.
- **Tailwind v4 → ESLint config** — a fixed sequence
  (`references/tailwind-and-eslint.md`): `pnpx @tailwindcss/upgrade`, bump
  `tailwind-merge`, audit what the codemod got wrong, review `globals.css`, then
  `@obusk/eslint-config-next@16.3`, then `cssConfigPath`. Lint **fails between
  the first and fifth steps and that is accepted** — carry on rather than fixing
  or reverting. If work stops in that window, say so in the report. The codemod
  silently drops JS plugins and over-applies utility renames, so this migration
  needs a **browser check** before commit; a green build does not catch either.
- **Other repo edits** (`<Analytics />`, workflows) — one concern per commit.
- **Platform changes** — calls in `references/vercel-api.md`. Re-read the
  project afterwards and confirm the field actually moved.

Verify each change with `pnpm lint` (which chains prettier) and `pnpm build`,
and **finish the checkup by running `pnpm lint` once more**. A green final lint
is the exit condition; the Tailwind purgatory is the only place a red one is
acceptable.
