---
name: ship-it
description: Create a PR, wait for automatic code reviews, address all review comments, and merge.
disable-model-invocation: true
argument-hint: [optional description of changes]
allowed-tools:
  - Bash(gh *)
  - Bash(git *)
  - Read
  - Grep
  - Glob
  - Edit
  - Write
---

# Ship It: PR + Review + Merge Workflow

Complete the end-of-work shipping workflow for the current branch.

## Context

$ARGUMENTS

## Steps

### 1. Create the Pull Request

- Run `git status` and `git log main..HEAD` to understand what's being shipped
- Create a PR targeting `main` using `gh pr create`:
  - Title: follow conventional commits (`feat:`, `fix:`, `chore:`)
  - Body: include summary bullets, test plan, and the Claude Code attribution line
  - If there is a related GitHub issue, include `Closes #N` in the body
- Capture the PR number from the output

### 2. Wait for Reviews

- Code reviews from Claude and Gemini are triggered automatically when the PR is created
- Poll for review comments using `gh pr view <number> --json reviews,comments` and `gh api repos/{owner}/{repo}/pulls/<number>/comments`
- Wait until at least one substantive review has been posted (check every 30 seconds, up to 10 minutes)
- Once reviews arrive, read ALL review comments carefully — from both Claude and Gemini (and any human reviewers)

### 3. Address Review Comments

- For each review comment or suggestion:
  - Read the relevant code in context
  - Make the requested fix or improvement (use Edit/Write tools)
  - If a suggestion doesn't make sense or would be harmful, explain why in a reply comment using `gh pr comment`
- After all changes are made, commit with a message like `chore: address review feedback`
- Push the changes with `git push`

### 4. Verify and Merge

- Check that CI is passing: `gh pr checks <number>`
  - If CI is failing, investigate and fix the issues, commit, and push again
- Once CI passes and review feedback is addressed, merge the PR:
  ```
  gh pr merge <number> --squash --delete-branch
  ```
- Report the final merged PR URL
