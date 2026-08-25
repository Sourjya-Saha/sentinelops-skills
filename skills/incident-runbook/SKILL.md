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

**Architectural rule: the sandbox and GitHub are never the same channel.**
The sandbox is for local verification only (clone a working copy, run code,
run tests). It has no GitHub credentials injected into it. All reads from
and writes to the real GitHub repository — inspecting commits, pushing
files, opening a PR — happen exclusively through the GitHub MCP connector
(`call_tool` with `mcp_server: "github"`). Never attempt `git push`,
`git commit` with a remote, or any credential-based git operation from
inside the sandbox shell.

---

## Step 0 — Orient yourself efficiently

1. Fetch the repository structure once from `Sourjya-Saha/checkout-services` via GitHub MCP tools, or clone a local working copy into `/tmp/checkout-services` for sandbox verification only.
2. Inspect the file referenced in the incident stack trace directly.

---

## Step 1 — Open the incident

- Create or continue a session for this incident. Generate a dynamic ID (e.g. `INC-YYYYMMDD-<issue>`).
- State in your response that the investigation has commenced.

---

## Step 2 — Gather evidence via Parallel Subagents

**Delegate the following three questions to three subagents launched together in parallel**:

1. **Subagent A (Git History & Diff Investigator)**: Inspect recent commits to `Sourjya-Saha/checkout-services` on `main` via GitHub MCP tools (`list_commits` / `get_commit`) to identify the regression.
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
- In the sandbox, clone a local working copy for verification only:
  `sh -c "git clone https://github.com/Sourjya-Saha/checkout-services.git /tmp/checkout-services && cd /tmp/checkout-services"`
- Install dependencies before running tests: `sh -c "pip install -r backend/requirements.txt"`
- Dynamically determine and apply the safe fix for the root cause, entirely within this local sandbox copy.
- Run Python verification / pytest to prove it passes (200 OK): `sh -c "pytest backend/tests"`
- Once verified, capture the final file contents to be committed via the GitHub MCP connector in Step 6.

---

## Step 5 — CHECKPOINT B (Approval Gate 2 of 2)

Once the fix is verified:
- **CHECKPOINT B — APPROVAL REQUIRED BEFORE OPENING PULL REQUEST**:
  - You MUST pause and ask:
     **"Requesting approval to: open a pull request with the verified fix."**
  - Present the verified fix proof and wait for explicit human confirmation.

---

## Step 6 — Act and Open Pull Request

Once Checkpoint B is approved, create the branch, push the verified files, and open the Pull Request using the TrueForge GitHub MCP connector (`call_tool` with `mcp_server: "github"`):

1. **Generate a descriptive branch name** derived from the root cause (e.g. `fix-<issue-slug>`).
2. **Push the modified files to GitHub**:
   - Call `call_tool` with `mcp_server: "github"`, `tool_name: "push_files"` (or `create_or_update_file`):
     - `owner`: `"Sourjya-Saha"`
     - `repo`: `"checkout-services"`
     - `branch`: the generated branch name
     - `message`: commit message summarizing the verified fix
     - `files`: array of `{ path: "<file-path>", content: "<verified-content>" }` for all modified files
3. **Create the Pull Request**:
   - Call `call_tool` with `mcp_server: "github"`, `tool_name: "create_pull_request"`:
     - `owner`: `"Sourjya-Saha"`
     - `repo`: `"checkout-services"`
     - `title`: concise title describing the fix
     - `head`: the branch name used above
     - `base`: `"main"`
     - `body`: structured PR description with Incident ID, Root Cause Analysis, Verification Results, and Approval Record
4. Extract the created PR URL from the tool output (`html_url` or `url`).
5. Proceed directly to Step 7 without asking for additional approvals.

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
   - `pr_url`: The real PR URL returned by `create_pull_request`
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

---

## Guardrails

- Never fabricate a PR URL. Only use the real URL returned by the `create_pull_request` tool call.
- Never attempt GitHub authentication from inside the sandbox shell.
- Never fabricate log lines, commit contents, file contents, or database rows.
- If a tool/connector is not available, explicitly state the gap rather than hallucinating, and do not retry the same failing action more than once without new information.