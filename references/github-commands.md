# GitHub CLI & API Command Reference

Cross-shell (`PowerShell` and `bash`/`zsh`) commands for `pr-flow`. Uses `gh api`'s built-in `{owner}` and `{repo}` placeholders (resolved from the current repository).

## 1. Shell Quoting Notes

- **Quote `"@copilot"`**: Always wrap `"@copilot"` in quotes; PowerShell treats an unquoted leading `@` as a splatting operator.
- **Single-quote GraphQL queries**: Always wrap the GraphQL `query='...'` argument in single quotes `'...'` so `$owner`, `$repo`, `$pr`, `$threadId`, and `$body` are passed literally to GraphQL instead of being expanded as shell variables.
- **Quote REST paths with `{owner}`/`{repo}`**: Wrap endpoints such as `'repos/{owner}/{repo}/pulls/<PR>/reviews'` in single quotes so braces are not treated specially by shells.

---

## 2. Detect Default Branch & Repo Metadata

```bash
# Git remote HEAD (fast local lookup; may require `git remote set-head origin -a` if missing)
git symbolic-ref refs/remotes/origin/HEAD

# GitHub CLI repository metadata & allowed merge strategies
gh repo view --json nameWithOwner,defaultBranchRef,mergeCommitAllowed,squashMergeAllowed,rebaseMergeAllowed
```

Extract the default branch directly:

```bash
gh repo view --json defaultBranchRef --jq '.defaultBranchRef.name'
```

---

## 3. Request Copilot Review

### Primary (`gh pr edit`)

```bash
gh pr edit <PR> --add-reviewer "@copilot"
```

### Fallback (Dedicated Copilot REST Endpoint)

```bash
gh api --method POST 'repos/{owner}/{repo}/pulls/<PR>/copilot-code-review'
```

Optional effort parameter (`"lite"`, `"balanced"`, `"max"`, `"adaptive"`):

```bash
gh api --method POST 'repos/{owner}/{repo}/pulls/<PR>/copilot-code-review' -f review_effort="balanced"
```

Response status codes:
- `201 Created`: Copilot code review queued.
- `409 Conflict`: A Copilot code review request is already pending for this PR.
- `422 Validation Failed`: Repository/actor ineligible, Copilot quota exhausted, or invalid parameters.

---

## 4. Check Copilot Pending Status & Submitted Reviews

Inspect PR status, `HEAD` SHA, pending review requests, and review decisions:

```bash
gh pr view <PR> --json headRefOid,reviewRequests,reviews,state,mergeable,mergeStateStatus,reviewDecision
```

Check currently pending reviewers via REST:

```bash
gh api 'repos/{owner}/{repo}/pulls/<PR>/requested_reviewers' --jq '.users[].login'
```

List submitted reviews with commit SHA and timestamp via REST:

```bash
gh api 'repos/{owner}/{repo}/pulls/<PR>/reviews' --jq '.[] | {id, user: .user.login, state, commit_id, submitted_at, body}'
```

**Matching Copilot logins** (compare case-insensitively):
- REST `user.login` / `requested_reviewers`: `copilot-pull-request-reviewer[bot]` (or `Copilot` / `copilot-pull-request-reviewer`).
- GraphQL `author.login`: `copilot-pull-request-reviewer` (or `Copilot` / `copilot-pull-request-reviewer[bot]`).

A Copilot review round for `head_sha` is complete when `reviews` contains a Copilot review with `commit_id == head_sha` (or `submitted_at` after the latest request for `head_sha`) and Copilot is no longer in pending reviewers.

---

## 5. Fetch Unresolved Review Threads (GraphQL)

Fetch unresolved review threads (include `--paginate` with `$endCursor` and `pageInfo` so PRs with more than 100 threads are handled completely):

```bash
gh api graphql --paginate -F owner='{owner}' -F repo='{repo}' -F pr=<PR> -f query='
query($owner: String!, $repo: String!, $pr: Int!, $endCursor: String) {
  repository(owner: $owner, name: $repo) {
    pullRequest(number: $pr) {
      reviewThreads(first: 100, after: $endCursor) {
        pageInfo {
          hasNextPage
          endCursor
        }
        nodes {
          id
          isResolved
          isOutdated
          path
          line
          comments(first: 50) {
            nodes {
              id
              body
              createdAt
              author {
                login
              }
            }
          }
        }
      }
    }
  }
}' --jq '.data.repository.pullRequest.reviewThreads.nodes[] | select(.isResolved == false)'
```

---

## 6. Reply to and Resolve a Review Thread (GraphQL)

### Reply to a Thread

```bash
gh api graphql -f threadId='<THREAD_ID>' -f body='Fixed in <COMMIT_SHA>: <brief explanation>' -f query='
mutation($threadId: ID!, $body: String!) {
  addPullRequestReviewThreadReply(input: {pullRequestReviewThreadId: $threadId, body: $body}) {
    comment {
      id
      url
    }
    thread {
      id
      isResolved
    }
  }
}'
```

### Resolve a Thread

```bash
gh api graphql -f threadId='<THREAD_ID>' -f query='
mutation($threadId: ID!) {
  resolveReviewThread(input: {threadId: $threadId}) {
    thread {
      id
      isResolved
    }
  }
}'
```

---

## 7. Check CI & Merge

```bash
# Inspect CI check status
gh pr checks <PR>

# Inspect failed workflow run logs if a check fails
gh run view <RUN_ID> --log-failed

# Merge (default preferred strategy) and delete branch
gh pr merge <PR> --merge --delete-branch

# Alternative strategies (when requested or required by repo settings)
gh pr merge <PR> --squash --delete-branch
gh pr merge <PR> --rebase --delete-branch

# Verify PR is merged
gh pr view <PR> --json state,mergedAt
```
