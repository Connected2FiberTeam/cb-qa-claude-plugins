# cb-qa-claude-plugins

QA automation Claude Code plugins for the Connectbase ACE team. Works across RemotePricing2 (RP2), RemotePricingAPI (RP1), and UAP.

## Install this marketplace

```
/plugin marketplace add Connected2FiberTeam/cb-qa-claude-plugins
```

## Available Plugins

| Plugin | Description |
|--------|-------------|
| `qa-addquote` | Add new quote test scenarios to the Karate integration test suite — auto-detects RP2, RP1, or UAP |
| `qa-apimapadd` | Generate Karate API mapping feature files from PSAM CSV files |
| `qa-coverage-analysis` | Analyze Jira ticket test coverage — requirements, code changes, Zephyr test cases, and Karate gaps |
| `qa-zephyr-review` | Review Zephyr Scale test cases for completeness, clarity, and platform standards — supports RP1, RP2, LMX, and UAP |
| `ace-runquote` | Run a live one-off quote against any ACE provider (RP2, RP1, or UAP) and return the full raw response |
| `ace-datavalidate` | Validate addresses and quote configurations by running live quote requests across all combinations |
| `ace-datadiscovery` | Search historical pricing databases (MongoDB for RP2/RP1, PostgreSQL for UAP) to find valid test addresses |

## Install individual plugins

```
/plugin install qa-addquote@qa-tools
/plugin install qa-apimapadd@qa-tools
/plugin install qa-coverage-analysis@qa-tools
/plugin install qa-zephyr-review@qa-tools
/plugin install ace-runquote@qa-tools
/plugin install ace-datavalidate@qa-tools
/plugin install ace-datadiscovery@qa-tools
```

## Prerequisites

### All plugins
- Connectbase VPN connected for Stage/UAT/Prod database access

### qa-addquote / qa-apimapadd
- `mongosh` (`brew install mongosh`) — for RP1/RP2 MongoDB lookups
- `mysql` 8.4 client (`brew install mysql@8.4`) — for transmission rate and country mapping queries
- The following env vars set in `~/.zshrc`:
  - `MYSQL_USER` — MySQL username
  - `MYSQL_HOST_STAGE` / `MYSQL_PASSWORD_STAGE` — Stage connection
  - `MYSQL_HOST_UAT` / `MYSQL_PASSWORD_UAT` — UAT connection
  - `MYSQL_HOST_PROD` / `MYSQL_PASSWORD_PROD` — Production connection

### qa-coverage-analysis
- `ZEPHYR_SCALE_TOKEN` set in `~/.zshrc` — for Zephyr Scale test case lookup
- Atlassian MCP connected — for Jira ticket fetch

### qa-zephyr-review
- `ZEPHYR_SCALE_TOKEN` set in `~/.zshrc` — optional (XML export works without it)

### ace-runquote / ace-datavalidate
No additional prerequisites beyond VPN — uses the existing Gradle/Karate test runner in the target repo.

### ace-datadiscovery
- `mongosh` (`brew install mongosh`) + the MongoDB env vars above — for RP2/RP1 database searches
- `database-operations` plugin or `psql` — for UAP PostgreSQL searches


## Repo Awareness

Skills in this marketplace auto-detect which repo they're running in by scanning for marker files:
- `rp2_base_template.json` → RP2 mode (RemotePricing2)
- `rp1_base_template.json` → RP1 mode (RemotePricingAPI)
- `wait_for_uap_result.js` → UAP mode

This means a single plugin installation works correctly in all repos without any configuration.

## Shared Testing Context

`context/qa-testing-context.md` contains shared QA reference data (provider lists, field options, platform standards) used by `qa-coverage-analysis` and `qa-zephyr-review`. These skills fetch it automatically from GitHub at runtime — no local setup required.

## Canonical Source

These skills are canonical in this repo. The per-repo `.claude/commands/` copies in RP2 and RP1 can be removed once this marketplace is installed.