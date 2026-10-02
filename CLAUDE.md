# CLAUDE.md — Safe Autopilot for Cloud Ops

Read `PROJECT_SPEC.md` for the full design. This file is the short version: what the project is, how to work in it, and the rules that must always hold.

## What this is

AI agents that diagnose cloud incidents and propose fixes, and can only act as far as they have earned trust. Four modules:

- **Black box recorder:** append-only event log of every prompt, tool call and decision, with replay and run diff.
- **Digital twin:** a second LocalStack instance; every fix is applied and health-checked there first.
- **Agent court:** prosecutor, defender and judge agents produce a verdict and a risk score.
- **Trust ladder:** a per-agent trust score controls what the agent may do (Level 0 suggest-only, Level 1 auto-apply low risk, Level 2 approval-gated medium risk).

Localhost first (Docker Compose + LocalStack), then AWS (ECS Fargate, RDS, IAM roles per trust level).

## Stack

Python 3.11+, FastAPI, SQLAlchemy 2 + Alembic, Postgres 16, LangGraph, boto3, Jinja2 + HTMX, pytest, ruff, Docker Compose, LocalStack. Terraform and GitHub Actions (OIDC) from Phase 6.

## Commands

```
make up        # start Postgres, LocalStack world (:4566), LocalStack twin (:4567)
make down
make test      # pytest
make lint      # ruff check + ruff format --check
make seed SCENARIO=public_bucket
make run       # start the API on :8000
```

If a command does not exist yet, add it to the Makefile as part of the task that needs it.

## Rules that must always hold

1. The LLM never executes arbitrary code. It proposes actions from the allowlisted catalog in `app/actions/catalog.py`; `executor.py` validates every action against the catalog, its parameter schema and the agent's trust level.
2. Default AWS endpoint is LocalStack. Real AWS only when `ENV=prod` and credentials are explicitly configured. Never hardcode credentials.
3. Every proposed action is applied and health-checked on the twin before it can run on the world.
4. `high` risk actions are never auto-applied, at any trust level.
5. Every state change is recorded as an event. `events` is append-only and `seq` is contiguous per run.
6. Redact secrets before writing logs or event payloads.
7. Support `DRY_RUN=true`: the executor logs and skips all writes.
8. All LLM outputs are validated with Pydantic. Retry once on invalid output, then fail the run with an `error` event.

## Conventions

- Type hints everywhere; keep functions small and testable.
- Config through `app/config.py` and environment variables; keep `.env.example` current.
- Database changes go through Alembic migrations, never manual DDL.
- Every new module gets tests. Mock the LLM in tests with fixed responses; do not call a real model in the test suite.
- Scenarios live in `scenarios/<name>/` with `setup.py`, `incident.json`, `health.py`, `expected.json`.
- Small commits with clear messages. One phase from the spec at a time.

## How to work

- Work phase by phase using the table in `PROJECT_SPEC.md` section 11 and finish a phase's acceptance criteria before starting the next.
- If a LocalStack community API a scenario needs is unsupported, say so and propose an alternative instead of silently changing the scenario.
- If a task is ambiguous or conflicts with the spec, ask before building on it.
- When a phase is done, update `README.md` with how to run and demo it.
