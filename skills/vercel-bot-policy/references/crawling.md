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

Nothing enforces it. It is a declaration, published by Cloudflare in September
2025 and not adopted by Google's parser, so propose it as a statement of intent
and a machine-readable reservation of rights — never as a control. The firewall
is still the only enforcement.

### Derive the list from the signals

**The signals are the decision; the agent list is generated from them.** Do not
maintain the two independently and try to keep them agreeing — three yes/no
answers are something a person can hold in their head, thirty user-agent strings
are not.

The vocabularies do not line up, so translate first. `Content-Signal`'s `search`
means **conventional indexing and explicitly excludes AI-generated summaries**,
which is the opposite of the `ai-search` tag in `ai-crawlers.txt`:

| Signal set to `no` | Purposes to block |
| --- | --- |
| `ai-train=no` | `training` |
| `ai-input=no` | `ai-search`, `user-agent` |
| `search=no` | none — that is `Disallow: /` for `*`, not an agent list |

Then generate:

```bash
BLOCK='training|ai-search|user-agent'   # ai-train=no and ai-input=no
BLOCK='training'                        # ai-train=no only, the common case

awk -v p="^($BLOCK)$" '$1 !~ /^#/ && $2 ~ p { print "    \"" $1 "\"," }' ai-crawlers.txt
```

Diff the output against the project's array. **Any difference is the finding**,
and the direction tells you which:

- In the generated list, missing from the project → the project under-enforces
  its own declaration. The usual case, and a `fix`.
- In the project, not in the generated list → either the project caught an agent
  before `ai-crawlers.txt` did, or it is blocking something its signals do not
  ask it to block. Check the agent's operator documentation before deciding
  which; `PetalBot` sat in this file until it turned out to be Huawei's Petal
  Search crawler rather than an AI agent (their AI trainer is `PanguBot`).

This also removes a false positive worth naming: blocking `OAI-SearchBot`
alongside `search=yes` looks contradictory and is not — it is `ai-input=no`
doing its job. Never report that as a conflict.

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

**Never both.** The dead one drifts, and which wins is not obvious from reading
the repo.

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

### What blocking each purpose costs

`ai-crawlers.txt` tags every entry with a purpose. The three carry very
different costs, and `Content-Signal` gives only two knobs to control them —
`ai-train` covers the first row, `ai-input` covers the other two. Use this to
name the cost when asking:

| `purpose` | Examples | What blocking costs you |
| --- | --- | --- |
| `training` | GPTBot, ClaudeBot, CCBot, Google-Extended | Nothing user-facing. The usual default to disallow |
| `ai-search` | OAI-SearchBot, PerplexityBot, Amazonbot | Removes you from AI answer engines — a real trade |
| `user-agent` | ChatGPT-User, Perplexity-User, DuckAssistBot | Blocks a fetch **a human just asked for** |

Most people who say "block AI crawlers" mean `training` only. Blocking all three
is a coherent choice — and the only coherent one once `ai_bots` is set to `deny`,
since the ruleset cannot express anything narrower.

To emit a training-only file, take the entries tagged `training`:

```bash
awk '$2 == "training" { print $1 }' ai-crawlers.txt
```
