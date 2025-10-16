---
name: Full Specification Audit
description: Orchestrate comprehensive audit of all specification files using spec-file-auditing skill
when_to_use: when doing initial Beads population, quarterly comprehensive reviews, after major refactoring, or when significant drift suspected (manual invocation only, not automatic)
version: 1.0.0
---

# Full Specification Audit

## Overview

Orchestrates a comprehensive audit of all specification files in `specs/` directory by calling the spec-file-auditing skill for each file. Designed for periodic refresh of implementation status, not continuous operation.

## Core Concept

Rather than running 43 parallel subagents simultaneously (expensive, resource-intensive), this skill:

- Discovers all spec files in `specs/` directory
- Calls spec-file-auditing skill sequentially or in small batches
- Aggregates results across all audits
- Provides comprehensive summary of implementation status
- Updates Beads issue tracker with all gaps found

## Quick Reference

```bash
# This skill is invoked manually, not automatically
# Manual trigger only:
# - Initial Beads population
# - Quarterly review
# - After major refactoring
# - When drift suspected

# NOT for continuous operation
# NOT every N Ralph iterations (that was the old system)
```

## When to Use This Skill

### Initial Beads Population
```markdown
User: "Initialize spec tracking for PicoTalk"
Assistant: *Uses full-spec-audit to audit all 42 specs*
Result: ~50-200 Beads issues created for gaps
```

### Quarterly Comprehensive Review
```markdown
# After 3 months of development
# Check if new gaps emerged or old gaps resolved
```

### After Major Refactoring
```markdown
# After significant codebase changes
# Verify implementation status hasn't regressed
```

### Suspected Drift
```markdown
User: "I think there's drift between Beads and actual implementation"
Assistant: *Runs full audit to reconcile*
```

### When NOT to Use
- ❌ Automatic every N iterations (use incremental spec-file-auditing instead)
- ❌ Before every feature (use just-in-time spec-file-auditing for one spec)
- ❌ Continuous operation (expensive and wasteful)

## Execution Strategy

### Sequential vs Parallel

**Sequential** (recommended, safer):
```markdown
Audit one spec file at a time:
- Lower cost (no parallel agent spawning)
- Predictable resource usage
- Easier to debug if issues occur
- Takes longer (~2-4 hours for 42 specs)
```

**Batched Parallel** (optional, faster):
```markdown
Audit 5-10 specs at a time in parallel:
- Faster completion (~30-60 minutes)
- Higher cost (multiple concurrent agents)
- Risk of rate limiting
- Harder to track progress
```

**NOT Recommended** (old approach):
```markdown
43 agents simultaneously:
- Very expensive
- Prone to rate limits
- Hard to debug failures
- High false-negative risk
```

## Implementation Workflow

### Step 1: Discover Specification Files

```bash
# Find all spec files
find specs -name "*.md" -type f | sort

# Expected structure:
specs/
  overview.md
  README.md
  language/
    grammar.md
    ast.md
    semantics.md
  object-model/
    core.md
    namespaces.md
    metadata.md
  runtime/
    scheduler.md
    continuations.md
    exceptions.md
    ...
  stdlib/
    kernel.md
    collections.md
    streams.md
    ...
  tooling/
    testing.md
    quality.md
    ...
  adr/
    ADR-0001-*.md
    ADR-0002-*.md
    ...
```

### Step 2: Filter Spec Files

```markdown
Not all .md files in specs/ are specification files:

INCLUDE:
- Files with requirement IDs (ST-XXX-XXX-NNN pattern)
- Core specs: grammar.md, ast.md, semantics.md, etc.
- Domain specs: stdlib/*.md, runtime/*.md, tooling/*.md
- ADR files: adr/ADR-*.md

EXCLUDE (or handle specially):
- overview.md (no requirements, just context)
- README.md (index file, not spec)
- Vision documents (if any in specs/)

Recommended: Audit all .md files except overview.md and README.md
```

### Step 3: Check Beads State (Optional)

```bash
# Before starting, optionally archive old audit state
bd list --json | jq '.[] | select(.tags | contains("gap") or contains("partial"))'

# Decision:
# - Keep existing issues (update during audit)
# - Close all and recreate (fresh start)
# - Let user decide

# Recommended: Keep existing, update during audit
```

### Step 4: Audit Each Spec File

For each spec file, call spec-file-auditing skill:

**Sequential approach:**
```markdown
For each spec_file in spec_files:
  Use Read tool to load spec-file-auditing skill
  Invoke skill with parameters:
    - spec_file: current file path
    - project_root: working directory
  Wait for completion
  Record results (issues created, implemented count, etc.)
  Continue to next file
```

**Batched parallel approach:**
```markdown
Batch spec_files into groups of 5:
  For each batch:
    Launch 5 parallel agents (Task tool, general-purpose)
    Each agent runs spec-file-auditing skill
    Wait for all 5 to complete
    Record aggregated results
    Continue to next batch
```

### Step 5: Aggregate Results

After all audits complete:

```bash
# Collect statistics
bd stats
bd list --json | jq '[.[] | select(.tags | contains("gap"))] | length'

# Generate summary
Total specs audited: 42
Total requirements found: ~500-1000
Implemented: XXX (YY%)
Partial: XXX (YY%)
Gaps: XXX (YY%)
Beads issues created: XXX
Beads issues updated: XXX
```

### Step 6: Commit Beads State

```bash
# Commit all JSONL changes
git add .beads/*.jsonl
git commit -m "Full specification audit: populate Beads with implementation gaps"
git push
```

## Output Format

```markdown
## Full Specification Audit Results

**Audit Date**: YYYY-MM-DD
**Execution Mode**: Sequential (or Batched Parallel)
**Total Specs Audited**: 42
**Duration**: X hours

### Aggregate Statistics

**Total Requirements**: 847
**Implementation Status**:
- IMPLEMENTED: 712 (84%)
- PARTIAL: 68 (8%)
- GAP: 67 (8%)

**Beads Issues**:
- Created: 135 new issues
- Updated: 23 existing issues
- Total open: 158

### By Domain

**Language** (3 specs):
- Implemented: 95%
- Gaps: 5%
- Critical gaps: 0

**Runtime** (7 specs):
- Implemented: 88%
- Gaps: 12%
- Critical gaps: 3

**Stdlib** (18 specs):
- Implemented: 82%
- Gaps: 18%
- Critical gaps: 8

**Tooling** (3 specs):
- Implemented: 90%
- Gaps: 10%
- Critical gaps: 1

**ADRs** (6 specs):
- Implemented: 78%
- Gaps: 22%
- Critical gaps: 2

### Priority Breakdown

**Critical Gaps**: 14 issues (needs immediate attention)
**Important Gaps**: 89 issues (needed for completeness)
**Minor Gaps**: 55 issues (polish/enhancement)

### Top Priority Tasks

Use `bd ready` to see unblocked work. Suggested priorities:

1. picotalk-XXX: [Critical gap description]
2. picotalk-YYY: [Critical gap description]
3. picotalk-ZZZ: [Critical gap description]

### Confidence Assessment

**Investigation Thoroughness**: HIGH
- Used Explore (very thorough) for all uncertain cases
- Cross-referenced with test files
- Checked coverage reports
- Applied project-specific patterns (CLAUDE.md)

**False Negative Risk**: LOW
- Multi-method investigation (5+ techniques)
- Test evidence prioritized
- Conservative gap classification

### Next Steps

1. Review critical gaps for immediate work
2. Use `bd ready` to see unblocked tasks
3. Implement high-priority items first
4. Re-audit specific domains as implementation progresses
5. Schedule next full audit in 3 months (or as needed)

### Beads Commands

```bash
# See all open gaps
bd list

# See unblocked work
bd ready

# See critical gaps
bd list --json | jq '.[] | select(.tags | contains("critical"))'

# See gaps by domain
bd list --json | jq '.[] | select(.tags | contains("stdlib"))'
```
```

## Best Practices

### Incremental Over Comprehensive

**Prefer incremental auditing**:
- Use spec-file-auditing for just-in-time checks
- Ralph audits specs when low on Beads tasks
- Full audit only when truly needed (quarterly, major milestones)

**Avoid automatic comprehensive audits**:
- Don't run every N iterations
- Don't run before every feature
- Resource-intensive for diminishing returns

### Communication with User

```markdown
Before starting:
"This will audit all 42 specification files against the codebase.
Expected duration: 2-4 hours (sequential) or 30-60 minutes (parallel).
Proceed?"

During execution:
"Auditing specs/stdlib/collections.md... (15/42)"

After completion:
[Provide aggregate summary as shown above]
```

### Error Handling

```markdown
If spec-file-auditing fails for a file:
- Log the failure
- Continue with remaining specs
- Report failed specs in summary
- Allow retry of failed specs only

Don't fail entire audit due to one spec failure.
```

### Resource Management

```markdown
Sequential mode:
- No special handling needed
- Natural rate limiting

Batched parallel mode:
- Monitor for rate limiting
- Reduce batch size if limits hit
- Add delays between batches if needed
```

## Integration with Ralph

Update Ralph prompt to use this skill appropriately:

```markdown
Ralph should:
- NOT invoke full-spec-audit automatically
- Only invoke when explicitly requested by user
- Prefer incremental spec-file-auditing for ongoing work

Exception:
- First time Beads is initialized (bootstrap)
- User explicitly requests comprehensive audit
```

## Anti-Patterns to Avoid

**Don't:**
- Run automatically every N iterations (wasteful)
- Run in tight loops (creates audit thrashing)
- Spawn all 43 agents simultaneously (expensive, unreliable)
- Skip aggregation step (summary is valuable)
- Forget to commit Beads JSONL files
- Run without user awareness (communicate duration)

**Red Flags:**
- "Let's audit everything just to be safe" → Only when needed
- "Run audit before every feature" → Use spec-file-auditing instead
- "Audit in continuous mode" → Defeats incremental approach
- Automatic scheduling → Manual invocation only

## Success Criteria

Full audit succeeds when:

1. ✅ All spec files discovered and processed (or failures reported)
2. ✅ Beads issues created for all gaps found
3. ✅ Aggregate statistics generated and reported
4. ✅ Beads JSONL files committed to git
5. ✅ User provided with clear summary and next steps
6. ✅ No duplicate issues created (updates existing when appropriate)
7. ✅ Confidence assessment provided (investigation thoroughness)

## Remember

Full specification audit is for **periodic comprehensive reviews**, not continuous operation:

- Use quarterly or after major milestones
- Prefer incremental spec-file-auditing for ongoing work
- Run sequentially for safety, parallel batches for speed
- Aggregate results provide valuable big-picture view
- Always commit Beads JSONL state after completion

Think of this as a "checkpoint" or "snapshot" of implementation status, not a real-time tracking system. Real-time tracking is Beads + incremental auditing.
