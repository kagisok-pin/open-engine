---
name: open-engine
description: Run the Open Engine queue in Linear (team Arc and Mantle). Use when asked to "run the queue", "run Open Engine", "check the agent queue", or run the agent loop — in either the Claude Code tab or Cowork. Determines which persona to run as, claims one assigned task, does the scoped work, leaves typed receipts, and updates the status ledger — exactly one task per run.
---

# Open Engine — runner (v1)

Cross-surface runner for the Open Engine queue. A persona-agent finds its assigned work in Linear, claims one task, does it, leaves typed receipts, and updates the ledger. Runs in **either** the Claude Code tab or the Cowork tab — everything goes through the account-level **Linear** connector, so no local files are required.

## Prerequisite

Linear must be connected as an **account connector** (Settings → Connectors → `https://mcp.linear.app/mcp`) so it is available in both Code and Cowork.

## Step 0 — Identify this run's persona (agent code)

This plugin is **persona-agnostic**. Determine the agent code for THIS run:
- If the operator named one ("run Open Engine as **forge**"), use it.
- Otherwise ask once: "Which persona should I run as?"

Valid codes (lowercase persona codenames): `lattice`, `forge`, `xhantsilo`, `vector`, `sable`, `bastion`, `studio`, `loom`, `ledger`, `signal`, `prism`, `catalyst`.

Use the chosen `<code>` for the whole run. Your ledger marker is `AGENT STATUS :: <code>`; your task filter is the second title bracket `[<code>]`.

## Config (engine v1) — shared across personas

- **Engine version:** `v1` (this plugin's version; compare against setup issue ARC-8's `version:` each run)
- **Linear team:** Arc and Mantle — id `883576b4-05a3-4738-af6b-dfdfa1f17af6`
- **Project:** Personal Agent Engine — id `67cf68a6-9eff-421e-8bd5-fde7d1e47588`
- **Filter label:** `agent-instructions`
- **Standing issues:** setup = `ARC-8` · status ledger = `ARC-6` · optional-skill directory = `ARC-7`
- **Runtime field:** set to the tab you are in — `Claude Code` or `Cowork`.

**Data rule (pointers-only):** issues may hold task instructions, outcomes, receipts, and references (project slugs, Open Brain thought IDs, file paths). NEVER put raw customer PII, deal financials, credentials/secrets, or entity-confidential detail in a Linear issue — keep those in Open Brain / GlobalContext and reference by pointer.

**Safety boundaries:** ask the human before publishing, emailing, posting anywhere public, deploying, deleting data, changing billing/credentials, or any customer-facing change. External/destructive actions need explicit issue-level approval.

**Linear tools** (account connector): `list_issues`, `get_issue`, `save_issue` (set `state` to move status), `save_comment` (receipts + ledger; pass `id` to update in place), `list_comments`, `list_issue_statuses`. Note: this MCP has **no issue-delete/archive** — never assume you can self-clean issues; cleanup is a human UI/GraphQL job.

## A run (heartbeat) — do these in order, then STOP

1. **Open the ledger** (ARC-6). Find *your* comment (first line `AGENT STATUS :: <code>`) via `list_comments`; if none exists, you will create it in step 11. Set `Last queue result: checking`.
2. **Standing preflight:** read setup issue ARC-8's `version:`. Compare to this plugin's version (`v1`, and to your ledger `Local context:` line). If ARC-8 is **ahead**, you are running a stale plugin — tell the operator to update it (`/plugin marketplace update` in Code, or re-sync the plugin on desktop), note it in the ledger, and do NOT fake `AGENT APPLIED`. If you are current, nothing to do. Record the applied version on your ledger `Local context:` line. *(Version lives on the ledger — no local state file.)*
3. **Optional-skill preflight:** only for optional skills you have already subscribed to (recorded on your ledger `Optional skills:` line). Apply same-scope updates and post `AGENT SKILL UPDATED` only after a real update. Do NOT browse/install unapproved optional skills on a routine run.
4. **Check AGENT HUMAN HOLD issues** (Agent Needs Input, your code, holding). If one now shows `AGENT HUMAN ANSWERED`: move it to Agent Working, post `AGENT RESUMED`, finish it, update ledger, STOP.
5. **Check AGENT BLOCKED issues** (Agent Needs Input, your code, blocked). If the answer is now on the issue: move to Agent Working, post `AGENT UNBLOCKED` then `AGENT RESUMED`, finish, update ledger, STOP.
6. **Check delegated issues** you handed to others; post `AGENT FOLLOW-UP` if state changed.
7. **Otherwise claim** the oldest eligible Agent Todo issue. Eligible = label `agent-instructions` + title starts `[agent instructions]` + second bracket == `<code>`. Move it to Agent Working, post `AGENT CLAIMED`. **Optimistic-claim check:** re-read; if a *different* code's `AGENT CLAIMED` has an earlier Linear-server timestamp, yield (move back to Agent Todo / skip). Otherwise proceed.
8. **Do only the scoped work.** Done, no human judgement needed → post `AGENT DONE`, move to **Agent Done**. Done but needs review/QA/approval/publishing → post `AGENT DONE`, move to **Agent Review** (a human moves Review → Agent Done).
9. **If you need an answer:** belongs on the Linear issue → ask ONE specific question, post `AGENT BLOCKED`, move to Agent Needs Input, ledger `blocked ISSUE-ID`, STOP. Belongs in the operator's own agent thread/app → post `AGENT HUMAN HOLD`, move to Agent Needs Input, ledger `holding ISSUE-ID`, STOP.
10. **If execution fails unexpectedly:** post `AGENT FAILED` with the last safe step + retry count.
11. **Update your ledger comment** (in place — never a new one) and STOP after exactly one task issue.

## Receipts (post as issue comments, verbatim tokens)

`AGENT CLAIMED` · `AGENT DONE` · `AGENT BLOCKED` · `AGENT UNBLOCKED` · `AGENT HUMAN HOLD` · `AGENT HUMAN ANSWERED` · `AGENT RESUMED` · `AGENT FAILED` · `AGENT APPLIED` · `AGENT SKILL SUBSCRIBED` · `AGENT SKILL INSTALLED` · `AGENT SKILL UPDATED` · `AGENT SKILL DECLINED` · `AGENT FOLLOW-UP` · `AGENT STATUS`.

## Ledger (ARC-6)

You own exactly ONE top-level comment, updated **in place** every run. First line is the marker so you find your own comment:

```
AGENT STATUS :: <code>
Agent: <code>
Human/operator: Kagiso
Runtime: <Claude Code | Cowork>
Automation: <name | manual>
Automation state: <installed | manual-required | blocked | paused>
Last heartbeat: <ISO8601>
Last queue result: <checking | none | claimed ISSUE-ID | completed ISSUE-ID | blocked ISSUE-ID | holding ISSUE-ID | resumed ISSUE-ID | failed ISSUE-ID>
Last successful run: <ISO8601 | unknown>
Local context: <plugin version>; <routing map version>
Optional skills: <none | skill-id@version subscribed>
Notes: <none | short blocker>
```
