---
name: create-jira-ticket
description: >
  Create a Jira ticket (Bug, Task, Sub-task, or Story) in the BAR project.
  Use this skill whenever the user says "create a jira ticket", "file a ticket",
  "open a jira", "create a bug", "create a task", "create a story", "ticket this up",
  or "create a retrospective ticket for this". Gathers context from the current
  conversation, git history, and any linked Jira ticket or PR. Drafts the ticket
  and shows it for review before creating it in Jira.
allowed-tools: Bash, AskUserQuestion, mcp__plugin_make-general_atlassian__createJiraIssue, mcp__plugin_make-general_atlassian__getJiraIssue, mcp__plugin_make-general_atlassian__searchJiraIssuesUsingJql
---

# Create Jira Ticket

## Step 1 — Gather context

Collect as much signal as possible **before asking the user anything**.

**From git (run in parallel):**
```
git log --oneline -10
git diff HEAD~1..HEAD --stat
git branch --show-current
```

**From the current conversation:** read what has already been discussed — problem descriptions, PR links, ticket IDs, implemented features.

**From linked references:** if the conversation mentions a `BAR-XXX` ticket ID or GitHub PR URL, fetch it:
- Jira ticket: `mcp__plugin_make-general_atlassian__getJiraIssue` with the issue key
- GitHub PR: `gh pr view <number-or-url> --json title,body,commits`

After gathering, silently synthesise a draft understanding of: the problem, any known solution, and the likely ticket type.

## Step 2 — Determine ticket type

If the type is clear from context (e.g. "something is broken" → Bug, "new feature" → Story, "refactor/chore" → Task), proceed with that type.

If ambiguous, ask once:

```
AskUserQuestion: "What type of ticket is this?"
Options: Bug | Task | Story | Sub-task
```

If **Bug** is selected, ask:

```
AskUserQuestion: "Use bug-specific format (steps to reproduce / actual / expected behavior)?"
Options:
  - Yes — steps to reproduce + actual/expected
  - No — standard Problem/Solution bullets
```

## Step 3 — Resolve parent ticket (Sub-tasks only)

If the type is Sub-task:
1. Look for a `BAR-XXX` reference in the current conversation or linked PR.
2. If found, propose: "I'll link this as a sub-task under BAR-XXX — confirm?"
3. If not found, ask once: "Parent ticket ID? (or skip to create as a standalone Task)"
4. If skipped, create as a **Task** instead.

## Step 4 — Ask about Epic link

Ask once:

```
AskUserQuestion: "Link to an epic? Provide a BAR-XXX ID or name, or skip."
```

If skipped, proceed without an epic link. Do not search Jira proactively.

## Step 5 — Draft the ticket

Generate a title and description following the rules below.

---

### Title
- Verb-first imperative phrase
- No ticket ID prefix
- Under 72 characters
- Examples: `Add bulk export to scenario logs` · `Fix null-check on session refresh` · `Migrate auth middleware to JWT`

---

### Description formats

**Standard (Bug with standard format, Task, Story):**
```
## Problem
- <present-tense gap — what doesn't exist or is broken>
- <additional bullet if needed>

## Solution
- <what was or will be done>
- <additional bullet if needed>
```
Omit `## Solution` entirely if no solution is known.

**Bug with bug-specific format:**
```
## Problem

**Steps to reproduce:**
- <step 1>
- <step 2>

**Actual behavior:**
- <what currently happens>

**Expected behavior:**
- <what should happen>

## Solution
- <fix, if known>
```
Omit `## Solution` entirely if no fix is known.

**Story (additional sections):**
```
As a <persona>, I want <X> so that <Y>.

## Problem
- <gap or pain point>

## Solution
- <feature or change>

## Acceptance Criteria
- [ ] <testable condition>
- [ ] <edge case handled>
```

---

### Retrospective framing rule
When the ticket covers already-implemented work, write the **Problem** section in present-tense gap framing — as if describing the gap that existed before the work:
- ✅ `Users have no way to bulk-export scenario logs`
- ❌ `We implemented bulk export because users couldn't export`

The Solution section can use natural past or present tense.

## Step 6 — Present draft for review

Print the full draft clearly:

```
---
Type:    <Bug | Task | Sub-task | Story>
Project: BAR
Epic:    <BAR-XXX | none>
Parent:  <BAR-XXX | n/a>

Title: <title>

<description>
---
```

Then ask:

```
AskUserQuestion: "Does this look right?"
Options:
  - Create it — submit to Jira as-is
  - Change something — describe what to adjust
```

If the user selects "Change something", incorporate their feedback and repeat Step 6. Loop until they approve.

## Step 7 — Confirm project

Before creating, confirm:

```
AskUserQuestion: "Create this in project BAR?"
Options:
  - Yes — BAR
  - Different project — specify key
```

## Step 8 — Create the ticket

Call `mcp__plugin_make-general_atlassian__createJiraIssue` with:
- `projectKey`: `BAR` (or user-specified)
- `issueType`: the selected type
- `summary`: the title
- `description`: the formatted description (ADF or markdown as accepted by the MCP)
- `parent`: parent issue key if Sub-task
- `epic`: epic link if provided

## Step 9 — Done

Print the created ticket ID and URL. Nothing else.

Example:
```
BAR-456: https://make.atlassian.net/browse/BAR-456
```
