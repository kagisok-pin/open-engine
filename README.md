# Open Engine (plugin)

Cross-surface runner for the **Open Engine** agent queue in Linear. One package, installs in both **Claude Code** and the **Claude desktop app** (Cowork + Chat).

This repo is simultaneously the **marketplace**, the **plugin**, and the **skill host**:

```
.claude-plugin/
  marketplace.json     # catalog (lists the one plugin)
  plugin.json          # the plugin manifest
skills/
  open-engine/
    SKILL.md           # the runner (persona-parameterized, surface-agnostic)
```

## Prerequisite — Linear as an account connector

Add Linear **once** as an account connector: **Settings → Connectors → Add custom connector → `https://mcp.linear.app/mcp`** (or the directory "Linear" → Connect). Account connectors are available in Code, Cowork, Chat, and mobile — so both surfaces share the same Linear workspace. *(Do not rely on `claude mcp add`; that is Code/CLI-local only.)*

## Install

**Claude Code:**
```
/plugin marketplace add kagisok-pin/open-engine
/plugin install open-engine@open-engine
/reload-plugins
```

**Desktop app (Cowork + Chat):** plugin manager → **Marketplaces** → Add from a repository → `kagisok-pin/open-engine` → **Discover** → install **open-engine**.

## Configure

The plugin ships **no workspace identifiers** — it is generic by design. On first run it resolves its config, first hit wins:

1. Named in the invocation — *"run Open Engine as `<code>` on `<team>`"*.
2. An `Open Engine config` block in the consuming project's adapter (`CLAUDE.md` / `AGENTS.md`).
3. Discovery — a single team carrying the eligibility label; standing issues found by their `[agent instructions][all agents][standing_*]` titles.
4. Otherwise it asks once.

Set up your Linear team with: an eligibility label (default `agent-instructions`), the agent workflow states (`Agent Todo`, `Agent Working`, `Agent Needs Input`, `Agent Review`, `Agent Done`), and three standing issues — setup/version, status ledger, and optional-skill directory. See `skills/open-engine/SKILL.md` for the full contract.

## Use

Say *"run the Open Engine queue as `<persona>`"* or invoke `/open-engine:open-engine`. It runs one task per invocation. Cadence is manual by default.

## Notes

- **Persona-agnostic:** the agent code is chosen per run, not baked in. One plugin serves every persona.
- **Surface-agnostic:** all state lives in Linear (the ledger issue's comments); no local files, so it runs the same in Code and Cowork.
- **No destructive ops:** the Linear MCP cannot delete/archive issues or labels — cleanup is a human UI/GraphQL job.
- Version bumps go in `plugin.json` only.
