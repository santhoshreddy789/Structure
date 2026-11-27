---
# ==============================================================================
# AR-FIX (AGENTIC) - RUNS IN THE PRODUCT / TARGET REPO
# ==============================================================================
# Installed in each onboarded product repo. Started by ar-release (central) through
# repository_dispatch. The run checks out THIS repo, so the agent sees the repo's
# code, AGENTS.md and any skills installed for it.
on:
  repository_dispatch:
    types: [ar-fix]
  bots:
    - "<remediation-app>[bot]"

# ==============================================================================
# DISPATCH PAYLOAD (client_payload)
# ==============================================================================
# jira_key, summary, cve, severity, vuln_type, framework, attempt, pr_number, pr_branch

# One run per repo at a time; later runs wait in line
concurrency:
  group: ar-fix-${{ github.repository }}
  cancel-in-progress: false

engine:
  id: copilot

# ==============================================================================
# READ-ONLY AGENT
# ==============================================================================
# The agent holds no write token and no Jira or ServiceNow credentials.
# All writes go through safe-outputs, which run as separate scoped jobs.
permissions:
  contents: read
  pull-requests: read

# ==============================================================================
# SKILLS FROM THE CENTRAL CATALOG (SCRIPT STEP, PINNED)
# ==============================================================================
# Installs only the skills this ticket needs: the base skill for the vulnerability
# type, plus guidance for the repo's framework. The agent chooses among these.
steps:
  - name: Fetch skills from central catalog
    uses: actions/checkout@v7
    with:
      repository: your-org/remediation-platform
      ref: ${{ vars.AR_CATALOG_TAG }}
      token: ${{ secrets.CATALOG_READ_TOKEN }}
      path: .ar-catalog
      sparse-checkout: skills
      persist-credentials: false
  - name: Write ticket file
    env:
      PAYLOAD: ${{ toJson(github.event.client_payload) }}
    run: printf '%s' "$PAYLOAD" > .ar-ticket.json
  - name: Install selected skills
    env:
      VULN_TYPE: ${{ github.event.client_payload.vuln_type }}
      FRAMEWORK: ${{ github.event.client_payload.framework }}
    run: |
      mkdir -p .ar-skills
      cp -r ".ar-catalog/skills/base/${VULN_TYPE}" .ar-skills/ || true
      cp -r ".ar-catalog/skills/guidance/${FRAMEWORK}" .ar-skills/ || true

tools:
  bash: true
  github:
    toolsets: [repos, pull_requests]

# ==============================================================================
# SAFE OUTPUTS (GUARDRAILS)
# ==============================================================================
# - Draft PR only, never ready for review (ar-ready marks it ready later)
# - Retries push to the existing Draft PR branch
# - Reviewer is started in this repo
# - Status goes to the control repo as an event; ar-writeback updates Jira
safe-outputs:
  create-pull-request:
    draft: true
    max: 1
  push-to-pull-request-branch:
    max: 1
  dispatch-workflow:
    workflows: [ar-review]
    max: 1
  dispatch-repository:
    notify_control:
      description: Send a status event to the control repo (ar-writeback)
      workflow: ar-writeback
      event_type: ar-event
      repository: your-org/remediation-platform
      inputs:
        jira_key:
          type: string
        event:
          type: choice
          options: [not-applicable, pr-opened, failed]
        detail:
          type: string
      max: 1

# ==============================================================================
# OBSERVABILITY (OTLP)
# ==============================================================================
observability:
  otlp:
    endpoint: ${{ secrets.OTLP_ENDPOINT }}
    headers:
      Authorization: ${{ secrets.OTLP_AUTHORIZATION }}
    attributes:
      workflow.role: fixer
      jira.key: ${{ github.event.client_payload.jira_key }}
      target.repo: ${{ github.repository }}
      remediation.attempt: ${{ github.event.client_payload.attempt }}
    resource-attributes:
      service.name: ar-remediation

env:
  OTEL_EXPORTER_OTLP_PROTOCOL: http/protobuf

network:
  allowed:
    - defaults
    - otlp.example.com
---

# Vulnerability fix

The ticket is in `.ar-ticket.json` (jira_key, summary, cve, vuln_type, framework, attempt, pr_number, pr_branch). Read it first, then `AGENTS.md` in this repo for build, test and constraints. Skills selected for this ticket are in `.ar-skills/`; use only those. Up to 3 attempts per ticket.

## First attempt (no `pr_number` in the ticket)

1. Check whether the finding is real and reachable in this repo. If it is a false positive or already fixed, call `notify_control` with event `not-applicable` and stop.
2. Apply a minimal fix for the root cause, following the skill in `.ar-skills/`.
3. Build and run the tests the way `AGENTS.md` describes. Iterate until they pass. If they cannot pass, call `notify_control` with event `failed` and stop.
4. Open a draft pull request titled `[<jira_key>] Remediate vulnerability`, with the finding, root cause, fix and test results in the body.
5. Call `notify_control` with event `pr-opened`, then dispatch `ar-review`.

## Retry after reviewer feedback (`pr_number` present)

1. Read the review comments on the existing draft PR.
2. Address every concern, rebuild and re-run the tests.
3. Push to the existing PR branch, then dispatch `ar-review`.

Never mark the PR ready, merge it, or edit `.ar-ticket.json`, `.ar-skills/` or `.ar-catalog/`.
