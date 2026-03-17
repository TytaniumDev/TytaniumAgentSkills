# Smart Review Polling Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the arbitrary fixed sleep in both ship skills with signal-based polling that stops as soon as both `claude[bot]` and `gemini-code-assist[bot]` have posted their reviews.

**Architecture:** Step 3 ("Wait for Reviews") in `ship-it/SKILL.md` and `ship-no-merge/SKILL.md` is replaced with a 20-cycle poll loop. Each cycle sleeps 15s first, then checks two GitHub APIs (issue comments + PR reviews) for each bot independently. The loop exits early when both bots have posted, or proceeds after 5 minutes with whatever exists.

**Tech Stack:** Markdown skill files, `gh` CLI (GitHub REST API), `jq`

**Spec:** `docs/superpowers/specs/2026-03-16-smart-review-polling-design.md`

---

## File Structure

| File | Action | What changes |
|------|--------|--------------|
| `plugins/tytanium-claude/skills/ship-it/SKILL.md` | Modify | Replace Step 3 block |
| `plugins/tytanium-claude/skills/ship-no-merge/SKILL.md` | Modify | Replace Step 3 block (identical content) |

No new files. No other files touched.

---

## Chunk 1: Update both skill files

> Note: These are declarative Markdown files — not code. There is no test suite. Verification is manual inspection of the diff.

### Task 1: Update `ship-it/SKILL.md`

**Files:**
- Modify: `plugins/tytanium-claude/skills/ship-it/SKILL.md`

- [ ] **Step 1: Read the current file**

```bash
cat plugins/tytanium-claude/skills/ship-it/SKILL.md
```

Locate the Step 3 block. It currently starts with:
```
### 3. Wait for Reviews

- **You MUST wait at least 1 minute** before checking for reviews.
```
and ends just before `### 4. Address Review Comments`.

- [ ] **Step 2: Replace Step 3**

Replace everything from `### 3. Wait for Reviews` through (but not including) `### 4. Address Review Comments` with:

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

- [ ] **Step 3: Verify the file looks correct**

```bash
cat plugins/tytanium-claude/skills/ship-it/SKILL.md
```

Expected: Step 3 now contains the poll loop. Steps 1, 2, 4, and 5 are unchanged.

---

### Task 2: Update `ship-no-merge/SKILL.md`

**Files:**
- Modify: `plugins/tytanium-claude/skills/ship-no-merge/SKILL.md`

- [ ] **Step 1: Apply the identical Step 3 replacement**

Same replacement as Task 1. The only structural difference in `ship-no-merge` is that Step 5 says "report the PR URL" instead of "merge" — Step 3 content is identical.

- [ ] **Step 2: Verify the file looks correct**

```bash
cat plugins/tytanium-claude/skills/ship-no-merge/SKILL.md
```

Expected: Step 3 matches the ship-it version exactly. Steps 1, 2, 4, and 5 are unchanged.

---

### Task 3: Commit

- [ ] **Step 1: Review the diff**

```bash
git diff plugins/
```

Confirm: only Step 3 changed in each file, nothing else.

- [ ] **Step 2: Commit**

```bash
git add plugins/tytanium-claude/skills/ship-it/SKILL.md \
        plugins/tytanium-claude/skills/ship-no-merge/SKILL.md
git commit -m "feat: replace fixed sleep with smart bot-detection polling in ship skills"
```
