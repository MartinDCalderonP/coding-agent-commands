Resolve OWNER, REPO, and PR number dynamically:

```bash
gh repo view --json owner,name   # → OWNER, REPO
gh pr view --json number          # → PR
```

Fetch the last $ARGUMENTS comments from three sources:

```bash
gh api repos/{OWNER}/{REPO}/pulls/{PR}/comments          # inline review comments
gh api repos/{OWNER}/{REPO}/issues/{PR}/comments         # general PR comments
gh api repos/{OWNER}/{REPO}/pulls/{PR}/reviews           # PR-level review summaries
```

Include **all** commenters — bots (`github-actions[bot]`, `copilot[bot]`, etc.) and human teammates alike. If $ARGUMENTS is empty, default to the **20** most recent.

## Output format

Group findings into named **Blocks** by theme or file area (e.g., "Block A — Type safety", "Block B — Test coverage"). For each block:

1. Author and whether they are a bot or a human teammate.
2. What they flagged (quote the relevant snippet).
3. Exact file and line(s) to change.
4. Action: **Fix** / **Consider** / **Ignore** — for Ignore, state the reason.

## Approval and execution flow

After presenting the block plan, **stop and wait**. Do not touch any code until the user explicitly approves the plan.

Once approved, work **one block at a time**: implement the block, report done, and wait for the user to explicitly say to continue before starting the next block. The user commits and pushes **only after all blocks in the round are complete** — pushing mid-round would re-trigger the bots before the round is finished.

## Resolving a block

When the user says to proceed with a block:

- Touch the minimum code required.
- **Do NOT commit or push.** The user supervises all git operations.
- After applying fixes: scan new/modified code for TS errors, ESLint violations, and Sonar smells introduced by the fix. Correct them before reporting done.
- Run only the tests related to the changed files and confirm they pass.

## After the user pushes the full round

Once the user confirms they have pushed all the commits for this round:

**Bot comments:** mark each addressed thread as resolved with no reply. Use the GitHub GraphQL API:

1. Fetch open review thread IDs:
   ```graphql
   query { repository(owner: OWNER, name: REPO) { pullRequest(number: PR) {
     reviewThreads(first: 100) { nodes { id isResolved
       comments(first: 1) { nodes { author { login } body } }
     }}
   }}}
   ```
2. For each unresolved bot thread addressed in this round:
   ```graphql
   mutation { resolveReviewThread(input: { threadId: "THREAD_ID" }) {
     thread { isResolved }
   }}
   ```

**Human comments:** for each one addressed in this round, draft a friendly and collaborative reply and present it to the user for approval. Do not post anything until approved. Once approved, post the reply — but **do not mark the thread as resolved** unless the user explicitly says to, since the conversation may continue.

## Closing the cycle (no more actionables)

This is the terminal state of the entire iterative process — reached after all rounds of comments have been worked through and nothing actionable remains. When every remaining comment is stale, out of scope, or the user decides not to act on it:

**Bot comments:**
1. Reply with a concise technical justification for not acting on it.
   - Inline: `gh api --method POST repos/{OWNER}/{REPO}/pulls/{PR}/comments/{comment_id}/replies -f body="..."`
   - General: `gh api --method POST repos/{OWNER}/{REPO}/issues/{PR}/comments -f body="..."`
2. Mark the thread as resolved.

**Human comments:**
1. Draft a friendly, collaborative reply explaining why no action was taken and present it to the user for approval. Do not post until approved.
2. Once approved, post the reply. Mark as resolved only if the user explicitly says to.

After all bot threads are resolved and human replies are approved: `gh pr ready`.
