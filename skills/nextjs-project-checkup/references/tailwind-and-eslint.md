# Tailwind v4 and the ESLint config

These two are one job, not two: the Tailwind upgrade and the
`@obusk/eslint-config-next@16.3` upgrade have a required order, and lint is
expected to fail in between.

## No Tailwind? Skip the Tailwind half

If no version of `tailwindcss` is present, every Tailwind item here is
inapplicable — not a finding, not a recommendation. Say nothing about it in the
report. The ESLint checks below still apply, minus `cssConfigPath`.

## Order of operations

The order is load-bearing. Doing it backwards leaves lint failing with no
version of the config that can pass.

1. **Tailwind v3 → v4** — `pnpx @tailwindcss/upgrade`
2. **`tailwind-merge`** — bump in the same change; its major tracks Tailwind's
3. **Audit what the codemod got wrong** — two known silent failures, below
4. **Review `globals.css`** — the codemod's output needs a human pass
5. **`@obusk/eslint-config-next@16.3`**
6. **Add `settings.tailwindcss.cssConfigPath`** — required by 16.3

### The purgatory between 1 and 5

**Lint will fail after the Tailwind upgrade and stay failing until
`@obusk/eslint-config-next@16.3` lands. That is expected and accepted.**

Do not try to fix those lint errors, do not revert the upgrade, and do not treat
it as a broken step. Carry on to step 5, which is what resolves them. If the
work stops mid-sequence for any reason, say plainly in the report that the
project is parked in that window and which step gets it out.

This is the **only** point in the whole checkup where a failing lint is
acceptable.

## Tailwind v3 → v4

```bash
pnpx @tailwindcss/upgrade
```

The codemod handles the mechanical work — config to CSS, renamed utilities,
PostCSS plugin swap. Afterwards confirm the end state:

- `tailwindcss` at `^4`
- PostCSS goes through `@tailwindcss/postcss`, not a `tailwindcss` plugin entry
- no leftover `tailwind.config.{js,ts}` holding config that v4 expects in CSS

### What the codemod gets wrong

It does the mechanical work well and gets two things wrong in ways that do not
announce themselves. Check both every time.

**It drops JS plugins without migrating them.** The v3 `plugins: [...]` array in
`tailwind.config.ts` is deleted and no v4 equivalent is emitted. Re-register each
one in CSS:

```css
@import "tailwindcss";
@plugin "@tailwindcss/typography";
```

`@tailwindcss/typography` is still a separate plugin on v4 — core ships no
`.prose` classes at all. What makes this one nasty is that `max-w-prose` *does*
survive, because `--max-width-prose` is a core theme variable. So the content
keeps its correct measure while every heading size, list marker, link colour and
vertical rhythm disappears. It looks laid out, merely unstyled, and the build and
lint both pass. Check `git diff tailwind.config.*` for a `plugins` array before
deleting the file.

**Its template pass renames strings it should not touch.** The v3→v4 utility
renames are applied by string match across source files, so an unrelated string
that happens to equal an old utility name gets rewritten too. Observed:
`"outline"` → `"outline-solid"` inside a TypeScript variant union, breaking the
build while the `cva` variant key it referred to stayed correct.

```ts
variant?: "default" | "outline-solid";   // codemod
variant?: "default" | "outline";         // correct
```

Anything with a variant-name union — `cva`, `tv`, hand-rolled prop unions — is
exposed. Read every non-CSS file the codemod touched and confirm each rename is
genuinely a class name. Real renames in `className` strings
(`bg-gradient-to-r` → `bg-linear-to-r`, `shadow-sm` → `shadow-xs`) are correct;
leave those.

### Verify in a browser

A green build proves nothing here — both bugs above pass typecheck, and the
first passes lint too. Before committing the migration, run the dev server and
look at a page that exercises the plugins, typically a content or MDX route for
`prose`. This is the one step in the checkup that cannot be automated away.

### Review `globals.css`

The codemod is correct but not tidy — it translates faithfully and leaves the
result in whatever shape the old config implied. Read the file afterwards and
report (do not silently restructure):

- duplicated or dead custom properties carried over from the v3 config
- `@theme` entries that restate a Tailwind default
- ordering: `@import "tailwindcss";` first, then `@theme`, then `@utility`, then
  plain CSS
- v3 leftovers the codemod could not translate — any `@tailwind` directive still
  present is the clearest sign

The v4 idiom for reference:

```css
@import "tailwindcss";

@theme {
    --color-ink: #1e2a44;
}

@utility text-balance {
    text-wrap: balance;
}
```

## ESLint config

Reference shape:

```js
import { defineConfig } from "eslint/config";
import nextObusk from "@obusk/eslint-config-next";

const eslintConfig = defineConfig([
    ...nextObusk,
    {
        settings: {
            react: {
                version: "19",
            },
            tailwindcss:
                /** @type {import('eslint-plugin-tailwindcss').PluginSettings} */
                ({
                    cssConfigPath: "./src/app/globals.css",
                }),
        },
    },
]);

export default eslintConfig;
```

Three rules:

1. **Always `defineConfig`**, imported from `eslint/config`. A bare exported
   array works but loses the type checking, so a config without it is a finding
   regardless of whether lint currently passes.
2. **`settings.react.version` is `"19"`** — the major alone. Do not write
   `"19.2"` or `"19.2.18"`; pinning the minor means it silently goes stale on
   every React bump. Only needed when **eslint v10** is installed — on eslint 9
   the setting is unnecessary, and adding it is not an improvement.
3. **`settings.tailwindcss.cssConfigPath`** is required from
   `@obusk/eslint-config-next@16.3` onward, pointing at the CSS entry that has
   the `@import "tailwindcss"` (`./src/app/globals.css`). Before 16.3 it does
   nothing; on 16.3 without it, lint fails. Omit it entirely on projects with no
   Tailwind.

The `@type` comment on the `tailwindcss` settings block is what makes the plugin
settings type-check; keep it when editing.

## Lint and prettier

One command covers both — `lint` chains `postlint`:

```bash
pnpm lint        # eslint, then prettier --check
pnpm lint-fix    # eslint --fix, then prettier --write
```

Do not add a separate `pnpm prettier` call; it already ran. Note prettier's
globs cover `md`, `yml`, `yaml` and `json` only — TS and TSX formatting comes
through eslint, so a clean prettier run says nothing about the source files.

`pnpm lint` must pass before every commit and, above all, at the very end of the
checkup — the one exception being the purgatory window described above.
