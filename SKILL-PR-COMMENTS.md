---
name: address-pr-comments
version: 1.0.0
description: Autonomously address review comments on GitHub PRs
triggers:
  - command: /address-pr-comments
  - command: /fix-pr-comments
  - command: /resolve-comments
permissions:
  - filesystem:read
  - filesystem:write
  - bash:execute
  - network:outbound
---

# Address PR Comments Skill

Autonomously implement changes requested in GitHub PR review comments with **zero human intervention**.

## What I Do

Transform PR review comments into implemented changes:
- Fetch all comments from a PR
- Analyze what changes are needed for each
- Check if already addressed (via git history)
- Implement unaddressed comments
- Create atomic commits for each change
- Push and optionally respond to comments

## When to Use Me

Use this skill when you want to:
- Address code review feedback automatically
- Implement requested changes from reviewers
- Batch-process multiple PR comments
- Keep PRs moving without manual intervention

## Usage

```
/address-pr-comments <pr-number> <repo> [--background]
```

Examples:
```
/address-pr-comments 43 tanvesh01/paper-portfolio
/address-pr-comments 43 tanvesh01/paper-portfolio --background
/fix-pr-comments 43 owner/repo
```

## Workflow

### Step 1: Fetch & Analyze Comments

```bash
gh pr view <pr-number> --repo <repo> --json comments,headRefName,baseRefName
```

The agent:
- Fetches all PR comments (review comments, not issue comments)
- Parses each to understand the request
- Categorizes by type:
  - Code change (refactoring, bug fix, feature)
  - Style/formatting
  - Testing (add missing tests)
  - Documentation
  - Questions (skip - needs human)

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

Create execution plan:
```
Comment Group 1: UI changes
  - File: src/components/Button.tsx
  - Changes: Increase size, add accessibility
  
Comment Group 2: Testing  
  - Files: src/**/*.test.ts
  - Changes: Add unit tests for theme switching
```

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
   git add <files>
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

2. Summary:
   ```
   ✅ Addressed 3 of 4 comments
   - @alice: Button size → Fixed in commit abc123
   - @bob: CSS variables → Fixed in commit def456
   - @charlie: Unit tests → Fixed in commit ghi789
   
   ⏭️ Skipped 1 comment:
   - @dave: "Should we use a different approach?" (Question - needs discussion)
   
   📝 All changes pushed to PR #<number>
   ```

## Comment Categories

| Category | Action |
|----------|--------|
| **Code Change** | Implement the requested change |
| **Style/Format** | Apply formatting rules |
| **Add Tests** | Write missing test coverage |
| **Documentation** | Update docs/comments |
| **Nitpick** | Address if straightforward |
| **Question** | **SKIP** - needs human discussion |
| **Outdated** | Skip if no longer relevant |

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
       
       ✏️ Implementing @alice's feedback...
       ✏️ Implementing @bob's feedback...
       ✏️ Implementing @charlie's feedback...
       
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

## Workflow Rules

1. **100% Autonomous** - Never ask user for clarification
2. **Skip Questions** - Questions need human discussion
3. **Atomic Commits** - Each addressed comment gets its own commit
4. **Test First** - Run tests before committing changes
5. **Clear Messages** - Commit messages reference comment author
6. **Signal Done** - Output `<promise>DONE</promise>` when complete

## Notes

- Uses `/ulw-loop` for persistence
- Leverages git history to detect already-addressed comments
- Groups related changes to minimize commits
- Skips questions and discussions automatically
- Can handle PRs with dozens of comments

## Troubleshooting

**"No actionable comments found"**
- All comments may already be addressed
- Comments might be questions requiring discussion
- Check PR manually to verify

**"Tests failing after changes"**
- Agent will retry with fixes
- If still failing, may need architectural changes
- Consider manual intervention

**"Branch has conflicts"**
- PR branch is behind base branch
- Run `/sync-pr <pr-number> <repo>` first (if available)
