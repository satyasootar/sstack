---
name: jira-solve
description: Understand a Jira ticket (including images embedded in it), reproduce the problem, write a plain-language plan with a diagram when needed, and after the user approves, implement the smallest possible fix with no code comments
argument-hint: <TICKET-KEY or URL> [optional notes, e.g. "only the frontend", "plan only"]
allowed-tools: mcp__claude_ai_Atlassian__getAccessibleAtlassianResources, mcp__claude_ai_Atlassian__getJiraIssue, mcp__claude_ai_Atlassian__fetch, AskUserQuestion
disable-model-invocation: true
---

Arguments: $ARGUMENTS (if empty or not substituted, take them from the user's message)

The first token is a Jira ticket key or full ticket URL (e.g. `PROJ-123` or `https://.../browse/PROJ-123`); extract the key. Anything after it is optional notes; follow them (e.g. "plan only" means stop after the plan).

Never write code before the user approves the plan.

## 1. Read the whole ticket

1. Resolve the `cloudId` via `getAccessibleAtlassianResources` (try the URL's own hostname first if one was given).
2. Call `getJiraIssue` with fields `summary`, `description`, `issuetype`, `priority`, `status`, `components`, `versions`, `fixVersions`, `attachment`, `comment`, `issuelinks`, `parent`, `subtasks`, `environment`, `labels`.
3. Read the description, **all comments** (later comments often correct the description), the parent ticket, and linked tickets that look related (blocks / is blocked by / duplicates / relates to — fetch them with `getJiraIssue`, summary + description only).

## 2. Get the images

Screenshots usually hold the most important detail: the exact error, the wrong value, which screen.

1. Find every image: media/inline images in the description and comments, and image files in the `attachment` field (`image/*` mime type, or `.png .jpg .jpeg .gif .webp`). Match inline references to attachments by filename or id.
2. Try to fetch each one through the Atlassian tools (`fetch` with the attachment's `content` URL or id). If you get the image, look at it.
3. If an image can't be fetched (tools return no binary, auth fails), **don't guess what it shows**. List the missing images by filename and ask the user to paste them into the chat, or save them to a path you can read. Continue reading the code while you wait, but don't finalize the plan without them unless the user says to skip.
4. For each image, write down what it actually shows: screen/page, the visible error text (copy it exactly), wrong vs. expected values, URL bar, console/network panel contents, timestamps.

Videos and other attachments: note them, and ask the user if one seems essential.

## 3. Understand the problem

Restate the ticket in your own words before touching code:

- **What the user sees** (bug) or **what is wanted** (feature/change).
- **Expected vs. actual**, from the description, comments, and images.
- **Where:** screen, endpoint, job, or command; environment and version if given.
- **Acceptance:** how we'll know it's done. Use the ticket's acceptance criteria if present; otherwise derive them and mark them as your assumption.

Then find the code: search for the exact error text, UI labels, route paths, field names, and component names seen in the ticket and screenshots. Follow the flow end to end (UI → API → service → DB, or whatever applies) and read every file the fix could touch.

## 4. Reproduce it

Prove the problem exists before planning a fix, using the cheapest reliable method:

1. A failing automated test that captures the bug, following the repo's existing test style. This is preferred: it doubles as the regression test.
2. Otherwise run it locally: the dev server, a script, a CLI call, or the existing test suite, with dummy data.
3. Otherwise trace it statically: point to the exact lines and the input that produces the wrong result.

Rules: never reproduce against production or shared environments, never use real customer data, never install new dependencies to reproduce.

If you **can't** reproduce it, say so plainly with what you tried. Don't invent a cause; list the facts you'd need (logs, exact steps, account/config) and ask.

For a feature or change ticket, "reproduce" means confirming the current behavior that has to change.

## 5. Draft the plan

Write it for a person who didn't read the code. Short sentences, plain words, no jargon without a one-line explanation. Use this structure:

```
## <TICKET-KEY>: <ticket summary>

### What's wrong (in plain words)
<2–4 sentences a non-developer could follow. What the user sees, and what should happen instead.>

### What the screenshots show
- <image name>: <what it shows, exact error text>

### Why it happens
<The root cause in simple terms, then the precise spot: path/to/file.ext:line.>
<How it was reproduced: test name / steps / trace, and the result.>

### The fix
<What will change, as a numbered list. Each item: file, and what changes in one sentence.>
<Why this is the smallest fix that solves the root cause.>

### How we'll check it works
- <the reproduction now passes / shows correct behavior>
- <existing tests that must still pass>

### Not changing
- <nearby things deliberately left alone, so the change stays small>

### Risks / questions
- <anything uncertain, or "none">
```

**Add a diagram when the bug is complex:** it crosses several files or services, involves ordering/timing (races, retries, async), or involves state changes. Use a Mermaid diagram (`sequenceDiagram` for who-calls-whom and timing, `flowchart` for branching logic, `stateDiagram-v2` for status changes). Show where it breaks with a clear label such as `❌ bug: tenant id not checked here`. Keep it under ~15 nodes. Simple, single-file bugs don't need one.

## 6. Get approval

Show the plan, then ask via `AskUserQuestion`: "Go ahead with this plan?" with options **"Go ahead"** / **"Change the plan"** / **"Plan only, stop here"**. If they want changes, revise and ask again. Do not edit any file until the answer is "Go ahead".

## 7. Implement

Change exactly what the approved plan says, nothing more:

- **Minimal:** the fewest lines that fix the root cause. Fix it once, in the shared place all callers go through, not with a patch in each caller.
- **No over-engineering:** no new abstractions, helpers, config options, feature flags, or "while I'm here" refactors. No new dependencies.
- **No comments:** don't add code comments, TODOs, or docstrings. Don't remove or edit existing comments unless the plan says so.
- **Match the surrounding code:** same naming, formatting, error-handling style, and patterns as the file you're editing.
- **Don't touch unrelated code:** no reformatting, import reordering, or renames outside the fix.

If the code turns out different from what the plan assumed, stop and tell the user what changed and the adjusted plan before continuing.

## 8. Verify and report

1. Re-run the reproduction from step 4; it must now show the correct behavior. If you wrote a failing test, it now passes and stays as the regression test.
2. Run the existing tests for the touched modules, plus the project's lint/typecheck command if one exists.
3. Report:

```
<TICKET-KEY>: done
Changed: <file:line — one line each>
Verified: <reproduction result> · <tests/lint/typecheck result>
Not done: <anything from the plan skipped, or "nothing">
```

Don't commit, push, or change the ticket's status. Suggest `/jira-commit <TICKET-KEY>` to commit and `/jira-log <TICKET-KEY>` to log the work.
