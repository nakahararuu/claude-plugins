---
name: todo-comment-responder
description: Scan the current git diff (uncommitted working-tree changes) for TODO comments the user has dropped into the code as inline instructions, implement each one, and commit them one at a time so that `git log` becomes the running conversation between the user and Claude. Use when the user asks to "work through the TODOs", "resolve the TODO comments", "implement the TODOs I left", "check the diff for TODOs", or wants Claude to keep picking up new TODO instructions as they're added over time. Pairs naturally with /loop for continuous, asynchronous pair-programming (slow TODOs are handed to a background subagent so each loop tick stays short): the user drops a TODO, walks away, and reviews Claude's commits later via git log.
---

# TODO Comment Responder

Turn TODO comments in the working tree into a conversation: the user writes a TODO as an instruction, Claude implements it and commits, the user reads `git log` to see what happened, and drops the next TODO whenever they have new input.

## Step 0: Is a background worker still running?

If a background worker you started in an earlier tick of this session (see "Inline or background?" below) hasn't finished yet, **stop here**: report that a worker is still running and end the tick. The worker owns the working tree until it's done — a second writer would mix its changes into the worker's per-TODO commits. TODOs added in the meantime are picked up by the first tick after the worker finishes.

If a worker finished since the last tick, include its outcome (commits, clarification notes, failures) in this tick's report.

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

To delegate, hand **the whole pass** (every TODO in the list, in order) to one background subagent with the Agent tool and `run_in_background: true`, then report which TODOs were delegated and end the tick — don't wait for it. Keeping the entire pass in one worker preserves the one-TODO-per-commit sequencing of Step 3, which only works with a single writer. You'll be notified when the worker completes.

If the Agent tool isn't available in this environment, fall back to doing the pass inline.

The worker starts with no context, so give it a self-contained prompt:

```
You are resolving TODO comments for the todo-comment-responder skill, in the background.

Working directory: {working_directory}, branch `{branch}`.
Handle exactly these TODOs, in this order (they are uncommitted additions in `git diff HEAD`):
1. {file}:{line} — "{full TODO text, including continuation lines}"
2. ...

For each TODO, one at a time, fully, before the next:
1. Understand what it asks; set up whatever environment is needed.
2. Implement only that TODO's change and remove the TODO comment itself.
3. Validate (build/tests as appropriate). Don't commit untested changes.
4. Commit immediately, staging only the files for this TODO:
   todo: <short description>

   Resolves: "<original TODO text>"
If a TODO is ambiguous, destructive, or out of scope, don't guess: append a `CLAUDE:` clarification note under it in the same comment style and leave it uncommitted.
Leave any other uncommitted changes (including TODOs not listed above) untouched.
After the last TODO: `git pull --rebase origin {branch}` (only if there are new commits upstream and the tree allows it) and `git push origin {branch}`.
Finish with a summary: each TODO → commit hash, clarification note, or failure reason.
```

## Step 3: Process TODOs one at a time, fully, before moving to the next

This sequencing is what keeps commits clean without needing interactive hunk-staging:

1. Read enough of the surrounding file (and related files) to understand what the TODO is asking for.
2. Implement **only** that TODO's change.
3. Remove the TODO comment itself once it's resolved — the code that replaces it is the answer, it shouldn't linger.
4. Run any quick local check that's feasible (build, lint, relevant test) before committing. Don't commit code you haven't sanity-checked.
5. Stage and commit **now**, before touching the next TODO:
   ```bash
   git add <files touched for this TODO>
   git commit -m "$(cat <<'EOF'
   todo: <short description of what changed>

   Resolves: "<the original TODO comment text>"
   EOF
   )"
   ```
   Because every other TODO in the diff is still untouched at this point, `git diff HEAD` for the files you staged reflects only this one TODO's change — that's what keeps the commit scoped to a single TODO even when several TODOs live in the same file.
6. Only after committing, move to the next TODO in the list and repeat from step 1.

Do not batch multiple TODOs into one commit, even if they're small or related — one commit per TODO is the point, since it's what makes `git log` readable as a turn-by-turn conversation.

## Step 4: Push

After all TODOs found in this pass have been committed (or skipped per Step 5), push:

```bash
git push origin <branch>
```

## Step 5: Handling ambiguous TODOs

If a TODO's intent is genuinely unclear, don't guess and don't commit a speculative change. Instead, leave the TODO in place but append a clarifying note directly below it in the same comment style, e.g.:

```
// TODO: fix the retry logic here
// CLAUDE: unclear which retry logic — there are two call sites (fetchUser, fetchOrders). Which one, or both?
```

Leave this uncommitted (it's not a code change, just a clarification request the user will see next time they look at the file or run `git diff`). Do not commit a clarification note — committing it would remove it from the visible diff, defeating the purpose. Move on to the next TODO.

## Step 6: When used with /loop

Each invocation naturally de-duplicates: once a TODO is implemented and committed, it's gone from `git diff HEAD`, so the next iteration won't see it again. The user adds new TODOs whenever they have new instructions; Claude picks them up on the next loop tick. The only state beyond the diff itself is whether a background worker you started in this session is still running (Step 0) — and if a worker dies without committing, its TODOs are still in the diff, so the next tick simply picks them up again.

Keep each tick short: check for a running worker, scan, then either resolve small TODOs inline or dispatch one worker, report, and end. Never wait for a worker inside a tick.

After each pass, report briefly: how many TODOs were found, how many were resolved inline (with commit hashes), how many were delegated to a background worker, whether a worker is still running, the outcome of any worker that finished since the last tick, and how many were left as clarification requests.

## Tips

- If a TODO asks for something destructive or clearly out of scope (e.g. "TODO: drop the users table"), don't do it — leave a clarification note instead and explain why.
- If implementing a TODO would require touching unrelated code far outside its context, prefer the smallest change that actually satisfies the instruction over a broad refactor.
- If the working tree has unrelated pre-existing uncommitted changes (not a TODO, just in-progress edits), leave them alone — only act on TODOs.
