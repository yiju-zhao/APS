# APS

A personal marketplace of Claude Code plugins. Each plugin lives in `plugins/<name>/`, and
`.claude-plugin/marketplace.json` lists them.

## Install a plugin

Type this at the prompt of a Claude Code session:

```
/plugin install billing-usage --marketplace yiju-zhao/APS
```

Answer `y` to add the marketplace, then pick a scope.

## Plugins

| Plugin | What it does |
| --- | --- |
| [billing-usage](plugins/billing-usage) | Per-day (UTC) token usage and API-list-price equivalent cost since a billing-cycle start date |

### billing-usage

```
/billing-usage:report 2026-10-08
/billing-usage:report 2026-10-08 --end 2026-10-31
```

The first argument is the first day of the billing cycle. The command prints one row per UTC day
and a total: requests, input, output, cache write, cache read, and `cost_usd`.

- It reads `~/.claude/projects/**/*.jsonl` (or `$CLAUDE_CONFIG_DIR/projects`), so it covers this
  machine and this config directory only. It excludes other devices, claude.ai, and sessions
  already removed by `cleanupPeriodDays`.
- A request is written to the transcript several times while it streams, so rows are
  deduplicated by `(message.id, requestId)`.
- `cost_usd` is what the same tokens would cost at the public API list price, not what a
  subscription is billed. The price table is `PRICES` in `scripts/billing_usage.py`, read from
  platform.claude.com/docs/en/about-claude/pricing on 2026-10-19. A model missing from the table
  is reported as a warning; its tokens are still counted.
- The transcript format is not a documented interface. After a Claude Code upgrade, compare the
  request count with `claude -p "/usage"` (its "Last 24h" and "Last 7d" lines).

## Add a plugin

1. Create `plugins/<name>/.claude-plugin/plugin.json`.
2. Add an entry to `.claude-plugin/marketplace.json` with `"source": "./plugins/<name>"`.
3. Run `claude plugin validate .` from the repository root.
