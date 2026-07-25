---
name: todo-comment-responder
description: Scan the current git diff (uncommitted working-tree changes) for TODO comments the user has dropped into the code as inline instructions, implement each one, and commit them one at a time so that `git log` becomes the running conversation between the user and Claude. Use when the user asks to "work through the TODOs", "resolve the TODO comments", "implement the TODOs I left", "check the diff for TODOs", or wants Claude to keep picking up new TODO instructions as they're added over time. Pairs naturally with /loop for continuous, asynchronous pair-programming: the user drops a TODO, walks away, and reviews Claude's commits later via git log.
---

# TODO Comment Responder

Turn TODO comments in the working tree into a conversation: the user writes a TODO as an instruction, Claude implements it and commits, the user reads `git log` to see what happened, and drops the next TODO whenever they have new input.

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

Each invocation naturally de-duplicates: once a TODO is implemented and committed, it's gone from `git diff HEAD`, so the next iteration won't see it again. The user adds new TODOs whenever they have new instructions; Claude picks them up on the next loop tick. No separate state-tracking is needed beyond the diff itself.

After each pass, report briefly: how many TODOs were found, how many were resolved (with commit hashes), and how many were left as clarification requests.

## Tips

- If a TODO asks for something destructive or clearly out of scope (e.g. "TODO: drop the users table"), don't do it — leave a clarification note instead and explain why.
- If implementing a TODO would require touching unrelated code far outside its context, prefer the smallest change that actually satisfies the instruction over a broad refactor.
- If the working tree has unrelated pre-existing uncommitted changes (not a TODO, just in-progress edits), leave them alone — only act on TODOs.
