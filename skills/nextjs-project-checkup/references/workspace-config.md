# Workspace config and repo hygiene

Covers `pnpm-workspace.yaml`, `tsconfig.json`, and the files that should never
have been committed. All of it drifts silently — nothing fails when these go
stale, which is why they need a periodic pass.

## Pinned `@types/react` / `@types/react-dom`

These are pinned in **two** places and must move together:

```yaml
# pnpm-workspace.yaml
overrides:
    "@types/react": 19.2.18
    "@types/react-dom": 19.2.4
```

```jsonc
// package.json
"@types/react": "19.2.18",
"@types/react-dom": "19.2.4",
```

The `devDependencies` entry governs the direct dependency; the `overrides` entry
forces every transitive consumer onto the same copy. Bumping one without the
other reintroduces the duplicate-React-types conflict the override exists to
prevent — and it fails as confusing structural type errors between two
identical-looking `ReactNode` types, not as a version warning.

Check both, and when upgrading, edit both in the same commit.

## Pruning stale entries

Version-scoped entries in `pnpm-workspace.yaml` go stale as the tree moves. Each
kind has its own test.

### Audit-remediation overrides

Entries like `postcss@<8.5.10: ^8.5.10` or `sharp@<0.35.0: ^0.35.0` were added to
force a vulnerable transitive dependency forward. Once upstream catches up, they
do nothing but add noise and pin risk.

A version comparison is **not** a valid test — seeing `sharp@0.35.3` in the tree
does not tell you whether the override is what put it there. Re-resolve without
the entry and compare:

```bash
cp pnpm-lock.yaml /tmp/lock.bak && cp pnpm-workspace.yaml /tmp/ws.bak
# remove the override entries under test from pnpm-workspace.yaml
pnpm install --lockfile-only --ignore-scripts
diff <(grep -oE '^  [a-z@/-]+@[0-9.]+:' /tmp/lock.bak | sort -u) \
     <(grep -oE '^  [a-z@/-]+@[0-9.]+:' pnpm-lock.yaml | sort -u)
cp /tmp/lock.bak pnpm-lock.yaml && cp /tmp/ws.bak pnpm-workspace.yaml
```

Empty diff = the override is dead, safe to delete. Any version moving *down* =
it is load-bearing, keep it.

`--lockfile-only --ignore-scripts` keeps this off `node_modules`, so it is safe
to run mid-checkup. Always restore both files, then confirm with `git status`
that the working tree is clean before moving on.

### `trustPolicyExclude` and version-pinned `allowBuilds`

These name exact versions, so they rot the moment the package moves:

```yaml
allowBuilds:
    simple-git-hooks@2.13.1: true   # pinned — stale after any bump

trustPolicyExclude:
    - semver@6.3.1
    - tailwind-merge@2.6.1
```

Test by checking whether that exact version is still resolved:

```bash
ls node_modules/.pnpm | grep -E "^tailwind-merge@[0-9]"
```

A version present in the tree means the entry is live. Absent means it is stale
— delete it. This is a much cheaper test than the override one because the
entries are keyed by exact version, so presence is the whole question.

Unversioned `allowBuilds` entries (`sharp: true`) only go stale when the package
leaves the tree entirely.

### `minimumReleaseAgeExclude`

Entries name packages you want exempt from the release-age delay, usually
because you track them closely. Stale here means the package is no longer a
dependency at all — check presence, same as above. Glob entries (`"@next/*"`)
are fine to leave.

## `next-env.d.ts`

Next regenerates this on every build, so it must be ignored **and** untracked.
The failure mode is a file that is listed in `.gitignore` but was committed
before the ignore rule landed — `.gitignore` has no effect on already-tracked
files, so it keeps showing up in diffs forever.

Check both halves independently:

```bash
grep -n "next-env.d.ts" .gitignore          # is it ignored?
git ls-files --error-unmatch next-env.d.ts  # is it tracked? (error = good)
```

Ignored-and-untracked is correct. Ignored-but-tracked is the finding:

```bash
git rm --cached next-env.d.ts
```

That stages a deletion without touching the file on disk. Mention that anyone
else on the repo gets the file deleted from their working copy on pull, and Next
regenerates it on their next build — harmless, but worth saying rather than
having it surprise someone.

## `AGENTS.md` and `CLAUDE.md`

**Commit them.** They are the opposite of `next-env.d.ts` above, and the
similarity is a trap: both are touched by Next, but only one is a build
artifact.

From Next 16.3, `next dev` writes a managed block into both files when it
detects an agent in the environment:

```md
<!-- BEGIN:nextjs-agent-rules -->
...
<!-- END:nextjs-agent-rules -->
```

Next's own guidance is explicit that this is not a file to ignore:

> This block is written and re-added by `next dev`. Removing it from a diff only
> re-creates the uncommitted change; committing it with your work keeps the tree
> clean.

So:

- **Never gitignore them.** They would regenerate on every dev run and show as
  untracked forever, which is the state that prompts someone to ignore them
  again.
- **Never strip the managed block** to tidy a diff. It comes straight back.
- **Content outside the block is yours** and is preserved on regeneration — Next
  upserts rather than overwrites. That is where project description, commands
  and conventions go.
- A regenerated block appearing mid-task is not a finding. Fold it into the
  commit you are already making and say so.

Untracked `AGENTS.md` / `CLAUDE.md` is itself the finding: commit them. This
skill reads a project's `AGENTS.md` for a `## Dependency holds` section
(`dependency-holds.md`), which only works if the file is in the repo.

## GitHub Actions: pnpm setup

Use **`pnpm/setup`**, not the older `pnpm/action-setup`. `pnpm/setup@v2`
installs pnpm 11 and newer from self-contained binaries; `action-setup` is only
needed for pnpm 10 or older, which none of these projects are on.

Correct shape:

```yaml
- name: Set up using pnpm
  uses: pnpm/setup@v2
  with:
      cache: true
```

- **`cache: true` is required, not optional.** It defaults to `false`, so
  omitting it silently costs you the store cache on every run. This is the
  single most common finding here.
- **`run_install` and `standalone` must be removed.** Both are `action-setup`
  inputs that `pnpm/setup` does not accept — `run_install` is superseded by
  `install`, which runs `pnpm install` by default, and `standalone` is
  meaningless now that the action always uses self-contained binaries. Carrying
  them over during a migration is the other common finding.
- There is no `version` input to set: pnpm's version comes from
  `packageManager` in `package.json`.

### `pnpm update` updates the actions

**Action versions are pnpm's job in these projects, not Dependabot's.**
`@obusk/pnpm-plugin-defaults` sets `updateConfig.githubActions = true`, so
`pnpm outdated` reports outdated actions from `.github/workflows/*.yml` alongside
packages, and `pnpm update` bumps them. Elsewhere the same behaviour is
`--include-github-actions` on either command.

pnpm writes the pinned form itself — exact commit SHA, release tag preserved as a
comment:

```yaml
uses: pnpm/setup@84cb39b217b10273981911c288cd62326dc7c6d2 # v2.0.2
```

So when editing a workflow by hand, **write the tag form** — `pnpm/setup@v2` —
and let the next `pnpm update` resolve and pin it. Never hand-resolve a SHA: it
is an error-prone detour and a wrong one is a broken workflow.

Two consequences worth holding onto:

- **Workflow diffs during a dependency upgrade are expected.** `pnpm update`
  touching `.github/workflows` is the configured behaviour, not contamination
  from another process. Do not attribute it to Dependabot or to a pulled commit,
  and do not report it as unexplained.
- **`pnpm outdated` rows are not all packages.** An action can appear in pass 1
  or pass 2 (`dependency-holds.md`) and is not a `package.json` entry — do not go
  hunting for it in `dependencies`.

### Remove `.github/dependabot.yml`

Superseded by the above. Report its presence as a `fix`.

**If it only declares `package-ecosystem: github-actions`** — the common case —
delete the whole file. `pnpm update` does that job, and two things bumping the
same pins produces duplicate PRs and pointless churn.

**If it declares other ecosystems**, remove only the `github-actions` entry and
say what remains. An `npm` entry is also covered by this checkup's dependency
passes, but dropping it is a larger decision — see the trade below — so surface
it rather than deleting it.

Be straight about what is lost: this is not pure redundancy. Dependabot opens PRs
**on a schedule with nobody present**; `pnpm update` runs when someone runs it.
Removing the config means action pins move only when a checkup happens. That is
the intended trade here — updates arrive in one reviewed batch instead of a
trickle of bot PRs — but it is a trade, and a repo that goes unvisited for a year
will show it.

Two things the file does **not** control, so deleting it does not disable them:

- **Dependabot security updates and alerts** are repository settings, not
  `dependabot.yml`. Vulnerability alerts keep arriving.
- **`dependency-review.yml`** (or any workflow using
  `actions/dependency-review-action`) is unrelated. Leave it.

This is a settled decision for these projects. Do not re-propose adding
Dependabot back on a later run.

## `tsconfig.json`

Two layers, and only one of them self-heals.

**Next-managed.** Next patches `tsconfig.json` on `dev` and `build`, adding
required entries as they change between releases. So the cheapest check is to
build and look:

```bash
pnpm build && git diff tsconfig.json
```

A non-empty diff means the config was behind and Next has just fixed it — review
and commit. Next only *adds* what it needs; it never removes stale options or
revises `target`.

**Hand-maintained.** Compare the rest against `tsconfig.reference.json` in this
directory. Report drift, do not apply it blindly — `types` is absent from the
reference on purpose (see `dependency-holds.md`; it is per-project and adding it
to a project that does not need it *removes* global types), and `paths` is a
per-project convention (these projects use `^/*` → `./src/*`, not the
`@/*` create-next-app default) and differences there are expected, not findings.

Markers worth confirming on a Next 16 + TypeScript 6 project:

| Setting | Expected | Why it matters |
| --- | --- | --- |
| `include` | contains `.next/dev/types/**/*.ts` | Next 16 dev types; absent on configs predating it |
| `include` | contains `**/*.mts` | otherwise `.mts` files are silently untyped |
| `moduleResolution` | `bundler` | `node` is the legacy value and resolves `exports` wrong |
| `plugins` | `[{ "name": "next" }]` | editor integration for the Next plugin |
