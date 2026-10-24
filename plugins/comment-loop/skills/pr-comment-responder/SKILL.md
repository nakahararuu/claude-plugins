---
name: pr-comment-responder
description: Monitor the GitHub PR for the current branch (auto-detected via `gh pr view` — no URL, argument-less) for new unaddressed comments and respond — answering questions, making code fixes with commits, and posting replies. Use when the user asks to "watch a PR", "babysit a PR", "respond to PR comments", "monitor PR activity", or "keep an eye on a PR". Works great with /loop for continuous monitoring: slow work (multi-file changes, environment setup, builds/tests) is handed to a background subagent so each loop tick stays short. All replies are signed "by ClaudeCode" so humans can distinguish them, and fix replies include the commit link.
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

**In-progress acknowledgements don't count as addressed on their own.** When heavy work is handed to a background worker (see Step 3), the acknowledgement reply ends with `— by ClaudeCode (in progress)`. Treat a comment whose only ClaudeCode reply is such an acknowledgement as:
- **addressed** while the worker you started for it in this session is still running — leave it alone;
- **unaddressed** once that worker has finished without posting a final reply, or if no such worker exists in this session (e.g. the session was restarted) — pick it up again.

## Step 3: Process each unaddressed comment

Read the full body carefully before responding, then decide **who** does the work. Under `/loop`, every minute spent inside a tick is a minute the loop can't notice new comments or answer anything else, so keep the main loop to fetching, triage, quick replies, and dispatching. Anything slow goes to a background subagent.

### Triage: inline or background?

Handle **inline** (in this tick):
- Questions and explanation requests that can be answered from the code/diff you can read quickly
- Small, local code fixes: a few lines in one or two files, with at most a quick, fast check (lint, a single fast unit test)
- Clarifying questions for ambiguous comments

Hand to a **background worker** when the work is likely to take more than a couple of minutes, for example:
- Changes spanning several files, a refactor, or a feature-sized request
- Anything that needs environment setup: installing dependencies, starting services, building containers, `docker run` reproduction
- Running a build, a test suite, or an end-to-end check before committing
- A question whose answer requires a long investigation (reading a lot of code, bisecting, reproducing a bug)

When in doubt, prefer the background worker — a slightly over-delegated small fix costs little, a blocked loop costs every other comment.

### If it's a question or explanation request (inline)

Draft a clear, direct answer and post it immediately.

### If it's a small code fix request (inline) (keyword signals: "駄目", "エラー", "失敗", "修正", [must], bug, error, fail)

Only if no background worker is currently running (see "One worker at a time" below) — otherwise defer it to a later tick.

1. **Understand first** — read the error output fully
2. **Fix the code**
3. **Commit and push:**
   ```bash
   git add <changed-files>
   git commit -m "fix(...): <description>"
   git push origin <branch>
   ```
4. **Get the commit SHA:** `git rev-parse --short HEAD`
5. **Reply with the commit link** (see reply format below)

### If it's heavy work (background worker)

1. **Check for a running worker.** If a worker you started earlier in this session hasn't finished yet, don't start another one and don't acknowledge this comment — leave it unaddressed so a later tick picks it up once the worker is done.
2. **Acknowledge immediately**, so the human knows it was seen and the next tick doesn't start it twice. Reply in the comment's language, e.g.:
   ```
   {one line on what will be done and why it runs in the background, e.g. "This needs the test suite to run, so I'm handling it in the background and will report back in this thread."}

   — by ClaudeCode (in progress)
   ```
3. **Spawn the worker** with the Agent tool and `run_in_background: true` (general-purpose subagent). The worker starts with no context, so the prompt must be self-contained — use the template below.
4. **Move on** to the next comment (questions can still be answered inline; further heavy or code-changing comments wait for a later tick) and finish the tick. You'll be notified when the worker completes.

If the Agent tool isn't available in this environment, fall back to handling the work inline.

#### Worker prompt template

```
You are handling one GitHub PR comment for the pr-comment-responder skill, in the background.

Repository: {owner}/{repo}, PR #{pr_number}, branch `{branch}` (already checked out in {working_directory}).
Comment: {review comment on {path}:{line} | issue comment}, id {comment_id}
Full comment body:
<<<
{comment body}
>>>
{Any thread context you already have: earlier replies, the diff hunk, related comments}

Do this:
1. Understand the request; reproduce the problem if relevant (set up whatever environment is needed).
2. Make the change. Touch only what this comment asks for.
3. Validate it (build/tests/reproduction as appropriate). Don't commit untested changes.
4. Commit with `fix(...): <description>`, `git pull --rebase origin {branch}`, then `git push origin {branch}`.
5. Post the final reply to the comment and end it with "— by ClaudeCode":
   - fix:      "{root cause and what changed}\n\nFixed in commit `{sha}` (https://github.com/{owner}/{repo}/commit/{full_sha}) — by ClaudeCode"
   - answer:   "{answer}\n\n— by ClaudeCode"
   - failure:  "{what was tried, what blocked it, what you need from the human}\n\n— by ClaudeCode"
   Write the body to a temp file first. Review comment: `gh api repos/{owner}/{repo}/pulls/{pr_number}/comments/{comment_id}/replies -X POST -F body="$(cat <file>)"` (on 404 fall back to `gh issue comment {pr_number} --body "$(cat <file>)"`). Issue comment: `gh issue comment {pr_number} --body "$(cat <file>)"`.
Always post a final reply — success or failure — so the comment isn't left showing only "(in progress)".
Finish with a one-paragraph summary: what changed, the commit SHA (if any), and whether the reply was posted.
```

### One worker at a time

Workers commit and push on the same checkout as the main loop, so concurrent writers would trample each other's working tree and commits. While a worker is running:
- Start no second worker.
- Make no inline code changes or commits yourself; only answer questions.
- Leave other code-changing comments unaddressed (no acknowledgement) so they're picked up after the worker finishes.

When a worker's completion notification arrives, check that its final reply was posted; if it wasn't, post it yourself from the worker's summary (or a failure reply).

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

On each invocation, the "already addressed" check (Step 2) is the de-duplication mechanism — you naturally skip comments that already have a ClaudeCode reply, and comments acknowledged as "(in progress)" while their worker is still running. The only state beyond GitHub itself is which background worker (if any) you started in this session, which you already know from your own task list and completion notifications.

Keep each tick short: triage, answer what's quick, dispatch at most one worker, report, and end. Never wait for a worker inside a tick.

After handling all new comments, report briefly: how many comments were checked, how many were addressed inline, which one (if any) was handed to a background worker, whether a worker is still running, and the outcome of any worker that finished since the last tick.

## Tips

- When multiple fixes are needed, handle them in a single commit if they're related; use separate commits if they're independent. Several related heavy comments can go to one worker together (acknowledge each of them)
- If a fix requires validation (e.g., testing in Docker), that's background-worker work — test before committing, don't commit untested fixes
- Don't respond to your own comments; skip anything containing "by ClaudeCode"
- If the PR has been merged or closed, report that and stop
