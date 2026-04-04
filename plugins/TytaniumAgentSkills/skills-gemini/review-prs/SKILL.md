---
name: review-prs
description: Review all open PRs in the current repo, triage for quality, fix issues, and enable automerge. Use this when the user wants to review all open PRs.
---

# Review PRs: Triage, Fix, and Automerge

Review every open non-draft PR in the current repo. For each PR: rebase onto main, run quality checks, fix CI/review issues, and enable automerge if worthy. Present a summary at the end.

## Step 1: List Open PRs

Run:
```
gh pr list --state open --draft=false --json number,title,headRefName,baseRefName,url --limit 100
```

If no PRs are returned, report "No open non-draft PRs found" and stop.

Save the list. You will need `number`, `title`, `headRefName`, `baseRefName`, and `url` for each PR.

## Step 2: Get Repo Info

Run:
```
gh repo view --json nameWithOwner --jq '.nameWithOwner'
```

Split on `/` to get `OWNER` and `REPO`. You will need these for API calls.

## Step 3: Process Each PR Sequentially

For EVERY PR from Step 1, process them one at a time following the Per-PR Instructions section below. Work through each PR completely before starting the next one.

For each PR, collect a structured result containing:
- `pr_number`, `pr_title`, `pr_url`
- `status`: one of `automerge_enabled`, `not_worthy`, `needs_human_attention`
- `reason`: short explanation

## Step 4: Present Summary

After ALL PRs are processed, present a summary to the user:

```
## PR Review Summary -- <OWNER>/<REPO>

### Automerge Enabled (N)
- #<number> -- "<title>" -- <reason> -- <url>

### Not Worthy (N)
- #<number> -- "<title>" -- <reason> -- <url>

### Needs Human Attention (N)
- #<number> -- "<title>" -- <reason> -- <url>
```

If a section has 0 items, omit it.

---

## Per-PR Instructions

**IMPORTANT:** Follow these instructions for each PR, substituting the PR-specific values.

You are processing PR #<NUMBER> ("<TITLE>") on branch `<BRANCH>` targeting `<BASE_BRANCH>` in <OWNER>/<REPO>.
URL: <URL>

Your job: rebase this PR onto <BASE_BRANCH>, run quality checks, fix any issues if worthy, and enable automerge. Record a structured result.

### Phase 1: Rebase onto Base Branch

1. Check out the PR branch:
   ```
   git checkout <BRANCH>
   ```

2. Fetch latest base branch:
   ```
   git fetch origin <BASE_BRANCH>
   ```

3. Attempt rebase:
   ```
   git rebase origin/<BASE_BRANCH>
   ```

4. If rebase succeeds with no conflicts, force-push:
   ```
   git push --force-with-lease
   ```

5. If rebase fails due to merge conflicts, resolve them directly:
   1. Run `git rebase --abort` to reset
   2. Run `git merge origin/<BASE_BRANCH>` instead
   3. For each conflicted file, use `read_file` to read the file, understand both sides, and resolve the conflict sensibly using `replace` or `write_file`
   4. After resolving all conflicts, run `git add .` and `git commit -m "chore: merge <BASE_BRANCH> into <BRANCH>"`
   5. Run `git push --force-with-lease`

   If the conflicts cannot be resolved automatically:
   - Comment on the PR: `gh pr comment <NUMBER> --body "Unable to automatically resolve merge conflicts with <BASE_BRANCH>. Manual resolution needed."`
   - Record: `{ status: "needs_human_attention", reason: "Unresolvable merge conflicts" }`
   - Move on to the next PR.

### Phase 2: Quality Gate

Run all three checks. A PR must pass ALL THREE to be worthy.

**Check 1: Code Review Quality**

Read the full PR diff:
```
gh pr diff <NUMBER>
```

Evaluate holistically: Do the changes make sense? Is the code correct, reasonably clean, and not introducing obvious bugs or security issues? Would a competent reviewer approve this?

**Check 2: No Duplication of Existing Functionality**

Look at what the PR adds (new functions, components, utilities, etc). For each significant addition, search the existing codebase using `run_shell_command` with `rg` or `find` to check if similar functionality already exists. Flag if the PR reimplements something that's already there.

**Check 3: No CI Naming Changes**

First check if any workflow files are changed. If none, this check passes. If workflow files are changed, read the full diff of those files and check for `name:` field modifications:
```
gh pr diff <NUMBER> --name-only | grep '.github/workflows/' || true
```

If workflow files are changed, check specifically for modifications to `name:` fields on jobs or workflows. Renaming CI jobs or workflows is an automatic rejection — it breaks existing automations. Adding new workflows or changing non-name fields is fine.

**If any check fails:**
- Post a review comment explaining which check(s) failed and why:
  ```
  gh pr comment <NUMBER> --body "<explanation of which checks failed and why>"
  ```
- Record: `{ status: "not_worthy", reason: "<brief summary of failures>" }`
- Move on to the next PR.

### Phase 3: Fix Issues (if worthy)

After passing the quality gate, check for issues that need fixing.

**Check CI status:**
```
gh pr checks <NUMBER>
```

**Check for CHANGES_REQUESTED reviews:**
```
gh api repos/<OWNER>/<REPO>/pulls/<NUMBER>/reviews --jq '[.[] | select(.state == "CHANGES_REQUESTED")] | length'
```

If CI is green AND there are no `CHANGES_REQUESTED` reviews, skip to Phase 4.

If there are issues to fix, work through up to 5 fix-push cycles directly:

For each cycle:
1. Read CI failure logs: `gh run list --branch <BRANCH> --limit 1 --json databaseId --jq '.[0].databaseId'` then `gh run view <RUN_ID> --log-failed`
2. Read unresolved review comments: `gh api repos/<OWNER>/<REPO>/pulls/<NUMBER>/comments`
3. Fix the issues in the code using `read_file`, `replace`, and `write_file` tools
4. Commit and push:
   ```
   git add <files you modified>
   git commit -m "fix: address CI failures and review feedback"
   git push
   ```
   Stage only the files you modified. Do NOT use `git add -A` or `git add .` — this skill runs across arbitrary repos and could accidentally stage sensitive files.
5. Wait 30 seconds for CI to start, then poll CI status up to 20 times (sleep 15s each):
   ```
   gh pr checks <NUMBER>
   ```
   Stop polling when all checks complete (pass or fail).
6. If CI passes and no more unresolved comments, proceed to Phase 4.
7. If CI still fails, start the next cycle.

After 5 failed cycles:
- Comment on the PR:
  ```
  gh pr comment <NUMBER> --body "<summary of what was tried and what's still broken>"
  ```
- Record: `{ status: "needs_human_attention", reason: "CI still failing after 5 fix attempts" }`
- Move on to the next PR.

### Phase 4: Enable Automerge

```
gh pr merge <NUMBER> --auto --squash
```

If automerge fails (e.g., the repo has automerge disabled), fall back to reporting the PR as ready for manual merge in the summary. Record: `{ status: "automerge_enabled", reason: "CI green, ready for merge (automerge unavailable — manual merge needed)" }`

If automerge succeeds, record: `{ status: "automerge_enabled", reason: "<brief note, e.g. CI green / fixed N issues>" }`
