---
description: Manage PR lifecycle — CI monitoring, auto-fix, review comments, merge
---

# Manage PR: $ARGUMENTS

You are managing the full lifecycle of a pull request. Your goal is to shepherd this PR from CI through review to merge-ready status, fixing issues along the way.

## Parse Arguments

`$ARGUMENTS` should be one of:
- `<PR_NUMBER>` — manage an existing PR from the start
- `--create` — commit current changes, create a new PR, then manage it

## Setup

### For `--create`:
1. Verify you're NOT on `main` (create a branch if needed)
2. Stage and commit changes (follow commit conventions)
3. Push: `git push -u origin $(git branch --show-current)`
4. Create PR against upstream: `gh pr create --repo msanvido/co-reader --head legalcodes:$(git branch --show-current) --base main --fill`
5. Extract the PR number from the output

### For `<PR_NUMBER>`:
1. Check out the PR branch: `gh pr checkout <PR_NUMBER> --repo msanvido/co-reader`
2. Get the PR URL: `gh pr view <PR_NUMBER> --repo msanvido/co-reader --json url --jq .url`

## Phase 1: Wait for CI

Poll CI status in a loop:
```bash
gh pr checks <PR_NUMBER> --repo msanvido/co-reader --watch
```

If `--watch` is not available or times out, poll manually:
```bash
gh pr checks <PR_NUMBER> --repo msanvido/co-reader
```
Retry every 30 seconds until all checks pass, fail, or 30 minutes elapse.

- **All checks pass**: Proceed to **Phase 2**.
- **Any check fails**: Proceed to **Phase 1a**.
- **PR is merged or closed**: Report to user and STOP.
- **Timeout (30 min)**: Report to user and STOP.

### Phase 1a: Auto-Fix CI Failure (max 5 attempts)

Track the attempt count. If this is attempt 6+, report to user and STOP.

1. **Read the failure logs**: `gh run view <RUN_ID> --repo msanvido/co-reader --log-failed`
2. **Fix the code**. Focus only on what's broken. No unrelated changes.
3. **Verify locally**:
   - Run tests: `npm test`
   - Build: `npm run build`
4. **Commit** the fix with a descriptive message. Do NOT amend previous commits.
5. **Push**: `git push`
6. **Go back to Phase 1** — wait for CI again.

**If the same failure repeats** (logs look identical to last attempt), it may be a flaky test or an issue that needs human attention. Report and STOP after 3 identical failures.

## Phase 2: Review Comments

Track the review cycle count (max 10 total cycles).

### 2a. Check for GitHub Review Comments

Check for review comments from Copilot or human reviewers:

```bash
gh api repos/msanvido/co-reader/pulls/<PR_NUMBER>/reviews --paginate
gh api repos/msanvido/co-reader/pulls/<PR_NUMBER>/comments --paginate
gh api repos/msanvido/co-reader/issues/<PR_NUMBER>/comments --paginate
```

### 2b. Address Comments

- If there are **unresolved comments**, address each one:
  1. For each comment, analyze and act:
     - **Valid and unaddressed**: Fix the code, reply with what was changed
     - **Invalid or mistaken**: Reply with technical justification
     - **Already addressed**: Reply noting which commit fixed it
     - **Acceptable/low-priority**: Acknowledge and explain the tradeoff
  2. Run tests and build after all fixes: `npm test && npm run build`
  3. Commit and push
  4. Increment review cycle count
  5. **Go back to Phase 1** (CI must pass after changes)
  6. Then **back to Phase 2** for the next review cycle

- If there are **no unresolved comments**: Proceed to **Phase 3**.

**Exhausted (10 review cycles):**
Report to the user that the PR has gone through too many cycles and needs human attention.

## Phase 3: Merge-Ready

1. Report to the user:
   - PR URL
   - Summary: all CI green, review comments addressed
   - Suggest merging (but do NOT merge automatically)
2. If the user says to merge:
   ```bash
   gh pr merge <PR_NUMBER> --repo msanvido/co-reader --squash --delete-branch
   ```

## Important Rules

- **Never amend commits** — always create new commits
- **Never make unrelated changes** — fix only what's broken
- **Always verify locally** before pushing (tests + build)
- **Always push after committing** so CI picks up the changes
- **Never merge without explicit user approval**
- **Never use `--admin`** to bypass branch protection
