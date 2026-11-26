---
name: vuln-fix
description: Implements minimal, idiomatic security patches without introducing breaking changes, workarounds, or leftover artifacts.
---

# Vulnerability Remediation Fix Skill

Apply a clean, minimal, production-safe fix for a confirmed `TRUE_POSITIVE` vulnerability.

## Core Remediation Rules
1. **Fix the Root Cause (No Suppression):**
   - Never silence security warnings using suppression annotations (`// nolint`, `# nosec`, `@SuppressWarnings`, `// eslint-disable`) or dummy `try/catch` blocks that hide errors.
   - Fix input validation, encoding, authorization checks, or dependency versions directly at the source.
2. **Minimal Blast Radius:**
   - Touch only the files required to fix the vulnerability and its associated unit/integration tests.
   - Do **not** reformat unrelated files, rename public APIs, or upgrade unrelated dependencies.
3. **Dependency Upgrades:**
   - Upgrade to the **minimum non-vulnerable patched version** (or minimal compatible minor/patch release) to avoid breaking major-version API changes.
   - Regenerate the project's lockfile (`package-lock.json`, `yarn.lock`, `go.sum`, `poetry.lock`, etc.) using the project's package manager—never hand-edit lockfiles.
4. **Add Regression Tests:**
   - Add or update a focused unit test that reproduces the vulnerable input/condition and asserts that the patched code blocks or handles it safely.
5. **Workspace Hygiene:**
   - Ensure temporary files and `.central-skills/` are **never** staged or committed into the target repository (`git status` must show only intentional source/lockfile/test changes).