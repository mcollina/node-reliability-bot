# CLAUDE.md

This file provides guidance to Claude Code for fixing flaky tests in the Node.js repository.

## Workflow Overview

This project automates the process of identifying and fixing flaky tests in Node.js by:
1. Fetching the latest reliability issue from the Node.js reliability tracker
2. Identifying the test with the most failures
3. Creating a fix and opening a PR

## Step 1: Fetch Latest Reliability Issue

Use the GitHub CLI to get the latest reliability issue:

```bash
# List recent issues from the reliability repo
gh issue list --repo nodejs/reliability --limit 10 --json number,title,createdAt

# Get the latest issue (the most recent one)
gh issue view <ISSUE_NUMBER> --repo nodejs/reliability --json body,title
```

The reliability issues contain a table of flaky tests with failure counts. Look for the **test with the highest failure count** to prioritize fixing.

### Parsing the Issue

The issue body contains markdown tables with test results. Look for patterns like:
- Test file paths (e.g., `test/parallel/test-*.js`)
- Failure counts or percentages
- Platform information (linux, macos, windows)

Select the test with the **most failures** for maximum impact.

### Verifying the Failure is on Main

**IMPORTANT**: Only fix tests that are flaky on the main branch. Before working on a test:

1. Check the detailed failure report to see which PRs triggered the failure
2. Verify the test exists and has the same code on main - the failure might be from a PR that changed the test
3. Skip failures that are infrastructure issues, not actual flaky tests:
   - `ENOSPC: no space left on device` - disk space issue on CI
   - `Failed to trigger node-test-commit` - CI infrastructure issue
   - `fatal: No rebase in progress?` - git issue on CI
   - `Directory not empty` - cleanup issue on CI

If the test file was recently added or modified by a PR, the flakiness might be in that PR's changes, not in main. In that case, skip to the next test.

### Reproducing the Flakiness

**IMPORTANT**: Before attempting any fix, you must first reproduce the flakiness locally:

```bash
# Run the test repeatedly to reproduce the failure
python3 tools/test.py --repeat 100 test/parallel/test-<name>.js
```

If the test passes consistently after many runs (100+), the flakiness may be:
- Platform-specific (e.g., only fails on macOS, Windows ARM64, or specific CI environments)
- Environment-specific (e.g., shared library builds, debug builds)
- Already fixed in main

**Do not attempt to fix a test if you cannot reproduce the flakiness.** Move on to the next test instead.

## Step 2: Prepare the Node.js Repository

Before starting work, always ensure you're on a clean, up-to-date main branch:

```bash
cd ./node

# Checkout main and pull latest from upstream
git checkout main
git fetch upstream
git reset --hard upstream/main

# Create a new branch for the fix
git checkout -b fix-flaky-<test-name>
```

**Important**: Always work on a separate branch for each fix.

## Step 3: Build Node.js

**IMPORTANT**: Always build Node.js before running tests. Changes to `lib/` or `src/` require a rebuild.

```bash
cd ./node

# Configure and build (first time or after clean)
./configure
make -j4

# Rebuild after changes (faster, only recompiles what changed)
make -j4
```

## Step 4: Analyze and Fix the Flaky Test

### Locating the Test

```bash
# Tests are in the test/ directory
# Most flaky tests are in test/parallel/ or test/sequential/
ls test/parallel/test-<pattern>*.js
```

### Common Flaky Test Causes

1. **Timing issues**: Hardcoded timeouts, race conditions
2. **Resource exhaustion**: File descriptors, ports, memory
3. **Order dependencies**: Tests assuming specific execution order
4. **Platform differences**: OS-specific behavior not accounted for
5. **Network issues**: Assuming localhost availability, port conflicts

### Running the Test

```bash
# Run the specific test
./node test/parallel/test-<name>.js

# Or use the test runner
tools/test.py test/parallel/test-<name>.js

# Run test multiple times to verify fix (use --repeat)
python3 tools/test.py --repeat 1000 test/parallel/test-<name>.js
```

## Step 5: Node.js Repository Reference

### Build Commands

```bash
# Configure and build
./configure
make -j4

# Build with ninja (faster)
./configure --ninja
make -j4

# Debug build
./configure --debug
make -j4
```

### Testing

```bash
# Run all tests
make -j4 test

# Run tests only (no linting)
make test-only

# Run a single test file
tools/test.py test/parallel/test-<name>.js
# Or directly:
./node test/parallel/test-<name>.js

# Run tests for a subsystem
tools/test.py <subsystem>

# Run tests matching a pattern
tools/test.py "test/parallel/test-stream-*"
```

### Linting

```bash
# Run all linters
make lint

# Fix auto-fixable JS lint errors
make lint-js-fix
```

### Architecture

- **`lib/`** - JavaScript implementation of Node.js core modules
  - `lib/internal/` - Internal modules not exposed in public API
- **`src/`** - C++ source code for Node.js runtime
- **`test/parallel/`** - Tests that run in parallel (most tests)
- **`test/sequential/`** - Tests that must run sequentially
- **`test/common/`** - Shared test utilities

### Test Patterns

- JavaScript modules in `lib/` use `primordials` for guaranteed built-in behavior
- Test files follow naming convention: `test-<subsystem>-<description>.js`
- Use `require('../common')` at the top of test files

## Step 6: Create the Pull Request

After fixing and verifying the test:

```bash
cd ./node

# Stage and commit changes
git add test/parallel/test-<name>.js
git commit -m "test: fix flaky <test-name>

Fixes timing issue causing intermittent failures.

Refs: nodejs/reliability#<ISSUE_NUMBER>"

# Push to your fork
git push -u origin fix-flaky-<test-name>

# Create PR
gh pr create --title "test: fix flaky <test-name>" --body "$(cat <<'EOF'
## Summary

Fixes flaky test `test/<path>/test-<name>.js`.

## Problem

[Describe the root cause of the flakiness]

## Solution

[Describe the fix]

## Verification

Verified with `python3 tools/test.py --repeat 1000` without failures.

Refs: nodejs/reliability#<ISSUE_NUMBER>
EOF
)"
```

### PR Requirements

- PRs need at least two collaborator approvals
- CI must pass before landing
- Wait 48 hours before landing (some PRs can be fast-tracked)

### Commit Guidelines

- **Never sign commits as Claude** - Do not add `Co-Authored-By: Claude` or similar attribution lines
- Follow Node.js commit message conventions (subsystem prefix, imperative mood)
- **Always run linters before committing**: Run `make lint` or `make lint-js-fix` to check and fix lint errors
- **Validate commits before pushing**: Run `npx core-validate-commit` to check commit format
  - Ignore errors about missing PR-URL (not available before PR is created)
  - Ignore errors about missing reviewers (added during review process)

## Quick Reference Commands

```bash
# Get latest reliability issue
gh issue list --repo nodejs/reliability --limit 1 --json number,title | jq '.[0]'

# View issue details
gh issue view <NUMBER> --repo nodejs/reliability

# Prepare node repo
cd ./node && git checkout main && git fetch upstream && git reset --hard upstream/main

# Create branch
git checkout -b fix-flaky-<test-name>

# Run test repeatedly to verify fix
python3 tools/test.py --repeat 1000 test/parallel/test-<name>.js

# Lint before committing
make lint-js-fix
```

## Key Learnings

This section documents important lessons learned from fixing flaky tests.

### Most Failures Are Not Reproducible Locally

The majority of failures in reliability reports fall into these categories:

1. **Infrastructure issues** (~40% of failures):
   - `ENOSPC: no space left on device` - ARM64 debug nodes running out of disk
   - `Failed to trigger node-test-commit` - CI job trigger failures
   - `fatal: No rebase in progress?` - Git state issues on CI
   - `Build timed out` - CI resource constraints

2. **Platform-specific failures** (~40% of failures):
   - macOS-only: watch mode tests, HTTP/2 timeouts, zlib chunk counts
   - Windows-only: path casing issues, crypto timeouts
   - ARM64-only: timing-sensitive tests on slower hardware

3. **Actually fixable on Linux x64** (~20% of failures):
   - These are the ones worth investigating

### Checking the Detailed Report is Essential

Always fetch the detailed markdown report to understand:
- Which **platforms** failed (Linux x64 vs macOS vs Windows vs ARM64)
- Which **PRs** triggered the failure (might be PR-specific, not main)
- The **actual error message** (ENOSPC vs assertion vs timeout)

```bash
# Get detailed report
https://raw.githubusercontent.com/nodejs/reliability/main/reports/YYYY-MM-DD.md
```

### Common Fixes That Worked

1. **Race conditions with async operations**: Use `setImmediate()` or `process.nextTick()` to allow pending operations to complete before assertions.

2. **File handle lifecycle issues**: Ensure file handles are properly closed before test cleanup/GC runs. Add explicit `closeHandle()` methods.

3. **Platform-specific skips**: Some tests legitimately cannot work on certain platforms. Use `common.skip()`:
   ```javascript
   if (common.isAIX)
     common.skip('AIX does not reliably capture syntax errors in watch mode');
   ```

4. **Case-insensitive path matching on Windows**: Use regex with `gi` flag for path replacements on Windows since drive letters can have different casing.

### PR Lifecycle

1. **Review process**: PRs need 2 approvals from Node.js collaborators
2. **CI must pass**: All platforms must pass before merge
3. **Time window**: After merge, the test still appears in reports for 1-2 days because reports cover a rolling window of CI runs
4. **Address feedback promptly**: Reviewers often have good suggestions (e.g., removing unnecessary `process.nextTick()`, removing unused event emissions)

### What NOT to Fix

- Infrastructure failures (ENOSPC, CI triggers) - these need CI team attention
- Platform-specific failures you can't reproduce - leave for maintainers with access to those platforms
- Tests that pass 100+ times locally - the flakiness is environment-specific

### Efficiency Tips

1. **Check multiple days of reports** - some tests appear consistently, others are one-offs
2. **Start with highest failure count** - more impact per fix
3. **Run reproduction tests in parallel** with other investigation
4. **Use `--repeat 50` first**, then `--repeat 1000` only for final verification
5. **Read the test code AND the helper modules** (e.g., `test/common/*.js`) to understand the full picture

### Always Run Linting Before Committing

**CRITICAL**: Always run `make lint-js` before committing changes. Common lint issues:
- Unused imports (e.g., removing code that used `os` but leaving `const os = require('os')`)
- Trailing whitespace
- Indentation inconsistencies
- Line length violations

```bash
# Check for lint errors
make lint-js

# Auto-fix what can be fixed
make lint-js-fix
```

Running the full linter takes time, so run it once before committing rather than after each edit.
