# SentinelOps Agent Skills: Autonomous Runbooks & Playbooks

[![TrueForge](https://img.shields.io/badge/TrueForge-Agent_Skills-DC2626)](https://truefoundry.com/)
[![Daytona](https://img.shields.io/badge/Daytona-Isolated_Sandbox-000000?logo=linux)](https://daytona.io/)
[![GitHub MCP](https://img.shields.io/badge/MCP-GitHub_Connector-181717?logo=github)](https://modelcontextprotocol.io/)
[![Supabase MCP](https://img.shields.io/badge/MCP-Supabase_PostgreSQL-3ECF8E?logo=supabase)](https://supabase.com/)
[![Qodo AI](https://img.shields.io/badge/Qodo_AI-PR_Review_Verified-10B981)](https://qodo.ai/)

> This repository contains the **TrueForge Agent Skills** and standard operating procedures (SOPs) for **SentinelOps**, an autonomous SRE incident response agent designed for enterprise production microservice ecosystems.

---

## Table of Contents
1. [Agent Skills Architecture](#1-agent-skills-architecture)
2. [Skill 1: Incident Runbook (`incident-runbook`)](#2-skill-1-incident-runbook-incident-runbook)
3. [Skill 2: Rollback Playbook (`rollback-playbook`)](#3-skill-2-rollback-playbook-rollback-playbook)
4. [Sandbox Isolation vs. GitHub MCP Boundary](#4-sandbox-isolation-vs-github-mcp-boundary)
5. [Two-Stage Human-in-the-Loop Approval Protocol](#5-two-stage-human-in-the-loop-approval-protocol)
6. [Visual Evidence & HUD Execution Stream](#6-visual-evidence--hud-execution-stream)
7. [TrueForge Agent Configuration (`agent.yaml` & `manifest.json`)](#7-trueforge-agent-configuration-agentyaml--manifestjson)
8. [Installation & Deployment](#8-installation--deployment)

---

## 1. Agent Skills Architecture

```mermaid
flowchart TD
    subgraph TriggerLayer ["1. INCIDENT INGESTION"]
        Alert["Production Alert / Error Spike<br/>(TypeError: float and dict)"] --> Dispatcher["TrueForge Event Router"]
    end

    subgraph SkillSelection ["2. SKILL ACTIVATION"]
        Dispatcher -->|Matches SRE Intent| Runbook["incident-runbook (SKILL.md)"]
        Dispatcher -->|Matches Rollback Intent| Playbook["rollback-playbook (SKILL.md)"]
    end

    subgraph SwarmExecution ["3. PARALLEL SUBAGENT SWARM"]
        Runbook -->|Step 2: Launch in Parallel| SubAlpha["Subagent Alpha<br/>(Git Commits & Branch Diffs)"]
        Runbook -->|Step 2: Launch in Parallel| SubBravo["Subagent Bravo<br/>(Exception Logs & Tracebacks)"]
        Runbook -->|Step 2: Launch in Parallel| SubCharlie["Subagent Charlie<br/>(Supabase DB Orders Correlation)"]
        
        SubAlpha --> Hypothesis["Single Root-Cause Hypothesis"]
        SubBravo --> Hypothesis
        SubCharlie --> Hypothesis
    end

    subgraph Gate1 ["4. CHECKPOINT A (APPROVAL GATE 1)"]
        Hypothesis --> CheckpointA{"CHECKPOINT A<br/>Human Approval to Reproduce & Fix"}
    end

    subgraph DaytonaIsolation ["5. DAYTONA LINUX SANDBOX (ISOLATED)"]
        CheckpointA -->|Approved by Commander| Daytona["Daytona Linux MicroVM<br/>- 1. Clone working copy<br/>- 2. pip install requirements<br/>- 3. Actively Reproduce Bug in Sandbox<br/>- 4. Apply candidate patch<br/>- 5. Re-run pytest backend/tests"]
        Daytona -->|All 9 Tests Pass| Proof["Sandbox Verification Proof (100% OK)"]
    end

    subgraph Gate2 ["6. CHECKPOINT B (APPROVAL GATE 2)"]
        Proof --> CheckpointB{"CHECKPOINT B<br/>Human Approval to Open GitHub PR"}
    end

    subgraph GitHubAndQodo ["7. GITHUB MCP & PERSISTENT MEMORY"]
        CheckpointB -->|Approved by Commander| GitHubMCP["TrueForge GitHub MCP Connector<br/>push_files + create_pull_request"]
        GitHubMCP --> QodoReview["Qodo AI Automated PR Review"]
        QodoReview --> SupabaseMemory[("Supabase PostgreSQL DB<br/>Table: incidents")]
    end

    classDef redGate fill:#fee2e2,stroke:#dc2626,stroke-width:2px,color:#991b1b;
    classDef darkNode fill:#18181b,stroke:#ffffff,stroke-width:2px,color:#ffffff;
    classDef greenProof fill:#dcfce7,stroke:#16a34a,stroke-width:2px,color:#166534;

    class CheckpointA,CheckpointB redGate;
    class Runbook,Playbook,Daytona,GitHubMCP darkNode;
    class Proof greenProof;
```

---

## 2. Skill 1: Incident Runbook (`incident-runbook`)

* **Path:** [`skills/incident-runbook/SKILL.md`](skills/incident-runbook/SKILL.md)
* **Target Repository:** `Sourjya-Saha/checkout-services`
* **Trigger:** Production error spikes, regional tax calculation exceptions, failed checkouts, or `/investigate` slash commands.

### Execution Lifecycle:

#### Step 0 — Target Isolation & Orientation
* Locks the investigation scope strictly to `Sourjya-Saha/checkout-services`.
* Extracts the impacted code files from the exception traceback (`backend/app/payment_processor.py`).

#### Step 1 — Session Opening & Incident Tagging
* Generates a unique incident ID format (e.g. `INC-20260826-checkout-500`).
* Emits real-time SSE telemetry to the SentinelOps Command HUD.

#### Step 2 — Parallel Subagent Swarm Evidence Gathering
The commander launches three specialized subagents simultaneously:
1. **Subagent Alpha (`GIT-SENTINEL`)**: Interrogates recent commits on `main` via GitHub MCP tools (`list_commits`, `get_commit`) and isolates commit `416e972`.
2. **Subagent Bravo (`LOG-TRACE`)**: Parses exception tracebacks and isolates runtime stack frame line 94 (`TypeError: unsupported operand type(s) for *: 'float' and 'dict'`).
3. **Subagent Charlie (`DATA-CORE`)**: Queries Supabase PostgreSQL `orders` and `users` tables to correlate failed transactions (`tax_region="US_CA"`).

#### Step 3 — Hypothesis Formulation & Checkpoint A
* Synthesizes findings into a unified root-cause hypothesis.
* **PAUSES EXECUTION for Checkpoint A**:
  > *"Requesting approval to: draft and test a fix in the sandbox."*

#### Step 4 — Sandbox Reproduction, Fix & Verification
Once Checkpoint A is approved:
1. **Clones working copy** into `/tmp/checkout-services` inside the Daytona Linux MicroVM.
2. **Installs dependencies**: `pip install -r backend/requirements.txt`.
3. **Actively reproduces the bug in the sandbox**: Runs `pytest backend/tests/test_checkout.py -k tax` to verify reproduction of the `TypeError` exception in the isolated environment.
4. **Applies candidate patch** in `payment_processor.py` to safely extract `.get("rate")` if the rate mapping is a structured dictionary.
5. **Verifies fix**: Executes `pytest backend/tests` to prove all 9 unit tests pass (100% OK).

#### Step 5 — Checkpoint B (PR Gate)
* **PAUSES EXECUTION for Checkpoint B**:
  > *"Requesting approval to: open a pull request with the verified fix."*

#### Step 6 — Act and Open GitHub Pull Request
Once Checkpoint B is approved:
* Directly invokes GitHub MCP `push_files` to branch `fix-regional-tax-rate-dict`.
* Invokes GitHub MCP `create_pull_request` referencing root cause, Daytona sandbox reproduction & fix proof, and verification logs.

#### Step 7 — Structured Postmortem Commitment
* Commits a standardized JSON postmortem record into the Supabase PostgreSQL `incidents` table.

---

## 3. Skill 2: Rollback Playbook (`rollback-playbook`)

* **Path:** [`skills/rollback-playbook/SKILL.md`](skills/rollback-playbook/SKILL.md)
* **Trigger:** Critical canary regression, deployment catastrophic failure, or `/rollback` command.

### Key Capabilities:
* Detects breaking merge commits within the last 3 release cycles.
* Synthesizes safe git revert commits without destructive force-pushes.
* Verifies pre-revert vs. post-revert unit test suites in Daytona before executing GitHub PR creation.

---

## 4. Sandbox Isolation vs. GitHub MCP Boundary

SentinelOps enforces a **strict physical separation** between sandbox compute and GitHub credentials:

```text
┌────────────────────────────────────────────────────────────────────────┐
│                        SENTINELOPS CONTROLLER                          │
├───────────────────────────────────┬────────────────────────────────────┤
│   DAYTONA LINUX SANDBOX           │   TRUEFORGE GITHUB MCP CONNECTOR   │
│   (Isolated Ephemeral Compute)    │   (Authenticated API Channel)      │
├───────────────────────────────────┼────────────────────────────────────┤
│ • Local working copy (/tmp)       │ • GitHub API Token Injection       │
│ • NO GitHub credentials injected  │ • Inspects commits & branch diffs  │
│ • Actively reproduces bug & tests │ • Pushes candidate file diffs      │
│ • Zero outbound push permissions  │ • Opens official Pull Requests     │
└───────────────────────────────────┴────────────────────────────────────┘
```

---

## 5. Two-Stage Human-in-the-Loop Approval Protocol

```text
                                  INCIDENT DETECTED
                                          │
                                          ▼
                             [Parallel Subagent Swarm]
                                          │
                         ┌────────────────┴────────────────┐
                         ▼                                 ▼
               [Hypothesis Formed]                [Target Isolated]
                         │
                         ▼
        ╔══════════════════════════════════════════════════════════╗
        ║  CHECKPOINT A // APPROVAL TO REPRODUCE & FIX IN SANDBOX  ║
        ║  (Presents: [TARGET REPO], [TARGET ERROR], [ACTION])     ║
        ╚══════════════════════════════════════════════════════════╝
                         │
                         ├─────────────────────────────┐
                         │ [DENY]                      │ [APPROVE]
                         ▼                             ▼
                  [Abort Runbook]            [Daytona Linux Sandbox]
                                             1. pip install deps
                                             2. Actively reproduce bug
                                             3. Apply candidate fix
                                             4. Run pytest suite (9/9 pass)
                                                       │
                                                       ▼
        ╔══════════════════════════════════════════════════════════╗
        ║  CHECKPOINT B // APPROVAL TO OPEN GITHUB PULL REQUEST    ║
        ║  (Presents: 100% Passed Sandbox Test Execution Logs)     ║
        ╚══════════════════════════════════════════════════════════╝
                         │
                         ├─────────────────────────────┐
                         │ [DENY]                      │ [APPROVE]
                         ▼                             ▼
                  [Abort Runbook]            [GitHub MCP PR Creation]
                                                       │
                                                       ▼
                                             [Qodo AI Code Review]
                                                       │
                                                       ▼
                                            [Supabase Postmortem DB]
```

---

## 6. Visual Evidence & HUD Execution Stream

### 1. SentinelOps Interactive Landing Experience
![SentinelOps Landing Experience](docs/landingpage.png)

---

### 2. Autonomous SRE Command Center Telemetry
![SentinelOps Telemetry HUD](docs/sentinleops_hub.png)

---

### 3. Live Approval Gates & Daytona Terminal Stream
![Two-Stage Approval Gates](docs/sentinleops_hub_2.png)

---

### 4. Interactive Human-in-the-Loop Approval in TrueForge
![Human-in-the-Loop Approval](docs/HITL.png)

---

### 5. Automated Pull Request Creation via GitHub MCP
![Pull Request Creation](docs/pr_req1.png)

---

### 6. Qodo AI Automated Code Review & Analysis
![Qodo AI Code Review](docs/pr_req2.png)

---

### 7. Supabase Persistent Memory Ledger
![Postmortem Incident Ledger](docs/sentinleops_incident.png)

---

### 8. TrueForge Agent Harness & Daytona MicroVM Instances
![TrueForge Runtime Interface](docs/trueforge_1.png)
![TrueForge Sandbox Compute](docs/trueforge_2.png)

---

## 7. TrueForge Agent Configuration (`agent.yaml` & `manifest.json`)

### `agent.yaml`
```yaml
name: sentinelops
display_name: SentinelOps Autonomous Incident Commander
version: "1.0.0"
model: gpt-4o

system_prompt: |
  You are SentinelOps, the Autonomous Incident Commander for production microservices.
  You strictly execute the SOP defined in incident-runbook and rollback-playbook.
  Never bypass Checkpoint A or Checkpoint B.

runtime:
  type: trueforge
  sandbox:
    provider: daytona
    image: python:3.11-slim
    shell: /bin/sh
    default_shell: sh
    working_directory: /tmp
    timeout_seconds: 300
    env:
      PATH: "/usr/local/bin:/usr/bin:/bin"
      PYTHONUNBUFFERED: "1"

skills:
  - name: incident-runbook
    path: ./skills/incident-runbook/SKILL.md
  - name: rollback-playbook
    path: ./skills/rollback-playbook/SKILL.md

mcp_servers:
  - name: github
    transport: stdio
    command: npx
    args: ["-y", "@modelcontextprotocol/server-github"]
    env:
      GITHUB_PERSONAL_ACCESS_TOKEN: "${GITHUB_TOKEN}"

  - name: postgres
    transport: stdio
    command: npx
    args: ["-y", "@modelcontextprotocol/server-postgres"]
    env:
      POSTGRES_CONNECTION_STRING: "${DATABASE_URL}"
```

### `manifest.json`
```json
{
  "name": "sentinelops",
  "version": "1.0.0",
  "display_name": "SentinelOps Autonomous Incident Commander",
  "description": "Autonomous SRE agent that investigates, sandboxes, verifies, and remediates production microservice outages with Two-Stage HITL approval gates and persistent postmortem memory.",
  "runtime": {
    "type": "trueforge",
    "model": "gpt-4o",
    "system_prompt": "You are SentinelOps, the Autonomous Incident Commander for production microservices. You strictly follow the SOP defined in incident-runbook and rollback-playbook. Always enforce Checkpoint A and Checkpoint B approval gates.",
    "sandbox": {
      "provider": "daytona",
      "image": "python:3.11-slim",
      "shell": "/bin/sh",
      "default_shell": "sh",
      "working_directory": "/tmp",
      "timeout_seconds": 300,
      "env": {
        "PATH": "/usr/local/bin:/usr/bin:/bin",
        "PYTHONUNBUFFERED": "1"
      }
    }
  },
  "skills": [
    {
      "name": "incident-runbook",
      "path": "./skills/incident-runbook/SKILL.md"
    },
    {
      "name": "rollback-playbook",
      "path": "./skills/rollback-playbook/SKILL.md"
    }
  ],
  "mcp_servers": [
    {
      "name": "github",
      "transport": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "${GITHUB_TOKEN}"
      }
    },
    {
      "name": "postgres",
      "transport": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"],
      "env": {
        "POSTGRES_CONNECTION_STRING": "${DATABASE_URL}"
      }
    }
  ]
}
```

---

## 8. Installation & Deployment

### Adding Skills to TrueForge Runtime
```bash
# Clone the skills repository
git clone https://github.com/Sourjya-Saha/sentinelops-skills.git

# Validate skill structure
trueforge skills validate ./skills/incident-runbook
trueforge skills validate ./skills/rollback-playbook

# Deploy agent with skills loaded
trueforge agent deploy --config agent.yaml
```
