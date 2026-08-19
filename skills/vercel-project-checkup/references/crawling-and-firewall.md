# Crawling and firewall recipes

Two layers, and they must agree:

- **robots** is the polite signal. Well-behaved crawlers honour it; nothing
  enforces it.
- **the firewall** is enforcement. It acts on identity at the edge regardless of
  what robots says.

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
grep -oE '^[A-Za-z0-9_-]+' references/ai-crawlers.txt | grep -v '^#' | sort -u > /tmp/canon
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
awk '$2 == "training" { print $1 }' references/ai-crawlers.txt
```

### Let crawlers reach `robots.txt`

Managed rulesets have **no path scoping**, so `ai_bots: deny` returns 403 on
`/robots.txt` too. Verified on a live project with both rulesets active:

```
GPTBot     robots.txt=403
ClaudeBot  robots.txt=403
```

The crawler cannot read the file that tells it not to crawl. The outcome is the
same while the ruleset stays on — they are blocked either way — but the
declaration layer is dead: the policy is unreachable, `Content-Signal` is
unreadable, operators who check robots from a different agent than their crawler
see a 403, and the moment the ruleset is turned off there is nothing that was
ever communicated.

Fix with a bypass custom rule. Custom rules run **before** managed rulesets, so
it takes effect over both:

```bash
cat <<'JSON' | pnpx vercel api "/v1/security/firewall/config?projectId=$PRJ&teamId=$TEAM" -X PATCH --input -
{
  "action": "rules.insert",
  "value": {
    "name": "Allow robots.txt",
    "active": true,
    "conditionGroup": [
      { "conditions": [{ "type": "path", "op": "eq", "value": "/robots.txt" }] }
    ],
    "action": { "mitigate": { "action": "bypass" } }
  }
}
JSON
```

Check it applies **before** the deny rules; `rules.priority` reorders if not.
Budget: this plus the `.php` deny is 2 of the 3 rules on Hobby. Verify after
publishing:

```bash
curl -sS -o /dev/null -w '%{http_code}
' -A "GPTBot/1.1" https://example.com/robots.txt
```

Report a project with `ai_bots: deny` and no such bypass as a `fix`. It is the
most common way this whole configuration ends up incoherent.

### Enforcement

Set the `ai_bots` managed ruleset to `deny` (see `vercel-api.md`). Vercel keeps
that list current itself, so new crawlers inherit the action with no change on
your side — which is exactly the part a hand-written robots list cannot do.

The ruleset is all-or-nothing: it cannot express the `training`-only split
above. Vercel documents it as covering AI bots that crawl "for training data,
search purposes, or user-generated fetches", with Log and Deny as the only
actions — so `deny` takes all three categories or none. Enforcing that split means leaving `ai_bots` off and writing custom rules
against the `bot_name` condition — but treat that as a maybe, not a plan:

- `bot_name` is not available on every plan. Confirm it is offered for this
  project before proposing rules built on it, rather than assuming the condition
  type exists because it appears in the API schema.
- Even where available it is several rules, against a budget of three on Hobby.

So on most projects the honest answer is that the split can be **requested** in
robots but not **enforced** at the edge. Say that plainly instead of proposing
rules the project cannot create.

## Deny `.php` probes

### A WAF custom rule

Use `rules.insert` from `vercel-api.md` — the payload there is already this
rule, matching `path` with `op: "suf"` on `.php`. Keeping it in the firewall
puts it alongside Bot Protection and the AI Bots ruleset, so the whole edge
policy reads from one place, and it takes effect on publish rather than waiting
for the next deploy.

It costs one rule against the plan budget — three on Hobby. Count the existing
`rules` before proposing it and say what it will consume.

### Why not `vercel.json`

`routes[].mitigate` expresses the same deny, is version-controlled, and applies
to previews. It also draws on a different budget — "Routes created per
Deployment", 2048 on Hobby and Pro — rather than the three WAF custom rules. That
pool is shared with every route the framework generates (dynamic segments,
`rewrites`, `redirects`, `headers`), so it is not free, just far from binding.

It is still not the default here: `routes` is the **legacy** routing property and
cannot coexist with `rewrites`, `redirects`, `headers`, `cleanUrls` or
`trailingSlash`. The JSON schema does not encode that constraint, so `$schema`
autocomplete stays silent and the deployment fails only when it is pushed.

Reach for it only when the custom-rule budget is genuinely full, and check for
those keys first:

```bash
node -e 'const c=require("./vercel.json");console.log(["rewrites","redirects","headers","cleanUrls","trailingSlash"].filter(k=>k in c))' 2>/dev/null
```

Anything listed rules it out. Never migrate existing routing into `routes` to
make room — that is a large, risky change to land as a side effect of a
scanner-noise cleanup.

### Verify

```bash
curl -s -o /dev/null -w '%{http_code}\n' https://example.com/wp-login.php
```

Remember the rule is a **draft** until published, so verify after publishing,
not after the write.

### Widening it

`{"type":"path","op":"suf","value":".php"}` catches `/wp-login.php` and
`/xmlrpc.php`, the overwhelming majority of the noise. Only widen if the logs
justify it:

- `op: "inc"` also catches `/index.php/foo`, at some risk of a false positive.
- Scanners also probe `.env`, `.git/config`, `.bak`, `.sql`, `/wp-admin`.
  Bundling them into one OR-group rule is more budget-efficient than one rule
  each — but propose it from observed traffic, not speculatively.

## Plan budgets

| Resource | Hobby | Pro | Enterprise |
| --- | --- | --- | --- |
| Custom firewall rules | 3 per project | 40 | 1000 |
| Project-level IP blocks | 3 | 100 | 1000 |
| `vercel.json` routes | 2048 per deployment | 2048 | custom |
| Rate-limit rules | 1 per project | 40 | 1000 |
| Rate-limit keys | IP, JA4 | IP, JA4 | + User Agent, arbitrary headers |
| Rate-limit algorithm | Fixed window | Fixed window | + Token bucket |
| Rate-limit window | 10s – 10min | 10s – 10min | 10s – 1hr |

Custom-rule and IP-block figures come from the limits table on
`/docs/vercel-firewall/vercel-waf`. The routes figure is a **deployment** budget,
not a firewall one, and it counts framework-generated routes too.

Count `rules` in the firewall config before proposing anything, and say what a
proposal will consume. On Hobby, three rules go fast.

## Rate limiting

Rate limiting is a cost- and abuse-control tool for specific endpoints, not a
crawler defence. Five reasons it fails at the latter:

- **Counters are per-region.** Vercel tracks the key per region, so the effective
  limit is your number multiplied by the regions serving the traffic.
  Distributed crawlers — the aggressive ones — are precisely what this misses.
- **Fixed window** is the only algorithm below Enterprise, so a burst straddling
  a window boundary passes roughly double the limit.
- **Static pages are CDN-cached** and often never reach a function, so the rule
  guards little on a marketing site.
- **Shared IPs** (CGNAT, corporate and campus NAT) put many real users behind one
  key, so false positives land on real traffic.
- **On Hobby it burns the only rate-limit slot**, which then cannot be spent on
  the endpoint that genuinely needs it later.

Bot Protection and the AI Bots ruleset already address aggressive crawling by
identity, which is more precise than by volume.

When it *is* warranted, scope it to the path:

```bash
cat <<'JSON' | pnpx vercel api "/v1/security/firewall/config?projectId=$PRJ&teamId=$TEAM" -X PATCH --input -
{
  "action": "rules.insert",
  "value": {
    "name": "Rate limit login",
    "active": true,
    "conditionGroup": [
      { "conditions": [{ "type": "path", "op": "pre", "value": "/api/auth" }] }
    ],
    "action": {
      "mitigate": {
        "action": "rate_limit",
        "rateLimit": {
          "algo": "fixed_window",
          "window": 60,
          "limit": 10,
          "keys": ["ip"],
          "action": "deny"
        }
      }
    }
  }
}
JSON
```

**Three different `action` fields appear in that payload.** They are not
interchangeable:

| Field | Value | Means |
| --- | --- | --- |
| top-level `action` | `rules.insert` | the API verb |
| `mitigate.action` | `rate_limit` | this rule rate-limits — never change it |
| `rateLimit.action` | `deny` | what happens once the limit is exceeded |

Vercel's advice, worth following: start with **`rateLimit.action` set to
`log`**, watch Firewall Observability for a few days, then move it to `deny` or
`challenge`. Only that field changes; `mitigate.action` stays `rate_limit`
throughout, and editing the wrong one produces an invalid rule.

`actionDuration` (a persistent block after the limit trips, e.g. `"15m"`) is
deliberately left out of the baseline. It is a separately gated feature, so it
is not safe to assume on an arbitrary project — add it only after confirming the
plan supports it, and expect the write to be rejected rather than silently
ignored if it does not.
