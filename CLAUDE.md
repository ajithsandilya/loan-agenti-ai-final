# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

---

## Project Overview

**Multi-Agent Agentic AI Loan Decisioning System** — A proof-of-concept that automates loan approval decisions using a five-layer architecture with four specialised agents coordinated via LangGraph.

**Key Constraints:**
- Proof of concept, not production-ready
- No regulatory or legal compliance
- Sequential agent execution (no parallelization yet)
- In-memory state storage (lost on restart)
- Manual startup of six separate services

---

## Architecture: Five Layers

```
Layer 1 (Presentation)    → Streamlit chatbot UI (port 8501)
Layer 2 (Microservice)    → FastAPI REST API (port 8000)
Layer 3 (Orchestration)   → LangGraph StateGraph workflow
Layer 4 (Agent)           → Four Python agents + Claude Sonnet 4.6 LLM
Layer 5 (Communication)   → Four MCP servers (ports 8001-8004)
```

### Data Flow
1. Streamlit UI sends POST /loan/apply to FastAPI
2. FastAPI validates schema and invokes LangGraph orchestrator (orchestration/graph.py)
3. Graph executes four nodes sequentially (applicant_profile → financial_risk → loan_decision → compliance)
4. Each agent calls its MCP server via HTTP to fetch data or perform analysis
5. Loan Decision Agent calls Claude Sonnet 4.6 for final reasoning
6. Results propagate back to Streamlit UI as JSON

### Four Agents
- **Applicant Profile Agent** → calls ApplicantDB (MCP 8001)
- **Financial Risk Analysis Agent** → calls RiskRulesDB (MCP 8002)
- **Loan Decision Agent** → calls DecisionSynthesis (MCP 8003), then Claude Sonnet
- **Compliance and Action Orchestrator Agent** → calls NotificationSystem (MCP 8004)

---

## Common Development Commands

### Installation & Setup
```bash
cd loan-agentic-ai
pip install --break-system-packages -r requirements.txt
# Or in a venv: python3 -m venv venv && source venv/bin/activate && pip install -r requirements.txt
cp .env.example .env
```

### Starting Services (each in its own terminal)
```bash
# MCP servers (must start first; agents depend on them)
python3 -m uvicorn mcp_servers.applicant_db.server:app --port 8001
python3 -m uvicorn mcp_servers.risk_rules_db.server:app --port 8002
python3 -m uvicorn mcp_servers.decision_synthesis.server:app --port 8003
python3 -m uvicorn mcp_servers.notification_system.server:app --port 8004

# FastAPI microservice
python3 -m uvicorn microservice.main:app --reload --port 8000

# Streamlit UI (opens at http://localhost:8501)
python3 -m streamlit run app/streamlit_app.py
```

### API Testing
- FastAPI docs: http://localhost:8000/docs (interactive Swagger UI)
- Example: `curl -X POST http://localhost:8000/loan/apply -H "Content-Type: application/json" -d '{...}'`

### Testing
```bash
# Run all 31 tests
python3 -m pytest tests/ -v

# Run tests in one file
python3 -m pytest tests/test_agents.py -v

# Run a specific test
python3 -m pytest tests/test_agents.py::test_applicant_profile_agent -v
```

### Mock LLM Mode
```bash
# Skip API key requirement; uses RiskRulesDB decision engine instead
MOCK_LLM_MODE=true python3 -m uvicorn microservice.main:app --reload --port 8000
```

---

## Directory Structure & Key Files

```
loan-agentic-ai/
├── agents/                           # Four agent implementations
│   ├── applicant_profile_agent.py
│   ├── financial_risk_analysis_agent.py
│   ├── loan_decision_agent.py
│   └── compliance_action_orchestrator_agent.py
├── mcp_servers/                      # Four MCP servers
│   ├── applicant_db/
│   ├── risk_rules_db/
│   ├── decision_synthesis/
│   └── notification_system/
├── orchestration/                    # LangGraph workflow
│   ├── graph.py                      # StateGraph definition & node implementations
│   ├── routing.py                    # Conditional routing logic
│   └── state.py                      # WorkflowState TypedDict
├── microservice/                     # FastAPI REST API
│   ├── main.py                       # POST /loan/apply, GET /loan/status/:id, /health
│   └── schemas.py                    # Pydantic request/response models
├── app/                              # Streamlit chatbot UI
│   └── streamlit_app.py
├── models/                           # Shared data models
│   ├── loan_application.py           # LoanApplication & ApplicantProfile Pydantic models
│   └── agent_outputs.py              # AgentResult Pydantic models
├── config/                           # Configuration
│   └── poc_rules.yaml                # Configurable decision thresholds & weights
├── prompts/                          # Agent prompt templates
├── tests/                            # 31 pytest tests
│   ├── conftest.py
│   ├── test_agents.py
│   ├── test_mcp_servers.py
│   ├── test_microservice.py
│   └── test_orchestration.py
├── .env.example                      # Template for environment variables
└── requirements.txt                  # Dependencies
```

---

## Key Architectural Patterns

### WorkflowState (orchestration/state.py)
- TypedDict that flows through the LangGraph StateGraph
- Carries loan_application, agent results, errors, status
- Accessed via `state["key"]` syntax

### Node Implementations (orchestration/graph.py)
- Each node runs sequentially: `node_applicant_profile() → node_financial_risk() → node_loan_decision() → node_compliance()`
- Nodes read from state, invoke their agent, update state with results
- Error handling: if an agent fails, node sets `status="error"` and stops

### Agent Pattern
All agents follow a similar pattern:
1. Take LoanApplication as input
2. Call their MCP server (via HTTP requests)
3. Return AgentResult Pydantic model with structured outputs

### MCP Servers
- Implemented as FastAPI apps (not a full MCP protocol stack)
- Each server runs on a dedicated port (8001-8004)
- Agents query them synchronously via HTTP

### Conditional Routing (orchestration/routing.py)
- `route_after_profile()` → decides if workflow should continue or error
- `route_after_risk()` → similar check
- `route_after_decision()` → check decision validity
- `route_after_compliance()` → final routing to END or error state

---

## Configuration & Rules

**config/poc_rules.yaml** is the single source of truth for decision thresholds:
- Income stability bands (high/medium/low)
- Employment risk types (low/medium/high)
- Credit score ranges (excellent/good/fair/poor)
- Debt-to-income (DTI) thresholds
- Loan amount risk bands
- Anomaly detection: loan-to-income ratio, age bounds, tenure minimums
- Decision synthesis: **weighted risk score** + **classification thresholds**
  - Approved: risk score ≤ 35
  - Manual Review: 35 < risk score ≤ 65
  - Rejected: risk score > 65

**Risk Score Formula (DecisionSynthesis):**
```
risk_score = (
  credit_score_weight * credit_risk +
  dti_weight * dti_risk +
  loan_amount_weight * loan_risk +
  employment_weight * employment_risk
)
```

---

## Environment Variables (.env)

```bash
# Anthropic API (required for real mode; ignored in mock mode)
ANTHROPIC_API_KEY=sk-ant-...
ANTHROPIC_MODEL_NAME=claude-sonnet-4-6

# Mock LLM mode: set to "true" to skip API key requirement
MOCK_LLM_MODE=true

# FastAPI URL (used by Streamlit)
FASTAPI_URL=http://localhost:8000

# MCP server ports (must match startup commands)
MCP_APPLICANT_DB_PORT=8001
MCP_RISK_RULES_DB_PORT=8002
MCP_DECISION_SYNTHESIS_PORT=8003
MCP_NOTIFICATION_SYSTEM_PORT=8004
```

---

## Testing

All tests are in `tests/`:
- **test_agents.py** (8 tests) — Agent unit tests with mock MCP responses
- **test_mcp_servers.py** (8 tests) — MCP server endpoints
- **test_microservice.py** (6 tests) — FastAPI endpoints and schemas
- **test_orchestration.py** (9 tests) — Graph nodes, routing, end-to-end workflows

Run tests with: `python3 -m pytest tests/ -v`

---

## Common Tasks

### Adding a New Risk Rule
1. Edit `config/poc_rules.yaml` → add threshold
2. Update `mcp_servers/risk_rules_db/server.py` → expose new rule in a route
3. Update agent to call the new route
4. Add test in `tests/test_mcp_servers.py`

### Adding a New Agent
1. Create `agents/new_agent.py` with `run_new_agent(application: LoanApplication) -> AgentResult`
2. Add MCP server in `mcp_servers/new_server/` (if needed)
3. Add node to `orchestration/graph.py`
4. Add routing in `orchestration/routing.py` if needed
5. Update WorkflowState in `orchestration/state.py` to carry results
6. Add tests in `tests/test_agents.py` and `tests/test_orchestration.py`

### Mock LLM Mode vs. Real Mode
- **Mock mode** (`MOCK_LLM_MODE=true`): Uses DecisionSynthesis rule engine, no API call needed, marks response as `[MOCK LLM MODE]`
- **Real mode** (`MOCK_LLM_MODE=false`): Calls Claude Sonnet 4.6 for final reasoning and explanation (requires `ANTHROPIC_API_KEY`)

---

## Proof-of-Concept Limitations to Know

1. **No persistent database** — ApplicantDB uses in-memory mock data
2. **No real notifications** — NotificationSystem simulates email/SMS/push
3. **No authentication/authorisation** — Any caller can submit applications
4. **No parallelism** — Agents run sequentially in the graph
5. **Single-process deployment** — All six services must be started manually in separate terminals
6. **No TLS, rate limiting, or secrets management** — Not production-ready
7. **In-memory state** — Application status lost on FastAPI restart
8. **Not legally/regulatorily compliant** — These rules are illustrative only

---

## Debugging Tips

- **Check MCP servers are running** — agents fail silently if a server is down
- **Read logs** — FastAPI, Streamlit, and graph execution log to stdout; check for exceptions
- **Use FastAPI /docs** — test microservice endpoints interactively at http://localhost:8000/docs
- **Mock LLM mode** — useful for testing without an API key; compare output structure between mock and real
- **Inspect state in routing** — add logging in `routing.py` to see state values at decision points
- **Run single test** — `pytest tests/test_agents.py::test_name -v` for isolated debugging
