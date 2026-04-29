---
name: create-pr
description: Create a GitHub pull request following Make conventions. Use this skill whenever the user says "create PR", "open PR", "make a PR", "submit a PR", "open a pull request", or similar. Generates a conventional-commit title, auto-summarises changes from commits, and includes a Jira ticket reference in the description.
allowed-tools: Bash, AskUserQuestion
---

# Create PR

Create a GitHub pull request following Make engineering conventions.

## Step 1 — Guard: existing PR

Run:
```
gh pr view --json url,title 2>/dev/null
```
If the command succeeds (exit 0), a PR already exists for this branch. Print the URL and **stop** — do not proceed further.

## Step 2 — Discover Jira ticket

Try in order:
1. `git branch --show-current` → parse for `[A-Z]+-\d+` pattern (e.g. `MAK-123`)
2. `git log --oneline -20` → scan each line for the same pattern
3. If still not found: ask the user for the ticket ID using AskUserQuestion

If the user skips or provides nothing: print a warning ("No Jira ticket found — description will use a placeholder URL") and set ticket to `TODO`. Continue.

## Step 3 — Read commit history

Run:
```
git log --oneline origin/HEAD..HEAD 2>/dev/null || git log --oneline -15
```
Use **commit messages only** (no diff) as the basis for all generated content.

## Step 4 — Ask for branch name

Derive a suggested branch name from the commits:
- Format: `{type}/{JIRA-ID}-{short-slug}` — all lowercase, dash-separated words
- Example: `feat/MAK-123-oauth2-login-flow`
- Use the current branch name as the suggestion if it already follows this convention

Ask the user to confirm or change the branch name using AskUserQuestion.

## Step 5 — Push if needed

Check whether the current branch has a remote tracking branch:
```
git rev-parse --abbrev-ref --symbolic-full-name @{u} 2>/dev/null
```
If this fails (no upstream), run:
```
git push -u origin <branch-name>
```
Do this silently — no confirmation needed.

## Step 6 — Generate PR title

Format: `type(scope): short description`

Rules:
- Derive type and short description from the commit messages
- Valid types: `feat`, `fix`, `chore`, `refactor`, `perf`, `docs`, `test`, `ci`, `build`, `revert`
- Scope is **optional** — include it only when clearly applicable (e.g. a changed module/package/area)
- No ticket ID in the title
- Keep it under 72 characters

Examples:
- `feat(auth): add OAuth2 login flow`
- `fix: handle null user on session expiry`
- `chore(deps): upgrade lodash to 4.17.21`

## Step 7 — Generate PR description

Write a single paragraph (2–4 sentences) that:
- Describes the problem or motivation
- Explains how it was solved

Base it entirely on the commit messages — do not read the file diff.

Append the Jira URL on its own line after the paragraph:
```
https://make.atlassian.net/browse/{JIRA-ID}
```
If no ticket was found, use:
```
https://make.atlassian.net/browse/TODO
```

Full description example:
```
Users were being silently logged out after token refresh failed due to a missing null-check on the session object. This PR adds a guard clause that falls back to re-authentication instead of dropping the session. The fix is covered by a new integration test for the token refresh path.

https://make.atlassian.net/browse/MAK-456
```

## Step 8 — Ask: draft or ready for review?

Ask the user using AskUserQuestion:
- "Ready for review" → create as a normal open PR
- "Draft" → add `--draft` flag

## Step 9 — Detect base branch

Run:
```
gh repo view --json defaultBranchRef --jq '.defaultBranchRef.name'
```

## Step 10 — Create the PR

Run `gh pr create` with:
- `--title` — the generated title (Step 6)
- `--body` — the formatted description (Step 7)
- `--base` — the default branch (Step 9)
- `--draft` — only if the user chose draft (Step 8)

## Step 11 — Done

Print the PR URL returned by `gh pr create`. Nothing else.
