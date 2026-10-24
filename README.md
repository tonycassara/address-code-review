# address-code-review

A Claude Code skill that works through the code review comments on your current pull request. It fetches the unresolved threads, decides for each one whether to fix it, push back, defer it to an issue, or ask the reviewer to clarify, then makes the fixes, replies on every thread, pushes, and re-requests review. It runs autonomously: no check-ins, no approval prompts, just a report at the end. It works in any repo because it reads the repo's own conventions (agent docs, package scripts, Makefile, CI) to find the lint, typecheck, and test commands and the commit and issue-label rules.

## How it works

0. Learn the repo's conventions: lint, typecheck, and test commands, commit-message style, and issue labels.
1. Identify the PR for the current branch.
2. Fetch unresolved, non-outdated review threads, review summaries, and PR comments.
3. Categorize each comment as must fix, should fix, or nice to have.
4. Decide one action per comment: Fix, Push back, Defer, or Clarification needed.
5. Log the plan (a progress report, not a prompt).
6. Implement every fix, as fixup commits squashed with autosquash.
7. Verify with the repo's lint and typecheck commands, plus tests when the fixes touch tested code.
8. Defer out-of-scope items to GitHub issues.
9. Push with `--force-with-lease`, reply on every thread, and resolve the fixed ones.
10. Re-request review from the human reviewers.
11. Print a summary of what was fixed, pushed back, deferred, and asked.

## Install

### Claude Code plugin marketplace

```
/plugin marketplace add tonycassara/address-code-review
/plugin install address-code-review@tonycassara
```

### Copy into a project

```bash
git clone https://github.com/tonycassara/address-code-review.git
# Project-level
cp -r address-code-review/skills/address-code-review .claude/skills/
# Or user-wide
cp -r address-code-review/skills/address-code-review ~/.claude/skills/
```

### Codex

Copy the same folder into `.codex/skills/`:

```bash
cp -r address-code-review/skills/address-code-review .codex/skills/
```

## Usage

Check out a branch that has an open PR and run:

```
/address-code-review
```

Nothing else needs to be configured.

## Requirements

- `gh` CLI installed and authenticated (check with `gh auth status`).
- `git`.
- A repo with a GitHub remote and an open PR for the current branch.
- Optional but recommended: a `CLAUDE.md`, `AGENTS.md`, or `CONTRIBUTING.md` that names your lint, typecheck, and test commands and your commit and label conventions. Without one, the skill falls back to `package.json` scripts, a `Makefile`, and your CI workflows.

## Decision rubric

| Severity tier | How it is detected | Default action | When it pushes back | When it defers |
| --- | --- | --- | --- | --- |
| Must fix | Latest human review is `REQUEST_CHANGES`, comment is marked blocking, security or correctness issue, or words like "must", "required", "critical" | Fix | Only when it has verified the claim is wrong (the bug cannot happen on any real path) and can say why | Rarely; must-fix items are fixed in the PR |
| Should fix | Better patterns, performance concerns, words like "should", "recommend", "consider" | Fix when correct and local to files the PR already touches | When the suggestion is not actually an improvement | When it is correct but would widen the PR into unrelated code |
| Nice to have | Nits, style preferences, comments starting with "nit:", "optional:", "minor:" | Fix plainly correct nits in files the PR already changes | When the preference contradicts the codebase or the repo's conventions | When it is valid but not worth touching here |

Bot findings (Cursor Bugbot, Claude Code Review, and similar) are treated as claims to verify, not orders. The skill traces a realistic path from a real caller or input to the failure before acting. A security finding is never pushed back on; it gets fixed.

## Example replies

Fix:

```
Fixed — batched the member lookup into a single query. abc1234
```

Push back:

```
Considered — keeping in-memory cache for now. At current scale (~100 orgs), memory footprint is negligible and avoids a new infrastructure dependency. Happy to revisit if we see cache pressure.
```

Defer:

```
Deferred — created #42 to track this
```

Clarification needed:

```
Clarification needed — are you suggesting we move this to a shared util, or keep it local but rename?
```

## Safety notes

- Fixes land as fixup commits and are squashed with `git rebase --autosquash`, so the branch history stays clean. After the rebase it diffs against the remote branch to confirm nothing was lost.
- Pushes use `--force-with-lease`, so it never clobbers someone else's push. If the lease is rejected, it fetches, rebases its fixes onto the new remote state, re-runs the checks, and pushes again.
- Only threads whose action was Fix get resolved. Push back, Defer, and Clarification needed threads stay open for the reviewer to answer.
- It is safe to re-run. It skips resolved threads, outdated threads, and threads whose latest comment is its own reply.
- It never commits or pushes while lint or typecheck are failing.
- The only hard stop is a `gh` authentication failure. Other failures (a single reply, a rejected push) are retried or reported in the summary.

## License

MIT. See [LICENSE](LICENSE).
