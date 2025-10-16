---
name: Using Beads Issue Tracker
description: Git-backed issue tracker for AI agents with dependency tracking and persistent task memory
when_to_use: when partner requests complex work with 3+ steps, when needing to track blockers and dependencies, when resuming work across sessions, or when coordinating parallel work with multiple tasks
version: 1.0.0
---

# Using Beads Issue Tracker

## Overview

Beads (bd) is a lightweight, git-backed issue tracker specifically designed for AI coding agents. It prevents "agent amnesia" during long conversations by maintaining persistent task memory with graph-based dependency tracking.

## Core Concept

Traditional issue trackers are built for human project managers. Beads assumes AI agents are the primary users and provides:

- **Persistent memory**: Tasks survive across sessions
- **Dependency tracking**: Four dependency types (blocks, related, parent-child, discovered-from)
- **Ready work detection**: Automatically identifies unblocked issues
- **Git integration**: JSONL records versioned with your code

## Architecture

Beads uses dual storage:
- **JSONL files** (.beads/*.jsonl): Source of truth, committed to git
- **SQLite database** (.beads/*.db): Ephemeral cache for fast queries, gitignored

Auto-export syncs changes to JSONL after CRUD operations. Auto-import syncs JSONL to SQLite when files are newer.

## Quick Reference

### Initialization
```bash
bd init              # Initialize in project (done once)
bd quickstart        # See tutorial
```

### Core Workflow
```bash
bd create "task description"         # Create new issue
bd create "task" --deps blocks:proj-1  # Create with blocker
bd list                              # Show all issues
bd ready                             # Show unblocked work (no blockers)
bd blocked                           # Show blocked issues
bd show proj-123                     # Show issue details
bd update proj-123 --title "new"    # Update issue
bd close proj-123                    # Close issue
```

### Dependency Management
```bash
bd dep add proj-123 proj-456 --type blocks         # 123 blocks 456
bd dep remove proj-123 proj-456                    # Remove dependency
bd dep add proj-123 proj-456 --type related        # Mark related
bd dep add proj-123 proj-456 --type parent-child   # 123 parent of 456
bd dep add proj-123 proj-456 --type discovered-from # 456 discovered from 123
bd dep tree proj-123                                # Show dependency tree
bd dep cycles                                       # Detect cycles
```

### JSON Output (for programmatic use)
```bash
bd list --json                       # Machine-readable output
bd show proj-123 --json
```

## When to Use Beads

### Start of Complex Tasks
When a user describes work with 3+ steps or dependencies:

```bash
# Example: "Add authentication with OAuth and session management"
bd create "Design authentication architecture"
bd create "Implement OAuth integration" --deps blocks:proj-1
bd create "Add session management" --deps blocks:proj-2
bd create "Write auth tests" --deps blocks:proj-2,blocks:proj-3
```

### During Implementation
When you discover new subtasks or blockers:

```bash
bd create "Fix bug in user model" --deps discovered-from:proj-5
bd dep add proj-7 proj-8 --type blocks    # proj-7 blocks proj-8
```

### Finding Ready Work
When deciding what to work on next:

```bash
bd ready             # Shows unblocked issues
bd show proj-15      # Check details before starting
bd update proj-15 --status in-progress
```

### Session Boundaries
At end of session, commit JSONL files:

```bash
git add .beads/*.jsonl
git commit -m "Update task tracking"
```

At start of session, check status:

```bash
bd ready             # What can I work on?
bd list              # Overall status
bd stats             # Show statistics
```

## Dependency Types

### Blocks
Task A must complete before Task B can start.
```bash
bd dep add proj-1 proj-2 --type blocks    # proj-1 blocks proj-2
# Or create with dependency:
bd create "Task B" --deps blocks:proj-1
```

### Parent-Child
Task B is a subtask of Task A.
```bash
bd dep add proj-1 proj-2 --type parent-child   # proj-1 is parent of proj-2
```

### Related
Tasks are connected but not dependent.
```bash
bd dep add proj-1 proj-2 --type related   # proj-1 and proj-2 related
```

### Discovered-From
Task B was discovered while working on Task A.
```bash
bd dep add proj-1 proj-2 --type discovered-from  # proj-2 discovered from proj-1
# Or create with dependency:
bd create "Fix found bug" --deps discovered-from:proj-1
```

## Status Values

- `open`: Ready to work (default)
- `in-progress`: Currently working
- `closed`: Completed

Change status:
```bash
bd update proj-123 --status in-progress
bd close proj-123                        # Mark closed
```

## Best Practices

### 1. Initialize Early
Run `bd init` at project start, before major work begins.

### 2. Break Down Complex Tasks
Convert user requests into atomic issues with dependencies:

```bash
# User: "Build a dashboard with charts and filters"
bd create "Design dashboard layout"
bd create "Implement data fetching" --deps blocks:proj-1
bd create "Add chart components" --deps blocks:proj-2
bd create "Add filter UI" --deps blocks:proj-2
bd create "Connect filters to charts" --deps blocks:proj-3,blocks:proj-4
```

### 3. Use Ready Work
Always check `bd ready` before starting new work.

### 4. Track Discovery
When debugging reveals new work:

```bash
bd create "Fix database connection pool" --deps discovered-from:proj-47
```

### 5. Commit JSONL Files
Add `.beads/*.jsonl` to git, ignore `.beads/*.db`:

```gitignore
# .gitignore
.beads/*.db
.beads/*.db-*
```

```bash
git add .beads/*.jsonl
git commit -m "Track new features and dependencies"
```

### 6. Visualize Dependencies
```bash
bd dep tree proj-123     # Show dependency tree
bd dep cycles            # Detect circular dependencies
```

## Integration with Other Tools

### With TodoWrite
Use Beads for **persistent** task tracking across sessions. Use TodoWrite for **ephemeral** task tracking within a single session.

- **Beads**: Project-level tasks, multi-session work, dependency tracking
- **TodoWrite**: Current session checklist, implementation steps

Example:
```bash
# Beads: Track feature work
bd add "Implement user authentication"
bd list ready

# TodoWrite: Track implementation steps in this session
# 1. Read auth library docs
# 2. Write auth middleware
# 3. Add tests
```

### With Git Worktrees
Each worktree shares the same Beads database via git:

```bash
dash new feature-x
bd ready                   # See unblocked issues
# Work on proj-15
bd close proj-15
git add .beads/*.jsonl && git commit -m "Complete proj-15"
```

### With Subagent Development
Assign issues to parallel subagents:

```bash
bd ready                   # Shows proj-10, proj-11, proj-12
# Assign proj-10 to subagent 1
bd update proj-10 --assignee subagent-1 --status in-progress
# Assign proj-11 to subagent 2
bd update proj-11 --assignee subagent-2 --status in-progress
```

## Common Patterns

### Feature Development
```bash
bd create "Feature: User profiles" --type epic
bd create "Design profile schema" --deps parent-child:proj-100
bd create "Implement profile CRUD" --deps parent-child:proj-100,blocks:proj-101
bd create "Add profile UI" --deps parent-child:proj-100,blocks:proj-102
bd create "Write profile tests" --deps parent-child:proj-100,blocks:proj-102,blocks:proj-103
```

### Bug Investigation
```bash
bd create "Bug: Login fails intermittently" --type bug
bd update proj-200 --status in-progress
# Investigation reveals root cause
bd create "Fix session timeout handling" --deps discovered-from:proj-200
bd dep add proj-200 proj-201 --type related
```

### Refactoring
```bash
bd create "Refactor: Extract auth module" --type epic
bd create "Move auth code to module" --deps parent-child:proj-300
bd create "Update imports" --deps parent-child:proj-300,blocks:proj-301
bd create "Update tests" --deps parent-child:proj-300,blocks:proj-301
```

## Troubleshooting

### Database out of sync
```bash
bd export        # Force export to JSONL
bd import        # Force import from JSONL
```

### View raw data
```bash
cat .beads/issues.jsonl
sqlite3 .beads/dash.db "SELECT * FROM issues"
```

### Merge conflicts
JSONL files may conflict in git. Resolve manually, then:
```bash
bd import        # Reload from resolved JSONL
```

## Advanced Usage

### Custom Queries
```bash
sqlite3 .beads/dash.db "SELECT id, title, status FROM issues WHERE status='in-progress'"
```

### Bulk Operations
```bash
# Mark all in-progress as open
for id in $(bd list --json | jq -r '.[] | select(.status=="in-progress") | .id'); do
  bd status $id open
done
```

### Export for Reports
```bash
bd list --json | jq '.[] | {id, title, status, created}'
```

## Key Benefits for AI Agents

1. **Survive context limits**: Tasks persist beyond conversation boundaries
2. **Prevent duplicate work**: See what's already tracked
3. **Handle interruptions**: Resume work after breaks
4. **Manage complexity**: Break down large tasks systematically
5. **Track discovery**: Record new work as you find it
6. **Enable parallelism**: Identify independent work for subagents

## Remember

Beads is designed for AI agents, not human project managers. Use it to:
- **Remember** tasks across sessions
- **Track** dependencies and blockers
- **Find** ready work automatically
- **Coordinate** with your human partner via git

Think of Beads as your external memory system that persists in the codebase itself.
