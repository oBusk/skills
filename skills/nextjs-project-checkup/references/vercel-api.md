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

### Everything about the project

```bash
pnpx vercel api "/v9/projects/$PRJ?teamId=$TEAM" --raw
```

Fields this checkup uses:

| Field | Means |
| --- | --- |
| `webAnalytics.enabledAt` | Web Analytics on. Absent (even with an `id`) = off |
| `defaultResourceConfig.fluid` | Fluid Compute. `false` is the finding |
| `defaultResourceConfig.functionDefaultMemoryType` | `standard_legacy` travels with fluid off |
| `security.managedRules` | Summary of the managed rulesets |
| `security.botIdEnabled` | BotID (separate product from Bot Protection) |
| `nodeVersion` | Compare with `engines.node` |
| `ssoProtection.deploymentType` | Vercel Authentication. `"all"` hides the custom domain; `"prod_deployment_urls_and_all_previews"` is the default and leaves it public; `null` is off |

Note the `security.managedRules` summary keys the bot ruleset as `bot_filter`,
while the firewall config below and the write API both call it `bot_protection`.
Same ruleset.

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

### Enable Fluid Compute

```bash
echo '{"defaultResourceConfig":{"fluid":true}}' \
| pnpx vercel api "/v9/projects/$PRJ?teamId=$TEAM" -X PATCH --input -
```

Re-read the project afterwards and confirm `fluid` flipped;
`functionDefaultMemoryType` should move off `standard_legacy`.

### Web Analytics

Enabling analytics from the API is not exposed as a documented endpoint. Do the
dashboard toggle, or `claude-in-chrome`, and confirm by re-reading
`webAnalytics.enabledAt`.

## Known breakage

- `vercel firewall overview` → `IP Bypass is unavailable for this plan (404)` on
  plans without IP Bypass, and the error takes the whole readout with it. Read
  the config through `vercel api` instead.
- `npx vercel` can fail on `EBADDEVENGINES` in repos that set `devEngines`.
  Use `pnpx vercel`.
