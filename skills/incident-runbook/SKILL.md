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
This is the only repository relevant to this runbook. If you cannot
access it, or a specific file/path within it doesn't exist, STOP and
report that clearly — do not fall back to searching all of GitHub for
similarly-named repositories, and do not guess at a different repo.

Your job is to INVESTIGATE, PAUSE FOR FIX APPROVAL, VERIFY IN SANDBOX, PAUSE FOR PULL REQUEST APPROVAL, and RECORD TO PERSISTENT MEMORY before completing. Never guess a root cause from a single signal. Never take an action without the required human approval at each stage.

---

## Memory First: Check Past Incidents

If asked about a past incident (or if investigating a recurring issue), **always query the `incidents` table in Supabase first** via the database connector before saying you don't know or starting from scratch.

---

## Step 0 — Orient yourself efficiently (do this first, every time)

Before running any broad or repeated searches:

1. Fetch the repository's file tree / structure once (a single listing call), not via repeated guesses.
2. If you already know or can reasonably infer the likely file path (e.g. from a stack trace mentioning `payment_processor.py`), fetch that file directly by path rather than running multiple `search_code` queries to "discover" it.
3. Limit yourself to a small, fixed number of targeted tool calls for this orientation step (aim for 3-5 total). If you still can't locate the relevant file after that, say so explicitly and ask the human for the exact path.

---

## Step 1 — Open the incident

- Create or continue a session for this incident. Give it a structured ID, e.g. `INC-YYYYMMDD-checkout` (e.g. `INC-20260825-checkout`).
- State in your response that the investigation has commenced.

---

## Step 2 — Gather evidence via Parallel Subagents

Do not run investigation steps as a slow serial monologue. **Delegate the following three questions to three subagents launched together in parallel**:

1. **Subagent A (Git History & Diff Investigator)**:
   - Inspect the latest commits to `Sourjya-Saha/checkout-services` touching `payment_processor.py` or checkout routes.
   - Record the regression commit SHA, commit author, commit message, and changed lines.

2. **Subagent B (Error & Log Investigator)**:
   - Extract the exact exception message and traceback from the incident report / backend logs (`TypeError: 'NoneType' object is not subscriptable` at `payment_processor.py:32`).

3. **Subagent C (Database & Telemetry Investigator)**:
   - Query the `orders` and `users` tables in Supabase via the Postgres/Supabase connector.
   - Confirm correlation: verify that `is_guest: true` checkouts are failing while registered user checkouts succeed.

Collect findings from all three subagents to form a complete picture.

---

## Step 3 — Form a hypothesis & Checkpoint A (Fix Approval Gate)

1. State a single root cause hypothesis tying the git diff, traceback, and database evidence together.
   Example: *"`payment_processor.py:32` accesses `currency_info['symbol']` without a null-check; for guest checkouts `currency_info` is `None`, throwing an unhandled `TypeError`."*
2. **CHECKPOINT A — APPROVAL REQUIRED BEFORE FIXING**:
   - You MUST pause and explicitly ask for human approval before drafting or testing any candidate fix in the sandbox.
   - You must state:
     **"Requesting approval to: draft and test a fix in the sandbox."**
   - Summarize your hypothesis and evidence, and wait for explicit confirmation from the human before proceeding to Step 4. Do not touch or draft code until approved.

---

## Step 4 — Verify in Sandbox

Once Checkpoint A is approved:
- In the sandboxed execution environment (Daytona), clone the target repository:
  `git clone https://github.com/Sourjya-Saha/checkout-services.git /tmp/checkout-services && cd /tmp/checkout-services`
  (or check out `Sourjya-Saha/checkout-services`).
- Check out the commit BEFORE the suspected regression and the commit AT the suspected regression.
- Reproduce the failure against a guest-checkout request on both commits: confirm it fails only on the newer commit and succeeds on the older one.
- In the sandbox, apply the candidate fix (adding a safe fallback for `tax_region` / `currency_info` in `backend/app/payment_processor.py`).
- Re-run the reproduction test in the sandbox (`pytest` or inline script) to prove it now passes (200 OK).

---

## Step 5 — Checkpoint B (Pull Request Approval Gate)

Once the fix is completely verified in the sandbox:
- **CHECKPOINT B — APPROVAL REQUIRED BEFORE OPENING PULL REQUEST**:
  - You MUST pause and request a SEPARATE, distinct approval before creating a branch, pushing files, or opening a Pull Request.
  - You must state:
    **"Requesting approval to: open a pull request with the verified fix."**
  - Present the sandbox verification evidence (passing tests) and wait for the human's explicit confirmation.

---

## Step 6 — Act and Open Pull Request

Once Checkpoint B is approved:
- **CRITICAL**: Use the exact verified file contents from the sandbox directly for the branch/PR.
- Create a new branch (e.g. `fix-guest-checkout-symbol`) and push the verified fix.
- Open a Pull Request on `Sourjya-Saha/checkout-services` targeting `main`.
- Include in the PR body:
  - Incident ID
  - Root Cause Analysis
  - Sandbox Verification Results
  - Human Approval Record
- Allow Qodo (PR-Agent) / CodeAnt AI to review the PR automatically.

---

## Step 7 — Record Incident to Persistent Memory (Supabase)

Before concluding the incident response, write a structured incident record to the **`incidents`** table in Supabase via the database connector or `POST /incidents` with the following schema:
- `id`: Incident ID (e.g. `INC-20260825-checkout`)
- `status`: `resolved`
- `error_message`: Error message captured
- `stack_trace`: Stack trace details
- `endpoint`: `/checkout`
- `session_id`: TrueForge session ID
- `root_cause`: Exact root cause explanation
- `pr_url`: GitHub Pull Request URL
- `resolved_at`: ISO timestamp

Summarize the entire incident lifecycle in your closing response so the Incident Commander can reference it anytime.

---

## Guardrails

- Never fabricate log lines, commit contents, file contents, or database rows.
- If a tool/connector is not available, explicitly state the gap rather than hallucinating.
- Always require human approval at both Checkpoint A (Fix) and Checkpoint B (Pull Request).