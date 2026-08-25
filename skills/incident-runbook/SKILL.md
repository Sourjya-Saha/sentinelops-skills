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

- Create or continue a session for this incident. Generate a dynamic ID (e.g. `INC-YYYYMMDD-<issue>`).
- State in your response that the investigation has commenced.

---

## Step 2 — Gather evidence via Parallel Subagents

**Delegate the following three questions to three subagents launched together in parallel**:

1. **Subagent A (Git History & Diff Investigator)**: Inspect recent commits to `Sourjya-Saha/checkout-services` on `main` to identify the regression.
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
- Dynamically determine and apply the safe fix for the root cause.
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
- Autonomously generate a descriptive branch name from the root cause (e.g. `fix-<issue-slug>`).
- In the sandbox `/tmp/checkout-services`, push the branch:
  `sh -c "cd /tmp/checkout-services && git checkout -b <branch-name> && git commit -am '<dynamic-commit-message>' && (git push https://${GITHUB_TOKEN}@github.com/Sourjya-Saha/checkout-services.git <branch-name> || git push origin <branch-name>)"`
- Or use GitHub MCP `create_pull_request`.
- Construct the GitHub PR link using your generated branch name:
  `https://github.com/Sourjya-Saha/checkout-services/pull/new/<branch-name>` (or the actual PR URL if created).
- Proceed directly to Step 7 without asking for additional approvals.

---

## Step 7 — Record Incident to Persistent Memory & Conclude

1. Write a structured incident record to the **`incidents`** table in Supabase via database connector or `POST /incidents`:
   - `id`: Dynamic Incident ID
   - `status`: `resolved`
   - `error_message`: Error message captured
   - `stack_trace`: Stack trace details
   - `endpoint`: `/checkout`
   - `session_id`: TrueForge session ID
   - `root_cause`: Autonomous root cause explanation
   - `pr_url`: The PR link constructed in Step 6
   - `resolved_at`: ISO timestamp

2. **Conclude your final response using this structured format:**

```markdown
Done — I drafted, verified, and opened a PR without merging it.

### What I changed:
- <concise summary of files and fix applied>

### Verification:
- Ran `pytest backend/tests/test_checkout.py`
- Result: <number> passed

### PR:
[<pr-url>](<pr-url>)

If you want, I can also summarize the root cause and fix in a short incident note for your team.
```