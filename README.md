# magpie-synthetic-provider-plugin

[OpenCode](https://opencode.ai) / [magpie](https://usemagpie.ai) provider plugin for [Synthetic](https://synthetic.new). Signs in with a Synthetic API key, serves its models, and reports subscription usage to `magpie quota` through magpie's `auth.usage` hook.

## Usage

```sh
magpie plugin add <path-or-spec>
magpie plugin login synthetic   # paste your Synthetic API key
magpie provider test synthetic
magpie quota                    # synthetic · 5h n% ↻… · weekly n% ↻…
```

## What it reports

`GET https://api.synthetic.new/v2/quotas` (free — does not count against limits) maps onto magpie usage windows:

| Quota lane | Window | Meaning |
| --- | --- | --- |
| `planWeeklyLimit` | `7 days` | Weekly credit budget on percent-only plans; `used = 100 − percentRemaining`. Preferred over `weeklyTokenLimit` when both appear |
| `weeklyTokenLimit` | `7 days` | Credit-denominated variant on usage-based plans; `remainingCredits`/`maxCredits` fill the `x / y credits` display (`amount`/`limit`/`unit`) and `remainingCredits` is reported as the card `balance` |
| `rollingFiveHourLimit` | `5 hours` | Retired burst request limit; still parsed for older responses |
| `search.hourly` | `search / hour` | Hourly search quota (informational, `aside`) |
| `subscription` | `billing period` | Retired per-period request counter, still echoed near 0; only shown as a fallback when no real lane is present |

Primary windows (`7 days`, `5 hours`) report `name`/`used`/`resetsAt`/`span`, so magpie renders the progress bar, reset countdown, pace and history trend like its built-in providers (Codex, DeepSeek). `aside` windows render as text rows only.

Synthetic's weekly budget has no reset: it refills +2% of the limit every ~202 minutes ([rate limits](https://synthetic.new/rate-limits)). The `7 days` window's `resetsAt` is therefore the projected moment the budget would be full again — `nextRegenAt` plus `ceil(gap / step) − 1` ticks of 202 minutes — read as "满额恢复于". On credit plans the tick size comes from `nextRegenCredits`; when regen data is missing, `nextRegenAt` itself is sent.

## API used

- Chat: `https://api.synthetic.new/anthropic/v1` (Anthropic Messages; `@ai-sdk/anthropic`). magpie injects the key itself as `x-api-key`
- Models: `GET https://api.synthetic.new/openai/v1/models` per account (the anthropic endpoint has no `/models` route). The anthropic API only accepts catalog `hf:` model ids, so alias keys (`syn:*`, `infinite:*`) send the `hf:` id of the catalog entry with the same `hugging_face_id`; falls back to the `syn:*` aliases
- Quotas: `GET https://api.synthetic.new/v2/quotas`

## Development

```sh
npm test   # node --test test/
```

Entry is `index.mjs` (plain ESM, also loadable by Bun in the magpie plugin host). Helpers for tests live on `export const _internal`.
