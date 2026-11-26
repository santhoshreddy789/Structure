---
# ==============================================================================
# WORKFLOW INPUTS (STATELESS HANDOFF)
# ==============================================================================
# Because every retry runs on a brand-new GitHub Actions runner with zero memory,
# all state (repo, Jira key, original summary, Draft PR number/branch, attempt)
# is passed explicitly between Fixer and Reviewer.
on:
  workflow_dispatch:
    inputs:
      target_repo:
        description: 'Target repository to remediate (owner/repo)'
        required: true
        type: string
      jira_key:
        description: 'Jira issue key (e.g., SEC-1042)'
        required: true
        type: string
      summary:
        description: 'Original vulnerability summary / CVE details'
        required: true
        type: string
      pr_number:
        description: 'Existing Draft PR number (populated on attempt 2 and 3)'
        required: false
        default: ''
        type: string
      pr_branch:
        description: 'Existing Draft PR branch name (populated on attempt 2 and 3)'
        required: false
        default: ''
        type: string
      attempt:
        description: 'Current remediation attempt counter (1, 2, or 3)'
        required: false
        default: '1'
        type: string

# ==============================================================================
# PER-REPOSITORY CONCURRENCY QUEUE
# ==============================================================================
# Tickets for DIFFERENT repositories run 100% in parallel.
# Tickets for the SAME repository wait in line (`cancel-in-progress: false`)
# so they don't collide or create merge conflicts on the same codebase.
concurrency:
  group: remediation-${{ github.event.inputs.target_repo }}
  cancel-in-progress: false

# ==============================================================================
# FIXER MODEL (MODEL A)
# ==============================================================================
engine:
  id: copilot
  model: claude-sonnet-4-6

# ==============================================================================
# WORKSPACE CHECKOUT (TARGET REPO + CENTRAL SKILLS)
# ==============================================================================
# Checks out the target repository into the workspace root (`current: true`).
# On Attempt 1 (`pr_branch == ''`), checks out the default branch.
# On Attempt 2+, checks out the existing Draft PR branch (`pr_branch`).
checkout:
  repository: ${{ github.event.inputs.target_repo }}
  ref: ${{ github.event.inputs.pr_branch }}
  github-token: ${{ secrets.ORG_REPO_CHECKOUT_TOKEN }}
  current: true

# Clones the Central Repository's `.github/skills/` folder into `.central-skills/`
# so target repositories remain clean and never need skill files stored inside them.
steps:
  - name: Fetch Central Repository Skills
    uses: actions/checkout@v4
    with:
      repository: ${{ github.repository }}
      path: .central-skills
      sparse-checkout: |
        .github/skills

# ==============================================================================
# TOOLS & MCP SERVERS
# ==============================================================================
tools:
  bash: true
  github:
    github-token: ${{ secrets.GH_AW_CROSS_REPO_PAT }}
    toolsets: [repos, pull_requests]
  mcp:
    jira:
      command: npx
      args: ["-y", "@sooperset/mcp-atlassian"]
      env:
        JIRA_URL: ${{ vars.JIRA_URL }}
        JIRA_USERNAME: ${{ secrets.JIRA_USERNAME }}
        JIRA_API_TOKEN: ${{ secrets.JIRA_API_TOKEN }}

# ==============================================================================
# SAFE OUTPUTS (GUARDRAILS)
# ==============================================================================
# - Can ONLY open a Draft PR (`draft: true`), never a ready-for-review PR.
# - Can push updates to the Draft PR branch on retries.
# - Can dispatch ONLY `reviewer-worker` once per run.
safe-outputs:
  github-token: ${{ secrets.GH_AW_CROSS_REPO_PAT }}
  create-pull-request:
    target-repo: ${{ github.event.inputs.target_repo }}
    draft: true
    max: 1
  push-to-pull-request-branch:
    target-repo: ${{ github.event.inputs.target_repo }}
    max: 1
  dispatch-workflow:
    workflows: [reviewer-worker]
    max: 1
---

# Vulnerability Remediation Fixer (Attempt ${{ github.event.inputs.attempt }} of 3)

- **Target Repository:** `${{ github.event.inputs.target_repo }}`
- **Jira Ticket:** `${{ github.event.inputs.jira_key }}`
- **Vulnerability Summary:** ${{ github.event.inputs.summary }}
- **Existing Draft PR Number:** `${{ github.event.inputs.pr_number }}`
- **Existing Draft PR Branch:** `${{ github.event.inputs.pr_branch }}`
- **Attempt Number:** `${{ github.event.inputs.attempt }}`

All central skill definitions are mounted at `.central-skills/.github/skills/`.

---

## Path A: Initial Attempt (`attempt == '1'`)

1. **Triage (`.central-skills/.github/skills/vuln-triage/SKILL.md`):**
   - Analyze whether the vulnerability is actually exploitable or present in `${{ github.event.inputs.target_repo }}`.
   - **False-Positive Exit:** If it is a false positive, unreachable dependency, or already patched:
     - Comment the technical triage findings on Jira `${{ github.event.inputs.jira_key }}` and transition it to `"Closed"` / `"Won't Fix"`.
     - **Stop immediately.** Do NOT create a Pull Request or dispatch `reviewer-worker`.

2. **Apply Fix (`.central-skills/.github/skills/vuln-fix/SKILL.md`):**
   - Implement a minimal, idiomatic fix for the root cause.
   - Ensure `.central-skills/` is untracked / ignored so it is never committed into the target repo.

3. **Run Unit & Integration Tests (`.central-skills/.github/skills/test-runner/SKILL.md`):**
   - Inspect the target repo's existing CI workflows (`.github/workflows/`) to identify build, unit test, and integration test commands.
   - Execute tests and iterate on the fix locally until all tests pass.
   - If tests cannot be fixed after reasonable effort, comment the failure logs on Jira `${{ github.event.inputs.jira_key }}`, transition Jira to `"Needs Manual Review"`, and **stop without creating a PR**.

4. **Raise Draft Pull Request:**
   - Call `create-pull-request` (`draft: true`) with:
     - **Title:** `[${{ github.event.inputs.jira_key }}] Remediate vulnerability`
     - **Body:** Include Triage analysis, Root Cause, Fix summary, and Test execution results.
   - Record the newly created `pr_number` and branch name (`pr_branch`).

---

## Path B: Retry After Reviewer Feedback (`attempt == '2'` or `'3'`)

1. **Read Reviewer Feedback:**
   - Fetch all review comments and review summaries on Draft PR `#${{ github.event.inputs.pr_number }}` in `${{ github.event.inputs.target_repo }}`.

2. **Address Feedback & Re-Test:**
   - Update the code on branch `${{ github.event.inputs.pr_branch }}` to resolve **every** concern raised by the Reviewer while keeping the original vulnerability fix intact.
   - Re-run all unit and integration tests per `.central-skills/.github/skills/test-runner/SKILL.md` until passing.

3. **Push Updated Commit:**
   - Call `push-to-pull-request-branch` to push the updated commit to Draft PR `#${{ github.event.inputs.pr_number }}`.

---

## Final Step (All Paths When PR Exists): Dispatch Independent Reviewer

Call `dispatch-workflow` for `reviewer-worker` passing **all 6 inputs**:
- `target_repo`: `${{ github.event.inputs.target_repo }}`
- `jira_key`: `${{ github.event.inputs.jira_key }}`
- `summary`: `${{ github.event.inputs.summary }}`
- `pr_number`: `<Draft PR number>`
- `pr_branch`: `<Draft PR branch name>`
- `attempt`: `${{ github.event.inputs.attempt }}`
