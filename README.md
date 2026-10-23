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
| [billing-usage](plugins/billing-usage) | Per-day (UTC) token usage and API-list-price equivalent cost since a billing-cycle start date, summed across your devices, as a report or a resident terminal UI |

### billing-usage

```
/billing-usage:report 2026-10-08
/billing-usage:report 2026-10-08 --end 2026-10-31
/billing-usage:setup
```

The first argument is the first day of the billing cycle. The command prints one row per UTC day
and a total: requests, input, output, cache write, cache read, and `cost_usd`.

- On each device it reads `~/.claude/projects/**/*.jsonl` (or `$CLAUDE_CONFIG_DIR/projects`). It
  excludes devices you did not list, claude.ai, and sessions already removed by
  `cleanupPeriodDays`. With no devices file it covers this machine only.
- A request is written to the transcript several times while it streams, so rows are
  deduplicated by `(message.id, requestId)`, per device.
- `cost_usd` is what the same tokens would cost at the public API list price, not what a
  subscription is billed. The price table is `PRICES` in `scripts/billing_usage.py`, read from
  platform.claude.com/docs/en/about-claude/pricing on 2026-10-19. A model missing from the table
  is reported as a warning; its tokens are still counted.
- The transcript format is not a documented interface. After a Claude Code upgrade, compare the
  request count with `claude -p "/usage"` (its "Last 24h" and "Last 7d" lines).

#### Multiple devices

Run `/billing-usage:setup` to build the devices file step by step. It lists candidates from
`tailscale status` and `~/.ssh/config`, asks which to include, writes the file, and tests each
connection, telling you how to fix any that fail.

Or write it by hand: list your devices in `~/.config/billing-usage/devices.json` (or pass
`--devices FILE`). Each device has a name, a connection method and, for `ssh`, an address:

```json
{"devices": [
  {"name": "this-machine", "method": "local"},
  {"name": "dev-box", "method": "ssh", "address": "100.64.0.10", "user": "alice"}
]}
```

[`devices.example.json`](plugins/billing-usage/devices.example.json) lists every field.
`ssh` also takes `port`, `identity_file`, `python` and `config_dir`. List every device you want
counted, this machine included.

- The `ssh` method sends the script to the device over standard input and reads back one line
  of JSON with the per-day numbers. Nothing is installed or written on the device, and no
  conversation content leaves it.
- It runs `ssh` with `BatchMode=yes`, so each device needs key-based or Tailscale SSH access
  that works without a prompt, and `python3`. Trust each device's host key once with
  `ssh <address>`; the plugin never turns host key checking off.
- Devices are read in parallel. The output has the combined per-day table and a per-device table.
- A device that cannot be read is marked `UNREACHABLE` and left out of the totals. A warning
  gives the reason and, for the common cases, a `fix:` line. It is never counted as zero.
- Each device keeps its own 30-day transcript history. If you sync `~/.claude/projects` between
  devices, the same request can appear on two of them and be counted twice.
- Keep `devices.json` out of this repository: it holds addresses and user names.

#### Terminal UI

A resident view of the same numbers: a stacked timeline and a share table, each headed by a sentence
whose words you can change (for example `花费 · 所有设备 · 按模型 · 每 4 小时 · 最近 7 天`). It reads the
same devices file as the report. It needs [uv](https://docs.astral.sh/uv/); `uv run --script` fetches
Python 3.11+ and Textual (pinned in the script header) on first use.

```
uv run --script ~/.claude/plugins/marketplaces/aps/plugins/billing-usage/scripts/billing_usage_tui.py
```

Suggested alias, for your shell rc file (the plugin never edits it):

```
alias bu='uv run --script ~/.claude/plugins/marketplaces/aps/plugins/billing-usage/scripts/billing_usage_tui.py'
```

The marketplace path works once this change is merged and the marketplace is updated; until then, or while
developing, use the same path under `~/Developer/repos/APS`. Options:
`--devices FILE`, `--config-dir DIR` and `--timeout SECONDS` (default 120 per ssh device).
`--once` refreshes, updates the status-line summary, prints one line (`本周期 … · 今日 … · n/m 台在线`,
then one line for each offline device) and exits.

| Key | Action |
| --- | --- |
| `r` | refresh now (it also refreshes every 60 s, and never starts a second refresh while one is running) |
| `q` | quit |
| `c` | set the billing-cycle start day, 1 to 28 (default 8; cycles start at 00:00 UTC) |
| `+` (or `=`), `-` | zoom the timeline in and out; the bucket size follows the zoom |
| `←` `→` | move the highlighted bar; once zoomed, pan the window |
| `Shift` + `←` `→` | always move the highlighted bar |
| `↑` `↓` | pick a legend entry |
| `i` | show only the picked entry; `Esc` lets go |
| `Tab`, `Enter`, or click | change a word in a sentence: metric, device, grouping, bucket size, range |

- The TUI and `/billing-usage:report` share one calculation (`billing_usage.py` reads, deduplicates
  and prices the requests for both), so the cycle total equals the report's `TOTAL` for the same
  devices at the same moment.
- For `ssh` devices the script runs with `--rows`: each request comes back as one row (time, model,
  effort, project folder name, skill, token counts, cost). Nothing is installed or written on the
  device, and no conversation content leaves it.
- A device that cannot be read shows as `<name> 离线 · 最后成功于 <time> · <fix hint>`. The TUI keeps
  running and keeps that device's last rows, so the other numbers do not change.
- Cache: `${XDG_CACHE_HOME:-~/.cache}/billing-usage/` holds `rows-<device>.json` (the last rows read
  from each device), `summary.json` and `limits.json`. State, meaning your sentence choices and the
  cycle day, is `${XDG_STATE_HOME:-~/.local/state}/billing-usage/tui.json`.
- `summary.json` holds today's and the cycle's cost (and `updated_at`) for a status line: one `jq` call
  can show `今日 … · 周期 …`, dimmed once it is older than 10 minutes. `limits.json` holds
  `seven_day_resets_at` (epoch seconds, Claude Code's `rate_limits.seven_day.resets_at`), which only a
  status line can record; until it is there, the range "自每周重置以来" is unavailable.

## Add a plugin

1. Create `plugins/<name>/.claude-plugin/plugin.json`.
2. Add an entry to `.claude-plugin/marketplace.json` with `"source": "./plugins/<name>"`.
3. Run `claude plugin validate .` from the repository root.
