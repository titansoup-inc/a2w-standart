# 🧾 A2W STANDARD v1.0

**Agent-as-Worker Interoperability Standard**
*Behavior, state, hierarchy, and telemetry standard for autonomous AI agents, built on top of Google A2A.*

## 0. Scope & Non-Goals

A2W defines:

* the agent state model;
* REST and WebSocket APIs for controlling an agent;
* weight and priorities between agents;
* the task format (Task Context);
* delegation between agents;
* logs, reports, telemetry, and errors;
* compatibility with A2A abilities.

A2W does **not** define:

* specific LLMs/models;
* specific planning algorithms;
* specific orchestration systems.

---

# 1. Terminology

**AGENT** — an autonomous task-executing entity.

**ABILITY** — an agent’s function defined according to the Google A2A standard.

**WEIGHT** — a numeric influence priority (0–100).

**TASK CONTEXT** — a formal structured description of a task.

**FSM** — the agent’s finite state machine.

**STATUS STREAM** — a WebSocket stream carrying state and telemetry updates.

**NEED SIGNAL** — a message indicating that the agent requires additional data or resources.


---

# 2. Transport & Encoding (MUST)

1. All A2W REST endpoints MUST use HTTP/1.1 or HTTP/2.
2. The body of all requests and responses MUST be JSON encoded in UTF-8.
3. All timestamps MUST use ISO 8601 in UTC (for example, `"2025-12-09T09:30:00Z"`).
4. WebSocket connections MUST use JSON messages encoded in UTF-8.
5. It is RECOMMENDED (SHOULD) to use a dedicated subprotocol, for example:

   * `Sec-WebSocket-Protocol: a2w.v1`.

---

# 3. Message Envelope (MUST)

All A2W REST responses and WS messages MUST use a common envelope:

```json
{
  "a2w_version": "1.2",
  "agent_id": "pr-agent",
  "message_type": "status_update|need_data|error|event|payload",
  "payload": { ... }
}
```

* `a2w_version` – string, protocol version.
* `agent_id` – agent identifier (MUST be unique per deployment).
* `message_type` – message type (fixed set of strings).
* `payload` – object depending on the message type.

HTTP error codes MUST follow standard semantics (4xx — client errors, 5xx — server errors).

---

# 4. Versioning & Extensibility (MUST)

1. A2W uses **semver**: `MAJOR.MINOR`.
2. The `a2w_version` field MUST be present in all responses/messages.
3. New fields MAY be added, but:

   * Clients MUST ignore unknown fields.
4. Changes that break backward compatibility REQUIRE a MAJOR version bump.

---

# 5. Task Context Schema (MUST)

```json
{
  "task_id": "uuid",
  "task_type": "string",
  "description": "string",
  "payload": {},
  "priority": 0,
  "deadline": "2025-12-09T09:30:00Z",
  "caller_agent": "string",
  "weight_caller": 0,
  "parent_task_id": "uuid|null",
  "retry_of": "uuid|null",
  "metadata": {}
}
```

* `priority` – 0–100 (0 – lowest, 100 – extremely high).
* `parent_task_id` and `retry_of` MAY be used to track task chains.

---

# 6. FSM – Agent States (MUST)

The list of agent states (same as before), plus **allowed transitions**:

**States:**

* `idle`
* `initializing`
* `running`
* `waiting`
* `blocked`
* `delegating`
* `finishing`
* `finished`
* `stopped`
* `terminated`

**Allowed transitions (subset):**

* `idle → initializing → idle|running|terminated`
* `running → waiting|blocked|delegating|finishing|terminated`
* `waiting → running|blocked|terminated`
* `blocked → waiting|terminated`
* `delegating → running|waiting|terminated`
* `finishing → finished|terminated`
* `finished → idle`
* `stopped` and `terminated` – terminal for the current task.

The agent MUST report its current state via `/status` and `/ws/status`.

---

# 7. Error Model (MUST)

Error format inside the envelope:

```json
{
  "a2w_version": "1.2",
  "agent_id": "pr-agent",
  "message_type": "error",
  "payload": {
    "code": "E013_MISSING_DATA",
    "http_status": 400,
    "severity": "high|medium|low",
    "recoverable": true,
    "message": "Data 'market_insights' is missing.",
    "details": {}
  }
}
```

Recommended (SHOULD) set of codes:

* `E001_INTERNAL`
* `E010_INVALID_INPUT`
* `E013_MISSING_DATA`
* `E020_PERMISSION_DENIED`
* `E030_LOW_WEIGHT`
* `E040_TIMEOUT`
* `E050_EXTERNAL_DEPENDENCY`

---

# 8. Weighted Hierarchy Model (MUST)

**Weight:** integer 0–100.

Recommended mapping table:

* 0–30: `junior`
* 31–60: `middle`
* 61–90: `senior`
* 91–100: `lead`

## 8.1 Priority Score (SHOULD)

To order requests, an A2W-compliant orchestrator SHOULD use:

```text
priority_score = α * weight + β * task_priority
```

where:

* `weight` – agent’s weight,
* `task_priority` – task priority,
* α and β — coefficients (α=1, β=1 recommended by default).

## 8.2 Rules (MUST)

1. The agent MUST expose its current `weight` via `/manifest`, `/status`, and `/ws/status`.
2. All else being equal, requests from agents with higher `weight` SHOULD be processed first.
3. An agent SHOULD NOT interrupt execution of a task initiated by an agent with significantly higher `weight` without explicit permission.

---

# 9. REST API (MUST)

Base prefix: `/a2w/v1`

Methods and key requirements (updated):

* `GET /manifest` – MUST be idempotent.
* `GET /capabilities` – A2A abilities + additional metadata.
* `POST /start` – accepts Task Context; repeated calls with the same `task_id` MUST be idempotent (must not create a duplicate execution).
* `GET /status` – returns FSM state, progress, weight, telemetry.
* `POST /stop` – graceful stop; MAY not finish instantly.
* `POST /terminate` – immediate stop; state becomes `terminated`.
* `GET /report` – final artifact for the given `task_id`.
* `GET /logs` – MAY use pagination (`cursor`, `limit`).
* `POST /insights` – requests artifacts/useful data.
* `POST /weight` – updates weight (if allowed by policy).

---

# 10. WebSocket API (MUST)

Base prefix: `/a2w/v1/ws`

## 10.1 `/status` (MUST)

Envelope with `message_type = "status_update"`:

```json
{
  "a2w_version": "1.2",
  "agent_id": "pr-agent",
  "message_type": "status_update",
  "payload": {
    "state": "running",
    "task_id": "uuid",
    "progress": 0.42,
    "weight": 70,
    "message": "Writing introduction",
    "telemetry": {
      "cpu": 32,
      "memory_mb": 512,
      "tokens": 3450,
      "eta_seconds": 15
    }
  }
}
```

## 10.2 `/needs` (MUST)

`message_type = "need_data"`:

```json
{
  "a2w_version": "1.2",
  "agent_id": "pr-agent",
  "message_type": "need_data",
  "payload": {
    "task_id": "uuid",
    "required": ["market_trends"],
    "description": "Need current market trends to continue.",
    "urgency": "high",
    "weight": 70
  }
}
```

## 10.3 `/errors` (MUST)

Transmits errors using the Error Model format.

## 10.4 `/events` (SHOULD)

Generic events channel, including:

* `delegation_started` / `delegation_finished`
* `weight_update`
* `task_retried`
* `health_warning`

---

# 11. Delegation Rules (MUST)

When delegating, the agent MUST:

1. Pass the full Task Context.
2. Provide `caller_agent` and `weight_caller`.
3. Provide the `reason` for delegation.

Example delegation message (via an A2A call or `/events`):

```json
{
  "a2w_version": "1.2",
  "agent_id": "orchestrator-agent",
  "message_type": "event",
  "payload": {
    "event_type": "delegation_started",
    "task": { ...task_context... },
    "delegate_to": "research-agent",
    "reason": "need_market_data"
  }
}
```

## 11.1 Cycle Avoidance (SHOULD)

An agent SHOULD avoid delegation cycles (A → B → A). For this:

* `metadata` in Task Context MAY contain the chain of agents that have already processed the task.
* An agent SHOULD refuse delegation if it is already present in that chain.

---

# 12. Security & Permissions (MUST)

At minimum:

* each agent has an `agent_id`;
* there is a mapping `agent_id → permissions`;
* there is a list of `allowed_agents` and/or `allowed_events`.

Example manifest fragment:

```json
{
  "identity": {
    "agent_id": "pr-agent",
    "allowed_callers": ["orchestrator-agent", "admin"],
    "allowed_callees": ["research-agent", "seo-agent"]
  }
}
```

Authentication and transport-level security (TLS, OAuth, mTLS) depend on the environment and are out of A2W scope, but using encrypted transport (`HTTPS`, `WSS`) is strongly RECOMMENDED (SHOULD).

---

# 13. Telemetry (SHOULD)

Telemetry is included in `/status` and may contain:

* `cpu` – CPU usage (%) for the current task;
* `memory_mb` – memory usage;
* `tokens` – LLM token usage;
* `eta_seconds` – estimated time to completion.

This simplifies orchestration and auto-scaling.

---

# 14. Compatibility with A2A (MUST)

* Any A2W-compliant agent MUST expose abilities in A2A format via `/capabilities`.
* Ability calls between agents SHOULD use the A2A RPC approach.
* A2W does not replace A2A; it extends it with a model of state, priorities, and telemetry.

---

# **15. Example Lifecycle (MUST for reference implementations)**

This section provides a complete example of a typical A2W-compliant agent lifecycle.
All messages follow the A2W message envelope and the agent FSM.
This example may be used for testing, debugging, or validating implementations.

---

## **15.1 Step 1 — Task Start**

A controller (or another agent) calls:

```
POST /a2w/v1/start
```

**Request:**

```json
{
  "task_id": "123e4567-e89b-12d3-a456-426614174000",
  "task_type": "content_generation",
  "description": "Write a 2-page article about distributed agents.",
  "payload": {
    "language": "en",
    "depth": "medium"
  },
  "priority": 70,
  "deadline": "2025-12-09T10:00:00Z",
  "caller_agent": "orchestrator-agent",
  "weight_caller": 90,
  "metadata": {}
}
```

Agent transitions:
`idle → initializing → running`

---

## **15.2 Step 2 — Status Stream Begins**

Agent sends a status update to `/ws/status`:

```json
{
  "a2w_version": "1.2",
  "agent_id": "writer-agent",
  "message_type": "status_update",
  "payload": {
    "state": "running",
    "task_id": "123e4567-e89b-12d3-a456-426614174000",
    "progress": 0.12,
    "weight": 65,
    "message": "Collecting high-level structure.",
    "telemetry": {
      "cpu": 14,
      "memory_mb": 412,
      "tokens": 980,
      "eta_seconds": 45
    }
  }
}
```

---

## **15.3 Step 3 — Agent Requires Data**

At some stage the agent lacks needed information and transitions:
`running → waiting`

It sends a **Need Signal** to `/ws/needs`:

```json
{
  "a2w_version": "1.2",
  "agent_id": "writer-agent",
  "message_type": "need_data",
  "payload": {
    "task_id": "123e4567-e89b-12d3-a456-426614174000",
    "required": ["latest_distributed_systems_trends"],
    "description": "Additional research required to continue writing.",
    "urgency": "medium",
    "weight": 65
  }
}
```

---

## **15.4 Step 4 — Providing Data (Insights Call)**

Another agent (e.g., research-agent) responds with:

```
POST /a2w/v1/insights
```

**Request:**

```json
{
  "task_id": "123e4567-e89b-12d3-a456-426614174000",
  "rating": "high",
  "payload": {
    "latest_distributed_systems_trends": [
      "multi-agent orchestration",
      "LLM-backed autonomous tools",
      "streaming compute pipelines"
    ]
  }
}
```

Writer agent transitions:
`waiting → running`

and sends a new status update:

```json
{
  "a2w_version": "1.2",
  "agent_id": "writer-agent",
  "message_type": "status_update",
  "payload": {
    "state": "running",
    "task_id": "123e4567-e89b-12d3-a456-426614174000",
    "progress": 0.55,
    "weight": 65,
    "message": "Incorporating research into draft.",
    "telemetry": {
      "cpu": 22,
      "memory_mb": 451,
      "tokens": 1500,
      "eta_seconds": 25
    }
  }
}
```

---

## **15.5 Step 5 — Task Near Completion**

The agent transitions:
`running → finishing`

Status update:

```json
{
  "a2w_version": "1.2",
  "agent_id": "writer-agent",
  "message_type": "status_update",
  "payload": {
    "state": "finishing",
    "task_id": "123e4567-e89b-12d3-a456-426614174000",
    "progress": 0.92,
    "weight": 65,
    "message": "Finalizing article structure and proofreading."
  }
}
```

---

## **15.6 Step 6 — Task Completed**

The agent transitions:
`finishing → finished`

And sends:

```json
{
  "a2w_version": "1.2",
  "agent_id": "writer-agent",
  "message_type": "status_update",
  "payload": {
    "state": "finished",
    "task_id": "123e4567-e89b-12d3-a456-426614174000",
    "progress": 1.0,
    "weight": 65,
    "message": "Task completed successfully."
  }
}
```

---

## **15.7 Step 7 — Retrieving the Final Report**

Controller calls:

```
GET /a2w/v1/report?task_id=123e4567-e89b-12d3-a456-426614174000
```

**Response:**

```json
{
  "a2w_version": "1.2",
  "agent_id": "writer-agent",
  "message_type": "payload",
  "payload": {
    "task_id": "123e4567-e89b-12d3-a456-426614174000",
    "artifacts": {
      "article_html": "<h1>Distributed Agents in Modern Systems</h1> ...",
      "keywords": ["distributed systems", "agents", "autonomous tools"]
    },
    "summary": "Article successfully generated with integrated research insights."
  }
}
```

---

## **15.8 Step 8 — Agent Returns to Idle**

Agent transitions:
`finished → idle`

Final optional WS message:

```json
{
  "a2w_version": "1.2",
  "agent_id": "writer-agent",
  "message_type": "event",
  "payload": {
    "event_type": "task_cycle_completed",
    "task_id": "123e4567-e89b-12d3-a456-426614174000"
  }
}
```

---

## ✔️ Complete Agent Lifecycle Sequence

**start → status → needs → insights → status → finishing → finished → report → idle**

This sequence demonstrates:

* task initiation,
* status streaming,
* requesting missing data,
* processing insights,
* finishing workflow,
* producing a final report,
* returning to idle.

It is **A2W-compliant** and suitable for use in documentation, testing suites, and reference implementations.

---

At this level, the standard is **sufficiently complete and realistic** to:

* publish it in a GitHub repository as `A2W Standard v1.0`;
* use it as the basis for a reference implementation;
* reference it in the context of “we built a multi-agent system compatible with A2A and extended with our own open A2W standard”.

If needed, further work can focus not on extending the core standard, but on writing:

* **A2W Reference Implementation Guide**,
* **A2W Orchestrator Best Practices**,
* **A2W Security Profiles**.

---

## Related useful next steps (very short)

* You can add a short “Motivation” section at the top for non-technical readers (VCs, PR, immigration lawyers).
* It’s worth creating a minimal Python reference agent that is 100% A2W-compliant and linking it from the spec.
* Consider registering a simple website like `a2w.dev` with this spec + diagrams.

---

## Terms (copy-ready)

A2W Standard – Agent-as-Worker protocol extending A2A with behavior, state, hierarchy, and telemetry.
A2A – Google’s Agent-to-Agent protocol for defining and calling agent abilities.
Task Context – Structured description of a task passed between agents.
FSM – Finite State Machine that formalizes allowed agent states and transitions.
Weighted Hierarchy – Priority model based on agent weight and task priority.
Status Stream – WebSocket stream with the agent’s current state and progress.
Need Signal – WebSocket message describing missing data or resources required by the agent.
Error Taxonomy – Standardized classification of error codes and their properties.
Telemetry – Technical metrics (CPU, memory, tokens, ETA) for orchestration and scaling.
Extensibility – Ability to evolve the standard with new fields and versions without breaking compatibility.
