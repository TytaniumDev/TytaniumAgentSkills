# Smart Review Polling for Ship Skills

**Date:** 2026-03-16
**Status:** Approved

## Problem

The `ship-it` and `ship-no-merge` skills currently wait an arbitrary fixed time before checking for code review comments. The current approach:

1. `sleep 60` — unconditional 1-minute wait
2. Poll every 60 seconds for up to 5 more minutes
3. Maximum wait: ~6 minutes regardless of how quickly bots respond

This is wasteful when reviews arrive quickly and unreliable as a signal that both reviewers have actually responded.

## Solution

Replace the fixed wait with a signal-based poll: check both the issue comments API and the PR reviews API every 15 seconds, stopping as soon as both `claude[bot]` and `gemini-code-assist[bot]` have posted — or after 5 minutes, whichever comes first.

## Design

### Scope

Update Step 3 ("Wait for Reviews") in both skills:
- `plugins/tytanium-claude/skills/ship-it/SKILL.md`
- `plugins/tytanium-claude/skills/ship-no-merge/SKILL.md`

**Replacement target:** In each file, replace the entire Step 3 block — everything from `### 3. Wait for Reviews` through (but not including) `### 4. Address Review Comments` — with the updated Step 3 text below.

No other steps change. Both files get identical Step 3 content.

### Bot Triggers

Discovered from real PR history (TytaniumDev/Wheelson#62):
- **Claude** (`claude[bot]`): triggered by the `@claude do a code review` comment posted in Step 2
- **Gemini** (`gemini-code-assist[bot]`): auto-triggers on PR creation as a GitHub App — no explicit mention required

No changes to Step 2 are needed.

### Runtime Values

- **PR number:** captured from the `gh pr create` output in Step 1
- **owner/repo:** resolved via `gh repo view --json nameWithOwner --jq '.nameWithOwner'`

### Polling Logic

The loop sleeps *before* each check so the first API call happens at t=15s (giving bots startup time) and the final timeout exits cleanly without an extra dead wait:

1. **Poll loop:** 20 cycles, each starting with `sleep 15` (20 × 15s = 300s = exactly 5 minutes)
2. **Per cycle, check both APIs independently for each bot:**
   - Issue comments: `gh api repos/<owner>/<repo>/issues/<pr>/comments`
   - PR reviews: `gh api repos/<owner>/<repo>/pulls/<pr>/reviews`
3. **Track per-bot:** mark `claude_done=true` if `claude[bot]` appears in either API; `gemini_done=true` if `gemini-code-assist[bot]` appears in either API
4. **Stop condition:** both `claude_done` and `gemini_done` are true — exit the loop immediately
5. **Timeout:** all 20 cycles complete — proceed with whatever exists, and note any bot that did not respond

### Why Both APIs

Bots may post as issue-level comments (observed on Wheelson#62) or as formal GitHub pull request reviews. Checking both ensures neither type is missed.

### jq Filters

Each check returns `true` or `false`:

```
gh api repos/<owner>/<repo>/issues/<pr>/comments \
  --jq 'any(.[]; .user.login == "claude[bot]")'

gh api repos/<owner>/<repo>/issues/<pr>/comments \
  --jq 'any(.[]; .user.login == "gemini-code-assist[bot]")'

gh api repos/<owner>/<repo>/pulls/<pr>/reviews \
  --jq 'any(.[]; .user.login == "claude[bot]")'

gh api repos/<owner>/<repo>/pulls/<pr>/reviews \
  --jq 'any(.[]; .user.login == "gemini-code-assist[bot]")'
```

A bot is considered done if either its issue-comments check OR its reviews check returns `true`.

### Token Cost

`gh api` calls hit GitHub's REST API — no Claude tokens consumed per poll cycle.

## Updated Step 3 Text

Replace the existing `### 3. Wait for Reviews` block in each skill file with:

```markdown
### 3. Wait for Reviews

- Note: Gemini auto-triggers on PR creation; Claude was triggered by the comment in Step 2.
- Get the repo's owner/name: `gh repo view --json nameWithOwner --jq '.nameWithOwner'`
- Poll up to 20 times (sleep 15 seconds before each check, 5 minutes total):
  - Each cycle starts with `sleep 15`, then checks both APIs for each bot:
    - `gh api repos/<owner>/<repo>/issues/<number>/comments --jq 'any(.[]; .user.login == "claude[bot]")'`
    - `gh api repos/<owner>/<repo>/issues/<number>/comments --jq 'any(.[]; .user.login == "gemini-code-assist[bot]")'`
    - `gh api repos/<owner>/<repo>/pulls/<number>/reviews --jq 'any(.[]; .user.login == "claude[bot]")'`
    - `gh api repos/<owner>/<repo>/pulls/<number>/reviews --jq 'any(.[]; .user.login == "gemini-code-assist[bot]")'`
  - Mark `claude_done=true` if either Claude check returns `true`; `gemini_done=true` if either Gemini check returns `true`
  - If both `claude_done` and `gemini_done` are true, stop polling immediately
- If 20 cycles complete without both bots posting, note which bot(s) did not respond and proceed with whatever comments/reviews exist
- Read ALL review comments and formal reviews carefully — from both Claude and Gemini (and any human reviewers)
```

## Non-Goals

- No changes to Step 2 (review triggers)
- No changes to how review comments are addressed (Step 4)
- No changes to merge/CI steps (Step 5)
