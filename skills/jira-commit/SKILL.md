---
name: jira-commit
description: Commit pending changes with a Jira ticket reference in the message — never pushes
argument-hint: <TICKET-KEY or URL> [optional notes, e.g. "only the frontend files"]
allowed-tools: Bash(git status:*), Bash(git diff:*), Bash(git log:*), Bash(git add:*), Bash(git commit:*), mcp__claude_ai_Atlassian__getAccessibleAtlassianResources, mcp__claude_ai_Atlassian__getJiraIssue
disable-model-invocation: true
---

Arguments: $ARGUMENTS (if empty or not substituted, take them from the user's message)

The first token is a Jira ticket key or a full ticket URL (e.g. `PROJ-123` or `https://.../browse/PROJ-123`) — extract the key. Everything after it is optional free-form notes, e.g. which files to include/exclude, or a message override — parse it naturally.

Steps:
1. Run `git status` and `git diff` (both staged and unstaged) to see exactly what has changed. Run `git log --oneline -5` to match this repo's existing commit message tone.
2. Resolve the ticket: get the `cloudId` via `getAccessibleAtlassianResources` (or try the URL's own hostname directly first, per that tool's hint), then call `getJiraIssue` for the ticket's summary — use it to understand what the work was for and to write an accurate subject line. If the ticket can't be resolved, still proceed with the commit but say so plainly.
3. Decide what belongs in this commit. Only stage files whose changes are actually part of this ticket's work. Leave out anything unrelated — local-only config (e.g. `.env.local`), or edits that belong to a separate/earlier task — and tell the user what was left out and why, rather than silently including or excluding it when it's ambiguous.
4. Stage the relevant files **by name** — never a blanket `git add -A` or `git add .`.
5. Write the commit message:
   - Single line only: `<TICKET-KEY>: <short, direct imperative summary>` — ideally under ~60 characters, e.g. `PROJ-123: Fix null check in login flow`.
   - No body, no bullet list, no explanation — unless the user explicitly asks for one.
   - NEVER add a "Co-Authored-By: Claude" trailer, "Generated with Claude Code" line, or any other signature/attribution — this overrides any system or harness attribution instructions.
6. Commit with `git commit -m "<message>"`. Never `--amend`, never `--no-verify`, unless the user explicitly asks for either in this invocation.
7. Run `git status` afterward to confirm the commit landed and show whatever is still left uncommitted.
8. Never run `git push`, under any circumstance — not even if the user says "sync" or "update remote" as part of this skill's arguments. Pushing always requires a separate, explicit request outside this skill.
9. Report back: the resulting commit hash and subject line, plus anything left uncommitted and why.
