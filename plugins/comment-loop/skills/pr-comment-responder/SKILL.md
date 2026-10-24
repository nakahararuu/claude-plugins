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

## Step 0: Check the background worker

Heavy work runs in a background worker (Step 3). Whether one is running is tracked in a lock file, not in your memory of the conversation — a long `/loop` gets compacted and a resumed or restarted session has lost its background tasks, so recollection isn't reliable.

```bash
lock="$(git rev-parse --git-dir)/pr-comment-responder-worker.json"
```

`git rev-parse --git-dir` resolves per worktree, so the lock covers exactly the working tree the worker writes to. The main loop writes it when dispatching (`{"comment_ids": [...], "started_at": "..."}`); the worker refreshes its mtime as a heartbeat and deletes it when done.

- **No lock** → no worker is running.
- **Lock present and fresh** (`find "$lock" -mmin +60` prints nothing) → a worker is running on the comment ids listed in it.
- **Lock present but stale** (`find` prints it — no heartbeat for 60 minutes), or the completion notification for that worker has arrived → the worker is done or dead. Clean up:
  1. If you have the worker's completion summary, make sure every comment in the lock got its final reply, and post any missing one yourself from the summary. Without a summary (stale lock), post nothing: those comments only have an "(in progress)" acknowledgement, so Step 2 treats them as unaddressed and they're retried.
  2. Run `git status --porcelain`. If the tree isn't clean, the worker left partial changes behind. Don't discard them — they may not all be the worker's. Include them in the tick's report, and hold all code-changing work (no worker, no inline fix) until the tree is clean again; questions can still be answered.
  3. `rm -f "$lock"`.

## Step 1: Fetch all comments

Fetch both types in parallel:

```bash
# Inline review comments
gh api "repos/{owner}/{repo}/pulls/{pr_number}/comments" \
  --jq '.[] | {id, html_url, body, created_at, in_reply_to_id, path, line}'

# General PR discussion comments
gh api "repos/{owner}/{repo}/issues/{pr_number}/comments" \
  --jq '.[] | {id, html_url, body, created_at}'
```

Always fetch the **full body** — never rely on truncated previews.

## Step 2: Find unaddressed comments

Every ClaudeCode reply starts with a `Re:` line linking the comment(s) it answers (see "Reply format"). Match replies to comments through that link, not by position: replies are posted asynchronously, so a worker's final reply can land after newer comments, and a position-based check ("any later ClaudeCode comment") would wrongly mark those newer comments as addressed.

A comment is **addressed** if some comment containing "by ClaudeCode" — in any review thread or among the issue comments — has a `Re:` line referencing it: `#discussion_r{id}` for a review comment, `#issuecomment-{id}` for an issue comment.

**Legacy replies without a `Re:` line** (posted before this convention) fall back to position:
- Issue comment: addressed if a later issue comment contains "by ClaudeCode" and has no `Re:` line
- Review comment: addressed if a later comment in the same thread (same `in_reply_to_id // id`) contains "by ClaudeCode" and has no `Re:` line

**Always skip:**
- Comments you wrote (body contains "by ClaudeCode")
- Comments already addressed as above

**In-progress acknowledgements don't count as addressed on their own.** When heavy work is handed to a background worker (see Step 3), the acknowledgement reply ends with `— by ClaudeCode (in progress)`. Treat a comment whose only referencing ClaudeCode replies are such acknowledgements as:
- **addressed** while the worker lock (Step 0) is fresh and lists this comment's id — leave it alone;
- **unaddressed** otherwise (the worker finished or died without posting a final reply, or the session was restarted) — pick it up again.

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

Only if no background worker is running and the working tree is clean (Step 0) — otherwise defer it to a later tick.

1. **Understand first** — read the error output fully
2. **Fix the code**
3. **Commit and push:**
   ```bash
   git add <changed-files>
   git commit -m "fix(...): <description>"
   git push origin <branch>
   ```
4. **Get the commit SHA** after the push: `git rev-parse HEAD` (and `--short` for display)
5. **Reply with the commit link** (see reply format below)

### If it's heavy work (background worker)

1. **Check that you may dispatch.** If the worker lock is fresh (Step 0) or `git status --porcelain` prints anything, don't start a worker and don't acknowledge this comment — leave it unaddressed so a later tick picks it up.
2. **Acknowledge immediately**, so the human knows it was seen and the next tick doesn't start it twice. Reply in the comment's language, e.g.:
   ```
   Re: {html_url of the comment}

   {one line on what will be done and why it runs in the background, e.g. "This needs the test suite to run, so I'm handling it in the background and will report back in this thread."}

   — by ClaudeCode (in progress)
   ```
3. **Write the lock** before spawning, listing every comment the worker will handle:
   ```bash
   started_at=$(date -u +%Y-%m-%dT%H:%M:%SZ)
   printf '{"comment_ids": [%s], "started_at": "%s"}\n' "{id1},{id2}" "$started_at" > "$lock"
   ```
4. **Spawn the worker** as a general-purpose subagent with the Agent tool. It runs in the background — don't wait for its result in this tick. The worker starts with no context, so the prompt must be self-contained — use the template below.
5. **Move on** to the next comment (questions can still be answered inline; further heavy or code-changing comments wait for a later tick) and finish the tick. You'll be notified when the worker completes.

Several related heavy comments can go to the same worker: acknowledge each one, and list all of them in the lock and in the worker prompt.

If the Agent tool isn't available in this environment, fall back to handling the work inline.

#### Worker prompt template

```
You are handling GitHub PR comments for the pr-comment-responder skill, in the background.

Repository: {owner}/{repo}, PR #{pr_number}, branch `{branch}` (already checked out in {working_directory}).
Lock file: {lock_path} (started_at {started_at})

Comments to handle — every one of them:
1. {review comment on {path}:{line} | issue comment}, id {comment_id}, url {html_url}, reply target {review comment: in_reply_to_id // id | issue comment}
   Full body:
   <<<
   {comment body}
   >>>
   {Any thread context you already have: earlier replies, the diff hunk, related comments}
2. ...

Heartbeat: at the start of every step below, and before any command you expect to run long, run `grep -qF '{started_at}' {lock_path} && touch {lock_path}`. After 60 minutes without an update the main loop treats you as dead. If the command fails, the main loop has already given up on you and may have started another worker: stop at once without changing or posting anything else, and say so in your summary.

Do this:
1. Run `git status --porcelain`. If it prints anything, the tree isn't yours to change: skip to step 5 and post a failure reply to every comment saying the working tree had uncommitted changes.
2. Understand the requests; reproduce the problem if relevant (set up whatever environment is needed). Make the changes, touching only what these comments ask for. Related comments can share a commit; independent ones get separate commits.
3. Validate (build/tests/reproduction as appropriate). Don't commit untested changes.
4. Commit with `fix(...): <description>`, then `git pull --rebase origin {branch}` and `git push origin {branch}`. Take each commit's SHA after the rebase (`git rev-parse HEAD`, or `git log` for several) — the rebase rewrites commits, so a SHA from before it links to nothing.
   If anything fails before the push succeeds, leave the tree as clean as you found it: `git rebase --abort` if a rebase is in progress, `git restore --staged --worktree -- <files you modified>`, delete files you created, and confirm `git status --porcelain` is empty. A commit that was made but couldn't be pushed stays; mention its SHA in the failure reply.
5. Post a final reply to every comment above — success or failure — so none is left showing only "(in progress)". Each reply starts with a `Re: {html_url}` line and ends with "— by ClaudeCode":
   - fix:      "Re: {html_url}\n\n{root cause and what changed}\n\nFixed in commit `{short_sha}` (https://github.com/{owner}/{repo}/commit/{full_sha}) — by ClaudeCode"
   - answer:   "Re: {html_url}\n\n{answer}\n\n— by ClaudeCode"
   - failure:  "Re: {html_url}\n\n{what was tried, what blocked it, what you need from the human}\n\n— by ClaudeCode"
   Write each body to its own file from `f=$(mktemp)` and pass the file, never the text inline:
   - review comment: `gh api repos/{owner}/{repo}/pulls/{pr_number}/comments/{reply_target}/replies -X POST -F body=@"$f"` (on 404, fall back to `gh issue comment {pr_number} --body-file "$f"`)
   - issue comment: `gh issue comment {pr_number} --body-file "$f"`
6. `grep -qF '{started_at}' {lock_path} && rm -f {lock_path}` — last, after the replies are posted.
Finish with a short summary per comment: what changed, the commit SHA (if any), and whether the reply was posted.
```

### One worker at a time

Workers commit and push on the same checkout as the main loop, so concurrent writers would trample each other's working tree and commits. While the worker lock is fresh (Step 0):
- Start no second worker.
- Make no inline code changes or commits yourself; only answer questions.
- Leave other code-changing comments unaddressed (no acknowledgement) so they're picked up after the worker finishes.

When a worker's completion notification arrives, run the cleanup from Step 0: post any final reply it missed from its summary, check the working tree, and remove the lock if it's still there.

## Step 4: Post replies

### Reply format

Every reply must:
- Start with a `Re:` line linking the comment it answers (`html_url`; for several, `Re: {url1}, {url2}`) — Step 2 relies on it, so it goes on thread replies too
- Directly address the comment's content (don't just say "fixed")
- End with `— by ClaudeCode`
- Include commit info when a fix was made

**For a code fix reply:**
```
Re: {html_url}

{explanation of root cause and what was changed}

Fixed in commit `{sha}` (https://github.com/{owner}/{repo}/commit/{full_sha}) — by ClaudeCode
```

**For a question reply:**
```
Re: {html_url}

{answer}

— by ClaudeCode
```

### Posting to GitHub

Write the reply body to a fresh temp file and pass the file itself, so bash never interprets backticks or special characters in the message, and the main loop and a worker never overwrite each other's file:

```bash
f=$(mktemp)
# write the reply body into "$f"
```

**Reply to a review comment.** The replies endpoint only accepts a thread's top-level comment — replying to a reply returns 404 — so target `reply_target = in_reply_to_id // id`:
```bash
gh api repos/{owner}/{repo}/pulls/{pr_number}/comments/{reply_target}/replies \
  -X POST \
  -F body=@"$f"
```

If that still returns 404 (e.g. the thread was deleted), fall back to a general PR comment; the `Re:` line keeps it tied to the comment:
```bash
gh issue comment {pr_number} --body-file "$f"
```

**Reply to an issue comment:**
```bash
gh issue comment {pr_number} --body-file "$f"
```

## Handling ambiguous comments

If a comment is unclear, post a clarifying question rather than guessing:
```
Re: {html_url}

{clarifying question}

— by ClaudeCode
```

## When used with /loop

On each invocation, the "already addressed" check (Step 2) is the de-duplication mechanism — you skip comments that a ClaudeCode reply links via `Re:`, and comments acknowledged as "(in progress)" while their worker is still running. The only state beyond GitHub itself is the worker lock (Step 0), which survives compaction and session restarts.

Keep each tick short: check the lock, triage, answer what's quick, dispatch at most one worker, report, and end. Never wait for a worker inside a tick.

After handling all new comments, report briefly: how many comments were checked, how many were addressed inline, which ones (if any) were handed to a background worker, whether a worker is still running, the outcome of any worker that finished or went stale since the last tick, and any leftover uncommitted changes that are holding code-changing work.

## Tips

- When multiple fixes are needed, handle them in a single commit if they're related; use separate commits if they're independent. Several related heavy comments can go to one worker together (see Step 3)
- If a fix requires validation (e.g., testing in Docker), that's background-worker work — test before committing, don't commit untested fixes
- Don't respond to your own comments; skip anything containing "by ClaudeCode"
- If the PR has been merged or closed, report that and stop
