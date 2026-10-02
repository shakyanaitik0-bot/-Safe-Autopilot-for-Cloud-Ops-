# Safe Autopilot for Cloud Ops — Project Spec

AI agents that diagnose cloud incidents and propose fixes, but can only act as far as they have earned trust. Built localhost-first (Docker Compose + LocalStack), then deployed to AWS.

> Assumptions: 2-month build, about 15-20 hours/week, comfortable with Python. If the 1-month version is chosen, treat Phases 1-5 as the deliverable and Phase 6-7 (AWS) as stretch.

---

## 1. Core ideas

| Module | What it does |
|---|---|
| **Black box recorder** | Logs every prompt, LLM response, tool call, decision and cost as ordered events. Supports step-by-step replay and diffing two runs. |
| **Digital twin** | A second, disposable LocalStack instance rebuilt from the scenario definition. Every proposed fix is applied there first and health-checked before touching the "real" environment. |
| **Agent court** | Prosecutor (argues the change is dangerous), defender (argues it is safe), judge (verdict + risk score). Transcript is stored with the proposal. |
| **Trust ladder** | Each agent has a trust score. Low trust = suggest-only. Higher trust unlocks auto-apply of low-risk actions, then approval-gated medium-risk actions. Mistakes lower the score. |

**Key design decision:** the LLM never executes arbitrary code or commands. It proposes actions from an **allowlisted action catalog** (typed boto3 operations). The executor validates each proposal against the catalog and the agent's current trust level before running anything.

---

## 2. Scope for the 2-month build

**In scope**
- 3 scripted incident scenarios (below)
- One LLM provider at a time, switchable by env var (Anthropic API, Bedrock or Ollama)
- Twin = second LocalStack container rebuilt from the scenario's `setup.py`
- Approvals through a page in our own web UI (no Slack)
- Boto3 allowlisted actions (Terraform-generated fixes are a stretch goal)
- Production on AWS: ECS Fargate, RDS Postgres, 2 IAM roles

**Out of scope (for now)**
- Generic "any incident" support
- Separate sandbox AWS account for the twin in production
- Slack approvals, OpenTelemetry, multi-cloud
- A React frontend (use FastAPI + Jinja2 + HTMX)

---

## 3. Tech stack

- Python 3.11+, FastAPI, Uvicorn
- SQLAlchemy 2.x + Alembic, Postgres 16 (SQLite allowed in unit tests)
- LangGraph for the agent workflow; a thin `llm/provider.py` abstraction over the model API
- boto3 for all AWS calls (endpoint overridable to LocalStack)
- Jinja2 + HTMX for the replay and approval pages
- Docker Compose, LocalStack (community), Makefile
- pytest, ruff
- Phase 6+: Terraform (use `tflocal` against LocalStack first), GitHub Actions with OIDC

---

## 4. Architecture

```
                 ┌──────────────────────────────────────────────┐
                 │                 FastAPI app                  │
 incident ─────► │  /incidents  /runs  /proposals  /agents      │
 (seed script    └───────┬──────────────────────────────┬───────┘
  or webhook)            │                              │
                         ▼                              ▼
                ┌─────────────────┐            ┌─────────────────┐
                │ LangGraph agent │            │  Web UI (HTMX)  │
                │  workflow       │            │ replay/approvals│
                └───┬─────────┬───┘            └─────────────────┘
                    │         │
        ┌───────────▼──┐   ┌──▼───────────┐     ┌──────────────────┐
        │ Black box    │   │ Action       │     │ Postgres         │
        │ recorder     │──►│ catalog +    │     │ incidents, runs, │
        └──────────────┘   │ executor     │     │ events, proposals│
                           └──┬────────┬──┘     │ agents, trust    │
                              │        │        └──────────────────┘
                   ┌──────────▼─┐  ┌───▼──────────┐
                   │ Twin       │  │ "World"      │
                   │ LocalStack │  │ LocalStack   │
                   │ :4567      │  │ :4566        │
                   └────────────┘  └──────────────┘
```

**Agent workflow (LangGraph nodes)**

```
ingest → diagnose → propose → twin_test → court → policy_gate
      → (suggest_only | await_approval | auto_apply) → verify → record_outcome → update_trust
```

**Policy gate rule.** An action is auto-applied only if all are true:
1. The agent's trust level allows the action's risk class
2. The twin health checks passed
3. The court verdict is `approve`
4. The judge's `risk_score` is at or below the threshold (default 40)

Otherwise it is routed to `await_approval` (if the level allows) or `suggest_only`.

---

## 5. Folder structure

```
safe-autopilot/
├── CLAUDE.md
├── PROJECT_SPEC.md
├── README.md
├── Makefile
├── docker-compose.yml
├── .env.example
├── pyproject.toml
├── app/
│   ├── main.py
│   ├── config.py
│   ├── db.py
│   ├── models.py
│   ├── api/            incidents.py, runs.py, proposals.py, agents.py
│   ├── agents/         graph.py, responder.py, court.py, prompts/
│   ├── blackbox/       recorder.py, replay.py, diff.py
│   ├── twin/           manager.py, health.py
│   ├── actions/        catalog.py, executor.py
│   ├── trust/          scoring.py, policy.py
│   ├── llm/            provider.py
│   └── web/            templates/, static/
├── scenarios/
│   ├── public_bucket/        setup.py, incident.json, health.py, expected.json
│   ├── sqs_backlog/          (same files)
│   └── open_security_group/  (same files)
├── tests/
├── scripts/            seed_scenario.py, reset_world.py
└── infra/              Terraform (Phase 6)
```

---

## 6. Data model

| Table | Key columns |
|---|---|
| `incidents` | id, scenario, source, payload (JSON), status, created_at |
| `runs` | id, incident_id, agent_id, status, trust_level_at_start, started_at, ended_at |
| `events` (black box) | id, run_id, seq, type, payload (JSON), tokens_in, tokens_out, cost_usd, created_at |
| `proposals` | id, run_id, actions (JSON), risk_class, rationale, twin_passed, court_verdict, court_risk_score, status, created_at |
| `human_decisions` | id, proposal_id, decision, edited (bool), note, created_at |
| `agents` | id, name, trust_score, trust_level, updated_at |
| `trust_events` | id, agent_id, delta, reason, proposal_id, created_at |

`events.type` values: `prompt`, `llm_response`, `tool_call`, `tool_result`, `decision`, `twin_result`, `court_message`, `human_decision`, `error`.
`proposals.status` values: `pending`, `auto_applied`, `approved`, `rejected`, `failed`, `suggested_only`.

Rules: `events` is append-only, and `seq` is contiguous per run. Redact secrets before writing any payload.

---

## 7. API

| Method | Path | Purpose |
|---|---|---|
| POST | `/incidents` | Create an incident (from a seed script or a webhook) |
| GET | `/incidents`, `/incidents/{id}` | List and view incidents |
| POST | `/incidents/{id}/run` | Start an agent run |
| GET | `/runs/{id}` | Run summary |
| GET | `/runs/{id}/events` | Black box timeline (JSON) |
| GET | `/runs/{id}/replay` | Step-by-step replay page (HTML) |
| GET | `/runs/diff?a={id}&b={id}` | Show where two runs diverged |
| GET | `/proposals?status=pending` | Approval queue |
| POST | `/proposals/{id}/decision` | Body: `{decision: "approve"\|"reject", note}` |
| GET | `/agents/{id}/trust` | Score, level, and history |
| GET | `/healthz` | Liveness check |

---

## 8. Agents and structured outputs

All LLM outputs use a JSON schema validated with Pydantic. Invalid output is retried once, then the run fails with an `error` event.

**Responder** (diagnose + propose)
```json
{
  "diagnosis": "string",
  "confidence": 0.0,
  "actions": [
    {"operation": "s3.put_public_access_block", "params": {"Bucket": "..."}, "risk_class": "low"}
  ],
  "rationale": "string"
}
```

**Prosecutor / Defender:** each returns `{"argument": "string", "key_risks_or_safeguards": ["..."]}`. Two rounds maximum.

**Judge**
```json
{
  "verdict": "approve | revise | reject",
  "risk_score": 0,
  "reasons": ["..."],
  "conditions": ["..."]
}
```

**Action catalog** (`actions/catalog.py`): each entry defines the operation name, a parameter schema, a risk class (`low`, `medium`, `high`), and the minimum trust level. Anything not in the catalog is rejected. `high` is never auto-applied.

---

## 9. Incident scenarios

Each folder in `scenarios/` has `setup.py` (builds the broken state in a LocalStack instance), `incident.json` (the alert payload), `health.py` (checks that prove the fix worked without breaking anything), and `expected.json` (the reference fix, used for evaluation).

| Scenario | Broken state | Expected fix | Health checks |
|---|---|---|---|
| `public_bucket` | S3 bucket with a public-read policy/ACL | Enable the public access block | Bucket is not public; the app role can still read objects |
| `sqs_backlog` | Queue with a tiny visibility timeout causing redelivery and no DLQ | Raise visibility timeout and add a DLQ with a redrive policy | Queue attributes updated; DLQ exists; simulated consumer drains the queue |
| `open_security_group` | Security group allowing SSH (22) from 0.0.0.0/0 | Revoke that ingress rule | No world-open port 22; the HTTPS rule is still present |

Verify that the installed LocalStack community version supports every API each scenario needs before building on it.

---

## 10. Trust ladder

| Level | Name | Permissions |
|---|---|---|
| 0 | Shadow | Suggest only. Read-only access. |
| 1 | Operator | May auto-apply `low` risk actions if the policy gate passes. |
| 2 | Senior | May also propose `medium` actions, applied after human approval. |

`high` risk actions always need a human, at every level.

**Score (0-100, new agents start at 0)**

| Event | Delta |
|---|---|
| Proposal approved by human unchanged | +5 |
| Approved after human edit | +2 |
| Rejected by human | -10 |
| Applied action fails verification or causes a health regression | -20 |

**Thresholds:** score >= 30 gives Level 1, score >= 70 gives Level 2. Dropping below a threshold demotes the agent immediately. All values live in `trust/policy.py` and are configurable.

---

## 11. Build phases with acceptance criteria

| Phase | When | Deliverable | Done when |
|---|---|---|---|
| 0 | Before week 1 (3-4 days) | Learn Docker Compose, FastAPI, LangGraph basics | You can run a hello-world version of each |
| 1 | Week 1 | Repo, Compose stack, Postgres, black box recorder, responder agent on `public_bucket` | `make seed SCENARIO=public_bucket && make run` produces a run with events in Postgres |
| 2 | Week 2 | Replay page, all 3 scenarios, action catalog + executor | Replay page shows every step; 3 scenarios produce valid proposals |
| 3 | Week 3 | Digital twin and health checks | Proposals are applied to the twin first; `twin_result` events are recorded |
| 4 | Week 4 | Agent court | Verdict and risk score stored on each proposal; transcript visible in the UI |
| 5 | Week 5 | Trust ladder, policy gate, approval UI | Same incident behaves differently at Level 0, 1 and 2. **Localhost version complete.** |
| 6 | Week 6 | Terraform for the platform; deploy to ECS Fargate + RDS | App reachable on AWS, running one scenario against a tiny real setup |
| 7 | Week 7 | IAM roles per trust level, read-only run, testing | Level 0 uses a read-only role, Level 1-2 use a scoped remediation role |
| 8 | Week 8 | Metrics, README, demo video, buffer | Metrics table filled with real numbers |

Cut a feature rather than extend the schedule if a phase runs over by 3 days. Record a short demo at the end of weeks 2, 5 and 8.

---

## 12. AWS production mapping

| Local | Production |
|---|---|
| FastAPI + agent container | ECS Fargate |
| Postgres container | RDS Postgres |
| Scenario seed / webhook | CloudWatch alarm → EventBridge → SQS → worker |
| LocalStack "world" | The real (small, dedicated) AWS account |
| LocalStack twin | Same LocalStack twin for the MVP (separate sandbox account is a later upgrade) |
| Trust level | IAM role assumed through STS: `autopilot-readonly` (Level 0), `autopilot-remediator` (Levels 1-2, scoped to catalog operations) |
| `.env` | Secrets Manager |
| Logs | CloudWatch Logs |
| LLM | Bedrock, or an external API key in Secrets Manager |
| Deploy | Terraform + GitHub Actions using OIDC (no stored AWS keys) |

Cost safety: set an AWS Budget alert on day one and stop the Fargate service when not demoing.

---

## 13. Safety rules (must hold in all phases)

1. Default endpoint is LocalStack. Real AWS is used only when `ENV=prod` and credentials are explicitly configured.
2. The executor runs only catalog operations, with params validated against the schema.
3. Support a `DRY_RUN=true` mode where the executor logs and skips every write.
4. No secrets, tokens or credentials in logs or event payloads.
5. `high` risk actions are never auto-applied.
6. An action must be applied and health-checked on the twin before it can run on the world.
7. Every state change is recorded as an event.

---

## 14. Metrics to report

| Metric | How to measure |
|---|---|
| Fix success rate on the twin | Proposals that pass health checks / total proposals |
| Court accuracy | How often the judge's verdict matches the human decision |
| Autonomous resolution rate | Incidents resolved with no human action / total incidents |
| Time to resolution | Incident created to verified fix |
| Cost per incident | Sum of `cost_usd` across the run's events |
| Safety | Bad fixes caught by the twin or court before reaching the world |

Run each scenario at least 10 times per trust level for stable numbers.

---

## 15. Suggested prompts for Claude Code (in order)

1. "Read CLAUDE.md and PROJECT_SPEC.md. Scaffold the repo structure from section 5, with a docker-compose.yml running Postgres, a LocalStack 'world' on 4566 and a LocalStack 'twin' on 4567. Add a Makefile with `up`, `down`, `test`, `lint`, `seed`, `run`."
2. "Implement the SQLAlchemy models from section 6 with Alembic migrations, and the black box recorder in `app/blackbox/recorder.py` with unit tests."
3. "Implement the `public_bucket` scenario (setup, incident.json, health, expected) and a seed script."
4. "Implement `llm/provider.py` and the responder node using LangGraph, with Pydantic-validated output. Record every prompt and response through the recorder."
5. "Implement the action catalog and executor with allowlist validation and DRY_RUN support. Add tests that reject unknown operations."
6. "Build the replay page with FastAPI + Jinja2 + HTMX showing the events of a run in order."

Then continue phase by phase (twin, court, trust ladder, approvals, AWS).

---

## 16. Open decisions

- LLM provider for development (Anthropic API, Bedrock or Ollama)
- Weekly hours available (the schedule above assumes 15-20)
- Whether Terraform-generated fixes are attempted after Phase 5 (stretch)
- A public project name (the working title is "Safe Autopilot for Cloud Ops")
