$ARGUMENTS can be either a GitHub issue number or a free-text task description.

- **If it's a number:** fetch the issue with `gh issue view $ARGUMENTS` and use its title and body as the task.
- **If it's a string:** create a new GitHub issue first using `gh issue create` with a clear title and description derived from the text. Assign it to `marcalderon`, apply the correct labels/type/Sprint Cycle from the available options, and set status to **In Progress**. Then use that issue as the task.
- **If $ARGUMENTS is empty:** ask the user for an issue number or task description before doing anything else.

Show a concise action plan and **stop**. Do not touch any code until the user explicitly approves the plan.

## Branch

Resolve OWNER and REPO dynamically with `gh repo view --json owner,name`.

Create a new branch from `main` named `fix/<issue-title-kebab-case>` or `feature/<issue-title-kebab-case>` based on the issue type.

If that branch already exists, append a short disambiguating suffix derived from the task (e.g., `-v2`, `-ui`, `-api`). If that also exists, keep appending incremental suffixes. As a last resort, append the current date (`-YYYYMMDD`). Repeat until a free name is found, then check it out.

## Implementation rules

- Touch the minimum code required to solve the task.
- Follow DRY. Do not add behavior not explicitly requested.
- If requirements are ambiguous, ask all necessary clarifying questions upfront and wait for answers before implementing. If still uncertain after clarification, default to the safest approach.
- Add unit tests for every new or changed behavior.
- If the issue is a bug, add a regression test that reproduces it.
- Fix only TS, ESLint, and Sonar issues **introduced by your changes** — ignore pre-existing ones.
- Run only the tests related to the files you changed.
- Sonar requires **80% coverage on new code**.
- **Do NOT commit or push.** The user supervises all git operations.

## After implementing

Before declaring done:
- Scan all new/modified code for TS errors, ESLint violations, and Sonar smells. Also verify the changes comply with the rules in the active CLAUDE.md files (global and project-level). Fix any violations found.
- Check coverage on new code. If below 80%, add unit tests until it meets the threshold. If above, leave it.
- Run related tests again to confirm green.

## When the user says "publicar"

- Do **not** create new commits — use the existing ones on the branch.
- Check if the branch already exists on the remote (`git ls-remote --heads origin <branch>`). If not, push it. If yes, skip the push.
- Open a **Draft PR** using the repo's PR template (`.github/PULL_REQUEST_TEMPLATE.md` if it exists).
- Answer the template's questions in **English**, 1–2 lines each, no links, plain text.
- Assign the PR to `marcalderon`.
- Add `Closes #<N>` in the PR body to link the issue.
- Return the Draft PR link.
