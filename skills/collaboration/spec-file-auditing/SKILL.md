---
name: Specification File Auditing
description: Audit one spec file against codebase implementation and update Beads issue tracker with gaps
when_to_use: when auditing a single specification file to identify implementation gaps, when Ralph needs more Beads tasks, or when doing just-in-time verification before working on a spec area
version: 1.0.0
---

# Specification File Auditing

## Overview

Audits a single specification file (from `specs/` directory) against the codebase implementation and creates/updates Beads issues for any gaps found. Designed for incremental specification tracking rather than monolithic batch audits.

## Core Concept

Traditional specification audits try to check everything at once (expensive, slow, high false-negative rate). This skill:

- Audits ONE specification file at a time
- Uses Explore subagent for thorough codebase investigation
- Leverages project-specific investigation patterns from CLAUDE.md
- Creates Beads issues only for actual gaps (not fully-implemented requirements)
- Supports both comprehensive audits (all specs) and just-in-time audits (before feature work)

## Quick Reference

```bash
# Audit single specification file
# (This skill is invoked by agent, not run directly via CLI)

# Inputs needed:
# - spec_file: Path to spec file (e.g., "specs/stdlib/collections.md")
# - project_root: Working directory (for Beads commands)

# Outputs:
# - Beads issues created for gaps (tagged with priority, domain, spec area)
# - Investigation notes in issue bodies
# - Summary of audit results
```

## When to Use This Skill

### Just-in-Time Auditing
Before working on a feature area:
```markdown
User: "I'm about to implement collections methods"
Assistant: "Let me audit specs/stdlib/collections.md first to see what's missing"
```

### Ralph Low-Beads Workflow
When task tracker is running low:
```bash
bd stats  # Shows: 5 open issues
# Ralph uses this skill to audit next unaudited spec
# Populates Beads with 10-20 more tasks
```

### Comprehensive Refresh
Periodic full audits (use full-spec-audit skill, which calls this skill 42 times):
```markdown
# Quarterly or after major refactoring
# Audit all 42 specs to refresh implementation status
```

## Investigation Protocol

### Step 1: Read Specification File

```markdown
Read the spec file completely:
- Extract all requirement IDs (ST-XXX-XXX-NNN pattern)
- Note requirement descriptions
- Identify dependencies between requirements
- Categorize by priority hints (if present in spec)
```

### Step 2: Check Existing Beads State

```bash
# Check if issues already exist for this spec
bd list --json | jq '.[] | select(.tags | contains("spec-file-name"))'

# Decision tree:
# - If issues exist and closed → Skip (already implemented)
# - If issues exist and open → Update with fresh investigation
# - If no issues exist → Investigate and create as needed
```

### Step 3: Investigate Each Requirement

For each requirement in the spec file, use **multi-method investigation**:

**Method 1: Explore Subagent (Primary)**
```markdown
Dispatch Explore subagent with thoroughness level based on confidence:

For obvious/common patterns:
  Thoroughness: "quick"
  Query: "Find implementation of <requirement description>"

For unclear/missing patterns:
  Thoroughness: "very thorough"
  Query: "Comprehensively search codebase for <requirement ID> or <description>.
          Check CLAUDE.md investigation patterns for project-specific search techniques.
          Include: grep with multiple term variations, test files, coverage reports,
          interactive verification hints."

Trust Explore's findings:
- If Explore finds implementation → Mark IMPLEMENTED
- If Explore exhaustively searches and finds nothing → Mark GAP
- If Explore is uncertain → Escalate to manual verification
```

**Method 2: Coverage Report Cross-Reference**
```bash
# Check if requirement appears in coverage reports
make requirement-coverage 2>/dev/null || echo "Coverage not available"
grep "ST-XXX-XXX-NNN" requirement-coverage.json
```

**Method 3: Direct Test Evidence**
```bash
# Search test files for requirement ID
grep -r "ST-XXX-XXX-NNN" tests/
# Passing test = implemented (even if code location unclear)
```

**Method 4: Project-Specific Patterns**
```markdown
CLAUDE.md in project root contains investigation patterns.
Common patterns to check:
- Bar-quoted symbol searches (|methodName:| not methodName)
- Multiple file locations (Smalltalk .st files, Lisp .lisp files)
- FFI/external library code
- Case-sensitive vs case-insensitive searches

Adapt investigation based on project patterns.
```

### Step 4: Categorize Requirements

After investigation, categorize each requirement:

**IMPLEMENTED** (skip, no Beads issue needed):
- Tests pass for this requirement
- Explore found working implementation
- Coverage report confirms implementation
- Interactive verification succeeds

**PARTIAL** (create/update Beads issue):
- Some aspects implemented
- Core functionality missing
- Implementation incomplete

**GAP** (create/update Beads issue):
- No evidence after comprehensive search (5+ methods)
- Tests don't exist or fail
- Coverage report shows missing
- Explore confirms absence

### Step 5: Create/Update Beads Issues

For PARTIAL and GAP requirements:

```bash
# Create new issue
bd create "ST-XXX-XXX-NNN: <short description>" \
  --tags gap,<domain>,<spec-area>,<priority>

# Or update existing issue (if found in Step 2)
bd update <issue-id> --title "Updated: ST-XXX-XXX-NNN: <description>"
```

**Issue Format:**
```markdown
Title: ST-STDLIB-COLL-045: String indexOf: method

Tags: gap, stdlib, collections, important

Body:
Requirement: ST-STDLIB-COLL-045
Spec: specs/stdlib/collections.md
Description: Implement indexOf: method for String class

Investigation (YYYY-MM-DD):
- Explore (very thorough): No implementation found
- Grep searches: Tried |indexOf:|, indexOf, index-of (all negative)
- Test files: No tests reference ST-STDLIB-COLL-045
- Coverage: Not in requirement-coverage.json
- CLAUDE.md patterns: Checked bar-quoted symbols, .st and .lisp files

Conclusion: GAP (not implemented)
Priority: IMPORTANT (needed for String API completeness)

Next Steps:
- Implement |indexOf:| method in String class
- Add tests referencing ST-STDLIB-COLL-045
- Verify with coverage report
```

## Priority Assignment

Determine priority based on requirement context:

**CRITICAL** (blocks core functionality):
- Core language features (parser, evaluator)
- Essential runtime systems (scheduler, exceptions)
- Blocking dependencies for other features
- Tag: `critical`

**IMPORTANT** (needed for completeness):
- Standard library methods
- Common use cases
- Non-blocking but expected functionality
- Tag: `important`

**MINOR** (polish/enhancement):
- Edge cases
- Optimizations
- Nice-to-have features
- Tag: `minor`

Heuristics:
- Runtime specs → often CRITICAL
- Stdlib specs → often IMPORTANT
- Tooling specs → varies (testing=CRITICAL, nice-to-have tools=MINOR)
- Check spec file for severity hints

## Tag Strategy

Use consistent tagging for Beads issues:

```bash
# Status tags
--tags gap          # Not implemented
--tags partial      # Partially implemented

# Priority tags (exactly one)
--tags critical     # Blocking core functionality
--tags important    # Needed for completeness
--tags minor        # Polish/enhancement

# Domain tags (exactly one)
--tags stdlib       # Standard library
--tags runtime      # Runtime system
--tags language     # Parser/AST/semantics
--tags tooling      # Build/test/dev tools

# Spec area tags (specific to spec file)
--tags collections  # specs/stdlib/collections.md
--tags scheduler    # specs/runtime/scheduler.md
--tags http         # specs/stdlib/http.md
# etc.
```

## False Negative Prevention

**CRITICAL**: Previous audits have suffered from 15-20% false negative rates (marking things as missing when they're actually implemented).

**Required safeguards:**

1. **Exhaust search methods** before concluding something is missing:
   - Explore subagent (very thorough mode)
   - Direct grep with 5+ term variations
   - Test file examination
   - Coverage report cross-reference
   - Project-specific patterns from CLAUDE.md

2. **Trust test evidence**:
   - If tests pass → it's implemented (even if you can't find code)
   - Don't mark as GAP if tests exist and pass

3. **Use Explore thoroughly**:
   - Start with "quick" mode for obvious checks
   - Escalate to "very thorough" before marking as GAP
   - Explore reads CLAUDE.md and knows project patterns

4. **Document investigation**:
   - Record ALL methods tried in Beads issue body
   - Note search patterns used
   - Explain why you concluded GAP vs IMPLEMENTED

5. **When uncertain**:
   - Create issue tagged as `needs-verification`
   - Don't guess - mark as needing human review

## Anti-Patterns to Avoid

**Don't:**
- Audit multiple spec files in one invocation (this skill does ONE file)
- Skip Explore subagent (it's the most reliable method)
- Search without checking CLAUDE.md patterns first
- Mark as GAP after only 1-2 search methods
- Create Beads issues for IMPLEMENTED requirements
- Guess at priority (use heuristics or mark as needing-triage)
- Ignore existing Beads issues (check first, update if found)

**Red Flags:**
- "Probably not implemented" → Investigate more
- "Can't find it" → Did you use Explore thoroughly?
- "Seems missing" → Evidence-based conclusions only
- Skipping test file checks → Always check tests
- Not reading CLAUDE.md → Project patterns are critical

## Success Criteria

Audit succeeds when:

1. ✅ All requirements in spec file investigated (not just sampled)
2. ✅ Beads issues created only for actual gaps (no false negatives)
3. ✅ Investigation methods documented in issue bodies
4. ✅ Priority and tags assigned consistently
5. ✅ Existing Beads issues updated (not duplicated)
6. ✅ Summary provided (X implemented, Y gaps, Z partial)
7. ✅ Confidence level stated (based on investigation thoroughness)

## Output Format

At completion, provide summary:

```markdown
## Specification Audit: specs/stdlib/collections.md

**Audit Date**: YYYY-MM-DD
**Total Requirements**: 45
**Investigation Methods**: Explore (very thorough), grep, tests, coverage

**Results:**
- IMPLEMENTED: 38 requirements (84%)
- PARTIAL: 4 requirements (9%)
- GAP: 3 requirements (7%)

**Beads Issues Created:**
- picotalk-101: ST-STDLIB-COLL-045 (gap, important, collections)
- picotalk-102: ST-STDLIB-COLL-052 (gap, minor, collections)
- picotalk-103: ST-STDLIB-COLL-061 (gap, critical, collections)

**Beads Issues Updated:**
- picotalk-85: ST-STDLIB-COLL-023 (updated investigation notes)

**Confidence**: HIGH (used Explore very thorough + 4 other methods)

**Next Steps**:
- Ralph can use `bd ready` to see unblocked collection tasks
- Prioritize picotalk-103 (critical) for next implementation
```

## Integration with Other Skills

**With full-spec-audit**:
- Full-spec-audit calls this skill 42 times (one per spec file)
- Provides orchestration and summary
- This skill focuses on single-file investigation

**With using-beads**:
- Creates and updates Beads issues
- Uses consistent tagging strategy
- Respects Beads workflow (ready work, dependencies, etc.)

**With systematic-debugging**:
- If investigation reveals broken implementation (not missing)
- Create bug issue instead of gap issue
- Tag as `bug` instead of `gap`

## Remember

This skill is for **incremental specification tracking**, not batch auditing. It:

- Investigates ONE spec file thoroughly
- Leverages Explore subagent + project patterns (CLAUDE.md)
- Creates Beads issues for gaps only (not implemented requirements)
- Prevents false negatives through multi-method investigation
- Supports both just-in-time and comprehensive audit workflows

Focus on accuracy over speed. Better to take time and get it right than rush and create false negatives.
