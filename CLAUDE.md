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
