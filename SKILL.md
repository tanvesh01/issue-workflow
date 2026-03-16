---
name: issue-workflow
version: 2.0.0
description: OpenClaw-native GitHub issue implementation and PR comment resolution workflow with persistent worktrees
triggers:
  - command: /feature-from-issue
  - command: /fix-issue
  - command: /address-pr-comments
  - command: /fix-pr-comments
  - command: /resolve-comments
permissions:
  - filesystem:read
  - filesystem:write
  - bash:execute
  - network:outbound
---

# Issue Workflow Skill

OpenClaw-native workflow for implementing GitHub issues and resuming work on PR comments.

This skill does not rely on helper scripts. OpenClaw owns the workflow, the main prompt, worktree lifecycle, and verification.

## Commands

```text
/feature-from-issue <issue-id> <repo>
/fix-issue <issue-id> <repo>
/address-pr-comments <pr-number> <repo>
/fix-pr-comments <pr-number> <repo>
/resolve-comments <pr-number> <repo>
```

## Core Model

- Use one persistent worktree per issue branch.
- Prefer delegating coding work to `claude`.
- If `claude` is unavailable, fall back to `opencode`.
- If neither external CLI is available, continue with OpenClaw's own tools.
- The delegated agent is responsible for implementation, testing, pushing commits, and creating the draft PR.
- OpenClaw is responsible for verification, retries, worktree reuse, and resume behavior.

## Persistent State

Store workflow state under a skill-managed root:

```text
~/.openclaw/issue-workflow/
```

Use this structure:

```text
~/.openclaw/issue-workflow/
  metadata.json
  worktrees/
    <repo-owner>/
      <repo-name>/
        issue-<id>/
```

`metadata.json` must track at least:

- `repo`
- `issue_number`
- `branch_name`
- `worktree_path`
- `pr_number` when known
- `pr_url` when known
- `last_synced_at`

Branch naming must be deterministic and reusable. Use:

```text
issue-<id>
```

## Issue Workflow

When invoked with `/feature-from-issue` or `/fix-issue`:

1. Fetch the issue:
   ```bash
   gh issue view <issue-id> --repo <repo> --json number,title,body,url
   ```
2. Load `metadata.json` and check whether a worktree already exists for this issue.
3. If it exists, reuse it.
4. If it does not exist:
   - ensure the target repo exists locally or clone it into the skill-managed area
   - create branch `issue-<id>`
   - create a new git worktree for that branch
   - persist metadata immediately
5. Sync the worktree before delegating:
   ```bash
   git fetch origin
   git checkout issue-<id>
   git pull --ff-only origin issue-<id>
   ```
   If the remote branch does not exist yet, continue on the local branch without pulling.
6. Change into the worktree and inspect the repository before delegating.
7. Compose the delegated prompt from:
   - the issue title/body/url
   - the repository path and branch
   - this skill's execution requirements
8. Delegate to `claude`, or to `opencode` if `claude` is unavailable.
9. Require the delegated agent to:
   - inspect the codebase first
   - produce a concrete implementation plan
   - implement the issue
   - add or update tests when appropriate
   - run the relevant verification commands
   - commit and push changes
   - create a draft PR
   - return the PR number or URL in its final output
10. Verify the outcome with GitHub and git state:
   - branch exists and is pushed
   - a draft PR exists for the branch
   - the PR matches the intended issue/repo
11. Update metadata with the PR details and sync timestamp.

## PR Comment Workflow

When invoked with `/address-pr-comments`, `/fix-pr-comments`, or `/resolve-comments`:

1. Fetch the PR:
   ```bash
   gh pr view <pr-number> --repo <repo> --json number,title,url,headRefName,baseRefName
   ```
2. Resolve the expected worktree using metadata by PR number, repo, or branch name.
3. If the tracked worktree exists, reuse it.
4. If it is missing, recreate it automatically from the PR branch and repair metadata.
5. Sync the worktree:
   ```bash
   git fetch origin
   git checkout <head-branch>
   git pull --ff-only origin <head-branch>
   ```
6. Fetch all PR comments, including general PR conversation comments and inline review comments on diffs.
7. Build the full comment payload from both:
   ```bash
   gh pr view <pr-number> --repo <repo> --json comments,reviews,reviewRequests
   gh api graphql -f query='
     query($owner: String!, $repo: String!, $number: Int!) {
       repository(owner: $owner, name: $repo) {
         pullRequest(number: $number) {
           reviewThreads(first: 100) {
             nodes {
               isResolved
               isOutdated
               path
               line
               comments(first: 100) {
                 nodes {
                   author { login }
                   body
                   createdAt
                   url
                 }
               }
             }
           }
         }
       }
     }' -F owner=<owner> -F repo=<repo-name> -F number=<pr-number>
   ```
   Include both the `gh pr view` payload and the review thread payload in the delegated context.
8. Compose the delegated prompt from:
   - PR metadata
   - full comment payload
   - repository path and branch
   - this skill's execution requirements
9. Delegate to `claude`, or to `opencode` if `claude` is unavailable.
10. Require the delegated agent to:
   - review all PR comments
   - review inline review-thread comments on diffs
   - decide which comments need code changes and which should be skipped with explanation
   - implement actionable feedback
   - add or update tests when appropriate
   - run the relevant verification commands
   - commit and push updates to the existing PR branch
11. Verify the result:
   - remote branch is updated or the delegated agent explicitly concluded no code changes were needed
   - the PR still points to the expected branch
12. Update metadata with the latest sync timestamp.

## Delegated Execution Requirements

For delegated runs, OpenClaw must tell the coding CLI:

- work only inside the prepared worktree
- inspect the repository before editing
- make autonomous decisions within the issue or PR-comment scope
- run appropriate tests, linting, and type checks for the repo
- fix failures introduced during the run
- keep commits coherent
- push changes before finishing
- create a draft PR for issue runs
- summarize changes, verification, and remaining risks in the final output

OpenClaw should prefer `claude` for delegated execution. If `claude` is unavailable, use `opencode` with the same behavioral contract.

## Verification Rules

Do not trust delegated output alone. OpenClaw must verify with direct commands.

Issue runs are complete only when:

- the branch exists locally and remotely
- a draft PR exists
- the PR targets the expected repo and branch

PR comment runs are complete only when:

- the correct branch is checked out and synced before delegation
- pushed updates are visible remotely, or no-op completion is justified

## Workflow Rules

1. Work autonomously and do not ask the user for clarification unless blocked by missing access or impossible ambiguity.
2. Never work directly on `main` or `master`.
3. Reuse the tracked issue worktree whenever possible.
4. Recreate missing worktrees automatically for PR-comment runs.
5. Treat all PR comments as in scope, not only code review comments.
6. Create draft PRs only.
7. OpenClaw owns verification and final completion decisions.

## Failure Handling

- If issue or PR fetch fails, stop and report the exact GitHub failure.
- If the worktree cannot be created or restored, stop and report the exact git failure.
- If delegated execution fails, retry with adjusted instructions or continue with OpenClaw's own tools.
- If verification fails after delegated success, do not mark the workflow complete; diagnose and continue.

## Completion Signal

When the workflow is fully verified, output:

```text
<promise>DONE</promise>
```
