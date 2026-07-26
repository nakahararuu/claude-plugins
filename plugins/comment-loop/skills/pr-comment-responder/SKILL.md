---
name: pr-comment-responder
description: Monitor the GitHub PR for the current branch (auto-detected via `gh pr view` — no URL, argument-less) for new unaddressed comments and respond — answering questions, making code fixes with commits, and posting replies. Use when the user asks to "watch a PR", "babysit a PR", "respond to PR comments", "monitor PR activity", or "keep an eye on a PR". Works great with /loop for continuous monitoring. All replies are signed "by ClaudeCode" so humans can distinguish them, and fix replies include the commit link.
---

# PR Comment Responder

Monitor a GitHub Pull Request for unaddressed comments, respond to questions, make code fixes, and post replies — all signed as ClaudeCode.

## Identifying the PR

The target is always the PR associated with the branch currently checked out in this session — fixes and replies only make sense for a PR you can actually push commits to, and that's the one under your feet. There is no way to target a different PR; mixing "the branch you can commit to" with "the PR named in the prompt" makes the resulting behavior unpredictable, so don't accept a URL override.

Derive it automatically:

```bash
gh pr view --json url,number,headRepositoryOwner,baseRepository \
  --jq '{url, number, owner: .headRepositoryOwner.login, repo: .baseRepository.name}'
```

From the result, derive:
- `owner` and `repo` from the GitHub URL path (or the `owner`/`repo` fields above)
- `pr_number` from the URL (or the `number` field above)

If this fails (no PR open for the current branch, detached HEAD, etc.), report the error to the user instead of guessing.

## Step 1: Fetch all comments

Fetch both types in parallel:

```bash
# Inline review comments
gh api "repos/{owner}/{repo}/pulls/{pr_number}/comments" \
  --jq '.[] | {id, body, created_at, in_reply_to_id, path, line}'

# General PR discussion comments
gh api "repos/{owner}/{repo}/issues/{pr_number}/comments" \
  --jq '.[] | {id, body, created_at}'
```

Always fetch the **full body** — never rely on truncated previews.

## Step 2: Find unaddressed comments

A comment is **unaddressed** if it has no subsequent "by ClaudeCode" reply.

**For issue comments** (linear thread):
- Sort by `created_at`
- A comment is addressed if any later comment in the list contains "by ClaudeCode"

**For review comments** (threaded):
- A comment is addressed if any comment with `in_reply_to_id == this_comment.id` contains "by ClaudeCode"
- Also skip root comments (no `in_reply_to_id`) that are followed by a reply from ClaudeCode

**Always skip:**
- Comments you wrote (body contains "by ClaudeCode")
- Comments already addressed as above

## Step 3: Process each unaddressed comment

Read the full body carefully before responding.

### If it's a question or explanation request

Draft a clear, direct answer and post it immediately.

### If it's a code fix request (keyword signals: "駄目", "エラー", "失敗", "修正", [must], bug, error, fail)

1. **Understand first** — read the error output fully, reproduce locally if possible (e.g., `docker run --rm ...` using the hint in the error message)
2. **Fix the code**
3. **Commit and push:**
   ```bash
   git add <changed-files>
   git commit -m "fix(...): <description>"
   git push origin <branch>
   ```
4. **Get the commit SHA:** `git rev-parse --short HEAD`
5. **Reply with the commit link** (see reply format below)

## Step 4: Post replies

### Reply format

Every reply must:
- Directly address the comment's content (don't just say "fixed")
- End with `— by ClaudeCode`
- Include commit info when a fix was made

**For a code fix reply:**
```
{explanation of root cause and what was changed}

Fixed in commit `{sha}` (https://github.com/{owner}/{repo}/commit/{full_sha}) — by ClaudeCode
```

**For a question reply:**
```
{answer}

— by ClaudeCode
```

### Posting to GitHub

**Reply to a review comment** (try first):
```bash
gh api repos/{owner}/{repo}/pulls/{pr_number}/comments/{comment_id}/replies \
  -X POST \
  -F body="$(cat /tmp/reply_body.txt)"
```

If that returns 404, fall back to a general PR comment:
```bash
gh issue comment {pr_number} \
  --body "$(cat /tmp/reply_body.txt)"
```

**Important**: Write the reply body to a temp file first (e.g., `/tmp/reply_body.txt`) to avoid bash interpreting backticks or special characters in the message string.

## Handling ambiguous comments

If a comment is unclear, post a clarifying question rather than guessing:
```
{clarifying question}

— by ClaudeCode
```

## When used with /loop

On each invocation, the "already addressed" check (Step 2) is the de-duplication mechanism — you naturally skip comments that already have a ClaudeCode reply. No separate state tracking is needed.

After handling all new comments, report briefly: how many comments were checked, how many were addressed, what actions were taken.

## Tips

- When multiple fixes are needed, handle them in a single commit if they're related; use separate commits if they're independent
- If a fix requires validation (e.g., testing in Docker), test before committing — don't commit untested fixes
- Don't respond to your own comments; skip anything containing "by ClaudeCode"
- If the PR has been merged or closed, report that and stop
