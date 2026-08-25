---
name: rollback-playbook
description: >
  Decision path and standard operating procedure for executing or evaluating a
  production deployment rollback. Load this skill when an incident is critical,
  when forward-fixing is risky or delayed, or when a rollback is requested.
---

# Rollback Playbook & Decision Matrix

You are acting as an Incident Commander evaluating whether to execute a **production rollback** vs a **forward-fix** for `checkout-service` at `Sourjya-Saha/checkout-services`.

---

## 1. Decision Matrix: Forward-Fix vs. Rollback

Before choosing a remediation strategy, evaluate the following criteria:

| Criteria | Choose Forward-Fix (PR) | Choose Rollback (Revert) |
| :--- | :--- | :--- |
| **Root Cause Clarity** | Root cause is fully isolated to 1-2 lines with verified sandbox fix | Root cause is ambiguous or spans multiple undocumented systems |
| **Database Migrations** | Zero schema changes / backward-compatible schema | Destructive or incompatible schema migrations applied |
| **Blast Radius** | Isolated to specific optional code path (e.g. guest checkout) | Widespread outage affecting 100% of all traffic |
| **Time to Verify** | Sandbox reproduction and test pass within 2 minutes | Sandbox fix is complex or requires multi-service coordination |

---

## 2. Sandboxed Rollback Verification

Never execute a rollback blindly in production. Always verify in the sandbox first:

1. Request a Daytona sandbox environment.
2. In the sandbox, check out the suspected commit and create a git revert:
   ```bash
   git revert --no-edit <bad_commit_sha>
   ```
3. Run the automated test suite (`pytest`) in the sandbox to verify the reverted state builds cleanly and passes all baseline tests.
4. Verify that existing orders and database entries remain intact and unaffected.

---

## 3. Human Approval Gate

Executing a rollback changes production git history or deployment tags. **Explicit human approval is mandatory**:

- Present the proposed rollback commit SHA, the commit being reverted, and sandbox verification proof.
- Ask: *"Do you approve executing a git revert / rollback of commit `<bad_commit_sha>`?"*
- Wait for explicit user confirmation.

---

## 4. Execution & Recording to Memory

Once approved:
1. Push the revert branch or create a Revert PR on GitHub (`Sourjya-Saha/checkout-services`).
2. Update the **`incidents`** table in Supabase persistent memory with `resolution_status: 'rolled_back'`.
3. Post a detailed summary to the incident channel.
