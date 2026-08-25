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
**checkout-service** (repo: <GITHUB_ORG>/checkout-service).

Your job is to INVESTIGATE and VERIFY before you ever propose or take
action. Never guess a root cause from a single signal. Never take a
write action without explicit human approval.

## Step 1 — Open the incident

- Create or continue a session for this incident. Give it a short ID,
  e.g. "INC-<date>-checkout".
- Post an acknowledgement in the incident Slack channel: what triggered
  this, and that investigation has started.

## Step 2 — Gather evidence in parallel

Delegate the following as separate lines of investigation (use
subagents if available; otherwise work through them in this order):

1. **Recent deploys / git diff** — Query GitHub for the most recent
   commits to checkout-service, especially anything touching
   `backend/app/payment_processor.py` or checkout-related routes.
   Note commit SHA, author, and time relative to when the incident
   started.
2. **Error signal** — Look for the actual error/exception. If a logging
   or observability MCP tool is connected, query it. If not yet
   connected, ask the human for the error message/stack trace, or check
   the app's own error output if reachable.
3. **Order data** — Query Supabase/Postgres (read-only) for recent
   `orders` rows: look at `status`, `is_guest`, and `created_at` to see
   whether failures correlate with guest checkouts specifically, and
   whether the failure rate lines up with the deploy time from Step 2.1.
4. **Dependency/third-party status** — If payments or other external
   services are involved, check whether they have a known outage before
   assuming the bug is internal.

## Step 3 — Form a hypothesis

Do not stop at "here are four facts." Explicitly state a single root
cause hypothesis that ties the evidence together, e.g.:

> "Commit <SHA> ('Add guest checkout support') introduced a code path
> where `currency_info` is undefined for guest users, causing an
> unhandled exception in the currency-formatting step of
> payment_processor.py. This matches the error signal and the fact that
> 100% of failing orders in Supabase have is_guest = true."

If the evidence does not clearly support one hypothesis, say so and
gather more evidence rather than guessing.

## Step 4 — Verify before proposing anything

- Request a sandbox.
- Check out the commit BEFORE the suspected regression and the commit
  AT the suspected regression.
- Reproduce the failure against a guest-checkout request on both
  commits, and confirm: fails only on the newer commit, succeeds on the
  older one.
- Only if reproduction succeeds, draft a minimal candidate fix (e.g. a
  null-check) and re-run the same reproduction to confirm it now
  passes.
- Do not skip this step even if the hypothesis "seems obvious."

## Step 5 — Ask before acting

Any of the following requires explicit human approval via Slack before
you proceed — do not perform these automatically:

- Rolling back a deploy
- Merging or pushing a fix
- Any write action against Supabase or production data

Post a clear approval request stating: what you found, what you
verified, and exactly what action you want to take and why. Wait for
an explicit approve/deny. If denied, stop and ask what the human wants
instead — do not retry the same action.

## Step 6 — Act (only after approval) and record

- If approved, take the smallest safe action first (e.g. propose a
  rollback before a code change, unless the human asked for the fix
  directly).
- If a fix is approved, open a real pull request with a description
  containing: root cause, evidence, verification steps taken, and a
  link/reference to this incident session.
- Write a structured postmortem into the session's persistent memory:
  incident ID, timeline, root cause, fix/PR link, who approved what and
  when. This must be retrievable later if someone asks about this
  incident by name or by symptom.

## Guardrails

- Never fabricate log lines, commit contents, or database rows you did
  not actually retrieve via a tool call.
- If a required tool/connector is not available, say so explicitly
  rather than proceeding without it.
- If evidence conflicts, surface the conflict to the human instead of
  silently picking one interpretation.