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

The plugin ships **no workspace identifiers** — it is generic by design. It bootstraps from ONE explicit value, the **team**, then reads everything else from the **setup issue**, whose body is the authoritative config block:

1. Resolve the `team` — from the invocation (*"run Open Engine as `<code>` on `<team>`"*), an `Open Engine config` block in the consuming project's adapter, or discovery (`list_issues(label="agent-instructions")`, reading `teamId` off the results; stop and ask if more than one distinct team appears). A declared config block is preferred over discovery. If none is unambiguous, it asks once.
2. Find the **setup issue** by the exact title bracket-token `[standing_skill]`, and read `project`, `label`, and the ledger + skills issue IDs from its body.
3. Resolve `personas` from the `persona` label group (minus the reserved `all`) and the claimable state (`Agent Todo`).

Resolution is per-key and **stops to ask** on any ambiguous or unverifiable result — it never proceeds on an unconfirmed guess. See `skills/open-engine/SKILL.md` → **Config** for the full contract.

To set up a new Linear team: add an eligibility label (default `agent-instructions`) and a `persona` label group; create the agent workflow states (`Agent Todo`, `Agent Working`, `Agent Needs Input`, `Agent Review`, `Agent Done`, plus `Standing`); and file three standing issues whose third title bracket-tokens are exactly `standing_skill` (setup/version — its body is the config block), `standing_status` (status ledger), and `optional_standing_skill_directory` (optional-skill directory).

## Use

Say *"run the Open Engine queue as `<persona>`"* or invoke `/open-engine:open-engine`. It runs one task per invocation. Cadence is manual by default.

## Notes

- **Persona-agnostic:** the agent code is chosen per run, not baked in. One plugin serves every persona.
- **Surface-agnostic:** all state lives in Linear (the ledger issue's comments); no local files, so it runs the same in Code and Cowork.
- **No destructive ops:** the Linear MCP cannot delete/archive issues or labels — cleanup is a human UI/GraphQL job.
- Version bumps go in `plugin.json` only.
