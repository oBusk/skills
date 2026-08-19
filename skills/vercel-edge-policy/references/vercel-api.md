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
plan budget), `ips`, and `changes` (unpublished draft edits).

`/v1/security/firewall/config` (without `/active`) returns active plus draft.

### Discovering endpoints

```bash
pnpx vercel api list | grep -i firewall
```

## Writes

Preview any mutation before running it:

```bash
pnpx vercel api "/v1/security/firewall/config?projectId=$PRJ&teamId=$TEAM" \
  -X PATCH -f action=managedRules.update -f id=bot_protection --generate=curl
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

## Draft vs published

Firewall writes land in a **draft**. They are not live until published.

```bash
pnpx vercel api "/v1/security/firewall/config?projectId=$PRJ&teamId=$TEAM" --raw   # inspect `changes`
```

Confirm with the user before publishing, and prefer `pnpx vercel firewall
publish` for that one step — it is the rung-3 command that does work, since it
does not touch the IP Bypass endpoint that breaks `firewall overview`.

## Known breakage

- `vercel firewall overview` → `IP Bypass is unavailable for this plan (404)` on
  plans without IP Bypass, and the error takes the whole readout with it. Read
  the config through `vercel api` instead.
- `npx vercel` can fail on `EBADDEVENGINES` in repos that set `devEngines`.
  Use `pnpx vercel`.
