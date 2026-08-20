# Canonical sections

Copied verbatim into a project's `AGENTS.md`, outside the Next.js markers. This
repo holds the canonical copy; projects carry it; `nextjs-project-checkup`
reports drift.

## Scope: only what is counterintuitive

Every line must correct a belief someone actually held. A model already knows
`pnpm install` installs dependencies — writing that down taxes every session in
the repo for no gain. Each entry below maps to a real mistake.

**State the rule, not the reasoning.** These files load in full on every session
in the repo, so a paragraph explaining _why_ a rule holds is paid for on every
turn and read on almost none. The reasoning belongs in the skill that acts on it
— `workspace-config.md` for why the lockfile cannot be grepped,
`dependencies.md` for the `pnpmfileChecksum` procedure. Keep enough here that
the rule is not mistaken for pedantry, and no more.

Both blocks below have been cut once already for exactly this. If a section is
drifting past roughly twenty lines, the surplus is almost certainly explanation
that has a better home.

## `## pnpm`

Scope this to pnpm 11 additions and the settings the plugin applies. Do not
broaden it into a general warning that pnpm is unfamiliar — the core CLI is
unchanged, and blanket doubt spends attention on commands that behave normally.

```markdown
## pnpm

> pnpm may have changed since your training data. The core CLI is unchanged; where syntax looks unfamiliar, check `pnpm help`.

### `@obusk/pnpm-plugin-defaults`

This project uses a config dependency `@obusk/pnpm-plugin-defaults` which sets some more opinionated defaults for security and stability. Config dependencies are not updated by `pnpm update` but with `pnpm add --config @obusk/pnpm-plugin-defaults`.

### Security policies

pnpm in this repository will...

- not resolve releases that are less than 72 hours old, unless they are exempt in `pnpm-workspace.yaml`.
- refuse to install packages that lack provenance or trusted publishers, if any older release does have it. This can also be overridden in `pnpm-workspace.yaml`.
- not execute install scripts by default, but must be explicitly enabled in `pnpm-workspace.yaml` for each package that needs it.

**Do not change any of these settings.** If an install is blocked, stop and ask
the developer. The block is the policy working; routing around it installs the
package the policy rejected.

- Avoid using `-i` or `--interactive` flags since they will hang.
- To execute locally installed packages, use `pnpm exec <package>`, this will ensure you run the locally installed package and don't accidentally download it anew.
- `pnpm update` and `pnpm outdated` cover more than `dependencies` and `devDependencies` — they also check `engines.node`, `devEngines.runtime` and the GitHub Actions pins in `.github/workflows/*.yml`.
- To find out whether a package is installed and at what version, use `pnpm why <package> --depth 0 --json`. It reports every resolved version and what depends on it. Do not parse `pnpm-lock.yaml` or read `node_modules/`.
- `pnpm clean` will delete all `node_modules` folders in the workspace, which can be useful if in a bad state.
```

## `## Code comments`

```markdown
## Code comments

Unless explicitly instructed, do not write code comments. Use names, types and structure to convey purpose. Do not use comments to justify decisions or respond to feedback.
```

Agents reach for comments as a way of _answering_ — narrating a fix, justifying
an approach, pointing at code that no longer exists. That is why the block names
those uses outright instead of leaving them to the general rule, and why the
exception is worded as *explicitly instructed*: the decision belongs to a human,
not to the agent's judgement of whether this comment is one of the good ones.
