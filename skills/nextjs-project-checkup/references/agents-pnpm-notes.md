# The `## pnpm` block for a project's `AGENTS.md`

Canonical copy. Projects carry this verbatim; the checkup diffs theirs against
it and reports drift, the same arrangement as `ai-crawlers.txt` in
`vercel-bot-policy`.

Paste it **outside** the `<!-- BEGIN:nextjs-agent-rules -->` block, which
`next dev` rewrites. Content outside that block is preserved.

## What belongs here

Only behaviour where the **default assumption is wrong**. A model already knows
`pnpm install` installs dependencies; writing that down is context tax on every
session in the repo. Every line below exists because it caused a real mistake.

Do not let this grow into a pnpm manual. If a new entry is not correcting a
belief someone actually held, leave it out.

## The block

```markdown
## pnpm

Behaviours that differ from other package managers. Everything else is ordinary.

- **`pnpx`, not `npx`.**
- **`configDependencies` need `pnpm add --config <pkg>@latest`.** `pnpm update`
  does not touch them. Follow with `pnpm clean && pnpm install` and confirm
  `pnpmfileChecksum` changed in the lockfile — a bump without it does nothing.
- **`pnpm update` rewrites `.github/workflows/*.yml`**, resolving action tags to
  commit SHAs with the release tag as a comment. Workflow diffs during a
  dependency upgrade are expected, not contamination from another process.
- **`pnpm outdated` rows are not all `package.json` entries.** `node` comes from
  `devEngines`, GitHub Actions from workflow files. Do not go hunting for them
  in `dependencies`.
- **Ask `pnpm why <pkg> --depth 0 --json` what is installed.** Never grep
  `pnpm-lock.yaml`: scoped packages are quoted and snapshots carry peer
  suffixes, so patterns miss much of the file without saying so. Never read
  `node_modules/.pnpm` either — long names are truncated to a hash.
- **`pnpm clean`** (alias `purge`) removes `node_modules`; `pnpm clean -l` also
  removes the lockfile.
- **Releases under three days old will not resolve** (`minimumReleaseAge`,
  strict). A brand-new version appearing not to exist is this, not a registry
  problem — `minimumReleaseAgeExclude` opts a package out.
```

## Checking a project

```bash
sed -n '/^## pnpm$/,/^## /p' AGENTS.md > /tmp/project-block
```

Compare against the fenced block above. Missing entirely is a `fix`. Drifted is
a `fix` in whichever direction is stale — if the project has a line this file
lacks, someone learned something; propose adding it here rather than deleting it
there.
