---
name: open-engine
description: Run the Open Engine queue in Linear. Use when asked to "run the queue", "run Open Engine", "check the agent queue", or run the agent loop — in either the Claude Code tab or Cowork. Determines which persona to run as, claims one assigned task, does the scoped work, leaves typed receipts, and updates the status ledger — exactly one task per run.
---

# Open Engine — runner (v1.3)

Cross-surface runner for the Open Engine queue. A persona-agent finds its assigned work in Linear, claims one task, does it, leaves typed receipts, and updates the ledger. Runs in **either** the Claude Code tab or the Cowork tab — everything goes through the account-level **Linear** connector, so no local files are required.

## Prerequisite

Linear must be connected as an **account connector** (Settings → Connectors → `https://mcp.linear.app/mcp`) so it is available in both Code and Cowork.

## Step 0 — Identify this run's persona (agent code)

This plugin is **persona-agnostic**. Determine the agent code for THIS run — first hit wins:
- If the operator named one ("run Open Engine as **`<code>`**"), use it.
- Else if the consuming project's adapter carries an `Open Engine config` block with an `agent code:` key, use that. A per-project adapter names exactly one persona, so this is deterministic — prefer it over asking.
- Otherwise ask once: "Which persona should I run as?"

Agent codes are lowercase persona codenames defined by your workspace (see `personas` in Config). Use the chosen `<code>` for the whole run. Your ledger marker is `AGENT STATUS :: <code>`; your task filter is the second title bracket `[<code>]`.

## Config — resolved at run start, never baked in

This plugin ships **no workspace identifiers**. It bootstraps from ONE explicit value — the **team** — then reads everything else from the **setup issue**, whose body is the authoritative config block. Resolution is **per key**, and any step that is ambiguous, empty, or unverifiable **stops and asks** — never proceed on a single-but-unconfirmed guess (a wrong-but-unique answer is indistinguishable from a right one, and this engine then *writes*: it claims issues and rewrites the ledger).

**Step A — resolve the `team`** (first hit wins; each must yield exactly ONE team or fall through):

1. **Invocation** — "run Open Engine as `<code>` on `<team>`".
2. **Adapter** — an `Open Engine config` block in the consuming project's `CLAUDE.md` / `AGENTS.md`.
3. **Discovery** — call `list_issues(label="<eligibility label>")` and read `teamId` off the results; this resolves the team in one call from zero prior workspace knowledge. **If more than one distinct `teamId` comes back, STOP and ask** — there is no safe tie-break, and guessing silently binds the plugin to the wrong queue. (Paginate — a busy label overflows one page; follow `cursor`, narrowing with `state`, until you have either seen a second `teamId` or confirmed a single one across all pages.)
4. If A1–A3 do not yield exactly one team, **ask once**.

> **Do not resolve the team via `list_issue_labels(name=…)`.** That call is scope-inconsistent: with no `team` argument it returns only *workspace-level* labels, so a team-scoped eligibility label comes back as `[]`, indistinguishable from "the label does not exist" (verified against the live connector 2026-07-19). The team is the bootstrap anchor — you cannot find it *by* an unscoped label query.
>
> **Discovery is the weakest rung — prefer step 2 (a declared config block).** A declared block cannot silently drift onto the wrong workspace; an inference can, and the multi-team tie-break above ships unproven — it has never run against a workspace with more than one team.
>
> (The eligibility *label* is `agent-instructions`, hyphen; the title's first *bracket* is `[agent instructions]`, space — different strings, both required.)

**Step B — find the `setup_issue`, then read its body as config.** Among the team's issues carrying the eligibility label, the three standing issues are keyed by the **exact** third bracket-token of their title — **exact string equality, never a glob or substring** (`standing_skill` is a substring of `optional_standing_skill_directory`, and `standing_*` does not match the `optional_`-prefixed one at all):

| Config key | Title 3rd-bracket token (match exactly) | Role |
|---|---|---|
| `setup_issue` | `standing_skill` | version + config block |
| `ledger_issue` | `standing_status` | status ledger |
| `skills_issue` | `optional_standing_skill_directory` | optional-skill directory |

The token names are historical and do **not** track the key names — map by this table, never by keyword ("skill" appears in two tokens; the one you want for `skills_issue` is the `*_directory` one). Open the `setup_issue`; its body states `version`, team, project, eligibility label, and the ledger + skills issue IDs. Take `project` and `label` from there.

**Cross-check, don't trust.** The `ledger_issue` / `skills_issue` found by bracket-token MUST equal the IDs the setup-issue body names. On any mismatch, empty result, or more-than-one match for any key → **stop and ask**.

**Step C — resolve the rest:**

- `personas` — children of the `persona` label group (`list_issue_labels(team=…)`), **excluding the reserved `all` token** — `all` is the broadcast target for `[all agents]` standing issues, not a claimable code (it is structurally identical to a real persona label; exclude it by token, not by shape). A code whose label description records an open lifecycle decision is **contested** — surface it with the caveat; neither silently include nor silently drop it.
- `claimable status` — `list_issue_statuses(team=…)`. **Only `Agent Todo` is claimable.** `Standing`, `Agent Working`, `Agent Needs Input`, `Agent Review`, `Agent Done`, `Backlog`, `Canceled`, `Duplicate` are not — never claim a `Standing` issue; those are the engine's own records (ledger / setup / skills).
- `ledger comment contract` — the `AGENT STATUS :: <code>` template in the **Ledger** section below; for a persona's first-ever run, there is no prior comment to copy — use that template.

**Pagination discipline.** Every `list_*` verb is paged with differing caps; one call is not a complete view. Narrow with `query` / `label` / `state` filters and follow `cursor`. For the eligibility sweep, filter `state="Agent Todo"` — an unfiltered `label` sweep can overflow the response.

| Key | Meaning | Source |
|---|---|---|
| `team` | Linear team hosting the queue | Step A |
| `setup_issue` | version + config block | Step B — bracket-token `standing_skill` |
| `project` | Linear project | setup-issue body |
| `label` | Eligibility label (default `agent-instructions`) | setup-issue body |
| `ledger_issue` | Status ledger | setup body + bracket-token `standing_status` |
| `skills_issue` | Optional-skill directory | setup body + bracket-token `optional_standing_skill_directory` |
| `personas` | Valid agent codes | `persona` label group minus `all` |
| `claimable status` | Which state may be claimed | `Agent Todo` only |

- **Engine version:** `v1.3` — compare against the setup issue's `version:` each run.
- **Runtime field:** the tab you are in — `Claude Code` or `Cowork`.

**Data rule (pointers-only):** issues may hold task instructions, outcomes, receipts, and references (project slugs, memory-system thought IDs, file paths). NEVER put raw customer PII, deal financials, credentials/secrets, or entity-confidential detail in a Linear issue — keep those in your memory/context store and reference by pointer.

**Safety boundaries:** ask the human before publishing, emailing, posting anywhere public, deploying, deleting data, changing billing/credentials, or any customer-facing change. External/destructive actions need explicit issue-level approval.

**Linear tools** (account connector): `list_issues`, `get_issue`, `save_issue` (set `state` to move status), `save_comment` (receipts + ledger; pass `id` to update in place), `list_comments`, `list_issue_statuses`. Note: this MCP has **no issue-delete/archive** — never assume you can self-clean issues; cleanup is a human UI/GraphQL job.

## Creating issues (engine contract)

Set these **at creation** — a backfill only fixes the past; the template is what stops the drift.

- **Title:** `[agent instructions][<code>][task] <summary>`. Plain text only — `<`, `>` and `&` are stored as HTML entities, render literally, and mis-key every title-parsing consumer.
- **`persona/<code>` label** — always, mirroring the title's 2nd bracket. Single-select group: Linear rejects a second persona server-side, so an ambiguous key is structurally impossible.
- **eligibility label** (`agent-instructions` by default) — ONLY if the issue is meant to be agent-claimable.

**Two deliberate off-queue idioms. Respect both; never "tidy" them away:**

1. **Blocked work** — set `blockedBy` to the real dependency. A blocked issue is not eligible: skip it rather than re-deriving the block every run.
2. **Human gates** — to file work agents must NOT auto-claim (a validation or approval gate), omit the eligibility label AND use an off-format title (e.g. `[VALIDATE][<code>]`). Still set `persona/<code>` — persona keying and eligibility are orthogonal.

**Never add the eligibility label to an existing issue as cleanup.** It is the queue's eligibility switch, not metadata; adding it silently converts a human gate into auto-claimable agent work.

**`save_issue`'s `labels` REPLACES the whole label set** — omitted labels are deleted, it does not append. Always read-modify-write (`get_issue` → union → write). Sending only the new label strips the eligibility label and silently makes issues unclaimable by every persona.

## A run (heartbeat) — do these in order, then STOP

1. **Open the ledger** (the ledger issue). Find *your* comment (first line `AGENT STATUS :: <code>`) via `list_comments`; if none exists, you will create it in step 11. Set `Last queue result: checking`.
2. **Standing preflight:** read the setup issue's `version:`. Compare to this plugin's version (the **Engine version** in Config above, and to your ledger `Local context:` line). If the setup issue is **ahead**, you are running a stale plugin — tell the operator to update it (`/plugin marketplace update` in Code, or re-sync the plugin on desktop), note it in the ledger, and do NOT fake `AGENT APPLIED`. If you are current, nothing to do. Record the applied version on your ledger `Local context:` line. *(Version lives on the ledger — no local state file.)*
3. **Optional-skill preflight:** only for optional skills you have already subscribed to (recorded on your ledger `Optional skills:` line). Apply same-scope updates and post `AGENT SKILL UPDATED` only after a real update. Do NOT browse/install unapproved optional skills on a routine run.
4. **Check AGENT HUMAN HOLD issues** (Agent Needs Input, your code, holding). If one now shows `AGENT HUMAN ANSWERED`: move it to Agent Working, post `AGENT RESUMED`, finish it, update ledger, STOP.
5. **Check AGENT BLOCKED issues** (Agent Needs Input, your code, blocked). If the answer is now on the issue: move to Agent Working, post `AGENT UNBLOCKED` then `AGENT RESUMED`, finish, update ledger, STOP.
6. **Check delegated issues** you handed to others; post `AGENT FOLLOW-UP` if state changed.
7. **Otherwise claim** the oldest eligible Agent Todo issue. Eligible = eligibility label + title starts `[agent instructions]` + second bracket == `<code>`. Move it to Agent Working, post `AGENT CLAIMED`. **Optimistic-claim check:** re-read; if a *different* code's `AGENT CLAIMED` has an earlier Linear-server timestamp, yield (move back to Agent Todo / skip). Otherwise proceed.
8. **Do only the scoped work.** Done, no human judgement needed → post `AGENT DONE`, move to **Agent Done**. Done but needs review/QA/approval/publishing → post `AGENT DONE`, move to **Agent Review** (a human moves Review → Agent Done).
9. **If you need an answer:** belongs on the Linear issue → ask ONE specific question, post `AGENT BLOCKED`, move to Agent Needs Input, ledger `blocked ISSUE-ID`, STOP. Belongs in the operator's own agent thread/app → post `AGENT HUMAN HOLD`, move to Agent Needs Input, ledger `holding ISSUE-ID`, STOP.
10. **If execution fails unexpectedly:** post `AGENT FAILED` with the last safe step + retry count.
11. **Update your ledger comment** (in place — never a new one) and STOP after exactly one task issue.

**Answered work stays in Agent Needs Input.** Steps 4–5 are the only resume path, and they scan *Agent Needs Input* only. Moving an answered issue to Agent Review takes it out of the loop — it then reads as completed work awaiting sign-off and no run will ever pick it up.

## Receipts (post as issue comments, verbatim tokens)

`AGENT CLAIMED` · `AGENT DONE` · `AGENT BLOCKED` · `AGENT UNBLOCKED` · `AGENT HUMAN HOLD` · `AGENT HUMAN ANSWERED` · `AGENT RESUMED` · `AGENT FAILED` · `AGENT APPLIED` · `AGENT SKILL SUBSCRIBED` · `AGENT SKILL INSTALLED` · `AGENT SKILL UPDATED` · `AGENT SKILL DECLINED` · `AGENT FOLLOW-UP` · `AGENT STATUS`.

## Ledger

You own exactly ONE top-level comment on the ledger issue, updated **in place** every run. First line is the marker so you find your own comment:

```
AGENT STATUS :: <code>
Agent: <code>
Human/operator: <operator>
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
