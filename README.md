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
| [billing-usage](plugins/billing-usage) | Sets up the devices file for [`bu`](https://github.com/yiju-zhao/billing-usage), the Claude Code usage terminal UI |

### billing-usage

```
/billing-usage:setup
```

Builds the devices file for [`bu`](https://github.com/yiju-zhao/billing-usage), a terminal UI for Claude Code
token usage and list-price cost across your devices. It lists candidates from `tailscale status` and
`~/.ssh/config`, asks which to include, writes `~/.config/billing-usage/devices.json`, and tests each connection
with `bu report`, telling you how to fix any that fail. Install `bu` first:

```
uv tool install git+https://github.com/yiju-zhao/billing-usage
```

The usage numbers themselves come from `bu` (or `bu report --start <cycle-start>`) in a terminal, with no model
call; see that repository's README.

## Add a plugin

1. Create `plugins/<name>/.claude-plugin/plugin.json`.
2. Add an entry to `.claude-plugin/marketplace.json` with `"source": "./plugins/<name>"`.
3. Run `claude plugin validate .` from the repository root.
