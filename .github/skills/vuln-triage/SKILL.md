---
name: vuln-triage
description: Validates whether a reported CVE or security finding is actually present, reachable, and exploitable in the target repository before writing any code.
---

# Vulnerability Triage Skill

Your goal is to determine whether the vulnerability described in the Jira ticket is a **True Positive** (requires a code/dependency change) or a **False Positive** (not applicable, unreachable, or already fixed).

## Step 1: Identify Finding Category
Classify the ticket into one of two categories:
1. **Dependency / SCA Vulnerability (CVE / GHSA):** A vulnerable third-party library version.
2. **Code / SAST Vulnerability (CWE):** An insecure coding pattern (e.g., SQL injection, XSS, path traversal, SSRF, hardcoded secret).

## Step 2: Verify Presence in the Codebase
- **For Dependency Findings:**
  - Inspect manifest and lockfiles (`package.json`, `package-lock.json`, `pom.xml`, `build.gradle`, `go.mod`, `requirements.txt`, `pyproject.toml`, `Cargo.toml`, etc.).
  - Check the **exact resolved version** in the lockfile—not just the loose range in the manifest. If the resolved version is already at or above the patched version, mark as **Already Patched**.
  - Check whether the vulnerable package is only used in `devDependencies` / test scopes (e.g., a test runner or linter not shipped to production) and whether the vulnerable function/module is actually imported or called in application code.
- **For Code / SAST Findings:**
  - Locate the exact file, function, and line numbers referenced in the ticket.
  - Trace data flow from untrusted user inputs (sources) to the sensitive operation (sinks). Check if sanitization, parameterized queries, or framework-level protections already neutralize the input.

## Step 3: Triage Decision
Output a structured triage report containing:
- **Verdict:** `TRUE_POSITIVE` or `FALSE_POSITIVE`
- **Affected Files / Packages:** Exact file paths and versions found
- **Reachability & Exploitability Reasoning:** Concise technical evidence supporting your verdict
- **Recommended Remediation Strategy:** (Only if `TRUE_POSITIVE`) What needs to change

### Exit Rule
- If `FALSE_POSITIVE`: **Stop immediately.** Do not modify files, create branches, or open a PR. Post this triage report to Jira so the ticket can be closed cleanly.