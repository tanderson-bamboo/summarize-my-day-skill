---
name: summarizing-my-day
description: Use when the user asks to "summarize my day", says "summarize", or runs /summarize — reconstructs work since the last summary from git, shell history, Claude sessions, and JIRA; writes it to a Markdown daily note; and proposes JIRA status comments.
---

# Summarizing My Day

## Overview

Reconstruct what the user did since their last summary, write it into their Markdown daily
note, and propose brief status comments on the JIRA tickets they worked. **Posting to JIRA
requires the user's explicit confirmation — never auto-post.**

## Config & state

Single file `~/.claude/summarizing-my-day.json`:
```json
{
  "version": 1,
  "dailyNotesDir": "",
  "noteFilename": "YYYY-MM-DD.md",
  "jiraCloudId": "",
  "gather": { "reposDir": "", "histFile": "", "claudeProjectsDir": "", "gitEmail": "" },
  "slack": { "enabled": false, "userId": "" },
  "lastRun": 0
}
```
- `gather.*` empty → `gather.sh` uses its built-in defaults; pass a flag only for non-empty values.
- `jiraCloudId` may be empty (note-only, or no Atlassian MCP) — the JIRA steps then skip.
- `slack.enabled` is opt-in (default off); when on, `slack.userId` holds the user's own Slack
  ID. Slack is optional and NOT part of the setup-complete test.
- **Setup-complete test:** file exists AND `version >= 1` AND `dailyNotesDir` non-empty.
  Otherwise run **Setup (first run)** before anything else.
- **Missing/zero `lastRun`:** the window starts at midnight today; tell the user.

## Setup (first run)

Run when the setup-complete test fails, or when the user asks to reconfigure
("reconfigure summarize", `/summarize setup`). Then continue into the Procedure.

1. **Daily-notes directory** — ask the user where their dated daily notes live (no preset
   path). Verify the directory exists; offer to create it if missing.
2. **gather.sh sources** — the built-in defaults are `~/repos`, `~/.zsh_history`,
   `~/.claude/projects`, and the global git email. Ask whether to override any; store only
   the changed values under `gather`.
3. **JIRA cloud id** — call `getAccessibleAtlassianResources`. One site → use its id.
   Multiple → ask which. MCP not connected → warn and store `""` (note-only mode). Save the
   id as `jiraCloudId`.
4. **Slack activity (optional)** — ask: "Include Slack activity in summaries? This reads your
   sent messages across all channel types plus the DMs you received, each run." If yes → call `slack_read_user_profile` (no args) to get
   the current user's own ID; set `slack.enabled` true and `slack.userId` to that id. If no, or
   the Slack MCP is not connected → set `slack.enabled` false.
5. Read the existing config first if present and carry its `lastRun` forward; then write the
   full JSON with all fields set from the user's answers. Confirm a one-line summary of what
   was saved, then proceed into the Procedure.

## Procedure

1. **Resolve the window** `since` (epoch): from `lastRun`, unless the user gave an override
   ("yesterday", "this week", a date or range) — compute `since` from that instead.
2. **Run the gather script** and read its markdown output. Pass stored `gather` overrides as
   flags — only the non-empty ones:
   `~/.claude/skills/summarizing-my-day/gather.sh --since <epoch> [--repos-dir DIR] [--histfile FILE] [--claude-projects DIR] [--git-email EMAIL]`
   Config key → flag: `reposDir`→`--repos-dir`, `histFile`→`--histfile`,
   `claudeProjectsDir`→`--claude-projects`, `gitEmail`→`--git-email`.
3. **Slack activity** — only if `slack.enabled` and `slack.userId` is set (else skip silently;
   also skip if the Slack MCP is unavailable, noting "Slack skipped"). Substitute `{slack.userId}`
   with the stored value (e.g. if it is `U012AB3CD`, the query is `from:<@U012AB3CD>`). Run two
   `slack_search_public_and_private` calls, both windowed with the tool's `after`/`before`
   Unix-timestamp params (`after` = `since`, `before` = now):
   - query `from:<@{slack.userId}>`, `channel_types=public_channel,private_channel,im,mpim` —
     everything the user authored (channel posts, thread replies, DMs they sent).
   - query `to:me`, `channel_types=im,mpim` — DMs the user received.
   Dedupe by `message_ts`; add any JIRA ticket IDs found in these messages to the evidence
   ticket IDs from step 2 (gather's "Evidence ticket IDs"). Label each message
   (completion / follow-up / drop) per "Slack filtering" below and keep the labeled list for step 5.
4. **Pull JIRA activity** (Atlassian MCP) using `jiraCloudId` from config. If `jiraCloudId`
   is empty, skip this step and note that JIRA was skipped (suggest running setup).
   - JQL: `assignee = currentUser() AND updated >= "<YYYY-MM-DD>"`.
   - `getJiraIssue` for each evidence ticket ID from the gather and Slack output.
5. **Synthesize** a first-person summary: **Shipped / In progress / Explored**. Read the
   specific Claude transcripts gather listed only as needed — skim, never dump them. Using the
   labeled Slack list from step 3, fold completions into the Shipped / In progress / Explored
   bullets and collect follow-ups for the "Follow-ups (from Slack)" line (see "Slack filtering").
6. **Write the daily note** (see "Daily note" below).
7. **Candidate tickets** = evidence IDs ∪ assigned tickets clearly reflected in the work.
   Draft a brief (2–4 sentence) first-person status comment for each.
8. **STOP and present.** Show the summary, then a table: `Ticket | Evidence | Draft comment`.
   Ask which to post: all / a subset / edit text / none.
9. **Only after explicit confirmation**, post chosen comments via `addCommentToJiraIssue`.
   Report what posted.
10. **Write `lastRun` = now** to the config file (last, so a mid-run error doesn't lose the window).

## Slack filtering

Applies only when the Slack step ran. Classify each candidate message conservatively — when
unsure, drop it:
- **Work-related completion** (e.g. "Approved. Does it need QA?", "merged the fix") → fold into
  **Shipped / In progress / Explored** alongside git/JIRA evidence.
- **Work commitment or open todo** (e.g. "I'll get the NYA PR up today"; a request the user
  clearly accepted or acted on) → add to a **"Follow-ups (from Slack)"** line under `## Work Summary`.
- **Drop**: personal/social ("I'll go to that taco place"), banter, "lol", pure FYI with no
  action, a request addressed to the user without their clear acceptance or action in the
  window, and anything ambiguous or not work-related.

## Daily note

- Path: `<dailyNotesDir>/<today, formatted per noteFilename>` (format tokens follow
  strftime/moment.js conventions: `YYYY` 4-digit year, `MM` month, `DD` day).
- Never overwrite the file or touch sections other than `## Work Summary`.
- **File doesn't exist:** create it with `# <Weekday, Month Dayth, Year>`
  (e.g. `# Friday, June 5th, 2026`), then a `## Work Summary` section holding the summary.
- **File exists, `## Work Summary` absent:** append the `## Work Summary` section.
- **File exists, `## Work Summary` present (e.g. an earlier run today):** append a new
  `### <h:mmam/pm>` sub-block under it; do not replace earlier sub-blocks.

## The hard rule — JIRA

**Nothing posts without the user's explicit go-ahead in step 8.**
- No auto-post. No "I'll just post the obvious ones." No posting because it seems clearly right.
- If unsure whether a ticket applies, leave it OUT of the post set and mention it.
- "Confirmed" = the user named the tickets, or said yes, *after* seeing the drafts.
- "The user said to just post / they trust me / they're in a hurry" is NOT confirmation of
  *which* tickets. Show the drafts and get a yes anyway — it takes one message.

## Edge cases

- Setup not complete → run Setup (first run) first.
- No evidence tickets and no assigned matches → present + save the summary, skip steps 7–9.
- `gather.sh` errors or is empty → still pull JIRA + report what you have.
- `jiraCloudId` empty → skip the JIRA steps; note it in the output.
- `slack.enabled` false / `slack.userId` empty / Slack MCP unavailable → skip Slack; note it.
- First run after setup → midnight-today window; announce the fallback.

## Common mistakes

- Overwriting the daily note → only edit the Work Summary section.
- Auto-posting to JIRA → always confirm first.
- Dumping full transcripts → skim and summarize.
- Including non-work Slack chatter → filter conservatively; drop if unsure.
- Writing `lastRun` early → write it last (step 10).
