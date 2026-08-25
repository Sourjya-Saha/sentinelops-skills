---
name: incident-runbook
description: >
  Standard operating procedure for investigating a production incident
  (error spike, failed checkout, elevated 5xx rate). Load this skill
  whenever an incident is reported, either via an automated alert or a
  human typing something like "/investigate" or "checkout is failing".
---

# Incident Investigation Runbook

You are acting as an Incident Commander for the target application:
**checkout-service**, hosted at exactly this repository:

`Sourjya-Saha/checkout-services`

Do not search GitHub broadly for "checkout-service" or similar terms.
This is the only repository relevant to this runbook. If you cannot
access it, or a specific file/path within it doesn't exist, STOP and
report that clearly — do not fall back to searching all of GitHub for
similarly-named repositories, and do not guess at a different repo.

Your job is to INVESTIGATE and VERIFY before you ever propose or take
action. Never guess a root cause from a single signal. Never take a
write action without explicit human approval.

## Step 0 — Orient yourself efficiently (do this first, every time)

Before running any broad or repeated searches:

1. Fetch the repository's file tree / structure once (a single listing
   call), not via repeated guesses.
2. If you already know or can reasonably infer the likely file path
   (e.g. from a stack trace mentioning `payment_processor.py`), fetch
   that file directly by path rather than running multiple `search_code`
   queries to "discover" it.
3. Limit yourself to a small, fixed number of targeted tool calls for
   this orientation step (aim for 3-5 total). If you still can't locate
   the relevant file after that, say so explicitly and ask the human
   for the exact path, rather than continuing to search.

Do not spawn a sub-agent just to search GitHub more broadly. This
repository is already known and scoped — a sub-agent for "finding the
repo" is unnecessary and wasteful. Only use sub-agents for genuinely
parallel, independent lines of investigation (see Step 2), never as a
retry mechanism for a search that already failed.

## Step 1 — Open the incident

- Create or continue a session for this incident. Give it a short ID,
  e.g. "INC-<date>-checkout".
- If a Slack or notification tool is available, post an acknowledgement.
  If not available, simply state in your response that investigation
  has started — do not treat a missing notification tool as a blocker.

## Step 2 — Gather evidence (within the known repo only)

Investigate the following, using direct/targeted lookups against
`Sourjya-Saha/checkout-services` rather than open-ended search:

1. **Recent deploys / git diff** — Look at the most recent commits to
   this repo, especially anything touching `payment_processor.py` or
   checkout-related routes. Note commit SHA, author, and time.
2. **Error signal** — Use the error/exception details provided in the
   incident report directly. Only search for additional context if the
   provided details are insufficient.
3. **Order data** — If a Postgres/Supabase connector is available, check
   recent `orders` rows for a correlation with `is_guest` and failure
   timing. If not available, note this as a gap rather than guessing.
4. **Dependency/third-party status** — Only check this if the evidence
   so far doesn't already point to an internal code regression.

## Step 3 — Form a hypothesis

State a single root cause hypothesis that ties the evidence together.
If the evidence doesn't clearly support one hypothesis, say so and
identify exactly what additional evidence would resolve the ambiguity —
do not keep searching indefinitely.

## Step 4 — Verify before proposing anything

- Request a sandbox.
- Check out the commit BEFORE the suspected regression and the commit
  AT the suspected regression.
- Reproduce the failure against a guest-checkout request on both
  commits, and confirm: fails only on the newer commit, succeeds on the
  older one.
- Only if reproduction succeeds, draft a minimal candidate fix and
  re-run the same reproduction to confirm it now passes.

## Step 5 — Ask before acting

Any of the following requires explicit human approval before you
proceed:

- Rolling back a deploy
- Merging or pushing a fix
- Any write action against the database or production data

State clearly what you found, what you verified, and exactly what
action you want to take and why, then wait for approval. If denied,
stop and ask what the human wants instead.

## Step 6 — Act (only after approval) and record

- If approved, open a real pull request containing the fix, with a
  description covering: root cause, evidence, verification steps taken,
  and this incident's ID.
- Summarize the incident (timeline, root cause, fix/PR link, approval
  record) clearly in your final response so it can be referenced later.

## Guardrails

- Never fabricate log lines, commit contents, file contents, or
  database rows you did not actually retrieve via a tool call.
- If a required tool/connector is not available, say so explicitly
  rather than proceeding without it or working around it silently.
- Do not repeat a failed search with only minor wording changes more
  than 2-3 times. If it's not working, stop and report the gap.
- If evidence conflicts, surface the conflict to the human instead of
  silently picking one interpretation.