# skills

Personal collection of agent skills, installable with [`pnpx skills`](https://npmx.dev/package/skills).

## Install

```bash
# Browse and pick a skill interactively
pnpx skills add oBusk/skills

# Install a specific skill directly
pnpx skills add oBusk/skills --skill vercel-project-checkup
```

## Skills

| Skill | What it does |
| --- | --- |
| [`vercel-project-checkup`](skills/vercel-project-checkup/) | Hygiene checkup for a Next.js project on Vercel — deps, audit, Tailwind v4, pnpm toolchain, Web Analytics, Fluid Compute, and crawling/firewall policy |

## Structure

Each skill lives in its own directory under `skills/`, with a `SKILL.md`
containing YAML frontmatter (`name`, `description`) and instructions in the
body.

```
skills/
  example-skill/
    SKILL.md
```

## Adding a skill

1. Create `skills/<skill-name>/SKILL.md`.
2. Give it a specific `description` — it's what discovery matches against.
3. Delete `skills/example-skill/` once you have real skills in place.
