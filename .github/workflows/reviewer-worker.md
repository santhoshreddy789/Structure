---
# ==============================================================================
# WORKFLOW INPUTS
# ==============================================================================
on:
  workflow_dispatch:
    inputs:
      target_repo:
        description: 'Target repository (owner/repo)'
        required: true
        type: string
      jira_key:
        description: 'Jira issue key'
        required: true
        type: string
      summary:
        description: 'Original vulnerability summary'
        required: true
        type: string
      pr_number:
        description: 'Draft PR number to review'
        required: true
        type: string
      pr_branch:
        description: 'Draft PR branch name'
        required: true
        type: string
      attempt:
        description: 'Current attempt number (1, 2, or 3)'
        required: true
        type: string

# ==============================================================================
# INDEPENDENT REVIEWER MODEL (MODEL B)
# ==============================================================================
# Uses a DIFFERENT model family than the Fixer (`claude-sonnet-4-6`) and runs
# with zero memory of the Fixer's internal reasoning to prevent self-review bias.
engine:
  id: copilot
  model: gpt-5.2

# ==============================================================================
# WORKSPACE CHECKOUT (DRAFT PR BRANCH + CENTRAL REVIEW SKILL)
# ==============================================================================
checkout:
  repository: ${{ github.event.inputs.target_repo }}
  ref: ${{ github.event.inputs.pr_branch }}
  github-token: ${{ secrets.ORG_REPO_CHECKOUT_TOKEN }}
  current: true

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
# SAFE OUTPUTS (QUALITY GATE GUARDRAILS)
# ==============================================================================
# Note: `max: 1` applies per individual workflow run (so each of the 3 attempts
# gets 1 review decision). The Reviewer cannot directly push code; it can only
# approve/promote the PR, request changes & bounce back to Fixer, or close it.
safe-outputs:
  github-token: ${{ secrets.GH_AW_CROSS_REPO_PAT }}
  create-pull-request-review-comment:
    target-repo: ${{ github.event.inputs.target_repo }}
    max: 10
  submit-pull-request-review:
    target-repo: ${{ github.event.inputs.target_repo }}
    max: 1
  mark-pull-request-as-ready-for-review:
    target-repo: ${{ github.event.inputs.target_repo }}
    max: 1
  close-pull-request:
    target-repo: ${{ github.event.inputs.target_repo }}
    max: 1
  dispatch-workflow:
    workflows: [fixer-worker]
    max: 1
---

# Independent Security PR Reviewer (Attempt ${{ github.event.inputs.attempt }} of 3)

- **Target Repository:** `${{ github.event.inputs.target_repo }}`
- **Jira Ticket:** `${{ github.event.inputs.jira_key }}`
- **Original Vulnerability Summary:** ${{ github.event.inputs.summary }}
- **Draft PR:** `#${{ github.event.inputs.pr_number }}` (Branch: `${{ github.event.inputs.pr_branch }}`)
- **Attempt:** `${{ github.event.inputs.attempt }}` of `3`

## Step 1: Strict Security & Quality Audit
1. Read `.central-skills/.github/skills/security-review/SKILL.md`.
2. Inspect the full diff of Draft PR `#${{ github.event.inputs.pr_number }}` and run tests if needed to verify behavior.
3. Verify strictly:
   - Does the patch completely fix the root cause described in `${{ github.event.inputs.summary }}` (without superficial workarounds)?
   - Were all existing unit/integration tests preserved without weakening or deleting assertions?
   - Are there any new security issues, regressions, breaking API changes, or leaked artifacts (such as `.central-skills`)?

## Step 2: Enforce Quality Gate Decision

### Case A: `APPROVED` (All checks pass)
1. Call `submit-pull-request-review` with event `APPROVE` and a concise verification summary.
2. Call `mark-pull-request-as-ready-for-review` on PR `#${{ github.event.inputs.pr_number }}` (this notifies human `CODEOWNERS` for final merge).
3. Comment the Ready-for-Review PR link on Jira `${{ github.event.inputs.jira_key }}` and transition Jira status to `"In Review"`.

### Case B: `CHANGES_REQUESTED` AND `attempt < 3` (Attempt 1 or 2)
1. For each issue found, call `create-pull-request-review-comment` with exact file path, line number, and actionable fix instructions.
2. Call `submit-pull-request-review` with event `REQUEST_CHANGES`.
3. Calculate `next_attempt = attempt + 1`.
4. Call `dispatch-workflow` for `fixer-worker` passing **all 6 inputs** intact:
   - `target_repo`: `${{ github.event.inputs.target_repo }}`
   - `jira_key`: `${{ github.event.inputs.jira_key }}`
   - `summary`: `${{ github.event.inputs.summary }}`
   - `pr_number`: `${{ github.event.inputs.pr_number }}`
   - `pr_branch`: `${{ github.event.inputs.pr_branch }}`
   - `attempt`: `<next_attempt>`

### Case C: `CHANGES_REQUESTED` AND `attempt == 3` (Max Retries Exhausted)
1. Call `submit-pull-request-review` with event `COMMENT` summarizing why automated remediation failed after 3 attempts and listing remaining blockers.
2. Call `close-pull-request` on Draft PR `#${{ github.event.inputs.pr_number }}` so unverified code is never left open.
3. Comment the full failure summary on Jira `${{ github.event.inputs.jira_key }}` and transition Jira status to `"Needs Manual Review"`.
