# Crawling policy

Two layers, and they must agree:

- **robots** is the polite signal. Well-behaved crawlers honour it; nothing
  enforces it.
- **the firewall** is enforcement, and lives in `firewall.md`. It acts on
  identity at the edge regardless of what robots says.

Setting one without the other is the most common finding in this checkup.

## Search engines

### Crawling wanted

Allow `*`. Do not add an `ai_bots` firewall rule that also catches search
crawlers — `OAI-SearchBot`, `PerplexityBot` and `Google-Extended` are *search*
surfaces for their platforms, so denying all AI bots does remove you from AI
search results. That is a deliberate trade, not a mistake, but name it when
proposing `deny`.

### Crawling unwanted

`src/app/robots.ts`:

```ts
import type { MetadataRoute } from "next";

export default function robots(): MetadataRoute.Robots {
    return {
        rules: [{ userAgent: "*", disallow: "/" }],
    };
}
```

robots alone will not keep a private site private. If the content genuinely must
not be public, say so and point at Deployment Protection rather than treating
robots as the control.

## AI crawlers

### The user-agent list

The list is **static, committed, and owned by this skill**. The canonical copy is
`ai-crawlers.txt` in this directory; each project carries its own copy in its
robots output. The checkup is the refresh mechanism — diff the project against
the canonical list every run and report drift in both directions.

Do not add a package dependency for this. `@geosuite/ai-crawler-bots` was
evaluated and rejected: as of 0.6.3 it ships 24 entries and is missing
`Claude-SearchBot` and `Claude-User`, so it blocks Anthropic training while
silently allowing the search and user-fetch surfaces. It is a 2-star,
single-maintainer repo with no outside contributors. It remains useful as *one*
cross-check source when refreshing `ai-crawlers.txt`, and nothing more.

This is best-effort by nature. Nothing enforces robots, the list can never be
complete, and `Content-Signal` (below) is the layer that actually covers agents
nobody has named yet. Precision beyond "refresh it when the checkup runs" is not
worth buying.

#### Checking a project against the list

```bash
grep -oE '^[A-Za-z0-9_-]+' ai-crawlers.txt | grep -v '^#' | sort -u > /tmp/canon
grep -oiE 'User-agent: *[A-Za-z0-9_-]+' public/robots.txt | sed 's/.*: *//' |
  sort -u > /tmp/project
comm -23 /tmp/canon /tmp/project   # in canon, missing from project
comm -13 /tmp/canon /tmp/project   # in project, not in canon
```

Missing-from-project is a `fix`. Present-in-project-but-not-canon is a prompt to
**update `ai-crawlers.txt`**, not to delete from the project — the project may
have caught something first.

#### Refreshing the canonical list

Only when the user asks, or when a project turns up an agent the list lacks.
Check operator documentation rather than aggregators, note the date in the file
header, and say in the report which entries moved. Retired tokens
(`anthropic-ai`, `Claude-Web`) stay: a directive for a dead agent costs nothing
and old robots.txt files in the wild still carry them.

### Content signals

`Content-Signal` states policy by **purpose** rather than by agent name, on
`User-agent: *`. That makes it the one part of a robots file that does not rot:
it covers crawlers nobody has heard of yet, which is exactly the gap a
maintained user-agent list cannot close.

| Signal | Means |
| --- | --- |
| `search` | index and return links/excerpts, excluding AI summaries |
| `ai-input` | real-time grounding — RAG, AI search answers |
| `ai-train` | training or fine-tuning |

Match the values to the agent list already in the file, or the two layers state
different policies. A project blocking all three AI categories by user agent
wants `search=yes,ai-input=no,ai-train=no`.

Nothing enforces it. It is a declaration, published by Cloudflare in September
2025 and not adopted by Google's parser, so propose it as a statement of intent
and a machine-readable reservation of rights — never as a control. The firewall
is still the only enforcement.

### Emitting the file

Either mechanism is fine now that the list is static. Do not churn a project
from one to the other; pick per project and leave it.

**`public/robots.txt`** — a plain committed file. Simplest to read, shows the
exact served bytes in a diff, and allows `#` comments if a project ever wants
them.

**`robots.ts` with a literal array** — keeps Next conventions and type checking.
`Content-Signal` goes through `rules[].other`, which passes arbitrary directives
verbatim:

```ts
{
    userAgent: "*",
    other: { "Content-Signal": "search=yes,ai-input=no,ai-train=no" },
    allow: "/",
}
```

Probe before relying on `other` — it is absent in older Next, and a silently
dropped directive looks identical to a working one:

```bash
F=node_modules/next/dist/build/webpack/loaders/metadata/resolve-route-data.js
grep -q "rule.other" "$F" && echo supported
```

**Never both.** A project with `public/robots.txt` *and* `app/robots.ts` is
serving one and silently ignoring the other. Check for the pair and report it —
the dead file will drift, and whichever wins is not obvious from reading the
repo.

Cost is not a factor either way: `robots.ts` prerenders to a static file, so
nothing runs per request and it spends one static route against a budget of
2048.

### Skip the policy preamble

Cloudflare's managed robots.txt prefixes the signals with a ~25-line comment
block defining each signal and asserting a reservation of rights under Article 4
of the EU copyright directive. **These projects do not include it.** Emit the
`Content-Signal` values alone.

The signals are a statement of preference here, not a rights reservation being
relied on, and the block is 25 lines of boilerplate in every robots.txt. Do not
re-propose it on a later run, and do not paraphrase it into a shorter notice —
a partial version carries none of the legal weight and all of the noise. If the
reservation ever does matter commercially for a project, that is a decision to
raise with the user, and the canonical wording gets copied verbatim rather than
summarised.

### "AI crawling" is really three questions

`ai-crawlers.txt` tags every entry with a purpose, and the three want different
answers. Offer the split when the user hesitates on the binary question:

| `purpose` | Examples | What blocking costs you |
| --- | --- | --- |
| `training` | GPTBot, ClaudeBot, CCBot, Google-Extended | Nothing user-facing. The usual default to disallow |
| `search` | OAI-SearchBot, PerplexityBot, Amazonbot | Removes you from AI search results — a real trade |
| `user-agent` | ChatGPT-User, Perplexity-User, DuckAssistBot | Blocks a fetch **a human just asked for** |

Most people who say "block AI crawlers" mean `training` only. Blocking all three
is a coherent choice, but say out loud what the other two cost before applying.

To emit a training-only file, take the entries tagged `training`:

```bash
awk '$2 == "training" { print $1 }' ai-crawlers.txt
```
