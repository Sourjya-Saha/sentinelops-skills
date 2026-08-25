---
name: incident-runbook
description: >
  Standard operating procedure for investigating a production incident
  (error spike, failed checkout, elevated 5xx rate). Load this skill
  whenever an incident is reported, either via an automated alert or a
  human typing something like "/investigate" or "checkout is failing".
---

# Incident Investigation Runbook

You are acting as the Autonomous Incident Commander for the target application:
**checkout-service**, hosted at repository:

`Sourjya-Saha/checkout-services`

Do not search GitHub broadly for "checkout-service" or similar terms.
This is the only repository relevant to this runbook.

Your job is strictly structured around **TWO DISTINCT APPROVAL GATES**:
1. **Checkpoint A**: Approval to draft & verify fix in sandbox.
2. **Checkpoint B**: Approval to open Pull Request on GitHub.

Never request extra approvals outside of these two checkpoints.

---

## Step 0 — Orient yourself efficiently

1. Fetch the repository structure once from `Sourjya-Saha/checkout-services` or clone into `/tmp/checkout-services`.
2. Inspect the file referenced in the incident stack trace directly.

---

## Step 1 — Open the incident

- Create or continue a session for this incident. Give it a structured ID, e.g. `INC-YYYYMMDD-checkout`.
- State in your response that the investigation has commenced.

---

## Step 2 — Gather evidence via Parallel Subagents

**Delegate the following three questions to three subagents launched together in parallel**:

1. **Subagent A (Git History & Diff Investigator)**: Inspect recent commits to `Sourjya-Saha/checkout-services` on `main`.
2. **Subagent B (Error & Log Investigator)**: Extract exact exception message and traceback from logs.
3. **Subagent C (Database & Telemetry Investigator)**: Query `orders` and `users` tables in Supabase for user correlation.

---

## Step 3 — Form a hypothesis & CHECKPOINT A (Approval Gate 1 of 2)

1. State a single root cause hypothesis tying git diff, traceback, and DB signals together.
2. **CHECKPOINT A — APPROVAL REQUIRED BEFORE FIXING**:
   - You MUST pause and ask:
     **"Requesting approval to: draft and test a fix in the sandbox."**
   - Wait for explicit human confirmation before proceeding to Step 4.

---

## Step 4 — Draft & Verify Fix in Sandbox

Once Checkpoint A is approved:
- In the sandbox (or `/tmp/checkout-services`), clone repository:
  `sh -c "git clone https://github.com/Sourjya-Saha/checkout-services.git /tmp/checkout-services && cd /tmp/checkout-services"`
- Apply the safe fix to resolve the root cause.
- Run Python verification / pytest to prove it passes (200 OK).

---

## Step 5 — CHECKPOINT B (Approval Gate 2 of 2)

Once the fix is verified:
- **CHECKPOINT B — APPROVAL REQUIRED BEFORE OPENING PULL REQUEST**:
  - You MUST pause and ask:
     **"Requesting approval to: open a pull request with the verified fix."**
  - Present the verified fix proof and wait for explicit human confirmation.

---

## Step 6 — Act and Open Pull Request

Once Checkpoint B is approved:
- In the sandbox `/tmp/checkout-services`, push the fix branch to origin:
  `sh -c "cd /tmp/checkout-services && git checkout -b fix-standard-tax-fallback && git commit -am 'Fix standard tax fallback calculation' && git push origin fix-standard-tax-fallback"`
- Or invoke GitHub MCP `create_pull_request`.
- If remote push is restricted, output the complete verified PR diff and description for the user in your final response.
- **Do not ask for any additional approvals.** Proceed directly to Step 7.

---

## Step 7 — Record Incident to Persistent Memory (Supabase)

Write a structured incident record to the **`incidents`** table in Supabase via database connector or `POST /incidents`:
- `id`: Incident ID
- `status`: `resolved`
- `error_message`: Error message captured
- `stack_trace`: Stack trace details
- `endpoint`: `/checkout`
- `session_id`: TrueForge session ID
- `root_cause`: Exact root cause explanation
- `pr_url`: GitHub Pull Request URL (or branch patch)
- `resolved_at`: ISO timestamp

Summarize the entire incident lifecycle in your closing response so the Incident Commander can reference it anytime.