---
name: Feedback Loop Design
description: Design automated validation that catches problems before users do
when_to_use: when building features without tests, discovering bugs tests didn't catch, or seeking maintenance work during autonomous loops
version: 1.0.0
languages: all
---

# Feedback Loop Design

## Overview

**Backpressure from failures is what makes autonomous systems self-correcting.**

Without feedback loops, you can break things silently. With them, breaks become loud signals that force fixes.

**Core principle:** Convert "might be broken" into "test fails" through automated validation.

## The Feedback Hierarchy

Not all feedback is equal. **Move validation up the hierarchy:**

### Tier 1 (Best): Immediate build/runtime failures
- Compiler errors, type errors, syntax errors, import failures
- ✅ Can't proceed without fixing
- ✅ Points exactly to the problem
- ✅ Zero human intervention required

### Tier 2 (Good): Automated test failures
- Unit tests, integration tests, linters, static analysis
- ✅ Automated validation
- ✅ Specific failure messages
- ⚠️ Can be skipped if not in CI

### Tier 3 (Okay): Manual verification
- Running the app and checking behavior
- ⚠️ Requires human effort every time
- ⚠️ Easy to forget or skip
- ⚠️ Doesn't scale with complexity

### Tier 4 (Bad): User reports
- Bug reports, crash reports, complaints
- ❌ Slow feedback (days or weeks later)
- ❌ Damages trust and reputation
- ❌ Expensive to fix after shipping

**Goal: Move everything possible from Tier 4/3 → Tier 2/1**

## Common Gaps (What Lacks Feedback)

What can break without tests failing?

1. **UI/UX**: Visual rendering, interactions, layout, responsive design, accessibility
2. **Integration**: External APIs, services, file I/O, network calls, databases
3. **Performance**: Speed regressions, memory leaks, resource exhaustion
4. **Workflows**: Multi-step user processes, edge cases, error recovery paths
5. **Configuration**: Different environments, flags, optional features, defaults
6. **Documentation**: Examples that don't run, outdated API docs, wrong commands
7. **Cross-platform**: Works on dev machine, breaks in production/different OS

## Gap Identification Process

```
BEFORE claiming "done" or when seeking maintenance work:

1. LIST: What can break in this system?
2. ASK: For each potential break, what creates backpressure?
3. FIND: What has NO automated validation?
4. PRIORITIZE: What's most likely to break? Most costly?
5. DESIGN: What mechanism converts silent failure → loud signal?
6. IMPLEMENT: Build the feedback loop
7. INTEGRATE: Make it unavoidable (CI, pre-commit, blocking)
```

**Use TodoWrite to track each step.**

## Feedback Mechanisms by Domain

### CLI Applications

**Common gaps:**
- Interactive flows break silently
- Output format changes unexpectedly
- Flags/options don't work as documented
- Error messages become confusing
- Edge cases in user input handling

**Feedback mechanisms:**
- **Expect scripts**: Automate interaction sequences, verify responses
- **Tmux/screen automation**: Send keys, capture output, assert expected states
- **Golden file testing**: Compare actual output to expected output files
- **Integration test suites**: Run full workflows end-to-end, verify results
- **Shell script harness**: Boot app, execute scenarios, validate behavior

**Example pattern:**
```bash
#!/usr/bin/env bash
# Integration test for CLI app

# Start app in tmux session
tmux new-session -d -s test_app "./myapp --interactive"

# Send commands and verify responses
tmux send-keys -t test_app "create project foo" Enter
sleep 1

# Capture output and validate
OUTPUT=$(tmux capture-pane -t test_app -p)
if echo "$OUTPUT" | grep -q "Project 'foo' created"; then
    echo "✓ Create command works"
else
    echo "✗ Create command failed"
    exit 1
fi

# Test error handling
tmux send-keys -t test_app "invalid command" Enter
sleep 1
OUTPUT=$(tmux capture-pane -t test_app -p)
if echo "$OUTPUT" | grep -q "Unknown command"; then
    echo "✓ Error handling works"
else
    echo "✗ Error handling broken"
    exit 1
fi

tmux kill-session -t test_app
echo "All integration tests passed"
```

### Web Applications

**Common gaps:**
- UI elements don't render correctly
- User interactions break (clicks, forms, navigation)
- Visual regressions (CSS changes break layout)
- Accessibility issues (keyboard nav, screen readers)
- Cross-browser compatibility problems

**Feedback mechanisms:**
- **E2E tests**: Playwright, Cypress, Selenium (automate user workflows)
- **Visual regression**: Screenshot comparison tools (Percy, BackstopJS, Chromatic)
- **Accessibility audits**: Automated a11y testing (axe-core, pa11y)
- **Performance budgets**: Lighthouse CI (fail builds if too slow/large)
- **Contract testing**: Verify frontend/backend API contracts match

**Example pattern:**
```javascript
// E2E test with Playwright
import { test, expect } from '@playwright/test'

test('user can complete signup flow', async ({ page }) => {
  // Navigate to signup
  await page.goto('/signup')

  // Fill form
  await page.fill('input[name="email"]', 'test@example.com')
  await page.fill('input[name="password"]', 'SecurePass123')

  // Submit
  await page.click('button[type="submit"]')

  // Verify success
  await expect(page.locator('.success-message')).toBeVisible()
  await expect(page).toHaveURL('/dashboard')
})

test('form validation shows errors', async ({ page }) => {
  await page.goto('/signup')

  // Submit empty form
  await page.click('button[type="submit"]')

  // Verify error messages
  await expect(page.locator('.error-email')).toContainText('Email required')
  await expect(page.locator('.error-password')).toContainText('Password required')
})
```

### APIs and Services

**Common gaps:**
- Endpoints return wrong data structure
- Contract changes break clients
- Performance degrades over time
- Error responses malformed or unhelpful
- Authentication/authorization edge cases

**Feedback mechanisms:**
- **Contract testing**: Pact, OpenAPI validation (ensure schema compliance)
- **Integration tests**: Real HTTP requests against running service
- **Load testing**: K6, wrk, artillery (catch performance regressions)
- **Example requests**: Run documentation examples as automated tests
- **Schema validation**: Automatic type/structure checking on responses

**Example pattern:**
```javascript
// API integration test
describe('User API', () => {
  test('GET /users/:id returns valid user', async () => {
    const response = await fetch('http://localhost:3000/users/123')
    const user = await response.json()

    // Validate structure
    expect(user).toHaveProperty('id')
    expect(user).toHaveProperty('email')
    expect(typeof user.email).toBe('string')

    // Validate contract
    expect(response.status).toBe(200)
    expect(response.headers.get('content-type')).toContain('application/json')
  })

  test('GET /users/:id returns 404 for missing user', async () => {
    const response = await fetch('http://localhost:3000/users/nonexistent')
    expect(response.status).toBe(404)

    const error = await response.json()
    expect(error).toHaveProperty('message')
  })
})
```

### Libraries and SDKs

**Common gaps:**
- Examples in documentation don't actually work
- API changes break downstream users
- Performance regressions go unnoticed
- Edge cases and error paths untested
- Breaking changes shipped accidentally

**Feedback mechanisms:**
- **Doc tests**: Run code examples from documentation (Python doctest, Rust doc tests)
- **Property-based testing**: Generate random inputs (QuickCheck, Hypothesis, fast-check)
- **Benchmark tests**: Fail builds if performance degrades beyond threshold
- **API compatibility checks**: Semver validation, public API surface monitoring
- **Example projects**: Maintain full example apps as integration tests

**Example pattern:**
```python
# Python doctest - documentation that's also a test
def parse_config(path: str) -> dict:
    """
    Parse configuration file.

    >>> parse_config('test.conf')
    {'host': 'localhost', 'port': 8080}

    >>> parse_config('nonexistent.conf')
    Traceback (most recent call last):
        ...
    FileNotFoundError: nonexistent.conf
    """
    with open(path) as f:
        return yaml.safe_load(f)

# Run with: python -m doctest mymodule.py
```

### System Reliability

**Common gaps:**
- Application crashes in specific environments
- Resource leaks manifest over time
- Failure recovery mechanisms broken
- Configuration errors fail silently
- Degraded performance under load

**Feedback mechanisms:**
- **Health checks**: Liveness/readiness probes (Kubernetes-style)
- **Metrics + thresholds**: Monitor CPU/memory/latency, alert on anomalies
- **Chaos engineering**: Deliberately inject failures, verify recovery
- **Smoke tests**: Boot in production-like environment, verify basic functions
- **Config validation**: Schema checks on startup, fail fast on invalid config

**Example pattern:**
```go
// Health check endpoint with actual validation
func healthHandler(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()

    // Check database connectivity
    if err := db.PingContext(ctx); err != nil {
        http.Error(w, "database unhealthy", http.StatusServiceUnavailable)
        return
    }

    // Check external dependencies
    if err := checkExternalAPI(ctx); err != nil {
        http.Error(w, "external API unhealthy", http.StatusServiceUnavailable)
        return
    }

    // All checks passed
    w.WriteHeader(http.StatusOK)
    json.NewEncoder(w).Encode(map[string]string{"status": "healthy"})
}
```

## Implementation Checklist

**Use TodoWrite to track each step:**

1. **Audit current feedback**
   - List all potential failure modes for the system
   - Identify what currently has automated validation
   - Document gaps where breaks would be silent

2. **Prioritize gaps**
   - What's most likely to break?
   - What's most costly if it breaks?
   - What's easiest to add validation for first?

3. **Design mechanism**
   - Choose appropriate tool (tmux, Playwright, integration tests, etc.)
   - Define what "pass" means concretely
   - Make failure messages specific and actionable

4. **Implement validation**
   - Write the test/check/script
   - **Verify it catches the problem** (make it fail first)
   - Fix the underlying issue
   - Verify the test passes when system is correct

5. **Integrate into workflow**
   - Add to CI pipeline
   - Add to pre-commit hooks
   - Make blocking (can't merge if validation fails)

6. **Maintain feedback loops**
   - Fix flaky tests immediately (flaky = no feedback)
   - Update validation when system changes
   - Remove obsolete checks that no longer apply

## Key Patterns

### Make it fail first
```
✅ Write validation → Break the thing → Verify test fails → Fix → Verify test passes
❌ Write validation → See it pass → Assume it works (might be false positive)
```

### Specific over vague
```
✅ "Login button not found on /signin page (expected: button.login-btn)"
❌ "Something went wrong" or "Test failed"
```

### Fast over slow
```
✅ Unit test fails in 0.1 seconds
❌ Manual QA finds bug 2 weeks later
```

### Automated over manual
```
✅ CI fails merge if performance budget exceeded
❌ Checklist: "Remember to check performance before merging"
```

### Unavoidable over optional
```
✅ Required CI check blocks merge
❌ Optional test that can be skipped
```

## Anti-Patterns (Stop Immediately)

| Excuse | Reality | Fix |
|--------|---------|-----|
| "Tests pass so it works" | Tests only verify what they test | Add integration/E2E tests |
| "I manually checked it" | Manual doesn't scale, you'll forget | Automate the check |
| "Works on my machine" | Environment-specific issues have no feedback | Test in CI environment |
| "It's just a UI change" | UI bugs are real bugs that frustrate users | Add E2E tests |
| "I'll add tests later" | Later = never | Add tests now or create task |
| "Test is flaky, skip it" | Skipped test = zero feedback | Fix flakiness or delete test |
| "Too hard to test" | Hard to test = easy to break silently | Invest in test harness |
| "We have unit tests" | Unit tests miss integration issues | Add integration layer |

## For Autonomous Agents: Maintenance Mode

**When you've run out of obvious work** (no open tasks, all specs appear implemented):

### Don't sit idle - Switch to maintenance mode

1. **Run gap audit** - Use this skill to find missing validation
2. **Test realistic scenarios** - Boot the system, try actual user workflows
3. **Identify issues** - Document bugs, UX problems, performance issues
4. **Build feedback loops** - Add automated tests for gaps discovered
5. **Fix bugs found** - Issues discovered = valuable work to do
6. **Remove tech debt** - Refactor, improve code quality
7. **Repeat** - Sustainable autonomous cycle

### Example maintenance cycle

**For a CLI application:**
```
Current state: No open tickets, all specs look complete

1. Audit gaps:
   - Does it handle invalid user input gracefully?
   - What about edge cases in file paths (spaces, unicode, etc.)?
   - Does error output help users fix problems?
   - Does it work across different terminal sizes?

2. Design validation:
   - Create integration test script
   - Use tmux to automate interaction sequences
   - Test error cases and edge conditions
   - Verify helpful error messages appear

3. Implement and discover:
   - Build test harness (e.g., test_integration.sh)
   - Run realistic workflows through automation
   - Find: Input validation missing, unclear errors, layout breaks

4. Fix discovered issues:
   - Each bug = new task to complete
   - Add regression test for each fix
   - Verify with integration test

5. Result: Continuous improvement cycle
```

**For a web application:**
```
Current state: No open tickets, all specs look complete

1. Audit gaps:
   - Can users actually complete key workflows?
   - Does UI work on mobile/tablet/desktop?
   - Are loading states and errors handled well?
   - What about slow network conditions?

2. Design validation:
   - E2E tests with Playwright/Cypress
   - Visual regression tests for UI changes
   - Accessibility audits (keyboard nav, screen readers)
   - Performance budgets and monitoring

3. Implement and discover:
   - Build E2E test suite for critical paths
   - Run visual regression baselines
   - Find: Form validation broken, mobile layout issues, slow API calls

4. Fix discovered issues:
   - Each gap = new task
   - Add tests that would have caught it
   - Verify fixes don't break other flows

5. Result: Continuous improvement cycle
```

**For an API service:**
```
Current state: No open tickets, all specs look complete

1. Audit gaps:
   - Do all documented endpoints actually work?
   - Error responses clear and helpful?
   - Performance under realistic load?
   - Contract matches what clients expect?

2. Design validation:
   - Integration tests calling real endpoints
   - Contract tests (OpenAPI/Pact validation)
   - Load tests to catch performance regressions
   - Example requests as automated tests

3. Implement and discover:
   - Build integration test suite
   - Add contract validation
   - Find: Response schema mismatch, slow queries, unclear errors

4. Fix discovered issues:
   - Each problem = new task
   - Add tests that prevent regressions
   - Monitor metrics to catch future issues

5. Result: Continuous improvement cycle
```

**The pattern is universal:** Audit → Design validation → Discover real issues → Fix → Repeat

**This creates sustainable autonomy:** build → verify → discover → fix → build.

## Why This Matters

**From real failures that proper feedback loops would have caught:**

- UI bug shipped, users couldn't complete signup → Lost customers (no E2E tests)
- Performance regression made app 10x slower → User complaints (no benchmark tests)
- API breaking change broke all mobile clients → Emergency rollback (no contract tests)
- CLI flag broken for 3 months, nobody noticed → Trust damage (no integration tests)
- Documentation examples didn't work → Bad first impressions (no doc tests)
- Configuration error crashed production → Downtime (no config validation)

**Every single one preventable with proper feedback loops.**

## When to Apply

**ALWAYS apply this skill when:**

- **Building new features** - Design feedback mechanisms before implementing
- **Discovering bugs** - Ask: "Why didn't tests catch this?" then add validation
- **Working without tests** - Recognize the risk, plan to address it
- **Claiming work complete** - Verify adequate feedback exists for what you built
- **Seeking maintenance work** - Find gaps, build validation, discover real issues
- **Operating autonomously** - Establish maintenance cycles to stay productive
- **Reviewing code** - Check if changes have appropriate feedback mechanisms

**Especially important for autonomous agents:**

When you complete your assigned tasks but are still looping:
1. Use this skill to audit for gaps
2. Build validation that discovers real issues
3. Fix the issues found
4. Repeat - never run out of valuable work

## The Bottom Line

**If it can break silently, you don't have enough feedback.**

Design validation. Make breaks loud. Create backpressure.

Without feedback loops, autonomous systems drift into broken states.
With feedback loops, they self-correct and improve continuously.

**Build the loops. Trust the process.**
