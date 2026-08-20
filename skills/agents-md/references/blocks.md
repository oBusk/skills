# Canonical sections

Copied verbatim into a project's `AGENTS.md`, outside the Next.js markers. This
repo holds the canonical copy; projects carry it; `nextjs-project-checkup`
reports drift.

## Scope: only what is counterintuitive

Every line must correct a belief someone actually held. A model already knows
`pnpm install` installs dependencies — writing that down taxes every session in
the repo for no gain. Each entry below maps to a real mistake.

**State the rule, not the reasoning.** These files load in full on every session
in the repo, so a paragraph explaining *why* a rule holds is paid for on every
turn and read on almost none. The reasoning belongs in the skill that acts on it
— `workspace-config.md` for why the lockfile cannot be grepped,
`dependencies.md` for the `pnpmfileChecksum` procedure. Keep enough here that
the rule is not mistaken for pedantry, and no more.

Both blocks below have been cut once already for exactly this. If a section is
drifting past roughly twenty lines, the surplus is almost certainly explanation
that has a better home.

## `## pnpm`

Scoped deliberately. pnpm's core CLI is unchanged and well known, so a blanket
"this is not the pnpm you know" would invite distrust of `pnpm install` too and
waste attention. Only the pnpm 11 additions genuinely post-date training data.

````markdown
## pnpm

pnpm 11 added settings and verbs that post-date most training data, and
`@obusk/pnpm-plugin-defaults` — a `configDependencies` entry in
`pnpm-workspace.yaml` — changes a dozen more defaults on top. Read its
`pnpmfile.cjs` in `node_modules/.pnpm-config` rather than assuming stock
behaviour. The core CLI is unchanged; the list below is what is not.

- **Run local binaries with `pnpm exec`, never `npx`** — `npx` silently fetches
  from the registry when it cannot resolve locally, so it can run something
  other than what is installed.
- **`configDependencies` update with `pnpm add --config <pkg>@latest`.**
  `pnpm update` does not touch them.
- **`pnpm update` also rewrites action pins in `.github/workflows/*.yml`.**
  Workflow diffs during a dependency upgrade are expected.
- **`pnpm outdated` lists more than `package.json`** — `node` comes from
  `devEngines`, GitHub Actions from workflow files.
- **Ask `pnpm why <pkg> --depth 0 --json` what is installed.** Never grep
  `pnpm-lock.yaml` or read `node_modules/.pnpm`; both under-report silently.
- **`pnpm clean`** (alias `purge`) removes `node_modules`; `-l` also removes the
  lockfile.
- **Releases under three days old will not resolve** (`minimumReleaseAge`). A
  new version appearing not to exist is this, not a registry problem.
````

`pnpx` versus `npx` for **one-off remote** tools is a personal preference, not a
repo rule — both fetch from the registry and both work. It belongs in a
machine-level config (`~/.claude/CLAUDE.md`), not here, so the repo does not
impose it on other contributors. The `pnpm exec` rule above is different: that
one is about correctness and does belong in the repo.

## `## Code comments`

````markdown
## Code comments

Do not write code comments by default. Convey meaning through names, types and
structure.

A comment is warranted only for something a reader cannot infer from the code
itself — a non-obvious constraint, an API quirk, a deliberate workaround. Surface
it for review rather than adding it silently.

Never write a comment that argues for a change, describes what you just did, or
references review feedback or a conversation. Comments are read years later by
someone with no knowledge of the change that introduced them; that context
belongs in the pull request.
````

Agents reach for comments as a way of *answering* — narrating a fix, justifying
an approach, pointing at code that no longer exists. That is the failure this
section prevents, and it is why the rule is a default rather than a ban.
