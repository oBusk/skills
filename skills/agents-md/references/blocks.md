# Canonical sections

Copied verbatim into a project's `AGENTS.md`, outside the Next.js markers. This
repo holds the canonical copy; projects carry it; `nextjs-project-checkup`
reports drift.

## Scope: only what is counterintuitive

Every line must correct a belief someone actually held. A model already knows
`pnpm install` installs dependencies — writing that down taxes every session in
the repo for no gain. Each entry below maps to a real mistake.

Do not let these grow into manuals. If a proposed addition is not fixing a wrong
assumption, leave it out.

## `## pnpm`

Scoped deliberately. pnpm's core CLI is unchanged and well known, so a blanket
"this is not the pnpm you know" would invite distrust of `pnpm install` too and
waste attention. Only the pnpm 11 additions genuinely post-date training data.

````markdown
## pnpm

pnpm 11 added settings and verbs that post-date most training data, and this
repo uses them. The core CLI is unchanged; the list below is what is not.

- **Run local binaries with `pnpm exec`, never `npx`.** When `npx` cannot
  resolve a package locally it silently fetches it from the registry, so a
  missing dependency or a typo runs something other than what is installed.
  `pnpm exec` only runs what is in the tree, and fails loudly when it is not.
- **`configDependencies` need `pnpm add --config <pkg>@latest`.** `pnpm update`
  does not touch them. Follow with `pnpm clean && pnpm install` and confirm
  `pnpmfileChecksum` changed in the lockfile — a bump without it does nothing.
- **`pnpm update` rewrites `.github/workflows/*.yml`**, resolving action tags to
  commit SHAs with the release tag as a comment. Workflow diffs during a
  dependency upgrade are expected, not contamination from another process.
- **`pnpm outdated` rows are not all `package.json` entries.** `node` comes from
  `devEngines`, GitHub Actions from workflow files. Do not go looking for them
  in `dependencies`.
- **Ask `pnpm why <pkg> --depth 0 --json` what is installed.** Never grep
  `pnpm-lock.yaml`: scoped packages are quoted and snapshots carry peer
  suffixes, so patterns miss much of the file without saying so. Never read
  `node_modules/.pnpm` either — long names are truncated to a hash.
- **`pnpm clean`** (alias `purge`) removes `node_modules`; `pnpm clean -l` also
  removes the lockfile.
- **Releases under three days old will not resolve** (`minimumReleaseAge`,
  strict). A brand-new version appearing not to exist is this, not a registry
  problem; `minimumReleaseAgeExclude` opts a package out.
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
