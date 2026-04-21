# cb-qa-claude-plugins

QA automation Claude Code plugins for the Connectbase ACE team. Works across RemotePricing2 (RP2) and RemotePricingAPI (RP1).

## Install this marketplace

```
/plugin marketplace add Connected2FiberTeam/cb-qa-claude-plugins
```

## Available Plugins

| Plugin | Description |
|--------|-------------|
| `qa-addquote` | Add new quote test scenarios to the Karate integration test suite — auto-detects RP1 vs RP2 |
| `qa-apimapadd` | Generate Karate API mapping feature files from PSAM CSV files |
| `qa-coverage-analysis` | Analyze Jira ticket test coverage — requirements, code changes, Zephyr test cases, and Karate gaps |
| `qa-zephyr-review` | Review Zephyr Scale test cases for completeness, clarity, and platform standards |

## Install individual plugins

```
/plugin install qa-addquote@qa-tools
/plugin install qa-apimapadd@qa-tools
/plugin install qa-coverage-analysis@qa-tools
/plugin install qa-zephyr-review@qa-tools
```

## Prerequisites

### All plugins
- Connectbase VPN connected for Stage/UAT/Prod database access

### qa-addquote / qa-apimapadd
- `mongosh` (`brew install mongosh`) — for RP1/RP2 MongoDB lookups
- `mysql` 8.4 client (`brew install mysql@8.4`) — for transmission rate and country mapping queries
- Credentials set in `~/.zshrc` — see CLAUDE.local.md for the full list

### qa-coverage-analysis
- `ZEPHYR_SCALE_TOKEN` set in `~/.zshrc` — for Zephyr Scale test case lookup
- Atlassian MCP connected — for Jira ticket fetch

### qa-zephyr-review
- `ZEPHYR_SCALE_TOKEN` set in `~/.zshrc` — optional (XML export works without it)

## Repo Awareness

Skills in this marketplace auto-detect which repo they're running in by scanning for marker files:
- `rp2_base_template.json` → RP2 mode (RemotePricing2)
- `rp1_base_template.json` → RP1 mode (RemotePricingAPI)

This means a single plugin installation works correctly in both repos without any configuration.

## Canonical Source

These skills are canonical in this repo. The per-repo `.claude/commands/` copies in RP2 and RP1 can be removed once this marketplace is installed.