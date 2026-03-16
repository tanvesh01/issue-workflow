# Issue Workflow Skill

An OpenClaw-native skill for two related jobs:

- implement GitHub issues in isolated persistent worktrees
- resume the same worktrees later to address PR comments

The repository no longer relies on helper scripts. The workflow lives in [SKILL.md](/home/tanvesh01/issue-workflow/SKILL.md), and OpenClaw is responsible for setup, delegation, verification, and resume behavior.

## What Changed

- one persistent worktree per issue branch
- deterministic branch naming with `issue-<id>`
- Claude-first delegation, OpenCode fallback
- delegated agent creates and pushes the draft PR
- OpenClaw verifies the PR and tracks metadata
- PR comment runs reuse or recreate the tracked worktree and continue on the same branch

## Commands

```text
/feature-from-issue <issue-id> <repo>
/fix-issue <issue-id> <repo>
/address-pr-comments <pr-number> <repo>
/fix-pr-comments <pr-number> <repo>
/resolve-comments <pr-number> <repo>
```

## Runtime Requirements

- `gh`
- `git`
- `claude` preferred
- `opencode` fallback

## Persistent State

The workflow uses a skill-managed state root:

```text
~/.openclaw/issue-workflow/
```

Expected contents:

```text
metadata.json
worktrees/<owner>/<repo>/issue-<id>/
```

`metadata.json` stores the issue-to-branch/worktree/PR mapping OpenClaw uses to resume work.

## Execution Model

Issue run:

1. Fetch issue from GitHub.
2. Reuse or create the deterministic worktree for `issue-<id>`.
3. Delegate coding to `claude`, else `opencode`, else continue with OpenClaw tools.
4. Require the delegated agent to implement, test, push, and create a draft PR.
5. Verify that the PR exists and update metadata.

PR comment run:

1. Fetch PR and resolve the tracked worktree.
2. Reuse it, or recreate it from the PR branch if missing.
3. Pull latest branch state.
4. Fetch all PR comments.
5. Delegate comment resolution on the existing branch.
6. Verify remote updates and refresh metadata.

## Notes

- All PR comments are in scope, not only review comments.
- OpenClaw is the owner of the prompt and the final completion decision.
- The delegated coding CLI is an implementation worker, not the workflow controller.

## License

MIT License. See [LICENSE](/home/tanvesh01/issue-workflow/LICENSE).
