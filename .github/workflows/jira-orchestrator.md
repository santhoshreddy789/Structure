---
# ==============================================================================
# TRIGGERS
# ==============================================================================
# Supports BOTH:
# 1. Real-time webhook from Jira Automation (`repository_dispatch`)
# 2. Periodic cron fallback (`schedule`) to drain bulk scanner queues safely
on:
  repository_dispatch:
    types: [jira-vuln-batch]
  schedule:
    - cron: '*/20 * * * *'   # Runs every 20 minutes to drain remaining backlog

# ==============================================================================
# CONCURRENCY LOCK (BULK DEDUPLICATION)
# ==============================================================================
# Prevents two Orchestrator runs from executing simultaneously and grabbing the
# exact same Jira tickets. If a bulk scanner triggers 50 webhooks in 2 seconds,
# GitHub runs the first one and collapses/queues the rest.
## as per copilot remove this
#concurrency:
  #group: jira-vuln-orchestrator
  #cancel-in-progress: false

# ==============================================================================
# ENGINE CONFIGURATION
# ==============================================================================
engine:
  id: copilot
  model: claude-sonnet-4-6

# ==============================================================================
# TOOLS & MCP SERVERS
# ==============================================================================
# Gives the Orchestrator read/write access to Jira (to query open tickets and
# transition status) and read-only GitHub access (to verify target repos/PRs).
tools:
  github:
    github-token: ${{ secrets.GH_AW_READ_ORG_TOKEN }}
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
# Hard rate limit: The Orchestrator can trigger at most 5 worker runs per batch,
# protecting GitHub API quotas and LLM token rate limits during bulk scans.

# workflows: [fixer-worker] → the system is allowed to dispatch the fixer-worker workflow.
# max: 5 → the dispatcher can launch at most 5 instances of that workflow during a single execution cycle, request, or batch.
safe-outputs:
  dispatch-workflow:
    workflows: [fixer-worker]
    max: 5
---

# Bulk Vulnerability Rollout Orchestrator

You are the **Vulnerability Rollout Orchestrator**. Your job is to pull a prioritized batch of open vulnerability tickets from Jira, lock them so they are not duplicated, and dispatch isolated `fixer-worker` runs.

## Step 1: Query Open Vulnerability Tickets
1. Query Jira using JQL:
   `labels = "ai-remediate" AND status = "Open" ORDER BY priority DESC, created ASC`
2. If **0 issues** are returned, exit immediately and output: `"No open remediation tickets found."`
3. Select strictly the **top 5 issues** from the sorted list.

<!-- ## Step 2: Validate, Lock & Dispatch Each Ticket
For each of the selected tickets (up to 5):

1. **Extract Metadata:**
   - `jira_key`: Jira Issue Key (e.g., `SEC-1042`)
   - `target_repo`: Target repository in `owner/repo` format (from the Repository custom field or `repo:owner/repo` label)
   - `summary`: Concise summary of the CVE / vulnerability description

2. **Idempotency Check (Duplicate Protection):**
   - Check `target_repo` for any **open or recently merged Pull Request** whose title starts with `[${jira_key}]`.
   - If a PR already exists, transition the Jira ticket to `"In Review"` (or `"Done"` if merged) and **skip** dispatching a worker.

3. **Lock the Ticket in Jira:**
   - Transition the Jira issue status from `"Open"` to `"AI In Progress"` (and add a comment: `"Queued for automated AI remediation."`).
   - Doing this *before* dispatching guarantees the next scheduled batch will never pick up the same ticket.

4. **Dispatch Fixer Worker:**
   - Call `dispatch-workflow` for `fixer-worker` with these exact inputs:
     - `target_repo`: `<owner/repo>`
     - `jira_key`: `<jira_key>`
     - `summary`: `<summary>`
     - `pr_number`: `""`
     - `pr_branch`: `""`
     - `attempt`: `"1"` -->

## Step 2: Acquire Lock, Validate & Dispatch

For each selected ticket (up to 5):

### Extract Metadata

- jira_key: Jira Issue Key (e.g., SEC-1042)
- target_repo: Target repository in owner/repo format
  (from the Repository custom field or repo:owner/repo label)
- summary: Concise summary of the CVE / vulnerability description

<!-- ### Acquire Jira Lock (Authoritative Ownership Check)

Immediately attempt to transition the Jira issue from:

Open → AI In Progress

Add comment:

"Queued for automated AI remediation.
Lock acquired by GitHub Actions run ${github.run_id}."

If the transition fails, or the issue is no longer in Open status:

- Assume another Orchestrator run has already claimed the ticket.
- Skip the issue and continue to the next ticket.

Only tickets successfully transitioned to AI In Progress are considered owned by the current Orchestrator run. -->

### Acquire Jira Lock (Authoritative Ownership Check)

Attempt to transition the Jira issue from:

Open → AI In Progress

Add comment:

"Queued for automated AI remediation.
Lock acquired by GitHub Actions run ${github.run_id}."

If the transition succeeds:

- The current Orchestrator run owns the ticket and may continue processing.

If the transition fails, or the issue is no longer in Open status:

- Another Orchestrator run has already claimed the ticket.
- Skip the issue and continue to the next ticket.

### Idempotency Check (Duplicate Protection)

Check target_repo for any open or recently merged Pull Request whose title starts with:

[${jira_key}]

If an open PR exists:

- Transition Jira to In Review.
- Add a comment containing the PR link.
- Skip worker dispatch.

If a merged PR exists:

- Transition Jira to Done.
- Add a comment containing the merged PR link.
- Skip worker dispatch.

### Dispatch Fixer Worker

Call dispatch-workflow for fixer-worker with:

- target_repo: <owner/repo>
- jira_key: <jira_key>
- summary: <summary>
- pr_number: ""
- pr_branch: ""
- attempt: "1"

## Step 3: Output Batch Summary
Summarize which tickets were dispatched, which were skipped (and why), and how many remain in the Jira backlog.