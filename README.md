# Issue Workflow Skill

An OpenClaw skill for autonomously implementing GitHub issues using the Full-Auto from PRD workflow.

## What It Does

Transforms GitHub issues into implemented features with **zero human intervention**:

1. **Planning** - Analyzes issue as a Product Requirements Document (PRD)
2. **Parallel Execution** - Spawns background workers for independent tasks
3. **Integration & Verification** - Combines work and runs comprehensive tests
4. **Delivery** - Creates atomic commits and draft pull request

## Installation

```bash
openclaw skills install https://github.com/tanvesh01/issue-workflow.git
```

Or manually clone to your OpenClaw skills directory:
```bash
git clone https://github.com/tanvesh01/issue-workflow.git ~/.openclaw/workspace/skills/issue-workflow
```

## Prerequisites

Ensure these tools are installed and configured:

- **GitHub CLI (`gh`)** - Must be authenticated (`gh auth login`)
- **Git** - Must have user.name and user.email configured
- **jq** - JSON parsing utility
- **OpenClaw or OpenCode** - The skill runs within OpenClaw runtime

## Skills Included

This repository contains two related skills:

1. **`feature-from-issue`** - Create features from GitHub issues (Full-Auto PRD workflow)
2. **`address-pr-comments`** - Address review comments on existing PRs

---

## Usage: feature-from-issue

### Basic Usage

```
/feature-from-issue <issue-id> <repo>
```

Example:
```
/feature-from-issue 42 tanvesh01/paper-portfolio
```

### Alternative Commands

```
/fix-issue <issue-id> <repo>      # Same workflow, for bug fixes
```

### Background Mode

Run without blocking your terminal:

```
/feature-from-issue 42 tanvesh01/paper-portfolio --background
```

Then check status:
```
# View all running jobs
omo-feature --list

# Check specific job status
omo-feature --status <pid>

# View logs
omo-feature --logs 42

# Kill a job
omo-feature --kill <pid>
```

---

## Usage: address-pr-comments

### Basic Usage

```
/address-pr-comments <pr-number> <repo>
```

Example:
```
/address-pr-comments 43 tanvesh01/paper-portfolio
```

### Alternative Commands

```
/fix-pr-comments <pr-number> <repo>      # Same workflow
/resolve-comments <pr-number> <repo>     # Same workflow
```

### What It Does

Automatically addresses review comments on PRs:

1. **Fetches all comments** from the PR
2. **Analyzes each comment** - determines if code change, question, or already addressed
3. **Groups related comments** - same file or topic
4. **Implements changes** - makes code changes, runs tests, creates commits
5. **Pushes commits** - each addressed comment gets an atomic commit
6. **Summarizes** - reports what was addressed vs skipped

### Comment Categories

| Type | Action |
|------|--------|
| Code Change | ✅ Implement the change |
| Style/Format | ✅ Apply formatting |
| Add Tests | ✅ Write missing tests |
| Documentation | ✅ Update docs |
| **Question** | ⏭️ Skip (needs human) |
| **Outdated** | ⏭️ Skip |

### Example Session

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

## How It Works

### Phase 1: Planning (PRD Analysis)

The agent:
- Fetches the GitHub issue using `gh issue view`
- Analyzes it as a Product Requirements Document
- Creates acceptance criteria and technical approach
- Breaks work into subtasks
- Saves plan to `.sisyphus/plans/issue-<id>-plan.md`

### Phase 2: Parallel Execution

The agent:
- Spawns background workers for each subtask using OpenClaw's `task` tool
- Each worker implements their assigned portion
- Workers write tests and run linting
- All workers report completion

### Phase 3: Integration & Verification

The agent:
- Integrates all worker changes
- Runs full test suite
- Runs linting and type checking
- Verifies against acceptance criteria
- Loops until everything passes (ralph-loop style)
- Updates plan with completion status

### Phase 4: Delivery

The agent:
- Creates atomic commits with conventional commit messages
- Pushes branch to origin
- Creates a **DRAFT** pull request using `gh pr create --draft`
- Outputs `<promise>DONE</promise>` on completion

## Workflow Rules

1. **100% Autonomous** - Never asks user for clarification
2. **Self-Healing** - Debugs and solves problems independently
3. **Test-First** - Runs all tests before creating PR
4. **Draft PRs Only** - Never creates non-draft PRs
5. **Isolated Branches** - Never touches main/master directly

## Example Session

```
User: /feature-from-issue 42 tanvesh01/paper-portfolio

Agent: 🔍 Fetching issue #42 from tanvesh01/paper-portfolio...
       📋 Issue: Implement dark mode toggle
       📁 Created temp workspace: /tmp/tmp.abc123/paper-portfolio
       🌿 Creating branch: feature-issue-42-1773571419
       
       🚀 Starting Full-Auto from PRD workflow...
       Phase 1: Planning...
       Phase 2: Parallel Execution...
       Phase 3: Integration & Verification...
       Phase 4: Delivery...
       
       ✅ Draft PR created: https://github.com/tanvesh01/paper-portfolio/pull/43
       <promise>DONE</promise>
```

## Configuration

The skill requires these permissions (automatically requested on first use):
- `filesystem:read` - Read repository files
- `filesystem:write` - Write changes and plans
- `bash:execute` - Run git and gh commands
- `network:outbound` - Fetch issue data from GitHub

## Troubleshooting

### "gh not authenticated"
Run `gh auth login` and authenticate with your GitHub account.

### "Permission denied"
The skill needs bash execution permissions. Grant them when prompted.

### Issue not found
Ensure the issue exists and is accessible. Check that you have access to the repository.

### Tests failing
The agent will automatically retry failed tests. If it can't resolve after max iterations, check the logs for details.

## Architecture

This skill implements the **Full-Auto from PRD** workflow pattern:
- `$ralplan` → Planning with structured analysis
- `$team` → Parallel execution with background workers
- `$ralph` → Persistence until verified complete

It uses OpenClaw's built-in tools:
- `bash` - Execute git and gh commands
- `task` - Spawn parallel background workers
- File operations - Read/write plans and code

## Contributing

Contributions welcome! Please ensure:
- SKILL.md uses imperative form (not "You should")
- Scripts are tested before submitting
- Documentation is updated for new features

## License

MIT License - see LICENSE file for details.

## Related

- [Oh My OpenCode](https://github.com/code-yeongyu/oh-my-openagent) - The orchestration layer powering this workflow
- [OpenClaw](https://openclaw.ai) - The AI assistant platform
