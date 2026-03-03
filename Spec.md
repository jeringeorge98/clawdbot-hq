OpenClaw HQ — MVP Specification

Project: OpenClaw HQ — Agent Orchestration Portal
Version: 0.1.0 (MVP)
Date: March 2026


1. Project Overview
OpenClaw HQ is a web portal that sits on top of the OpenClaw Gateway, providing a visual interface to orchestrate AI agents. The MVP covers five capabilities: gateway connectivity, agent token usage summary, model switching per agent, a Kanban task board, and cost/model guardrails.
Tech Stack: React + TypeScript + Tailwind + shadcn/ui (frontend), FastAPI + Python via uv (backend), SQLite via SQLAlchemy + Alembic (database), async websockets library (gateway connection), FastAPI WebSocket (real-time to frontend).

2. Database Schema
Five tables: agents (cached from gateway), tasks, token_usage, guardrails, guardrail_events — as we defined in the conversation above.

3. Features & Specs

Feature 1: Gateway Connection
Spec 1.1: WebSocket Handshake & Authentication
Backend connects to ws://127.0.0.1:18789 on startup. Receives connect.challenge, sends connect with role operator, scopes ["operator.read", "operator.write", "operator.admin", "operator.approvals"], and auth token from OPENCLAW_GATEWAY_TOKEN. On hello-ok, connection is established. On failure, retries with exponential backoff (1s → 30s cap).
Acceptance Criteria:

Connects to gateway WebSocket on startup
Handles connect.challenge → sends connect with correct role, scopes, token
Parses hello-ok and persists device token if issued
GET /api/gateway/status returns { connected: bool, uptime_seconds: int }
Auto-reconnects with exponential backoff on disconnect
Gateway URL and token configurable via env vars

Spec 1.2: Request/Response Handling
Async send_request(method, params) sends {type: "req", id, method, params}, waits for matching response by ID, returns payload. 30s timeout.
Acceptance Criteria:

Generates unique request IDs
Correlates responses to requests by ID
Returns payload on ok: true, raises exception on ok: false
Times out after 30 seconds

Spec 1.3: Event Subscription
Listens for {type: "event"} frames, dispatches to registered handlers by event name.
Acceptance Criteria:

Event loop runs concurrently with request/response
Handlers registered per event name
Unhandled events logged, don't crash the client
Events forwarded to frontend clients via /ws


Feature 2: Token Summary Per Agent
Spec 2.1: Agent Discovery & Sync
On connect, query gateway for agents, populate local agents table. Re-sync every 60 seconds.
Acceptance Criteria:

GET /api/agents returns agents with: id, name, current_model, status, token_summary
token_summary includes: total_input_tokens, total_output_tokens, total_cost_estimate, period (default 24h)
Refreshes from gateway every 60 seconds
Status reflects gateway presence data
New agents auto-added, removed agents marked offline (not deleted)

Spec 2.2: Token Usage Collection
Parse token usage from gateway events, persist to token_usage with agent_id, session_key, model, computed cost.
Acceptance Criteria:

Gateway token events parsed and stored
Cost computed from configurable model pricing table
GET /api/agents/{id}/usage returns usage filterable by ?from=&to=
Response includes per-model breakdown
Response includes time-series array for charting

Spec 2.3: Dashboard Summary View (Frontend)
Agent cards showing name, status dot, current model, token summary.
Acceptance Criteria:

Dashboard shows all agents on load
Each card: name, online/offline dot, model, tokens (24h), cost (24h)
Time period selectable: 24h / 7d / 30d
Auto-updates via WebSocket
Loading spinner and empty state handled


Feature 3: Model Switching Per Agent
Spec 3.1: Available Models List
Query gateway for models, expose via API.
Acceptance Criteria:

GET /api/models returns models with: id, name, provider
Cached, refreshed every 5 minutes

Spec 3.2: Model Switch Endpoint
Accept model change, validate against guardrails, apply via config.patch.
Acceptance Criteria:

PATCH /api/agents/{id}/model accepts { model_id }
Checks allowed_models guardrail; returns 403 if blocked
Sends config.patch to gateway
Updates local agents table on success
Returns 502 on gateway error
Emits WebSocket event to frontend

Spec 3.3: Model Switcher UI (Frontend)
Dropdown on agent detail page.
Acceptance Criteria:

Dropdown with current model pre-selected
Blocked models shown disabled with tooltip
Confirmation dialog on change
Calls PATCH endpoint on confirm
Success/error toast
Immediate UI update


Feature 4: Kanban Task Board
Spec 4.1: Task CRUD
Acceptance Criteria:

POST /api/tasks with: title, description, priority, agent_id (optional)
GET /api/tasks filterable by ?status=&agent_id=
GET /api/tasks/{id} full detail with result and token usage
PATCH /api/tasks/{id} update fields
DELETE /api/tasks/{id} soft-delete
UUID assigned on creation, timestamps auto-set

Spec 4.2: Task Dispatch
Acceptance Criteria:

POST /api/tasks/{id}/dispatch sends task to assigned agent
Blocked if no agent_id (400)
Blocked if status not queued/assigned (400)
Pre-dispatch guardrail checks: daily_budget, monthly_budget, allowed_models
If guardrail blocks: status → guardrail_blocked, failure_reason set, guardrail_events row, returns 403
On pass: sends chat message to gateway, stores session_key, status → in_progress
Status flow: queued → assigned → in_progress → done/failed

Spec 4.3: Task Status Tracking
Acceptance Criteria:

Gateway events matching task's session_key update the task:

Completion → status: done, result populated
Error → status: failed, failure_reason populated


Token usage during execution recorded with task_id
Mid-execution guardrails checked (cost_per_task, max_tokens_per_task):

Over limit → halt, status: failed with guardrail reason


auto_downgrade: patches model via gateway, logs event, agent continues
All changes broadcast via WebSocket

Spec 4.4: Kanban Board UI (Frontend)
Acceptance Criteria:

5 columns: Queued | Assigned | In Progress | Done | Failed
Cards show: title, priority badge, agent name or "Unassigned", cost if dispatched
guardrail_blocked tasks in Failed column with distinct badge
Create Task form: title, description, priority, agent (optional)
Drag between Queued ↔ Assigned columns
Dispatch button on Assigned tasks
In Progress cards show live cost via WebSocket
Click card → detail panel with full output, usage, guardrail events
Auto-updates via WebSocket, no polling


Feature 5: Agent Guardrails
Spec 5.1: Guardrail CRUD
Acceptance Criteria:

GET /api/agents/{id}/guardrails list guardrails
POST /api/agents/{id}/guardrails create with { type, value, enabled }
PATCH /api/guardrails/{id} update value or enabled
DELETE /api/guardrails/{id} remove
One guardrail per type per agent (409 on duplicate)
Value validated per type schema

Spec 5.2: Pre-Dispatch Checks
Acceptance Criteria:

daily_budget: sum today's usage, block if >= limit
monthly_budget: sum this month, block if >= limit
allowed_models: check current_model against list, block if not in list
On block: status → guardrail_blocked, event logged, 403 returned

Spec 5.3: Mid-Execution Monitoring
Acceptance Criteria:

cost_per_task: cumulative cost check, halt if exceeded
max_tokens_per_task: cumulative token check, halt if exceeded
auto_downgrade: patch model on threshold, log event, continue on cheaper model
All triggers emit WebSocket events

Spec 5.4: Guardrails UI (Frontend)
Acceptance Criteria:

Toggle switches per guardrail (enabled/disabled)
Add Guardrail form with type-specific inputs (dollar amounts, token counts, model checkboxes)
Live progress bars for budget guardrails ($X.XX / $Y.YY)
Color coding: green (<50%), yellow (50-75%), orange (75-90%), red (>90%)
Guardrail events log with timestamp, action, task, details
Sortable and filterable


4. API Summary
18 endpoints total: GET /api/gateway/status, full CRUD for agents/tasks/guardrails/models, POST /api/tasks/{id}/dispatch, GET /api/guardrail-events, WS /ws

5. WebSocket Events (Backend → Frontend)
agent.status_changed, agent.usage_updated, agent.model_changed, task.status_changed, task.cost_updated, guardrail.triggered

6. Environment Variables
OPENCLAW_GATEWAY_URL (default ws://127.0.0.1:18789), OPENCLAW_GATEWAY_TOKEN (required), DATABASE_URL (default sqlite), CORS_ORIGINS, LOG_LEVEL

7. Implementation Order
Step 1: Gateway Client Library (Specs 1.1, 1.2, 1.3)
Step 2: Backend Skeleton + DB (schema, FastAPI app, Specs 2.1, 3.1)
Step 3: Token Usage + Model Switching (Specs 2.2, 3.2)
Step 4: Task Lifecycle (Specs 4.1, 4.2, 4.3)
Step 5: Guardrails Engine (Specs 5.1, 5.2, 5.3)
Step 6: React Frontend (Specs 2.3, 3.3, 4.4, 5.4)

8. Backlog (Post-MVP)
Interactive approval queue, context window health monitor, A/B model testing, agent performance analytics, cost forecasting, ClawHub skill browser, model cost calculator, task dependencies, task templates, notification system, multi-gateway support.