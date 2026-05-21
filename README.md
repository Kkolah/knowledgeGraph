# Jira-to-PR

A multi-agent pipeline built on the **Google Agent Development Kit (ADK)** that
turns a Jira ticket into a production-ready pull request. It chains specialized
agents — Spec → Design → Dev → QA → Review → Security → Performance →
Documentation → PR — with two human checkpoints: clarifying ambiguous
requirements, and reviewing/merging the final PR.

It is designed for **adding work to an existing repository**: agents read your
repo's own conventions and source files so generated code matches your patterns
instead of inventing a new architecture.

- **Architecture & rationale:** [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md)
- **Flow diagrams:** [`docs/FLOW.md`](docs/FLOW.md)
- **Conventions file template (for your target repo):** [`docs/CONVENTIONS_TEMPLATE.md`](docs/CONVENTIONS_TEMPLATE.md)

## How it works (in one paragraph)

The pipeline is composed of ADK workflow agents. A `SequentialAgent` runs the
stages in order; two `LoopAgent`s provide bounded feedback loops (Dev↔QA until
tests pass, and a quality gate that fixes review/security/performance blockers).
Agents communicate only through `session.state`: each writes one `output_key`
and the next reads it. File access is sandboxed to the configured target repo.
No agent can merge — the pipeline stops with a PR draft for a human.

## Requirements

- Python 3.10+
- A Google Gemini API key (AI Studio) **or** Vertex AI access
- A target repository containing a conventions file (default `AGENTS.md`)

## Install

```bash
git clone <this-repo> && cd jira-to-pr
python -m venv .venv && source .venv/bin/activate
pip install -e ".[dev]"
cp .env.example .env       # then edit .env with your API key + repo path
```

## Configure

All settings are environment variables prefixed `J2PR_` (see
`jira_to_pr/config/settings.py`). The essentials:

| Variable                   | Meaning                                   | Default          |
|----------------------------|-------------------------------------------|------------------|
| `GOOGLE_API_KEY`           | Gemini API key (or use Vertex vars)       | —                |
| `J2PR_MODEL`               | Model for all agents                      | `gemini-2.5-flash` |
| `J2PR_TARGET_REPO_PATH`    | Path to the repo being modified           | `./target_repo`  |
| `J2PR_CONVENTIONS_FILENAME`| Conventions doc name in the target repo   | `AGENTS.md`      |
| `J2PR_DEV_QA_MAX_ITERATIONS` | Dev↔QA loop ceiling                     | `3`              |
| `J2PR_QUALITY_GATE_MAX_ITERATIONS` | Quality-gate loop ceiling         | `2`              |

Put a conventions file at the **root of your target repo**. Start from
[`docs/CONVENTIONS_TEMPLATE.md`](docs/CONVENTIONS_TEMPLATE.md) — the richer it
is, the less the agents improvise.

## Run

### CLI

```bash
# Run a ticket (from a file)
python -m jira_to_pr.cli --ticket-id PROJ-123 \
    --ticket-file examples/ticket_PROJ-123.md --verbose
```

If the spec agent needs clarification, it prints questions and exits with code
`2`. Answer them in a file and resume:

```bash
python -m jira_to_pr.cli --ticket-id PROJ-123 \
    --ticket-file examples/ticket_PROJ-123.md \
    --answers-file examples/clarification_answers_PROJ-123.md --verbose
```

On success it prints the spec and the PR draft (ending in
`AWAITING HUMAN REVIEW AND MERGE.`).

### ADK Dev UI (recommended while developing)

The build pipeline exposes a module-level `root_agent`, so you can inspect every
agent, tool call, and state mutation step-by-step:

```bash
adk web
```

Then select the `jira_to_pr` agent. (Seed `jira_ticket` in the session state to
exercise the build pipeline; the clarification gate is driven by the CLI.)

## Project layout

```
jira_to_pr/
  config/settings.py       Environment-driven configuration
  tools/repo_tools.py      Sandboxed file read/write/list + run_checks
  tools/flow_tools.py      exit_loop, record/clear blockers
  agents/prompts.py        All instruction text
  agents/stage_agents.py   One builder per LlmAgent (scoped tools + state)
  agents/pipeline.py       Sequential/Loop composition; root_agent
  pipeline_runner.py       Two-run driver implementing the human gates
  cli.py                   Command-line entrypoint
docs/                      ARCHITECTURE.md, FLOW.md, conventions template
tests/                     Tool-layer unit tests (no LLM required)
examples/                  Sample ticket + clarification answers
```

## Test

```bash
pytest -q          # tool sandbox + flow-tool tests (no API key needed)
ruff check .       # lint
```

## Safety notes

- File operations are confined to `J2PR_TARGET_REPO_PATH`; traversal is
  rejected and writes are limited to an extension allowlist.
- Review/Security/Performance agents are read-only (plus blocker recording).
- The pipeline never merges; a human reviews the PR draft.
- Loops are bounded by `max_iterations`; they cannot run forever.

## Limitations & next steps

This is a working foundation, not a turnkey production system. See
[`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) §7–§8 for hardening (persistent
sessions, real VCS/PR integration, ADK eval sets, parallelizing the
review/security/performance stages) and known limitations. Verify ADK import
paths and method signatures against your installed ADK version, as the API has
changed across releases.

## License

Apache-2.0.
