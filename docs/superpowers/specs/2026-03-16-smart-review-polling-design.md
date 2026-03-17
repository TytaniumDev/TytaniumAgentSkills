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

No other steps change. Both files get identical Step 3 content.

### Bot Usernames

Discovered from real PR history (TytaniumDev/Wheelson#62):
- Claude: `claude[bot]`
- Gemini: `gemini-code-assist[bot]`

### Polling Logic

1. **Initial sleep:** `sleep 15` — give bots time to trigger before first check
2. **Poll loop:** every 15 seconds, up to 5 minutes total elapsed
3. **Per cycle, check both APIs:**
   - Issue comments: `gh api repos/{owner}/{repo}/issues/<pr>/comments`
   - PR reviews: `gh api repos/{owner}/{repo}/pulls/<pr>/reviews`
4. **Filter each response** for entries where `user.login` is `claude[bot]` or `gemini-code-assist[bot]`
5. **Stop condition:** both bot usernames found in the combined results from either API
6. **Timeout condition:** 5 minutes elapsed — proceed with whatever comments exist

### Why Both APIs

Bots may post as issue-level comments (observed on Wheelson#62) or as formal GitHub pull request reviews. Checking both ensures neither type is missed.

### jq Filters

Extract bot usernames from issue comments:
```
gh api repos/{owner}/{repo}/issues/<pr>/comments \
  --jq '[.[] | select(.user.login == "claude[bot]" or .user.login == "gemini-code-assist[bot]")] | map(.user.login) | unique'
```

Extract bot usernames from PR reviews:
```
gh api repos/{owner}/{repo}/pulls/<pr>/reviews \
  --jq '[.[] | select(.user.login == "claude[bot]" or .user.login == "gemini-code-assist[bot]")] | map(.user.login) | unique'
```

Merge and deduplicate the two arrays in subsequent logic. When the combined unique set equals `["claude[bot]", "gemini-code-assist[bot]"]` (2 entries), stop polling.

### Token Cost

`gh api` calls hit GitHub's REST API — no Claude tokens consumed per poll cycle.

## Updated Step 3 Text

```markdown
### 3. Wait for Reviews

- Sleep 15 seconds to give bots time to trigger: `sleep 15`
- Then poll every 15 seconds for up to 5 minutes total, checking both APIs each cycle:
  - Issue comments: `gh api repos/{owner}/{repo}/issues/<number>/comments --jq '[.[] | select(.user.login == "claude[bot]" or .user.login == "gemini-code-assist[bot]")] | map(.user.login) | unique'`
  - PR reviews: `gh api repos/{owner}/{repo}/pulls/<number>/reviews --jq '[.[] | select(.user.login == "claude[bot]" or .user.login == "gemini-code-assist[bot]")] | map(.user.login) | unique'`
- Combine and deduplicate the results from both API calls
- Stop polling as soon as both `claude[bot]` and `gemini-code-assist[bot]` appear in the combined results
- If 5 minutes elapse before both have posted, proceed with whatever comments/reviews exist
- Once done waiting, read ALL review comments and formal reviews carefully — from both Claude and Gemini (and any human reviewers)
```

## Non-Goals

- No changes to any other step in either skill
- No changes to how review comments are addressed (Step 4)
- No changes to merge/CI steps (Step 5)
