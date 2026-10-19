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
| [billing-usage](plugins/billing-usage) | Per-day (UTC) token usage and API-list-price equivalent cost since a billing-cycle start date, summed across your devices |

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

## Add a plugin

1. Create `plugins/<name>/.claude-plugin/plugin.json`.
2. Add an entry to `.claude-plugin/marketplace.json` with `"source": "./plugins/<name>"`.
3. Run `claude plugin validate .` from the repository root.
