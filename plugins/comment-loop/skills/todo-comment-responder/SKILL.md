---
name: todo-comment-responder
description: Scan the current git diff (uncommitted working-tree changes) for TODO comments the user has dropped into the code as inline instructions, implement each one, and commit them one at a time so that `git log` becomes the running conversation between the user and Claude. Use when the user asks to "work through the TODOs", "resolve the TODO comments", "implement the TODOs I left", "check the diff for TODOs", or wants Claude to keep picking up new TODO instructions as they're added over time. Pairs naturally with /loop for continuous, asynchronous pair-programming (slow TODOs are handed to a background subagent so each loop tick stays short): the user drops a TODO, walks away, and reviews Claude's commits later via git log.
---

# TODO Comment Responder

Turn TODO comments in the working tree into a conversation: the user writes a TODO as an instruction, Claude implements it and commits, the user reads `git log` to see what happened, and drops the next TODO whenever they have new input.

## Step 0: Is a background worker still running?

Whether a background worker (see "Inline or background?" below) is running is tracked in a lock file, not in your memory of the conversation — a long `/loop` gets compacted and a resumed or restarted session has lost its background tasks, so recollection isn't reliable.

```bash
lock="$(git rev-parse --git-dir)/todo-comment-responder-worker.json"
```

`git rev-parse --git-dir` resolves per worktree, so the lock covers exactly the working tree the worker writes to. The main loop writes it when dispatching (`{"todos": [...], "started_at": "..."}`); the worker refreshes its mtime as a heartbeat and deletes it when done.

- **No lock** → no worker is running; go on to Step 1.
- **Lock present and fresh** (`find "$lock" -mmin +60` prints nothing) → **stop here**: report that a worker is still running and end the tick. The worker owns the commits until it's done — a second writer would mix its changes into the worker's per-TODO commits. TODOs added in the meantime are picked up by the first tick after the worker finishes.
- **Lock present but stale** (`find` prints it — no heartbeat for 60 minutes), or the completion notification for that worker has arrived → the worker is done or dead:
  1. Include its outcome in this tick's report: from its summary if you have one (commits, clarification notes, failures). Without a summary (stale lock), check each TODO listed in the lock — committed (`git log --grep` its text), still in `git diff HEAD`, or neither. Report any in the last group as lost so the user can re-add it, and mention that partial edits from the dead worker may sit near the TODOs still in the diff.
  2. `rm -f "$lock"`, then continue with Step 1 — TODOs still in the diff are picked up again.

## Step 1: Look at the current diff

Check the uncommitted diff against HEAD — this is where the user's freshly-written instructions live:

```bash
git status
git diff HEAD
```

Only lines **added** in this diff are instructions. A TODO already sitting in HEAD (i.e. not part of this diff) is old news — it was either already handled in a previous run or intentionally left alone by the user, so leave it untouched.

## Step 2: Find TODO comments in the added lines

Look for added (`+`) lines containing a TODO marker, in whatever comment syntax the file uses (`// TODO`, `# TODO`, `<!-- TODO -->`, `TODO(claude)`, etc.). A TODO instruction may span multiple comment lines — if the added lines directly above/below a `TODO:` line are also comments continuing the thought, treat them as one instruction block.

List every distinct TODO found, in file order top to bottom. If none are found, report that the diff has no pending TODOs and stop.

## Inline or background?

Under `/loop`, a tick that spends twenty minutes implementing a TODO is twenty minutes in which the loop is deaf. Before implementing anything, estimate the work for the whole list:

- **Inline** (do Steps 3–5 in this tick) when every TODO is small and local: a few lines in one or two files, with at most a quick check (lint, a single fast unit test).
- **Background worker** when any TODO is likely to take more than a couple of minutes — changes across several files or a feature-sized request, environment setup (installing dependencies, starting services, building containers), running a build or test suite before committing, or a long investigation. When in doubt, delegate.

To delegate, hand **the whole pass** (every TODO in the list, in order) to one background worker, then report which TODOs were delegated and end the tick — don't wait for it. Keeping the entire pass in one worker preserves the one-TODO-per-commit sequencing of Step 3, which only works with a single writer.

1. **Write the lock** before spawning:
   ```bash
   started_at=$(date -u +%Y-%m-%dT%H:%M:%SZ)
   printf '{"todos": [%s], "started_at": "%s"}\n' '"{file}:{line} {TODO text}", ...' "$started_at" > "$lock"
   ```
2. **Spawn the worker** as a general-purpose subagent with the Agent tool. It runs in the background — don't wait for its result in this tick. You'll be notified when it completes.

If the Agent tool isn't available in this environment, fall back to doing the pass inline.

The worker starts with no context, so give it a self-contained prompt:

```
You are resolving TODO comments for the todo-comment-responder skill, in the background.

Working directory: {working_directory}, branch `{branch}`.
Lock file: {lock_path} (started_at {started_at})
Handle exactly these TODOs, in this order (they are uncommitted additions in `git diff HEAD`):
1. {file}:{line} — "{full TODO text, including continuation lines}"
2. ...

The user keeps editing this working tree while you work: new TODOs, notes, other in-progress changes. Anything that isn't part of the TODO you're on belongs to them. Never stage a whole modified file, and never stash, pull, rebase, reset, or check out files.

Heartbeat: at the start of every TODO and every step, and before any command you expect to run long, run `grep -qF '{started_at}' {lock_path} && touch {lock_path}`. If it fails, the main loop has given up on you and may have started another worker: stop at once without changing anything else, and say so in your summary.

For each TODO, one at a time, fully, before the next:
1. Understand what it asks; set up whatever environment is needed.
2. Implement only that TODO's change. Leave the TODO comment in place for now.
3. Validate (build/tests as appropriate). Don't commit untested changes.
4. If it works: remove the TODO comment, stage only this TODO's change, and commit:
   - For each modified file: `p=$(mktemp); git diff -- <file> > "$p"`. Edit "$p" so it holds only this TODO's change: delete `+` lines that aren't yours (other TODOs, CLAUDE: notes, the user's edits), turn `-` lines that aren't yours into context lines (replace the leading `-` with a space), and drop hunks left with no `+`/`-` lines. Then `git apply --cached --recount "$p"`.
   - New files created only for this TODO: `git add <file>`.
   - Check `git diff --cached` shows exactly this TODO's change, then commit:
     todo: <short description>

     Resolves: "<original TODO text>"
   If it fails, or the TODO is ambiguous, destructive, or out of scope: undo your own edits for this TODO, keep the TODO comment, append a `CLAUDE:` note under it in the same comment style (what blocked it, or what you need to know), and leave that uncommitted.
After the last TODO: `git push origin {branch}`. If the push is rejected, don't pull, rebase, or stash (the working tree holds the user's uncommitted TODOs) — just report the rejection.
Then, last: `grep -qF '{started_at}' {lock_path} && rm -f {lock_path}`.
Finish with a summary: each TODO → commit hash, clarification note, or failure reason; and whether the push succeeded.
```

## Step 3: Process TODOs one at a time, fully, before moving to the next

The user may keep editing the working tree while you work, and the other TODOs of this pass are themselves uncommitted additions — often in the same files. So never stage a whole modified file: stage only the lines that make up this TODO's change.

1. Read enough of the surrounding file (and related files) to understand what the TODO is asking for.
2. Implement **only** that TODO's change. Leave the TODO comment in place for now.
3. Run any quick local check that's feasible (build, lint, relevant test) before committing. Don't commit code you haven't sanity-checked. If it doesn't work out, undo your edits for this TODO and leave a `CLAUDE:` note as in Step 5 — the TODO stays in the diff for the user to see.
4. Remove the TODO comment itself — the code that replaces it is the answer, it shouldn't linger. The comment was never in HEAD, so removing it leaves no trace in the diff.
5. Stage **only this TODO's change**. For each modified file:
   ```bash
   p=$(mktemp)
   git diff -- <file> > "$p"
   ```
   Edit `"$p"` so it holds only this TODO's change: delete `+` lines that aren't yours (other TODOs, `CLAUDE:` notes, the user's edits), turn `-` lines that aren't yours into context lines (replace the leading `-` with a space), and drop hunks left with no `+`/`-` lines. Then:
   ```bash
   git apply --cached --recount "$p"
   git diff --cached    # must show exactly this TODO's change, nothing else
   ```
   A new file created only for this TODO can be staged with `git add <file>`.
6. Commit **now**, before touching the next TODO:
   ```bash
   git commit -m "$(cat <<'EOF'
   todo: <short description of what changed>

   Resolves: "<the original TODO comment text>"
   EOF
   )"
   ```
7. Only after committing, move to the next TODO in the list and repeat from step 1.

Do not batch multiple TODOs into one commit, even if they're small or related — one commit per TODO is the point, since it's what makes `git log` readable as a turn-by-turn conversation.

## Step 4: Push

After all TODOs found in this pass have been committed (or skipped per Step 5), push:

```bash
git push origin <branch>
```

If the push is rejected because the remote has new commits, don't `git pull`, rebase, or stash — the working tree always holds the user's uncommitted TODOs, so that would either fail or tangle their changes. Report the rejection and leave it to the user.

## Step 5: Handling ambiguous TODOs

If a TODO's intent is genuinely unclear, don't guess and don't commit a speculative change. Instead, leave the TODO in place but append a clarifying note directly below it in the same comment style, e.g.:

```
// TODO: fix the retry logic here
// CLAUDE: unclear which retry logic — there are two call sites (fetchUser, fetchOrders). Which one, or both?
```

Leave this uncommitted (it's not a code change, just a clarification request the user will see next time they look at the file or run `git diff`). Do not commit a clarification note — committing it would remove it from the visible diff, defeating the purpose. Move on to the next TODO.

## Step 6: When used with /loop

Each invocation naturally de-duplicates: once a TODO is implemented and committed, it's gone from `git diff HEAD`, so the next iteration won't see it again. The user adds new TODOs whenever they have new instructions; Claude picks them up on the next loop tick. The only state beyond the diff itself is the worker lock (Step 0). A TODO comment is removed only right before its commit, so if a worker fails or dies, its unfinished TODOs are still in the diff and the next tick picks them up again.

Keep each tick short: check the lock, scan, then either resolve small TODOs inline or dispatch one worker, report, and end. Never wait for a worker inside a tick.

After each pass, report briefly: how many TODOs were found, how many were resolved inline (with commit hashes), how many were delegated to a background worker, whether a worker is still running, the outcome of any worker that finished or went stale since the last tick, how many were left as clarification requests, and whether the push was rejected.

## Tips

- If a TODO asks for something destructive or clearly out of scope (e.g. "TODO: drop the users table"), don't do it — leave a clarification note instead and explain why.
- If implementing a TODO would require touching unrelated code far outside its context, prefer the smallest change that actually satisfies the instruction over a broad refactor.
- If the working tree has unrelated pre-existing uncommitted changes (not a TODO, just in-progress edits), leave them alone — only act on TODOs, and keep them out of your commits (Step 3, step 5).
