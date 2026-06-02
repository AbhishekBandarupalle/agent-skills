# Agentic Architecture

Two-agent system for deploying and validating code changes on Sylva OKD management clusters.

## Workflow

```mermaid
flowchart TD
    User([User]):::user -->|triggers deploy / repair / upgrade| ReadEnv

    subgraph deploy ["sylva-cluster-deploy"]
        ReadEnv[Read Environment<br/><i>cluster name, node IP, disk config</i>]
        InitSession[Init Session Context<br/><i>write goal + mode to .agent-session.md</i>]
        Diagnose[Diagnose / Fix<br/><i>monitor cluster, trace failures, edit code</i>]
        WriteCtx[Write Fix Attempt<br/><i>append problem + approach to .agent-session.md</i>]
        Commit[git add + commit<br/><i>stage and commit locally, no push</i>]
        CallCV[Call code-validate sub-agent]
        Push[git push]
        Revise[Revise or Justify<br/><i>fix code or re-submit with JUSTIFICATION</i>]
        RunScript[Run bootstrap / apply<br/><i>deploy changes via tmux</i>]

        ReadEnv --> InitSession --> Diagnose --> WriteCtx --> Commit --> CallCV
    end

    subgraph validate ["code-validate"]
        ReadShared[Read Shared Memory<br/><i>.agent-session.md + .code-validate-log.md</i>]
        ReadDiff[Read Commit + Diff<br/><i>git log + git diff</i>]
        Review[Review Against 6 Criteria<br/><i>purpose, scope, regressions, history...</i>]
        Decision{Decision}
        WriteReview[Write Review Notes<br/><i>append verdict to .agent-session.md</i>]

        ReadShared --> ReadDiff --> Review --> Decision
    end

    CallCV --> ReadShared

    Decision -->|APPROVED| WriteReview
    Decision -->|CONTRADICTION| WriteReview
    Decision -->|REJECTED| WriteReview

    WriteReview -->|APPROVED| Push
    WriteReview -->|CONTRADICTION / REJECTED| Revise

    Push --> RunScript
    RunScript -->|new failure| Diagnose
    Revise -->|re-commit| Commit

    classDef user fill:#166534,color:#fff,stroke:#22c55e
    classDef default fill:#1e293b,color:#e2e8f0,stroke:#334155
    style deploy fill:none,stroke:#3b82f6,stroke-width:2px,color:#60a5fa
    style validate fill:none,stroke:#a855f7,stroke-width:2px,color:#c084fc
```

### Shared Memory

```mermaid
flowchart LR
    subgraph files ["Shared Files in ~/sylva-core/"]
        Session[".agent-session.md<br/><i>session goal, fix attempts,<br/>review notes, decisions</i>"]
        Log[".code-validate-log.md<br/><i>clean audit trail of<br/>approved commits only</i>"]
    end

    Deploy([sylva-cluster-deploy]):::deploy
    Validate([code-validate]):::validate

    Deploy -->|writes session header,<br/>fix attempt entries| Session
    Deploy -.->|reads prior decisions<br/>+ reviewer feedback| Session
    Validate -->|writes review entries<br/>for all verdicts| Session
    Validate -.->|reads session goal<br/>+ fix context| Session
    Validate -->|writes approved<br/>commit entries| Log
    Validate -.->|reads commit history<br/>for contradiction check| Log

    classDef deploy fill:#1e3a5f,color:#93c5fd,stroke:#3b82f6
    classDef validate fill:#2e1065,color:#d8b4fe,stroke:#a855f7
    style files fill:none,stroke:#eab308,stroke-width:2px,color:#facc15
```

### Decision Outcomes

```mermaid
flowchart LR
    A[APPROVED]:::approved -->|log + push| Push[git push → run script]
    C[CONTRADICTION]:::contradiction -->|conflicting commits + detail| Choose{Deploy agent}
    Choose -->|unintentional| Revise[Revise code]
    Choose -->|intentional| Justify[Re-submit with JUSTIFICATION]
    R[REJECTED]:::rejected -->|reason + action| Fix[Reset commit, fix, re-submit]

    classDef approved fill:#166534,color:#fff,stroke:#22c55e
    classDef contradiction fill:#713f12,color:#fff,stroke:#eab308
    classDef rejected fill:#7f1d1d,color:#fff,stroke:#ef4444
```

## Agents

| Agent | Role | Skill file |
|-------|------|------------|
| **sylva-cluster-deploy** | Deploys, diagnoses, and fixes cluster issues. Commits locally and requests validation before pushing. | `sylva-cluster-deploy/SKILL.md` |
| **code-validate** | Gate-keeper that reviews every commit before push. Returns APPROVED, CONTRADICTION, or REJECTED. | `code-validate/SKILL.md` |

## Shared Memory

Both agents read and write shared files in `~/sylva-core/`:

| File | Purpose | Deploy writes | Validate writes |
|------|---------|---------------|-----------------|
| `.agent-session.md` | Session goal, fix attempts, review notes, decisions | Session header, fix attempt entries | Review entries (all verdicts with reasoning) |
| `.code-validate-log.md` | Clean audit trail of approved commits | — | Approved commit entries only |

## Review Criteria

The code-validate agent evaluates each commit against:

| # | Check | Question |
|---|-------|----------|
| 1 | Purpose alignment | Does every changed file relate to the stated issue? |
| 2 | Completeness | Does the change fully address the issue? |
| 3 | Scope limitation | Are there unrelated modifications (scope creep, debug leftovers)? |
| 4 | No regressions | Could the change break an existing unit or deployment step? |
| 5 | Commit message | Does the message accurately describe the change? |
| 6 | History consistency | Does this conflict with or revert a previous fix? |

## Decision Outcomes

**APPROVED** — Change is correct and scoped. Logged to audit trail. Deploy agent pushes and runs bootstrap/apply.

**CONTRADICTION** — Conflicts with a prior approved fix. Deploy agent either:
- **Revises** the code to avoid the conflict, or
- **Re-submits** with a `JUSTIFICATION:` explaining why the contradiction is necessary

**REJECTED** — Change has issues. Deploy agent resets the commit, fixes the code, and re-submits.

## Retry Loop

After a successful push and script run, the deploy agent resumes monitoring the cluster. If a new failure appears, it loops back through diagnosis → fix → commit → validation until all Sylva units report ready.
