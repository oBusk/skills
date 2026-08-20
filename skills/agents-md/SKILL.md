---
name: agents-md
description: Create or update a repository's AGENTS.md and CLAUDE.md — the Next.js managed block, the CLAUDE.md @AGENTS.md reference, and the shared pnpm and code-comment sections. Use when asked to set up, add, fix, or refresh AGENTS.md or CLAUDE.md, or when a repo has no agent instructions.
---

# AGENTS.md

Builds the agent instruction files for a repo. Everything here is additive:
never delete a section you did not just write, and never rewrite prose the user
authored.

Both files are **committed**. Untracked or gitignored is the defect this skill
exists to fix — `next dev` regenerates its block every run, so an ignored file
shows as dirty forever, which is what prompts someone to ignore it again.

## Step 0 — What kind of repo is this?

```bash
test -f package.json && node -p "Object.keys({...require('./package.json').dependencies,...require('./package.json').devDependencies}).includes('next')"
test -f pnpm-lock.yaml && echo pnpm
ls -la AGENTS.md CLAUDE.md 2>/dev/null
```

Each step below is gated on one of these. A repo that is neither Next nor pnpm
still gets Step 3 and Step 5.

## Step 1 — The Next.js managed block

Next projects only. Skip entirely otherwise — the block tells agents to read
`node_modules/next/dist/docs/`, which is nonsense in a repo without Next.

```bash
grep -q "BEGIN:nextjs-agent-rules" AGENTS.md 2>/dev/null && echo present
```

Present → nothing to do. Absent → **let Next write it**, do not hand-copy it.
From Next 16.3 `next dev` generates the block when it detects an agent in the
environment, which is exactly the situation you are in:

```bash
pnpm exec next dev &
DEV=$!
for i in $(seq 1 30); do
  grep -q "BEGIN:nextjs-agent-rules" AGENTS.md 2>/dev/null && break
  sleep 1
done
kill $DEV
grep -q "BEGIN:nextjs-agent-rules" AGENTS.md && echo generated || echo "not generated"
```

`next dev` does not exit on its own — start it in the background, poll for the
marker, and kill it. Leaving it running blocks the rest of the session.

If the marker never appears, the agent detection did not fire. **Do not
improvise the block.** Its text is version-matched to the installed Next, so a
copy pasted from anywhere else goes stale silently. Report that it could not be
generated and let the user run `pnpm dev` in their own terminal once.

The block is Next's to own. Never edit inside the markers — `next dev` upserts
it on every run, replacing whatever is between them, and content outside is
preserved. That is also why a regenerated block appearing mid-task belongs in
the commit rather than in a report.

## Step 2 — `CLAUDE.md` references `AGENTS.md`

`CLAUDE.md` should be a pointer, not a copy, and the pointer is an `@` import:

```md
@AGENTS.md
```

This matches what Next writes when it creates the file, so the two do not fight.

**Replace symlinks.** A `CLAUDE.md` symlinked to `AGENTS.md` works on one
machine and is a known source of trouble across platforms and in Git. Check and
convert:

```bash
test -L CLAUDE.md && echo "symlink — replace"
```

Replacing it is a delete plus a write, so say so before doing it. Any prose
already in `CLAUDE.md` that is not a copy of `AGENTS.md` moves into `AGENTS.md`
first — it is content someone wrote, and Claude is not the only agent that
should see it.

## Step 3 — The shared sections

Both live in `references/blocks.md`, verbatim, and go **outside** the Next
markers. Add each only when the repo warrants it.

- **`## pnpm`** — when `pnpm-lock.yaml` exists. Behaviour that post-dates most
  training data, plus the local-binary rule.
- **`## Code comments`** — always.

If a section already exists, diff it against `references/blocks.md` and report
drift rather than overwriting. A project line the canonical copy lacks means
someone learned something on that repo: propose adding it to this skill instead
of deleting it there.

## Step 4 — Commit

One commit, both files. Then confirm they are tracked and not ignored:

```bash
git ls-files --error-unmatch AGENTS.md CLAUDE.md
grep -nE "AGENTS\.md|CLAUDE\.md" .gitignore
```

A hit in `.gitignore` is a finding to fix, not to work around.

## Step 5 — Offer a repo review

The sections above are the shared baseline. What makes an `AGENTS.md` earn its
context is the project-specific half: what the project *is*, the commands that
actually exist, and conventions a newcomer would get wrong.

Offer it, do not assume it — it is a much larger job than the rest of this
skill, and the user may only have wanted the baseline:

> Want me to review the repo and fill in the project-specific sections —
> Project, Commands, Architecture, and any conventions worth writing down?

If they accept, read the repo rather than guessing: `package.json` scripts for
Commands, the directory layout and entry points for Architecture, and existing
config for conventions. Write only what you verified. An `AGENTS.md` that
describes a script which does not exist is worse than one that omits it.
