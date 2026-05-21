# Pipeline flow

This file describes the runtime flow. The diagrams use [Mermaid](https://mermaid.js.org/),
which renders natively on GitHub/GitLab and most Markdown viewers.

## End-to-end flow

```mermaid
flowchart TD
    J[Jira ticket] --> SPEC[spec_agent]
    SPEC -->|NEEDS_CLARIFICATION| HG1{{Human gate 1:\nanswer questions}}
    HG1 -->|answers injected| SPEC
    SPEC -->|spec ready| DESIGN[design_agent]

    DESIGN --> DEVQA

    subgraph DEVQA[Dev/QA loop  max_iterations=3]
        direction TB
        DEV1[dev_agent] --> QA1[qa_agent]
        QA1 -->|tests fail| DEV1
    end

    DEVQA -->|QA exit_loop: green| REVIEW[review_agent]
    REVIEW --> SEC[security_agent]
    SEC --> PERF[performance_agent]

    PERF --> QG

    subgraph QG[Quality-gate loop  max_iterations=2]
        direction TB
        DEV2[dev_agent: fix blockers] --> GATE[quality_gate_agent]
        GATE -->|blockers remain| DEV2
    end

    QG -->|gate exit_loop: no blockers| DOCS[docs_agent]
    DOCS --> PR[pr_agent]
    PR --> HG2{{Human gate 2:\nreview & merge}}
    HG2 --> DONE([Merged PR])
```

## How a stage hands off to the next

Every stage writes one `output_key` into shared `session.state`; the next stage
reads it by name. No files are passed between agents — the session is the
artifact store.

```mermaid
flowchart LR
    A[spec_agent] -->|writes spec| S[(session.state)]
    S -->|reads spec| B[design_agent]
    B -->|writes design| S
    S -->|reads spec, design| C[dev_agent]
    C -->|write_repo_file appends| TF[touched_files in state]
```

## Loop termination

A `LoopAgent` stops when a child sets `escalate=True` (via the `exit_loop` tool)
**or** `max_iterations` is reached.

```mermaid
flowchart TD
    START[Loop iteration] --> RUN[Run sub-agents in order]
    RUN --> CHK{Child called\nexit_loop?}
    CHK -->|yes| STOP([Exit loop])
    CHK -->|no| MAX{Reached\nmax_iterations?}
    MAX -->|yes| STOP
    MAX -->|no| START
```

## Blocker routing

Review, Security, and Performance do not loop individually. They record
blockers into shared state; a single quality-gate loop afterward fixes and
re-checks them. This avoids three independent loops contending over the Dev
agent.

```mermaid
flowchart TD
    REVIEW[review_agent] -->|record_blocker| OB[(open_blockers)]
    SEC[security_agent] -->|record_blocker| OB
    PERF[performance_agent] -->|record_blocker| OB
    OB --> GATE[quality_gate_agent]
    GATE -->|empty?| EXIT[exit_loop -> docs]
    GATE -->|not empty| DEVFIX[dev_agent fixes -> clear_blockers]
    DEVFIX --> GATE
```

## Textual sequence (no rendering required)

1. Driver seeds `session.state` with `jira_ticket`.
2. **Run 1** executes `spec_agent`.
   - If output starts with `NEEDS_CLARIFICATION:` → stop, return questions
     (**human gate 1**). Human answers → inject `clarification_answers` → re-run.
   - Else continue.
3. **Run 2** executes the `build_pipeline` `SequentialAgent`:
   1. `design_agent` reads conventions + sibling code, writes `design`.
   2. **Dev/QA loop** until QA `exit_loop` or 3 iterations.
   3. `review_agent`, then `security_agent`, then `performance_agent` —
      each may `record_blocker`.
   4. **Quality-gate loop** until gate `exit_loop` or 2 iterations.
   5. `docs_agent` updates documentation.
   6. `pr_agent` assembles the PR draft and stops (**human gate 2**).
4. A human reviews and merges.
