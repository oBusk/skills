# `vercel api` recipes

All commands verified against Vercel CLI 59.1.4. Run from the project root so
the CLI picks up the link, and use `pnpx vercel`.

`vercel api` is an authenticated passthrough to `api.vercel.com`. It is marked
beta but is the only path that reads and writes the settings this checkup needs.

Set these once per run:

```bash
PRJ=prj_xxxxxxxxxxxxxxxxxxxxxxxx   # .vercel/project.json or .vercel/repo.json
TEAM=team_xxxxxxxxxxxxxxxxxxxxxx   # orgId in the same file
```

## Reads

### Full firewall config

```bash
pnpx vercel api "/v1/security/firewall/config/active?projectId=$PRJ&teamId=$TEAM" --raw
```

Returns `managedRules` (`bot_protection`, `ai_bots`, `owasp`, `vercel_ruleset`),
`crs` (the OWASP sub-rules), `rules` (custom rules — count these against the
plan budget), `ips`, and `changes` (edits staged in the dashboard, if any).

`/v1/security/firewall/config` (without `/active`) returns active plus draft.

### Discovering endpoints

```bash
pnpx vercel api list | grep -i firewall
```

## Writes

Preview a mutation before running it. Build the preview from the **complete**
payload, including the nested `value` — `-f` flags cannot express it, so a
flag-built preview shows a different request than the one that gets sent:

```bash
echo '{"action":"managedRules.update","id":"bot_protection","value":{"active":true,"action":"challenge"}}' | pnpx vercel api "/v1/security/firewall/config?projectId=$PRJ&teamId=$TEAM" -X PATCH --input - --generate=curl
```

`-f` / `-F` only build flat bodies. Anything with a nested `value` object must
come from stdin via `--input -`.

### Bot Protection → Challenge

```bash
echo '{"action":"managedRules.update","id":"bot_protection","value":{"active":true,"action":"challenge"}}' \
| pnpx vercel api "/v1/security/firewall/config?projectId=$PRJ&teamId=$TEAM" -X PATCH --input -
```

`action` accepts `log` or `challenge` for `bot_protection`.

### AI Bots → Deny

```bash
echo '{"action":"managedRules.update","id":"ai_bots","value":{"active":true,"action":"deny"}}' \
| pnpx vercel api "/v1/security/firewall/config?projectId=$PRJ&teamId=$TEAM" -X PATCH --input -
```

`action` accepts `log` or `deny` for `ai_bots`. To turn a ruleset off entirely,
send `{"active":false}`.

### Insert a custom rule

```bash
cat <<'JSON' | pnpx vercel api "/v1/security/firewall/config?projectId=$PRJ&teamId=$TEAM" -X PATCH --input -
{
  "action": "rules.insert",
  "value": {
    "name": "Deny .php probes",
    "active": true,
    "conditionGroup": [
      { "conditions": [{ "type": "path", "op": "suf", "value": ".php" }] }
    ],
    "action": { "mitigate": { "action": "deny" } }
  }
}
JSON
```

Other write actions follow the same envelope: `rules.update` (with `id`),
`rules.remove`, `ips.insert`, `managedRules.update`.

### Insert a rate-limit rule

**Starts in `log`.** Rate-limit rules go live on write with no staging step, so a
first rule that denies can block real clients before anyone sees a graph. Observe
first, then tighten.

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
          "action": "log"
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
| `rateLimit.action` | `log` | what happens once the limit is exceeded |

Watch Firewall Observability for a few days, then move **only**
`rateLimit.action` to `deny` or `challenge` via `rules.update`. `mitigate.action`
stays `rate_limit` throughout; editing that one instead produces an invalid rule.

## Condition reference

Confirmed from the published OpenAPI spec.

**Operators:** `eq`, `neq`, `ex`, `nex`, `pre` (prefix), `suf` (suffix),
`sub` (substring), `inc`, `ninc`, `re` (regex), `gt`, `gte`, `lt`, `lte`, `list`.
All case-insensitive.

**Types:** `host`, `path`, `raw_path`, `target_path`, `route`, `method`,
`header`, `query`, `cookie`, `ip_address`, `region`, `protocol`, `scheme`,
`environment`, `user_agent`, `geo_continent`, `geo_country`,
`geo_country_region`, `geo_city`, `geo_as_number`, `ja3_digest`, `ja4_digest`,
`bot_name`, `bot_category`, `bot_status`, `bot_protection`, `server_action`,
`rate_limit_api_id`, `ruleset`, `shared_condition`.

`path` excludes the query string; `raw_path` includes it. Use `path` with `suf`
for extension matching.

## Writes are live immediately

**A `PATCH` to `/v1/security/firewall/config` takes effect at once.** There is no
staging step and nothing to publish. Verified on a live project: inserting a rule
moved `/config` and `/config/active` to the same new version together, with
`changes: []`, and the rule was enforcing on the next request.

The draft-and-publish flow people describe is the **dashboard**, which batches
edits behind *Review Changes → Publish*. That does not apply to the API, and
`vercel firewall publish` publishes dashboard-staged edits, not yours.

So the confirmation has to come **before the write**. Do not tell the user a
change is staged, and do not offer to publish it later — by the time you would
ask, production has already changed. Re-read `/config/active` afterwards to
confirm what landed.

`changes` on `/config` reflects edits staged in the dashboard. Empty is the
normal state for API-only work; a non-empty `changes` means someone has unapplied
edits in the UI, which is worth surfacing before writing over them.

## Known breakage

- `vercel firewall overview` → `IP Bypass is unavailable for this plan (404)` on
  plans without IP Bypass, and the error takes the whole readout with it. Read
  the config through `vercel api` instead.
- `npx vercel` can fail on `EBADDEVENGINES` in repos that set `devEngines`.
  Use `pnpx vercel`.
