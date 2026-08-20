# Dependencies

Detecting what is out of date, and moving it. Two halves:

- **The three passes** below find drift. They exist in that shape because some
  packages are deliberately held on an older major — reporting those as findings
  every run is noise, and suppressing them naively stops checking them at all.
- **Upgrading** is an ordered sequence, not a command. The config dependency goes
  first so everything resolves under it, and the fresh lockfile goes last.

Both halves interleave with `workspace-config.md`: the `@types/react` override
moves in the same commit as its package bump, and stale-entry pruning can only
run once the updates have landed.

## The three-pass check

```bash
pnpm outdated --compatible   # pass 1: in-range drift
pnpm outdated                # pass 2: everything outdated — includes pass 1
                             # pass 3: held packages, see below
```

**Pass 1 — `--compatible`** prints only versions that satisfy the range already
declared in `package.json`. Everything it lists sits inside the range you chose,
so it is safe to take and needs no discussion. Empty output means every
dependency is at the top of its declared range. This is the actionable list.

**Pass 2 — plain `pnpm outdated`** compares against the absolute latest,
ignoring ranges. It is a **superset of pass 1**, not a majors-only list —
anything below the top of its range is also below the latest, so every pass 1
row appears here too. Subtract before reading it:

```
majors = pass 2 − pass 1 − held packages
```

Skipping the subtraction reports a routine in-range patch as a major migration
and sends Step 2 to run `--latest` on a package that only needed `pnpm update`.

Rows here are also not all `package.json` entries: `node` arrives from
`devEngines`, and GitHub Actions from `.github/workflows` because the plugin
enables `updateConfig.githubActions` (`workspace-config.md`).

What survives the subtraction is the genuine major-upgrade candidate list. Held
packages appearing in pass 2 are **expected** and must not be reported.

**Pass 3 — held packages** exists because passes 1 and 2 have a blind spot
between them. A package held with `~` has its minors excluded from pass 1 (out
of range) *and* suppressed in pass 2 (it is in the hold table), so the available
minor inside the held major is reported by neither. Query it directly:

```bash
latest_in_major() {
  out=$(npm view "$1@^$2" version --json) || { echo "unknown: npm view failed"; return 1; }
  printf '%s' "$out" | node -e '
    let s="";process.stdin.on("data",d=>s+=d).on("end",()=>{
      if(!s.trim())return console.log("none");
      let v=JSON.parse(s); v=Array.isArray(v)?v:[v];
      const key=x=>x.split(".").map(Number);
      v.sort((a,b)=>{const A=key(a),B=key(b);return A[0]-B[0]||A[1]-B[1]||A[2]-B[2]});
      console.log(v[v.length-1]);
    })'
}

latest_in_major typescript 6      # -> highest 6.x
latest_in_major lucide-react 0    # -> highest 0.x
```

The sort is load-bearing: `npm view` returns versions in **publish order, not
semver order**, so taking the last element raw picks up backport patches
(`react@^19` yields `19.0.8` unsorted, `19.2.8` sorted).

Run it for every held package and compare against the installed version. A gap
is not a finding to fix silently — it is a **decision to surface**, because the
whole point of a minor hold is that the minor is a conscious step.

Report the three passes separately. "Safe to take now", "a major migration
someone must decide on", and "your held package has a new minor waiting" are
three different asks.

## Range hygiene

The range encodes *what kind* of hold it is. Both kinds are legitimate — read
the range as intent rather than correcting it toward a house style:

| Declared | Holds | Pass 1 surfaces | Use when |
| --- | --- | --- | --- |
| `^6.0.3` | major | 6.1.0, 6.2.0, 6.0.9 | minors are routine and safe to take |
| `~6.0.3` | major + minor | 6.0.9 only | minors carry real breakage risk |
| `6.0.3` | everything | nothing | reproducibility, or an `overrides` pin |

Only flag a range when it contradicts the `Range` column of the hold table — a
`~` on a package held major-only means minors are being frozen by accident. The
tilde on `typescript` is listed there and is never a finding; pass 3 is what
keeps it visible. And check `pnpm-workspace.yaml` `overrides` before calling an
exact pin a finding; resolving a transitive conflict is a valid reason to pin
hard.

## Held packages

Shared baseline across oBusk projects:

| Package | Held at | Range | Why |
| --- | --- | --- | --- |
| `node` | 24 | `^` | Runtime target; also `engines.node` and `devEngines.runtime.version` |
| `@types/node` | 24 | `^` | Must track the `node` major, not lead it |
| `typescript` | 6 | `~` | Not semver — minors add checks and change inference. Moving to 6 may need a `types` array *if the project uses global types*; see below |
| `lucide-react` | 0 | `^` | v1+ dropped icons still in use |

`node` is declared in `devEngines.runtime.version` (`^24.19.0`) and
`engines.node` (`24.x`), never in `dependencies`. pnpm 11 reads `devEngines` and
reports it in pass 2 anyway, as a `(dev)` row:

```
│ node (dev)        │ 24.19.0 │ 26.7.0 │
```

That is observed output, not an assumption — so the Node row in pass 2 is
expected and suppressed by the hold table like any other held package. Keep
`engines.node` and `devEngines.runtime.version` in agreement, and move
`@types/node` only when `node` itself moves.

## Upgrading

The checkup reports; it does not upgrade until the user picks. Once they do,
work in the order below. Each step has its own command and they are **not**
interchangeable — the detection passes above tell you *what* needs doing, these
steps tell you *how*.

### Step 0 — the pnpm config dependency

Do this first, before any other dependency work, so everything that follows
resolves under the current config.

`@obusk/pnpm-plugin-defaults` is a `configDependencies` entry in
`pnpm-workspace.yaml`, pinned to an exact version. It has its own install verb:

```bash
pnpm add --config @obusk/pnpm-plugin-defaults@latest
```

The lockfile records a hash of the resolved pnpmfile:

```yaml
configDependencies:
    "@obusk/pnpm-plugin-defaults":
        ...
pnpmfileChecksum: sha256-Byc4o2K9Q0jEno1uhV9yV0RWnD8+ccaYjKp3HeAMYOE=
```

A plain install against a warm `node_modules` may not re-derive that checksum,
so the version bump lands without the hash moving and CI resolves against the
old config. Follow the bump with:

```bash
pnpm clean && pnpm install
```

Then confirm `pnpmfileChecksum` actually changed in `git diff pnpm-lock.yaml`
before committing. A `configDependencies` bump with an unchanged checksum is the
finding — the commit looks right and does nothing.

The plugin also sets `verifyDepsBeforeRun: 'install'`, so scripts reconcile deps
on the next run regardless; that is a safety net, not a substitute for
committing a correct lockfile.

### Step 1 — in-range updates

Takes what pass 1 found. One command, one commit:

```bash
pnpm update && pnpm install
```

`pnpm update` respects declared ranges, so it cannot cross a hold. Safe as a
batch.

### Step 2 — majors

Takes **pass 2 minus pass 1 minus held**, never pass 2 raw — the raw list
repeats everything Step 1 already handled. One package at a time, never batched,
so a failure is attributable:

```bash
pnpm update <pkg> --latest
```

Verify after each. `--latest` **ignores ranges**, so never point it at a held
package — it will happily jump `typescript` to 7 and `lucide-react` to 1,
defeating the hold.

### Step 3 — a new minor inside a held major

Takes what pass 3 found. Neither command above will do it: `pnpm update` is
blocked by the range and `--latest` overshoots the major. Edit the range in
`package.json` by hand, then install:

```jsonc
"typescript": "~6.1.0"   // was ~6.0.3
```

```bash
pnpm install
```

Its own commit, its own verification — a held package's minor is held precisely
because it is the step most likely to break something.

#### TypeScript 6 and the `types` array

The known blocker for the `typescript` hold. Symptom: the build fails with
global types from a test framework no longer resolving — `describe`, `it`,
`expect` from `@types/jest`, or the equivalent for another runner — in files that
compiled fine on TypeScript 5.

Cause is the `types` compiler option. Left unset, TypeScript decides for itself
which `@types/*` packages to include globally; on TypeScript 6 that no longer
picks up what it used to under pnpm's isolated `node_modules`. Naming them
explicitly fixes it:

```jsonc
// tsconfig.json — only for projects that rely on global types
"types": ["jest", "node"]
```

Two things to get right:

- **`types` is a whitelist, not an addition.** Setting it *excludes* every
  `@types/*` package not listed. A project that did not need it before can be
  broken by adding it, which is why `tsconfig.reference.json` has no `types` key
  and why this is not a blanket recommendation.
- List what the project actually uses. Global-type packages are the test runner
  and `node`; everything imported by name needs no entry.

Reaching TypeScript 6 is a Step 3 hand edit and the `types` change belongs in
the same commit — the build does not pass in between.

#### The `0.x` caret trap

Caret on a `0.x` version locks the **minor**, not the major: `^0.577.0` means
`>=0.577.0 <0.578.0`. It still admits patches, so it behaves exactly like `~`,
not like an exact pin.

For `lucide-react` the practical effect is stronger still, because it publishes
only `0.N.0` releases and no patches — so `^0.577.0` resolves to `0.577.0` alone
in fact, though not by rule. Do not generalise that to other `0.x` packages.

Either way a `0.x` hold behaves as a minor hold whichever range character is
used, and moving 0.577 → 0.578 is a Step 3 hand edit rather than a
`pnpm update`. That is why pass 3 covers every held package, not only the `~`
ones.

### Step 4 — a fresh lockfile

Once every `package.json` edit is done, regenerate the lockfile from scratch:

```bash
pnpm clean -l && pnpm install
```

`pnpm clean` (alias `purge`) removes `node_modules`; `-l` / `--lockfile` also
removes `pnpm-lock.yaml`. Resolving from nothing picks up **subdependency**
updates that an in-place `pnpm install` leaves pinned at their locked versions.

Treat it as its own commit, and be honest in the report about what it is: every
transitive dependency floats to its newest matching version at once, which is a
far wider change than the direct bumps that preceded it. It needs the full
verification below, and if something breaks the cause is likely a transitive you
never named. `minimumReleaseAge: 4320` (3 days, strict) and
`trustPolicy: no-downgrade` come from `@obusk/pnpm-plugin-defaults`, so a fresh
resolve will not pick up a release published in the last three days — that gate
is what makes this safe to do routinely.

### Verification

After any dependency change:

```bash
pnpm install
pnpm lint     # chains postlint → prettier --check
pnpm build
```

`pnpm build` is the one that matters for `typescript` and `@types/node` — a new
TypeScript minor surfaces as build errors, not lint errors. `pnpm lint` already
runs prettier through its `postlint` hook, so no separate prettier call is
needed. Commit `pnpm-lock.yaml` alongside `package.json`; a lockfile left out of
the commit is its own finding on the next run.

Both must pass at every step, with one exception: the window between the
Tailwind v4 upgrade and `@obusk/eslint-config-next@16.3`, where a failing lint
is expected (`tailwind-and-eslint.md`).

If a held package's minor breaks the build, revert the range rather than
chasing the errors inline, and report what broke — deciding whether to absorb
that work is the user's call, not a side effect of a checkup.

## Project-local holds

The table above is the shared baseline, not the whole truth. Before reporting a
major bump, check the project's own `AGENTS.md` or `CLAUDE.md` for a
`## Dependency holds` section and merge those entries in.

When pass 2 turns up a major bump on a package that is neither in the table nor
in the project's file, treat it as a genuine candidate and ask. If the user says
to hold it, offer to record it — project-specific reasons go in the project's
`AGENTS.md`, and a hold that applies everywhere belongs in the table above so
every project inherits it.
