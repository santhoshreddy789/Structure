---
name: test-runner
description: Discovers and executes the target repository's existing unit, integration, and build checks to verify a security patch causes zero regressions.
---

# Test & Build Verification Skill

Before opening or updating any Pull Request, verify that the codebase builds cleanly and passes all relevant unit and integration tests.

## Step 1: Discover Exact CI Commands
Do not guess build or test commands. Inspect the target repository in this order:
1. `.github/workflows/*.yml` (look for `run:` steps executing `test`, `build`, `lint`, `verify`, `check`)
2. `Makefile`, `package.json` (`scripts`), `pom.xml`, `build.gradle`, `pyproject.toml`, or `go.mod`

## Step 2: Execute Checks in Order
1. **Compile / Build Check:** Run the build step first to catch syntax, type, or dependency resolution errors immediately.
2. **Unit Tests:** Run the full unit test suite (or targeted package tests first, followed by the full suite).
3. **Integration Tests:** If integration tests are defined in CI and runnable in the current environment (e.g., Testcontainers, in-memory DB, mocked HTTP), execute them. If an external live cloud environment is required and unavailable, document why in the PR body.
4. **Linter / Static Checks:** Run configured linters/formatters so the PR passes target repo CI checks on the first push.

## Step 3: Failure Handling
- If any test fails due to your patch, diagnose the failure and refine the fix (respecting `vuln-fix` rules—**never** delete or weaken existing test assertions just to make tests pass).
- If tests fail due to an unresolvable environment limitation or deep breaking dependency conflict, **abort before creating a PR** and report the exact failure logs to Jira.