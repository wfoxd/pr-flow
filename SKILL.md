---
name: pr-flow
description: >-
  Runs the end-to-end GitHub pull request delivery workflow: wait for current
  work to finish, run iterative local code review and fix rounds until clean,
  commit and open a PR, run the GitHub Copilot review-and-fix loop until clean,
  merge the PR, and switch local checkout back to an updated default branch
  ready for new work. Use when the user invokes /pr-flow or asks to run the PR
  flow or ship changes through local review, PR, Copilot review, merge, and
  reset to main.
---

# PR Flow

Execute this end-to-end delivery workflow in order. Do not skip verification gates or move to the next stage while unresolved issues remain.

For cross-shell (`PowerShell` and `bash`/`zsh`) `gh` CLI, REST, and GraphQL commands, see [references/github-commands.md](./references/github-commands.md).

## Global Guardrails

- **Dynamic default branch**: Detect the repository's default branch via `git symbolic-ref refs/remotes/origin/HEAD` or `gh repo view --json defaultBranchRef --jq .defaultBranchRef.name`, falling back to `main` if unset. Use `<default-branch>` everywhere below.
- **No shell `sleep` for waiting**:
  - When waiting for background tasks, subagents, or swarm workers in Step 0, stop calling tools and let reactive wakeup resume you.
  - When waiting for GitHub Copilot reviews or CI checks, use the `schedule` tool (one-shot timer, e.g., `DurationSeconds: 45` or `60`), then end the turn so the timer wakes you up.
- **Loop safeguard**: Cap iterative review loops (Step 1 and Step 3) at **5 rounds** by default. If actionable issues still remain after round 5, stop and present the remaining issues to the user rather than looping indefinitely.
- **Branch & repository safety**:
  - Never commit feature work directly to `<default-branch>`.
  - Never force-push (`--force` / `--force-with-lease`) or run `git reset --hard` without explicit user approval.
  - Never stage secrets, `.env` files, build artifacts, or unrelated edits.
  - Never attempt to bypass branch protection rules or required checks.

---

## Step 0: Wait for Current Work to Complete

1. **Check active asynchronous work**:
   - Inspect running background tasks (`manage_task` with `Action: "list"`).
   - Inspect active subagents (`manage_subagents` with `Action: "list"`).
   - If operating in a swarm, check worker status (`swarm` with `action: "status"`).
   - If any work-in-progress task, subagent, or swarm worker is still running, stop calling tools and wait for reactive wakeup until it completes.
2. **Confirm requested work is complete**:
   - Ensure no requested edits or implementation tasks in the current conversation are unfinished or half-applied.
3. **Verify there are changes to ship**:
   - Run `git status` and check commits ahead of `origin/<default-branch>` (`git log origin/<default-branch>..HEAD --oneline`).
   - If there are no uncommitted changes, no commits ahead of `<default-branch>`, and no open PR for the current branch, stop and ask the user what work should be shipped.
4. **Verify green baseline**:
   - Run the project's relevant build, typecheck, lint, and test commands before starting code review. Fix any pre-existing failures in the current work first.

---

## Step 1: Iterative Local Code Review & Fix Loop

Repeat until a full review round finds **0 actionable issues** (max 5 rounds):

1. **Inspect the full change set**:
   - Review `git status` (including untracked files) and `git diff origin/<default-branch>` (plus working-tree/staged diffs).
2. **Conduct an independent, skeptical code review**:
   - When the diff is non-trivial, invoke a fresh read-only `research` subagent on the changed files and diff so the review has clean context; for small diffs, perform a strict independent reviewer pass.
   - Audit for:
     - **Correctness**: logic bugs, edge cases, null/error handling, race conditions, state consistency.
     - **Security & safety**: injection, auth/permission flaws, exposed secrets, unsafe file/shell handling.
     - **Design & scope**: API contract mismatches, dead code, unnecessary duplication, unintended scope creep.
     - **Tests**: missing coverage for new/changed behavior, weak or tautological assertions.
     - **Hygiene**: formatting, lint warnings, type errors, leftover debug logs or temporary files.
3. **Separate actionable issues from nitpicks**:
   - Fix genuine bugs, security/correctness risks, missing tests, and clear maintainability defects. Do not invent artificial nitpicks just to extend the loop.
4. **Exit or iterate**:
   - **Exit condition**: If **0 actionable issues** are found and build, lint, and tests are green, proceed to Step 2.
   - **If actionable issues exist**: Fix them cleanly, re-run build, lint, and tests to confirm no regressions, increment the local review round counter (stop and ask the user if > 5 rounds), and start a fresh review round on the updated diff.

---

## Step 2: Commit to GitHub & Open the PR

1. **Verify GitHub CLI auth and repository metadata**:
   - Confirm `gh auth status` succeeds and fetch `owner/repo` and `<default-branch>`.
2. **Ensure a feature branch**:
   - Check `git branch --show-current`.
   - If currently on `<default-branch>`, create and switch to a descriptive branch before committing:
     `git checkout -b <type>/<short-slug>` (e.g., `feat/...`, `fix/...`, `chore/...`).
3. **Sync cleanly with `origin/<default-branch>`**:
   - Run `git fetch origin <default-branch>` and check whether the branch integrates cleanly with `origin/<default-branch>`.
   - If there are merge conflicts, rebase onto or merge `origin/<default-branch>`, resolve conflicts, re-run tests, and do a quick review check on the conflict resolution.
4. **Stage and commit**:
   - Inspect `git status` and `git diff` to ensure temporary, secret, or unrelated files are excluded.
   - Stage intended changes and create a clear, descriptive commit. (Skip committing if all intended changes are already cleanly committed on the feature branch.)
5. **Push branch**:
   - Run `git push -u origin <branch>`.
6. **Open or reuse PR**:
   - Check `gh pr view --json number,url,state`.
   - If no open PR exists for `<branch>`, create one:
     `gh pr create --base <default-branch> --title "<clear title>" --body "<concise summary of changes and local verification>"`.
   - Record the PR number (`<PR>`) and URL.

---

## Step 3: GitHub Copilot Review Loop

Repeat until GitHub Copilot completes a review on the latest pushed `HEAD` commit SHA with **0 unresolved actionable issues** (max 5 rounds). See [references/github-commands.md](./references/github-commands.md) for exact commands.

1. **Check current commit and Copilot review state**:
   - Record `head_sha` (`git rev-parse HEAD`).
   - Check whether Copilot (`copilot-pull-request-reviewer[bot]` / `copilot-pull-request-reviewer`) already has a pending review request or a completed review for `head_sha`.
2. **Request Copilot review (if not already pending or reviewed for `head_sha`)**:
   - **Primary**: `gh pr edit <PR> --add-reviewer "@copilot"` (always quote `"@copilot"` for PowerShell compatibility).
   - **Fallback REST endpoint**: `gh api --method POST 'repos/{owner}/{repo}/pulls/<PR>/copilot-code-review'`.
     - `201`: review queued.
     - `409`: review request already pending — proceed to wait.
     - `422`: Copilot review unavailable or quota exhausted — stop, inform the user, and ask whether to proceed to merge or pause.
3. **Wait for Copilot review completion (no shell `sleep`)**:
   - Check PR reviews and pending `requested_reviewers` for `head_sha`. Completion means a Copilot review exists for `head_sha` (or submitted after the request timestamp) and Copilot is no longer listed as pending.
   - If still pending, call `schedule` with a one-shot timer (`DurationSeconds: 45` or `60`) prompting to re-check PR `<PR>` Copilot review status for `<head_sha>`, then end your turn.
   - If still pending after ~5 minutes (5–6 timer checks), inspect whether the review request timed out or failed, re-request once if needed, or ask the user how to proceed.
4. **Fetch all review feedback**:
   - Fetch Copilot review summaries (`pulls/<PR>/reviews`) and all unresolved PR `reviewThreads` via GraphQL.
   - Also inspect any unresolved human reviewer comments or requested changes — never silently ignore human feedback.
5. **Evaluate every unresolved comment against the actual code**:
   - **Valid / actionable issue**: Fix locally, run relevant build, lint, and tests, commit, and push to `<branch>`.
   - **False positive / intentional design**: Verify carefully before dismissing; never apply broken or counterproductive suggestions. Reply concisely on the thread explaining why no change is needed and resolve the thread.
6. **Exit or iterate**:
   - **Exit condition**: If there are no unresolved actionable issues for `head_sha` and all Copilot review threads are resolved (and no unresolved human blockers remain), exit Step 3 and proceed to Step 4.
   - **If fixes were pushed**:
     - Reply briefly on each addressed thread with the fix commit SHA and summary (`addPullRequestReviewThreadReply`).
     - Resolve each addressed thread (`resolveReviewThread`).
     - Increment the Copilot review round counter (stop and ask the user if > 5 rounds).
     - Re-request Copilot review on the new `HEAD` SHA (Step 3.2) and wait again (Step 3.3).

---

## Step 4: Merge the PR

1. **Check merge readiness and CI**:
   - Run `gh pr view <PR> --json state,mergeable,mergeStateStatus,statusCheckRollup,reviewDecision` and `gh pr checks <PR>`.
2. **Handle in-progress checks**:
   - If required checks are still running, wait using `schedule` (one-shot timer, `DurationSeconds: 30`–`60`) and end your turn.
3. **Handle failed checks**:
   - If any required check failed, inspect logs (`gh run view <run-id> --log-failed`), fix locally, run tests, commit, push, and return to **Step 3** (Copilot review loop) before merging.
4. **Handle merge conflicts or protection blocks**:
   - If `mergeable == CONFLICTING`, sync `<branch>` with `origin/<default-branch>`, resolve conflicts, re-test, push, and return to **Step 3**.
   - If branch protection blocks merge (e.g., awaiting mandatory human approval), stop and inform the user clearly instead of trying to work around protections.
5. **Select merge strategy and merge**:
   - Check repository settings (`gh repo view --json mergeCommitAllowed,squashMergeAllowed,rebaseMergeAllowed`).
   - Prefer `gh pr merge <PR> --merge --delete-branch` unless the user requested `--squash` or `--rebase`, or the repository disallows merge commits (in which case fall back to an allowed strategy with `--delete-branch`).
6. **Verify merged state**:
   - Confirm `gh pr view <PR> --json state,mergedAt` reports `state == "MERGED"`.

---

## Step 5: Checkout Default Branch Locally, Update, and Clean Up

1. **Verify clean working tree**:
   - Run `git status`. If unexpected local modifications remain, stash them or alert the user — never discard unknown edits.
2. **Switch to `<default-branch>`**:
   - Run `git checkout <default-branch>`.
3. **Prune and fast-forward update**:
   - Run `git fetch --prune origin` followed by `git pull --ff-only origin <default-branch>`.
   - If `--ff-only` fails due to local divergence on `<default-branch>`, stop and alert the user instead of force-resetting.
4. **Delete the merged local feature branch**:
   - Run `git branch -d <feature-branch>`.
   - If the PR was squash-merged or rebased and `-d` refuses solely due to differing commit SHAs, first verify `gh pr view <PR> --json state` confirms `MERGED`, then delete with `git branch -D <feature-branch>`.
5. **Verify ready state**:
   - Confirm `git branch --show-current` is `<default-branch>`, `git status` is clean, and `git log -1 --oneline` reflects the merged PR commit.
6. **Report final summary**:
   - Provide a concise summary containing:
     - Merged PR number and URL
     - Local review rounds completed (Step 1)
     - Copilot review rounds completed (Step 3)
     - Current local branch and `HEAD` commit, confirming the workspace is clean and ready for new work.
