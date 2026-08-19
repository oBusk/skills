# Firewall policy

Enforcement at the edge. Payloads for every write are in `vercel-api.md`; this
file is when to reach for each and what it costs. Crawl policy — robots,
`Content-Signal`, the agent list — is in `crawling.md`.

Every write here changes production the moment it is sent. Confirm before
writing, not after.

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

## Let crawlers reach `robots.txt`

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
Budget: one rule. Verify:

```bash
curl -sS -o /dev/null -w '%{http_code}
' -A "GPTBot/1.1" https://example.com/robots.txt
```

Report a project with `ai_bots: deny` and no such bypass as a `fix`. It is the
most common way this whole configuration ends up incoherent.

## AI Bots ruleset

Set the `ai_bots` managed ruleset to `deny` (see `vercel-api.md`). Vercel keeps
that list current itself, so new crawlers inherit the action with no change on
your side — which is exactly the part a hand-written robots list cannot do.

The ruleset is all-or-nothing. Vercel documents it as covering AI bots that
crawl "for training data, search purposes, or user-generated fetches", with Log
and Deny as the only actions — so `deny` takes all three categories or none.
That is why the edge decision is taken *first* in Step 1: once it is `deny`, the
finer split in robots would be a claim the enforcement layer contradicts.

Enforcing the split at the edge means leaving `ai_bots` off and writing custom
rules against the `bot_name` condition — but treat that as a maybe, not a plan:

- `bot_name` is not available on every plan. Confirm it is offered for this
  project before proposing rules built on it, rather than assuming the condition
  type exists because it appears in the API schema.
- Even where available it is several rules, against a budget that is three on
  the smallest plan.

So on most projects the honest answer is that the split can be **requested** in
robots but not **enforced** at the edge. Say that plainly instead of proposing
rules the project cannot create.

## Deny `.php` probes

### A WAF custom rule

Use `rules.insert` from `vercel-api.md` — the payload there is already this
rule, matching `path` with `op: "suf"` on `.php`. Keeping it in the firewall
puts it alongside Bot Protection and the AI Bots ruleset, so the whole edge
policy reads from one place, and it takes effect at once rather than waiting for
the next deploy.

It costs one rule. Count the existing `rules` first and say what it will
consume.

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

Verify straight after the write — there is no publish step to wait for.

### Widening it

`{"type":"path","op":"suf","value":".php"}` catches `/wp-login.php` and
`/xmlrpc.php`, the overwhelming majority of the noise. Only widen if the logs
justify it:

- `op: "inc"` also catches `/index.php/foo`, at some risk of a false positive.
- Scanners also probe `.env`, `.git/config`, `.bak`, `.sql`, `/wp-admin`.
  Bundling them into one OR-group rule is more budget-efficient than one rule
  each — but propose it from observed traffic, not speculatively.

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
- **It burns the only rate-limit slot below Pro**, which then cannot be spent
  on the endpoint that genuinely needs it later.

Bot Protection and the AI Bots ruleset already address aggressive crawling by
identity, which is more precise than by volume.

The payload and the three-`action`-field trap are in `vercel-api.md`.

Vercel's advice, worth following: start in `log`, watch Firewall Observability
for a few days, then move to `deny` or `challenge`.

`actionDuration` (a persistent block after the limit trips, e.g. `"15m"`) is
deliberately left out of the baseline. It is a separately gated feature, so it
is not safe to assume on an arbitrary project — add it only after confirming the
plan supports it, and expect the write to be rejected rather than silently
ignored if it does not.
