---
name: release-update
description: Generate a Slack release-update post for the product-alphas (we.make.com / beta) or product-releases (make.com / live) channels following Make's product-team template. Use this skill whenever the user says "create a release update", "draft a product-alphas post", "announce a feature in Slack", "write a release-notes message", or shares a PR/Jira ticket and asks for the standard announcement. Pulls context from a Jira ticket, GitHub PR, or the current conversation, fills the four template sections (What's new, Why important, When and Who, Learn more), prints the result, and optionally creates a Slack draft.
allowed-tools: Bash, Read, AskUserQuestion
---

# Release Update

Generate a release announcement for `#product-alphas` (features on `we.make.com` that need feedback) or `#product-releases` (features live on `make.com`) using Make's standard product-team template. Print the message for review, then optionally save it as a Slack draft.

Never auto-send — the team requires human review before broadcasting. There has previously been an incident where a `we.make.com`-only feature was announced in `#product-releases` and Marketing posted to LinkedIn before the feature was actually live; this skill exists partly to prevent that.

## Step 1 — Detect the source

Scan the user's invocation arguments **and** their most recent message for:

- A Jira issue key matching `[A-Z]+-\d+` (e.g. `UP-2017`, `MAK-456`)
- A GitHub PR URL matching `https://github.com/[^/]+/[^/]+/pull/\d+`

Then:
- **Exactly one match** → use it directly, skip the picker.
- **Both** → ask the user via `AskUserQuestion` which one to use as the primary source.
- **Neither** → ask via `AskUserQuestion` with these four options:
  1. **Jira ticket** — paste the issue key
  2. **GitHub PR** — paste the URL, or use the current branch's PR (try `gh pr view --json url,title,body` and offer that)
  3. **Current conversation** — summarise the feature we just built together in this session
  4. **From scratch** — fill all fields manually

## Step 2 — Gather raw context

### If source is a Jira ticket

Call `mcp__plugin_make-general_atlassian__getJiraIssue` for the ticket. From the response, extract:

- `summary`
- `description` — look for `As a … I want … so that …` (user story) and `Given … when … then …` (acceptance criteria)
- `status`
- `components`, `labels`
- Any custom fields that look like documentation links

Do **not** use `fixVersions` — Make doesn't use that Jira feature, so it won't be reliable.

### If source is a GitHub PR

Run:
```
gh pr view <url-or-number> --json title,body,commits,headRefName,baseRefName,mergedAt,url
```

If the body, branch, or title contains a Jira key, also fetch that ticket via `getJiraIssue` and merge the context.

### If source is the current conversation

Use what is already in this session's context. Do **not** start searching the codebase unless the user explicitly asks — they just told you what they built.

### If source is "from scratch"

Skip to Step 4 and ask the user for every field.

## Step 3 — Choose target channel(s)

Print this reminder verbatim before asking, so the user can sanity-check their choice:

> Reminder: features only on we.make.com must NOT be posted to #product-releases — Marketing has previously LinkedIn-announced features that weren't live yet. Use #product-alphas for we.make.com / beta, and #product-releases only after it ships to make.com.

Then ask via `AskUserQuestion`:

- `#product-alphas` — feature is on `we.make.com`, want feedback / testing
- `#product-releases` — feature is **live on `make.com`** (production)
- *Both (alphas now, releases later)* — post to alphas now; the same message will be copy-pasted to releases when full release ships

## Step 4 — Fill the template

Render this layout exactly. Use Slack-flavored formatting: single asterisks for bold, `•` for bullets, literal emoji codes (`:new:` etc. — Slack will render them).

```
:new: *What's new?*
• <user benefit 1>
• <user benefit 2>

:fire: *Why is it important?*
*BEFORE:* <how it worked before>
*AFTER:* <how the feature solves it now>

:date: *When and Who?*
*Date of release:* <date or beta+full dates>
*Type of release:* <e.g. beta for X users / full big-bang / phased>
*Audience/Plans:* <Free / Core / Pro / Enterprise / etc.>

:books: *Learn more*
• <documentation link>
• <other supporting assets>
• <dedicated feedback Slack channel>
```

Pre-fill from gathered context when you can:

| Field | Source |
|---|---|
| User benefits | The `so that <outcome>` clause of the Jira user story; PR description bullets |
| BEFORE / AFTER | A `Before:` / `After:` block in the PR description; otherwise `[FILL IN]` |
| Date of release | **Always confirm with the user via `AskUserQuestion`.** Suggest the PR `mergedAt` date as the default option (since that's usually when the feature ships), but never use it without confirmation — release date and merge date are not the same thing for staged rollouts, beta/full splits, or features held behind a flag. |
| Type of release | Inferred from chosen channel (alphas → "beta on we.make.com"; releases → "full release on make.com"); user can override |
| Audience/Plans | Jira labels / components if they encode plan info; otherwise `[FILL IN]` |
| Documentation link | Jira "documentation link" field if present; otherwise `[FILL IN]` |
| Feedback channel | Default to `#product-alphas` for beta, otherwise `[FILL IN]` |

Show the draft once with `[FILL IN]` markers for everything missing, then ask the user to fill the gaps in **a single round** — not a per-field wizard. Accept their reply, render the final version, move on.

## Step 5 — Print, then offer Slack draft

1. Print the final message in a fenced code block so the user can copy-paste it manually:
   ```
   <formatted message>
   ```

2. Ask via `AskUserQuestion`:
   - **Yes — create Slack draft** → for each chosen channel, call `mcp__plugin_slack_slack__slack_send_message_draft` with the message body and the channel name. If the channel ID isn't known, look it up first via `mcp__plugin_slack_slack__slack_search_channels`.
   - **No — print only** → done.

**Never** call `mcp__plugin_slack_slack__slack_send_message` (auto-send). The team requires a human to review the draft inside Slack before it goes out.

## Step 6 — Jira hygiene reminder

Only when the source was a Jira ticket: check whether the description contains **both**

- A line matching `As a .* I want .* so that` (case-insensitive), and
- A line matching `Given .* when .* then` (case-insensitive)

If either is missing, print this nudge after the output (one line, non-blocking):

> Heads up — this Jira ticket is missing the user-story format that Marketing's automation reads (`As a … I want … so that …` / `Given … when … then …`). Consider editing the description so social posts can be generated automatically.

Do **not** auto-edit the ticket — just nudge.

## Done

After printing the message and (optionally) creating Slack drafts, stop. No summary, no extra commentary.
