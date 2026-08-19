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

Maintaining a hand-written list rots — operators split, rename and retire agents
constantly (`anthropic-ai` and `Claude-Web` gave way to `ClaudeBot`;
`OAI-SearchBot` and `ChatGPT-User` split off from `GPTBot`). Source it from
[`@geosuite/ai-crawler-bots`](https://npmx.dev/package-code/@geosuite/ai-crawler-bots),
a curated list where each entry links to the operator's own documentation.

It ships the data as a JSON export, so read the agents at build time rather than
pasting them:

```bash
pnpm add -D @geosuite/ai-crawler-bots
```

```ts
import type { MetadataRoute } from "next";
import bots from "@geosuite/ai-crawler-bots/bots.json";

const aiCrawlers = bots.map((bot) => bot.uaToken ?? bot.name);

export default function robots(): MetadataRoute.Robots {
    return {
        rules: [
            { userAgent: "*", allow: "/" },
            { userAgent: aiCrawlers, disallow: "/" },
        ],
    };
}
```

`uaToken ?? bot.name` is deliberate, not defensive. `Google-Extended` and
`Applebot-Extended` carry `uaToken: null` because they are policy-only tokens
with no separate HTTP user-agent — they exist *only* as robots.txt directives.
Mapping over `uaToken` alone silently drops both, which are the two entries that
opt you out of Gemini and Apple Intelligence training. Verify the shape against
the installed version (as of 0.6.3: `id`, `name`, `ua`, `uaToken`, `owner`,
`purpose`, `docsUrl`, `robotsDirective`, `notes`; 24 entries).

The package also has a CLI worth running once against the deployed site:

```bash
pnpx @geosuite/ai-crawler-bots robots https://example.com
```

It fetches the live `robots.txt` and scores it per bot, which is the fastest way
to confirm the deployed result matches the intent.

### A hand-maintained array is a finding

A hardcoded `const aiCrawlers = [...]` still works, so nothing is failing today
— but it was accurate on the day it was written and has been decaying since.
Report it as `fix` and propose the package. Do not leave it alone on the grounds
that it is not broken; going stale silently is the whole problem.

**Diff before replacing.** The package is not a superset of what a given project
already lists — hand-written lists accumulate agents the package has not picked
up, and dropping them is a real regression:

```bash
npm pack @geosuite/ai-crawler-bots && tar xzf geosuite-ai-crawler-bots-*.tgz
node -e 'const b=require("./package/bots.json").map(x=>(x.uaToken??x.name).toLowerCase());
const hard=require("./repo-list.json");
console.log("only in repo:", hard.filter(s=>!b.includes(s.toLowerCase())));
console.log("only in pkg :", b.filter(s=>!hard.map(h=>h.toLowerCase()).includes(s)));'
```

Take the **union**, so the package drives the maintained core and the extras
survive:

```ts
import bots from "@geosuite/ai-crawler-bots/bots.json";

const extras = ["Claude-SearchBot", "Claude-User", "Ai2Bot"];
const aiCrawlers = [
    ...new Set([...bots.map((bot) => bot.uaToken ?? bot.name), ...extras]),
];
```

Keep `extras` short and say in the report what went into it and why, so the next
run can retire an entry once the package covers it. Retired tokens are not worth
pruning by hand — the package still ships `anthropic-ai` and `Claude-Web`, and a
robots directive for a dead agent costs nothing.

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

### Choosing the robots output mechanism

Three ways to emit the file. Pick the highest rung that covers what the project
needs, and do not move a project down a rung without a reason.

**1. `robots.ts` with `other`** — the default. Arbitrary directives pass through
verbatim, so `Content-Signal` needs no static file:

```ts
{
    userAgent: "*",
    other: { "Content-Signal": "search=yes,ai-input=no,ai-train=no" },
    allow: "/",
}
```

Probe before relying on it — `other` is not in older Next releases, and a
silently dropped directive looks identical to a working one:

```bash
grep -q "rule.other"   node_modules/next/dist/build/webpack/loaders/metadata/resolve-route-data.js   && echo supported
```

**2. `app/robots.txt/route.ts`** — when the file needs `#` comment lines. The
serializer emits `Key: value` and nothing else, so a policy preamble is
impossible from `robots.ts`. A route handler returning `text/plain` keeps the
agent list generated from the package while controlling every byte. Next treats
`/robots.txt` as dynamic-capable, so this does not conflict with metadata
routing — but build once and confirm it landed in the **prerender** manifest. A
route handler that does not prerender is a function invocation on every
`robots.txt` request, which is the one real cost difference on this ladder.

**3. `public/robots.txt`** — the agent list becomes hand-maintained again, which
is the problem the package exists to solve. Reach for it only when there is no
build-time access to the list.

### What the ladder is and is not about

It ranks by **maintenance**, not cost. All three are effectively free:
`robots.ts` prerenders to a static file — confirm with

```bash
node -e 'console.log(Object.keys(require("./.next/prerender-manifest.json").routes))'   | grep robots
```

— so nothing runs per request, and it registers one static route against a
budget of 2048. `public/` registers none. That difference is not a reason to
choose anything.

Do not argue for rung 3 on cost grounds. Report a project sitting there with a
hardcoded list as a `fix`; a project on rung 2 for the preamble is a deliberate
choice, not drift.

### The policy preamble

Cloudflare's managed robots.txt prefixes the signals with a ~25-line comment
block defining each signal and asserting a reservation of rights under Article 4
of the EU copyright directive. Including it is a judgement call, not a default:

- **Include it** when the site is EU-based and the reservation matters
  commercially. Article 4(3)'s opt-out wants machine-readable reservation, and
  bare signal values carry that weight only by reference to a policy the crawler
  has to go and find.
- **Skip it** when the signals are a statement of preference. It is 25 lines of
  boilerplate on every request for a file most crawlers read mechanically.

If included, copy the canonical wording verbatim rather than paraphrasing, and
note the date it was taken so a later checkup can diff it. An abbreviation
invites argument about whether it is an express reservation; matching the
widely-deployed text does not. Flag to the user that this is the one item in the
checkup with a legal dimension, and that a paraphrase is worth a lawyer's eye if
the answer matters.

### "AI crawling" is really three questions

`bots.json` carries a `purpose` field, and the three values want different
answers. Offer the split when the user hesitates on the binary question:

| `purpose` | Examples | What blocking costs you |
| --- | --- | --- |
| `training` | GPTBot, ClaudeBot, CCBot, Google-Extended | Nothing user-facing. The usual default to disallow |
| `search` | OAI-SearchBot, PerplexityBot, Amazonbot | Removes you from AI search results — a real trade |
| `user-agent` | ChatGPT-User, Perplexity-User, DuckAssistBot | Blocks a fetch **a human just asked for** |

Most people who say "block AI crawlers" mean `training` only. Blocking all three
is a coherent choice, but say out loud what the other two cost before applying.

```ts
const aiCrawlers = bots
    .filter((bot) => bot.purpose === "training")
    .map((bot) => bot.uaToken ?? bot.name);
```

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
