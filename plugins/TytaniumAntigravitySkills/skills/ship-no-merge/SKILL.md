---
name: ship-no-merge
description: Create a PR, request code reviews from Gemini, and address all review comments (no merge). Use this when the user wants to prepare a PR and address feedback but keep it open.
---

# Ship (No Merge): PR + Review Workflow

Complete the end-of-work shipping workflow for the current branch, but leave the PR open for manual merge.

## Steps

### 1. Create the Pull Request

- Run `git status` and `git log main..HEAD` to understand what's being shipped.
- Create a PR targeting `main` using `gh pr create`:
  - Title: follow conventional commits (`feat:`, `fix:`, `chore:`)
  - Body: include summary bullets, test plan, and a note that this PR was prepared by an AI assistant.
  - If there is a related GitHub issue, include `Closes #N` in the body.
- Capture the PR number from the output.

### 2. Wait for Reviews

- Note: Gemini auto-triggers on PR creation.
- Get the repo's owner/name: `gh repo view --json nameWithOwner --jq '.nameWithOwner'`
- Substitute the PR number (from Step 1) for `<number>`, and split the `nameWithOwner` output on `/` to get `<owner>` and `<repo>` for all API paths below.
- Poll up to 20 times (sleep 15 seconds before each check, 5 minutes total):
  - Each cycle starts with `sleep 15`, then checks both APIs for the Gemini bot using `run_command`:
    - `gh api repos/<owner>/<repo>/issues/<number>/comments --jq 'any(.[]; .user.login == "gemini-code-assist[bot]")'`
    - `gh api repos/<owner>/<repo>/pulls/<number>/reviews --jq 'any(.[]; .user.login == "gemini-code-assist[bot]")'`
  - Mark `gemini_done=true` if either Gemini check returns `true`.
  - If `gemini_done` is true, stop polling immediately.
- If 20 cycles complete without the Gemini bot posting, note that Gemini did not respond and proceed with whatever comments/reviews exist.
- Read ALL review comments and formal reviews carefully — from Gemini (and any human reviewers) using `view_file` or `run_command` (e.g. `gh pr view`).

### 3. Address Review Comments

- For each review comment or suggestion:
  - Read the relevant code in context using `view_file`.
  - Make the requested fix or improvement using `replace_file_content` or `write_to_file`.
  - If a suggestion doesn't make sense or would be harmful, explain why in a reply comment using `gh pr comment`.
- After all changes are made, commit with a message like `chore: address review feedback`.
- Push the changes with `git push`.

### 4. Verify CI and Report

- Check that CI is passing: `gh pr checks <number>`.
  - If CI is failing, investigate and fix the issues, commit, and push again.
- Once CI passes and review feedback is addressed, report the PR URL and note that it is ready for manual merge.
- Do NOT merge the PR.
