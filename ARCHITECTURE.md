# Architecture

This document explains how the Jira-to-PR pipeline is built, why it is built
that way, and how data flows through it. For the visual flow, see
[`FLOW.md`](./FLOW.md). For setup and usage, see the root [`README.md`](../README.md).

## 1. Goal and scope

Turn a single Jira ticket into a production-ready pull request by passing it
through a chain of specialized agents: Spec → Design → Dev → QA → Review →
Security → Performance → (Quality gate) → Documentation → PR. A human is
involved at exactly two points — clarifying ambiguous requirements, and
reviewing/merging the final PR — and nowhere else.

This is a code-generation assistant for adding work to an **existing**
repository. It is deliberately not fully autonomous: it stops before merge.

## 2. Why Google ADK

The pipeline is a fixed, predictable sequence with two bounded retry loops.
ADK's workflow agents map onto this exactly, so we get orchestration for free
instead of hand-rolling a state machine:

| Need in this system            | ADK primitive            |
|--------------------------------|--------------------------|
| Run stages in a fixed order    | `SequentialAgent`        |
| Iterate Dev↔QA until tests pass| `LoopAgent`              |
| Each individual stage worker   | `LlmAgent`               |
| Pass data between stages       | `session.state` + `output_key` |
| Break a loop on success        | `tool_context.actions.escalate` |
| Confine file ops to the repo   | Custom function tools    |

`SequentialAgent` passes one shared `InvocationContext` (and thus shared session
state) down the chain, so a stage simply reads the keys written by earlier
stages. `LoopAgent` re-runs its sub-agents until a child escalates or
`max_iterations` is hit — this is the entire mechanism behind our feedback loops.

## 3. Module layout

```
jira_to_pr/
  config/
    settings.py          # env-driven Settings (pydantic-settings)
  tools/
    repo_tools.py        # sandboxed file ops + run_checks
    flow_tools.py        # exit_loop, record_blocker, clear_blockers
  agents/
    prompts.py           # all instruction strings (versionable separately)
    stage_agents.py      # one builder per LlmAgent, scoped tools + state
    pipeline.py          # composition into Sequential/Loop; exposes root_agent
  pipeline_runner.py     # two-run driver implementing the human gates
  cli.py                 # argparse entrypoint
docs/                    # this file, FLOW.md, conventions template
tests/                   # tool-layer unit tests (no LLM needed)
examples/                # sample ticket + clarification answers
```

The separation is intentional: **structure** (pipeline.py) is independent of
**behavior** (prompts.py) which is independent of **capability** (tools/). You
can rewrite a prompt without touching wiring, or tighten a tool's sandbox
without touching prompts.

## 4. The state contract

All inter-agent communication is through session state. Each stage writes one
`output_key`; downstream stages read it via `{key}` templating in their
instruction (a trailing `?` marks a key optional so first-pass reads don't
error). The full contract:

| Stage          | Reads (state keys)                         | Writes (`output_key`) |
|----------------|--------------------------------------------|-----------------------|
| spec           | `jira_ticket`, `clarification_answers?`    | `spec`                |
| design         | `spec`                                     | `design`              |
| dev            | `spec`, `design`, `qa_report?`, `open_blockers?` | `dev_output`    |
| qa             | `spec`, `dev_output`                       | `qa_report`           |
| review         | `design`, `dev_output`                     | `review`              |
| security       | `dev_output`                               | `security`            |
| performance    | `dev_output`                               | `perf`                |
| quality_gate   | `open_blockers?`                           | `quality_gate`        |
| docs           | `spec`, `design`                           | `docs`                |
| pr             | `spec`, `design`, `touched_files?`, `review?`, `security?`, `perf?`, `docs?` | `pr_draft` |

Two state keys are written by **tools**, not `output_key`:
- `touched_files` — appended to by `write_repo_file` every time a file changes,
  giving the PR agent an accurate manifest.
- `open_blockers` — appended to by `record_blocker` (review/security/perf) and
  cleared by `clear_blockers` (dev), driving the quality-gate loop.

## 5. Control flow and the two loops

The pipeline is split into two runnable roots because of the human gates:

- **Run 1 (`spec_root`)** is just the spec agent. If it emits the
  `NEEDS_CLARIFICATION:` token, the driver stops and returns the questions —
  this is **human gate #1**. The human's answers are injected into
  `clarification_answers` and Run 1 re-executes.
- **Run 2 (`build_root`)** is the `SequentialAgent` containing design, the two
  loops, and the review/security/perf/docs/pr stages. It ends by producing a PR
  draft and stopping — **human gate #2** (review and merge).

**Dev↔QA loop** (`LoopAgent`, `max_iterations=3` by default): Dev implements,
QA tests against the spec. QA calls `exit_loop` only when every acceptance
criterion passes; otherwise the loop repeats and Dev sees the prior `qa_report`.

**Quality-gate loop** (`LoopAgent`, `max_iterations=2`): review, security, and
performance each `record_blocker` for serious findings. The loop runs Dev (to
fix) then the quality gate (to re-check). The gate calls `exit_loop` only when
`open_blockers` is empty.

Both loops are **bounded** — on `max_iterations` exhaustion the pipeline
proceeds with the findings surfaced to the human at the PR gate rather than
looping forever. This boundedness is the single most important safety property
of the design.

## 6. Safety model

- **Filesystem sandbox.** Every path is resolved against the configured target
  repo and rejected if it escapes (`_safe_path`). Writes are limited to an
  extension allowlist. This is unit-tested independently of any LLM.
- **Least privilege per agent.** Review/Security/Performance get read-only repo
  tools (plus `record_blocker`); only Dev/QA/Docs/Refactor can write. No agent
  can push or merge.
- **Least context per agent.** Each instruction names only the state keys it
  needs, which keeps later stages from drowning in upstream artifacts and keeps
  token cost bounded.
- **Human gates.** No autonomous merge; ambiguous requirements escalate.

## 7. Production hardening (what to change before relying on it)

- Swap `InMemorySessionService` for a persistent `SessionService` (e.g.
  database-backed) so sessions survive restarts and the clarification gate can
  resume the *same* session rather than recreating it.
- Replace the textual PR draft with a real VCS integration (open an actual PR
  via the GitHub/GitLab API) — keep the human-merge gate.
- Add ADK evaluation sets to regression-test agent trajectories, not just final
  output.
- Add a `ParallelAgent` for review/security/performance if latency matters;
  they are independent reads of the same diff.
- Pin the model and add per-stage model overrides (cheap model for spec, a
  stronger one for design/dev).

## 8. Known limitations

- `output_key` holds a single string; structured handoffs (manifests, file
  lists) rely on tools writing to state rather than parsing prose.
- The Dev↔QA loop overwrites `qa_report` each iteration; long failure histories
  are not accumulated unless you add a tool to append them.
- ADK's API has shifted across releases — verify import paths and the
  `escalate`/session method signatures against your installed version.
