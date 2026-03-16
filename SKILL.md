---
name: issue-workflow
version: 1.0.0
description: Full-Auto from PRD workflow - autonomously develop features from GitHub issues and address PR comments
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

Autonomously implement GitHub issues using the Full-Auto from PRD workflow, and automatically address PR review comments.

## Skills Included

1. **feature-from-issue** - Create features from GitHub issues
2. **address-pr-comments** - Address review comments on existing PRs

## What I Do

Transform GitHub issues into implemented features with zero human intervention:
- Fetch issue details from GitHub
- Plan implementation using PRD-style analysis
- Execute with parallel worker delegation
- Verify with comprehensive testing
- Create draft pull request

## When to Use Me

Use this skill when you want to:
- Implement a feature from a GitHub issue autonomously
- Fix a bug reported in GitHub issues
- Refactor code based on issue requirements
- Any task that can be fully described in a GitHub issue

## Usage

```
/feature-from-issue <issue-id> <repo> [--background]
/fix-issue <issue-id> <repo> [--background]
```

## Full-Auto from PRD Workflow

### Phase 1: Planning (PRD Analysis)

When invoked with an issue ID and repository:

1. Fetch the issue using `gh issue view`:
   ```bash
   gh issue view <issue-id> --repo <repo> --json title,body,url,number
   ```

2. Analyze the issue as a Product Requirements Document (PRD):
   - Extract acceptance criteria (what does "done" look like?)
   - Map technical approach (which files to change)
   - Identify dependencies and risks
   - Break into subtasks with clear deliverables

3. Create a detailed plan:
   ```markdown
   # Implementation Plan for Issue #<id>
   
   ## Requirements
   [Issue title and description]
   
   ## Acceptance Criteria
   - [ ] Criterion 1
   - [ ] Criterion 2
   
   ## Technical Approach
   - Files to modify: [list]
   - Dependencies: [list]
   - Breaking changes: [yes/no]
   
   ## Subtasks
   1. [Task 1]: [specific deliverable] → [files]
   2. [Task 2]: [specific deliverable] → [files]
   
   ## Risk Assessment
   - [Risk 1]: [mitigation]
   ```

4. Save plan to `.sisyphus/plans/issue-<id>-plan.md`

### Phase 2: Parallel Execution

Execute the plan using background task delegation:

1. For each independent subtask:
   - Spawn a background agent using `task` tool
   - Assign specific files and deliverables
   - Agents should:
     * Implement their assigned changes
     * Write/update tests
     * Run linting and type checking
     * Report completion

2. Monitor all workers and wait for completion:
   - Check task status periodically
   - Handle failures by retrying or adjusting approach
   - Collect all completed work

### Phase 3: Integration & Verification

1. Integrate all changes from parallel workers:
   - Combine file modifications
   - Resolve any conflicts
   - Ensure cohesive implementation

2. Run comprehensive verification:
   ```bash
   # Run all tests
   npm test
   # or
   cargo test
   # or
   pytest
   
   # Run linting
   npm run lint
   # or
   cargo clippy
   
   # Type checking
   npx tsc --noEmit
   ```

3. Verify against acceptance criteria from Phase 1

4. If anything fails:
   - Debug and identify the issue
   - Fix the problem
   - Re-run verification
   - Loop until everything passes (ralph-loop style)

5. Update the plan file with completion status

### Phase 4: Delivery

1. Create atomic commits with conventional commit messages:
   ```bash
   git add <files>
   git commit -m "feat: implement feature for issue #<id>
   
   - [Change 1]
   - [Change 2]
   
   Closes #<id>"
   ```

2. Push branch to origin:
   ```bash
   git push origin <branch-name>
   ```

3. Create DRAFT pull request using GitHub CLI:
   ```bash
   gh pr create --draft \
     --title "Fix #<id>: [brief summary]" \
     --body "Closes #<id>
   
   ## Summary
   [Implementation summary]
   
   ## Changes
   - [Change 1]
   - [Change 2]
   
   ## Testing
   - [Test coverage]
   "
   ```

## Implementation Requirements

### Prerequisites

Ensure these tools are available:
- `gh` (GitHub CLI) - authenticated
- `git` - configured with user.name and user.email
- `jq` - for JSON parsing
- `opencode` or OpenClaw runtime

### Workspace Setup

1. Create temp workspace or use provided workdir
2. Clone repository if not exists
3. Create feature branch: `fix-issue-<id>-<timestamp>`
4. Configure git user if not set

### Background Mode

When `--background` flag is provided:
- Run workflow in background using `nohup`
- Log output to `~/.local/share/omo-fix/omo-fix-issue-<id>-<timestamp>.log`
- Return immediately with PID for status checking

## Error Handling

- If issue fetch fails: Report error and exit
- If clone fails: Check permissions and repository access
- If tests fail: Debug, fix, and retry (up to max iterations)
- If PR creation fails: Report branch name for manual PR creation

## Completion Signal

When workflow completes successfully, output:
```
<promise>DONE</promise>
```

## Example Session

```
User: /feature-from-issue 42 tanvesh01/paper-portfolio

Agent: 🔍 Fetching issue #42 from tanvesh01/paper-portfolio...
       📋 Issue: Implement dark mode toggle
       📁 Created temp workspace: /tmp/tmp.abc123/paper-portfolio
       🌿 Creating branch: fix-issue-42-1773571419
       
       🚀 Starting Full-Auto from PRD workflow...
       Phase 1: Planning...
       Phase 2: Parallel Execution...
       Phase 3: Integration & Verification...
       Phase 4: Delivery...
       
       ✅ Draft PR created: https://github.com/tanvesh01/paper-portfolio/pull/43
       <promise>DONE</promise>
```

## Workflow Rules

1. Work 100% autonomously - DO NOT ask user for clarification
2. If stuck, debug and solve independently
3. Use all available tools: task, bash, file operations
4. Create plan file before implementing
5. Run ALL tests before creating PR
6. Must create DRAFT PR, not regular PR
7. Signal completion with `<promise>DONE</promise>`

## Notes

- This workflow uses `/ulw-loop` internally for persistence
- Parallel execution uses OpenClaw's `task` tool
- Git operations use isolated branches (never touch main/master)
- All changes are committed before PR creation

---

# Address PR Comments Workflow

Autonomously implement changes requested in GitHub PR review comments.

## Usage

```
/address-pr-comments <pr-number> <repo>
/fix-pr-comments <pr-number> <repo>
/resolve-comments <pr-number> <repo>
```

## Workflow

### Step 1: Fetch & Analyze Comments

```bash
gh pr view <pr-number> --repo <repo> --json comments,headRefName,baseRefName
```

The agent:
- Fetches all PR review comments
- Parses each to understand the request
- Categorizes by type: Code change / Style / Tests / Docs / Question / Outdated

### Step 2: Check Already Addressed

For each comment:
- Check recent git commits for references to the comment
- Look for commit messages mentioning the comment author or topic
- Skip if clearly already implemented
- Mark as "needs action" if not addressed

### Step 3: Group & Plan

Group related comments:
- Same file → Group together
- Same topic (e.g., all CSS variables) → Group
- Independent → Handle separately

Create execution plan with specific changes needed.

### Step 4: Execute Changes

For each comment group:

1. **Checkout PR branch**:
   ```bash
   git fetch origin
   git checkout <pr-branch>
   ```

2. **Make changes**:
   - Implement requested code changes
   - Add/update tests if requested
   - Update documentation if needed

3. **Verify**:
   ```bash
   npm test  # or equivalent
   npm run lint
   ```

4. **Commit** with descriptive message:
   ```bash
   git commit -m "fix: address PR feedback - increase button size
   
   Resolves comment by @alice on PR #<number>
   
   Changes:
   - Increased button size from 24px to 48px
   - Added focus styles for accessibility"
   ```

### Step 5: Push & Complete

1. Push all commits:
   ```bash
   git push origin <pr-branch>
   ```

2. Summary output showing addressed vs skipped comments.

## Comment Categories

| Category | Action |
|----------|--------|
| **Code Change** | ✅ Implement the requested change |
| **Style/Format** | ✅ Apply formatting rules |
| **Add Tests** | ✅ Write missing test coverage |
| **Documentation** | ✅ Update docs/comments |
| **Nitpick** | ✅ Address if straightforward |
| **Question** | ⏭️ **SKIP** - needs human discussion |
| **Outdated** | ⏭️ Skip if no longer relevant |

## Workflow Rules

1. **100% Autonomous** - Never ask user for clarification
2. **Skip Questions** - Questions need human discussion
3. **Atomic Commits** - Each addressed comment gets its own commit
4. **Test First** - Run tests before committing changes
5. **Clear Messages** - Commit messages reference comment author
6. **Signal Done** - Output `<promise>DONE</promise>` when complete

## Example Session

```
User: /address-pr-comments 43 tanvesh01/paper-portfolio

Agent: 🔍 Fetching PR #43 from tanvesh01/paper-portfolio...
       📋 Found 4 review comments
       
       📊 Analysis:
       ├─ @alice: "Button too small" → Code change needed
       ├─ @bob: "Use CSS variables" → Code change needed
       ├─ @charlie: "Add unit tests" → Testing needed
       └─ @dave: "Why this approach?" → Question (skip)
       
       🚀 Addressing 3 actionable comments...
       
       ✅ All changes committed:
          - abc123: fix: address PR feedback - increase button size
          - def456: refactor: use CSS variables for theming  
          - ghi789: test: add unit tests for theme switching
       
       📤 Pushed to PR #43
       
       Summary:
       ✅ 3 comments addressed
       ⏭️ 1 comment skipped (needs discussion)
       
       <promise>DONE</promise>
```

## Error Handling

- **PR not found**: Report error and exit
- **No comments**: Report "No actionable comments found"
- **Already addressed**: Skip gracefully
- **Tests fail**: Debug and retry (up to 3 attempts)
- **Push rejected**: Report branch conflict, needs manual resolution
- **All comments are questions**: Report "No actionable comments - all require discussion"
