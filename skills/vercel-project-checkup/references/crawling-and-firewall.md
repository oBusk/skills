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

A project already carrying a hand-maintained array is not broken — replacing it
with the package is a maintenance improvement, so propose it as one rather than
as a fix.

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
above. If the user wants that split enforced rather than merely requested, leave
`ai_bots` off and write custom rules against the `bot_name` condition type
instead — then check it against the plan's custom-rule budget, since that is
several rules on a plan that allows three.

## Deny `.php` probes

### Preferred: `vercel.json`

Version-controlled, reviewable, applies to previews, no publish step. Schema
confirmed: `routes[].mitigate.action` accepts only `challenge` or `deny`.

```json
{
    "$schema": "https://openapi.vercel.sh/vercel.json",
    "routes": [
        {
            "src": "^/.*\\.php$",
            "mitigate": { "action": "deny" }
        }
    ]
}
```

Takes effect on the next deploy. Verify afterwards:

```bash
curl -s -o /dev/null -w '%{http_code}\n' https://example.com/wp-login.php
```

### Alternative: WAF custom rule

When it must take effect without a deploy — the `rules.insert` call in
`vercel-api.md`. Counts against the plan's custom-rule budget; the `vercel.json`
form is deployment config rather than WAF config, so it is the cheaper choice on
Hobby. Verify the count either way after applying.

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
| Custom firewall rules | 3 per project | — | — |
| Rate-limit rules | 1 per project | 40 | 1000 |
| Rate-limit keys | IP, JA4 | IP, JA4 | + User Agent, arbitrary headers |
| Rate-limit algorithm | Fixed window | Fixed window | + Token bucket |
| Rate-limit window | 10s – 10min | 10s – 10min | 10s – 1hr |

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
        },
        "actionDuration": "15m"
      }
    }
  }
}
JSON
```

Vercel's own advice, worth following: create it with `"action": "log"` first,
watch Firewall Observability for a few days, then switch to `deny` or
`challenge` once the traffic shape is known.
