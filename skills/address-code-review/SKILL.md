---
name: address-code-review
version: 1.0.0
description: >
  Fetch and address code review comments on the current PR autonomously.
  Decides which feedback is worth acting on, implements fixes, replies to
  every thread, pushes, and re-requests review — no check-ins with the user.
user_invocable: true
---

# Address Code Review

Fetch and address code review comments on the current PR, end to end, without stopping to ask the user.

**This skill is autonomous.** Do not use AskUserQuestion. Do not pause for approval before fixing, replying, pushing, or re-requesting review. You decide what feedback is worth doing, draft the replies, do the work, and report what happened at the end. The only time you stop early is when there is nothing to do (no PR, no unresolved comments) or when a hard blocker makes the whole run impossible (e.g. `gh` is not authenticated).

## Instructions

### Step 0: Learn the host repo's conventions

Before touching any review comment, work out how this repo expects code to be checked and committed. Discover, in order of preference:

1. **Agent and contributor docs.** Read whichever of these exist at the repo root: `CLAUDE.md`, `.claude/CLAUDE.md`, `AGENTS.md`, `CONTRIBUTING.md`. Look for:
   - the lint command (e.g. `yarn lint`, `npm run lint`, `make lint`)
   - the typecheck command
   - the test command (prefer a non-watch / CI mode variant)
   - commit-message conventions (e.g. Conventional Commits, required scope prefixes)
   - whether GitHub issues require specific labels, and which ones
2. **`package.json` scripts.** If the docs don't name a command, check for `lint`, `typecheck` (or a `tsc` script), and `test` scripts, and run them with the package manager the repo uses (look at the lockfile).
3. **`Makefile`.** Look for `lint`, `test`, and `check` targets.
4. **CI workflows.** Read the files under `.github/workflows/`. The commands CI runs are the commands to run locally.

Record what you found as:

- `{lint_cmd}` — the lint command
- `{typecheck_cmd}` — the typecheck command
- `{test_cmd}` — the test command
- `{commit_convention}` — the commit-message convention (may be empty)
- `{issue_labels}` — the label convention for new issues (may be empty)

Use these placeholders in the later steps. Do not invent commands. If a command can't be found, leave it empty. If no lint, typecheck, or test command can be found at all, skip Step 7 and say so in the Step 11 summary.

Also keep any code-style rules from these docs in mind; they are "the repo's conventions" referred to below.

### Step 1: Identify the current PR

Run `gh pr view --json number,headRefName,url,baseRefName` to get the current PR. If no PR exists for the current branch, report that and exit.

### Step 2: Fetch all review comments (unresolved only)

Fetch comments using the GitHub CLI, and **only keep unresolved, current feedback**.

**Preferred: review threads via GraphQL** (captures Cursor/Bugbot threads and resolution state):

```bash
# Review threads with resolution + outdated info + thread IDs for replying later
# Paginate with `after` cursor if hasNextPage is true
# IMPORTANT: Inline all values directly — never use GraphQL $variable syntax (zsh mangles it)

gh api graphql -f query='{ repository(owner:"<OWNER>", name:"<REPO>") { pullRequest(number:<PR_NUMBER>) { reviewThreads(first:50) { pageInfo { hasNextPage endCursor } nodes { id isResolved isOutdated path line originalLine comments(first:100) { nodes { id databaseId author { login } body createdAt } } } } } } }'

# For subsequent pages, inline the endCursor value:
gh api graphql -f query='{ repository(owner:"<OWNER>", name:"<REPO>") { pullRequest(number:<PR_NUMBER>) { reviewThreads(first:50, after:"<END_CURSOR>") { pageInfo { hasNextPage endCursor } nodes { id isResolved isOutdated path line originalLine comments(first:100) { nodes { id databaseId author { login } body createdAt } } } } } } }'
```

**Important:** Save the thread `id` (the node ID on each `reviewThreads.nodes[]` entry) alongside each comment. This ID is needed in Step 9 to reply directly — no second GraphQL call required.

Filter rules:

- Only include threads where `isResolved` is `false`.
- Skip threads where `isOutdated` is `true`.
- Use the **latest comment** in each thread (`createdAt`).
- For Cursor/Bugbot, only keep the **latest comment per thread** (ignore older versions).
- Skip threads whose latest comment is your own (a reply you already posted on an earlier run).

**Fallback: REST review comments** if GraphQL returns nothing:

```bash
gh api repos/{owner}/{repo}/pulls/{pr_number}/comments --paginate
```

If using the REST fallback, you must **also fetch thread IDs** via a separate GraphQL call so Step 9 can reply to threads:

```bash
# Paginate with `after` if hasNextPage is true
gh api graphql -f query='{ repository(owner:"<OWNER>", name:"<REPO>") { pullRequest(number:<PR_NUMBER>) { reviewThreads(first:100) { pageInfo { hasNextPage endCursor } nodes { id comments(first:100) { nodes { databaseId } } } } } } }'
```

Match each REST comment's `id` to **any** `databaseId` within a thread's `comments.nodes[]` to associate thread IDs. Fetching all comments (not just the first) ensures reply comments also match correctly.

**Always fetch review summaries and issue comments**:

```bash
# Review summaries with APPROVE/REQUEST_CHANGES/COMMENT
# Use the latest review per reviewer (humans only)
# Ignore older/dismissed reviews and approvals

gh api repos/{owner}/{repo}/pulls/{pr_number}/reviews --paginate

# PR issue comments (general conversation)
gh pr view {pr_number} --json comments
```

Always check **unresolved human reviews**: if a reviewer's latest review is `REQUEST_CHANGES` or `COMMENT`, include its summary comment even if there are no inline notes.

### Step 3: Categorize comments by severity

Analyze each **unresolved** comment and categorize into:

1. **Must fix** - Blocking issues that must be resolved before merge:
   - Comments from latest human reviews with `REQUEST_CHANGES`
   - Comments explicitly marked as blocking
   - Security issues, bugs, or correctness problems
   - Comments containing words like "blocking", "must", "required", "critical"

2. **Should fix** - Important but not blocking:
   - Suggestions for better patterns or approaches
   - Performance concerns
   - Comments with "should", "recommend", "consider"

3. **Nice to have** - Optional improvements:
   - Nits, style preferences
   - Comments starting with "nit:", "optional:", "minor:"
   - Suggestions that are clearly optional

If there are **no unresolved comments**, report that and exit.

### Step 4: Decide what to do with each comment

For each categorized comment, read the relevant code (3-5 lines around the comment location, plus whatever else you need to understand the claim) and **make the call yourself**. Every comment gets exactly one action:

- **Fix** — the comment is correct and the change is in scope for this PR.
- **Push back** — the comment is wrong, based on a misreading, or the fix costs more than it buys. You must be able to state the concrete reason.
- **Defer** — the comment is valid but out of scope for this PR (pre-existing issue, unrelated cleanup, larger refactor). Track it in an issue.
- **Clarification needed** — you genuinely cannot tell what the reviewer is asking for after reading the code. Ask the reviewer on the thread; do not ask the user.

Decision rules:

- **Must fix** items default to **Fix**. Only push back when you have verified the claim is incorrect (e.g. the "bug" cannot happen on any real path) and can explain why on the thread. Never push back on a security finding — fix it.
- **Should fix** items: Fix when the suggestion is correct and the change is local to files this PR already touches. Defer when it is correct but would widen the PR into unrelated code. Push back when it is not actually an improvement.
- **Nice to have** items: Fix plainly correct nits that touch files this PR already changes (renames, imports, sizing classes, typos) — they are cheap and reviewers notice when they are ignored. Push back on style preferences that contradict the codebase or the repo's conventions (CLAUDE.md / AGENTS.md / CONTRIBUTING.md). Defer anything that is valid but not worth touching here.
- **Bot findings** (Cursor Bugbot, Claude Code Review, etc.) are claims to verify, not orders. Trace a realistic path from a real caller or input to the failure. If the path is real and the fix is proportionate, Fix. If the path is far-fetched or the fix costs more than it prevents, Push back with the path you traced.
- When a fix is correct but you are unsure how to implement it, that is **not** a reason to stop. Read more code, look for existing patterns in the repo, and implement it. Reserve **Clarification needed** for genuinely ambiguous asks, and reply to the reviewer, never the user.
- Two comments asking for contradictory things: fix the one that matches the repo's conventions (CLAUDE.md / AGENTS.md / CONTRIBUTING.md) and the surrounding code, and explain the choice on the other thread.

For each item also record:

- **Effort**: trivial / moderate / involved (informational only — effort never changes the action)
- **Draft reply** (see templates below)

Response templates:

- **Fix:** `Fixed — {what changed}. {sha}` (sha filled in after commit)
- **Push back:** `Considered — {concrete justification for keeping current approach}`
- **Defer:** `Deferred — created #{issue} to track this` (issue number filled in later)
- **Clarification needed:** `Clarification needed — {specific question back to reviewer}`

### Step 5: Log the plan (do not wait for approval)

Print the plan so the user can see what you decided, then **immediately continue to Step 6**. This is a progress report, not a prompt.

```
## PR #{number} Code Review Comments ({total})

### Must fix ({count})

**1.** src/api/users.ts:45 · @reviewer · trivial
Use batched query instead of N+1
→ Action: Fix
→ Plan: Refactor to single WHERE IN query
→ Response: Fixed — batched the member lookup into a single query

---

**2.** Review summary · @reviewer
Should use Redis for caching
→ Action: Push back
→ Justification: In-memory cache is sufficient at our scale; Redis adds operational complexity with no measurable benefit yet.
→ Response: Considered — keeping in-memory cache for now. At current scale (~100 orgs), memory footprint is negligible and avoids a new infrastructure dependency. Happy to revisit if we see cache pressure.

---

### Should fix ({count})

**3.** src/utils/format.ts:12 · @reviewer · moderate
Extract into shared utility
→ Action: Fix
→ Plan: Move to `src/utils/slug.ts`, update imports
→ Response: Fixed — extracted to formatSlug() in src/utils/slug.ts

---

### Nice to have ({count})

**4.** src/components/Card.tsx:8 · @reviewer · trivial
nit: prefer size-4 over w-4 h-4
→ Action: Fix
→ Plan: Swap the classes
→ Response: Fixed — switched to size-4
```

Metadata line format: `**{#}.** {location} · @{author} · {effort}`
- Omit effort for Push back items (use `· @{author}` without trailing effort)
- Use `---` between items within a section and between sections
- If a section has zero items, show `### Must fix (0) (none)` as the header

Action row: `→ Action: {Fix | Push back | Defer | Clarification needed}` — always the first `→` line after the comment summary.

Detail line guidelines (prefixed with `→`):
- **Fix** items: `Action: Fix` + `Plan: {what will change}` + `Response: {draft}`
- **Push back** items: `Action: Push back` + `Justification: {why we disagree}` + `Response: {full draft}`
- **Defer** items: `Action: Defer` + `Justification: {why not now}` + `Response: {draft}`
- **Clarification needed**: `Action: Clarification needed` + `Question: {what's unclear}` + `Response: {draft asking reviewer}`

### Step 6: Implement every Fix item

For each item marked Fix:

1. Read the relevant file(s) and understand the context
2. Implement the fix following the project's code style
3. Use fixup commits when the fix belongs to an existing commit in the PR:
   - Run `git log {base}..HEAD --oneline` to find the relevant commit
   - Use `git commit --fixup {sha}` for the fix
   - After all fixes are committed, squash fixups with `GIT_SEQUENCE_EDITOR=: git rebase -i --autosquash {base}`
   - After the rebase, run `git diff origin/{branch_name}` and confirm the only differences are your intended fixes — nothing lost
4. If a fix requires substantial new work, create a regular commit. Regular commits must follow `{commit_convention}` when Step 0 found one.

If a fix turns out to be wrong or impossible once you are in the code (the reviewer misread, the change would break tests, the code no longer exists), do **not** stop. Downgrade the item to Push back or Clarification needed, update its draft reply with what you found, and move on.

### Step 7: Verify lint and typecheck pass

Before pushing, ensure the code passes all checks. Run lint and typecheck with the commands discovered in Step 0:

```bash
{lint_cmd}
{typecheck_cmd}
```

If your fixes touch code covered by the test suite, also run tests:

```bash
{test_cmd}
```

If any of them fail:

1. Fix the issues
2. Create additional fixup commits as needed
3. Squash with `GIT_SEQUENCE_EDITOR=: git rebase -i --autosquash {base}`
4. Re-run the failing commands to confirm they pass

Do not push until `{lint_cmd}` and `{typecheck_cmd}` (and `{test_cmd}`, where run) complete without errors. Skip any command Step 0 could not find, and if none were found, skip this step entirely and say so in the Step 11 summary.

### Step 8: Handle deferred items

For each item marked Defer, create a GitHub issue. Only pass `--label` flags when Step 0 found that the repo requires labels (`{issue_labels}`); if no label convention was found, create the issue without labels:

```bash
gh issue create --title "{brief description}" \
  --label "{label from {issue_labels}}" \
  --body "From PR #{pr_number} code review:

**Original comment by @{author}:**
> {full_comment}

**File:** {file}:{line}
**Context:** {brief context about what was being discussed}

---
Discovered during code review of #{pr_number}"
```

When labels are required, pick labels from the repo's convention, matching the comment's nature (defect vs cleanup vs new behaviour) and, where the convention has priority labels, use the lower priority unless the deferred item is a real bug in shipped code.

Note the issue number — fill it into the drafted response for this item.

### Step 9: Push changes and respond

1. Push the changes. Because autosquash rewrites the branch history, use a lease so you never clobber someone else's push:

   ```bash
   git push --force-with-lease origin {branch_name}
   ```

   If the lease is rejected, someone else pushed to the branch. Fetch, rebase your fixes on top of their work (`git fetch origin {branch_name} && git rebase origin/{branch_name}`), re-run Step 7, and push again.

2. Reply to **every** comment you processed using the **thread ID saved from Step 2** and the drafted response (updated with commit SHA / issue number as needed). No thread is left silent — Fix, Push back, Defer, and Clarification needed all get a reply.

   **Important:** Inline values directly in the mutation — do NOT use GraphQL `$variable` syntax (e.g. `$threadId`, `$body`). Shells like zsh interpret `$` before `gh` sees it, producing `Expected VAR_SIGN, actual: UNKNOWN_CHAR` errors. Single-quoting the `-f query='...'` argument does not reliably prevent this.

   ```bash
   # Inline both thread ID and response body directly in the mutation
   gh api graphql \
     -f query='mutation { addPullRequestReviewThreadReply(input: {pullRequestReviewThreadId: "<THREAD_ID>", body: "<RESPONSE>"}) { comment { id } } }'
   ```

   **Special characters in response bodies:** If the response contains quotes or newlines, write it to a temp file and substitute with `$(cat /tmp/response.txt)`, or escape quotes as `\"` in the inline string.

   For review summary comments and PR issue comments (which have no thread), reply as a single PR comment that addresses each point:

   ```bash
   gh pr comment {pr_number} --body-file /tmp/review-reply.md
   ```

   Response examples:
   - `Fixed — batched the member lookup into a single query. abc1234`
   - `Considered — keeping in-memory cache for now. At current scale (~100 orgs), memory footprint is negligible and avoids a new infrastructure dependency. Happy to revisit if we see cache pressure.`
   - `Deferred — created #42 to track this`
   - `Clarification needed — are you suggesting we move this to a shared util, or keep it local but rename?`

3. After replying to a thread with a "Fixed" response, **resolve the thread** so re-runs don't double-reply:

   ```bash
   gh api graphql \
     -f query='mutation { resolveReviewThread(input: {threadId: "<THREAD_ID>"}) { thread { isResolved } } }'
   ```

   Only resolve threads where the action was **Fix**. Do not resolve threads for Push back, Defer, or Clarification needed — those should remain open for the reviewer to respond.

### Step 10: Re-request review

Identify unique human reviewers who left comments and re-request their review:

```bash
gh pr edit {pr_number} --add-reviewer {reviewer1},{reviewer2}
```

Skip bot accounts (Cursor, Bugbot, Claude Code Review, etc.).

### Step 11: Summary

Provide a summary to the user. This is the one place you talk to them, and it is a report, not a question:

```
Done! Here's what I did:

**Fixed:**
- {description of fix 1} (fixup -> commit {sha})
- {description of fix 2} (new commit {sha})

**Pushed back:**
- {description}: {brief justification}

**Deferred:**
- Created #{issue_number}: {issue title}

**Asked for clarification:**
- {description}: {question asked}

**Responded to {n} comments, resolved {m} threads**

Pushed to origin/{branch_name}
Re-requested review from @{reviewer1}, @{reviewer2}

Ready for re-review!
```

If anything could not be completed (a check you could not run, a push that was rejected, a reply that failed), say so plainly in the summary with what you tried. If Step 0 found no lint, typecheck, or test command, say that verification was skipped for that reason. Never report a step as done that was skipped.

## Error handling

- If `gh` commands fail, check authentication with `gh auth status`. If auth is broken, stop and report — this is the one blocker you cannot work around.
- If there are merge conflicts after fixes or rebase, resolve them yourself: merge or rebase onto the base branch, regenerate lockfiles and generated files with the repo's own tooling rather than editing them by hand, re-run Step 7, and continue. Only report a conflict unresolved if both sides changed the same logic and picking either loses behaviour; in that case push everything else and describe the conflict in the summary.
- If a comment references code that no longer exists, reply on the thread saying so (`Considered — this code was removed in {sha}`) and treat the item as Push back.
- If a single reply or thread resolution fails, retry once, then continue with the remaining items and list the failure in the summary. One failed API call must not stop the run.
- If unsure how to fix an item, read more of the codebase for the existing pattern and implement it. Do not ask the user. If the ask itself is ambiguous, use Clarification needed and ask the reviewer on the thread.
