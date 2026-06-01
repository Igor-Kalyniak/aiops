# DevOps Tech Lead Interview Preparation
## AI Agent Platform - Agentic Observability

> **Role:** Senior/Staff DevOps Engineer (7+ years)  
> **Focus:** AI Platform Agentic Observability  
> **Stack:** AWS, Kubernetes, Terraform, CI/CD, IaC, Monitoring/Observability, SRE, AIOps

---

## Table of Contents

1. [Core Concepts](#1-core-concepts)
2. [MCP (Model Context Protocol)](#2-mcp-model-context-protocol)
3. [AI Observability Enhancement](#3-ai-observability-enhancement)
4. [Environment Architecture](#4-environment-architecture)
5. [Kubernetes / EKS Deep Dive](#5-kubernetes--eks-deep-dive)
6. [Temporal Workflow Orchestration](#6-temporal-workflow-orchestration)
7. [Agent Architecture](#7-agent-architecture)
8. [Enterprise Connector Security](#8-enterprise-connector-security)
9. [CI/CD Pipeline](#9-cicd-pipeline)
10. [Security Considerations](#10-security-considerations)
11. [Observability Stack](#11-observability-stack)
12. [AWS Services Reference](#12-aws-services-reference)
13. [Incident Response](#13-incident-response)
14. [DR Failover Procedures](#14-dr-failover-procedures)
15. [Mock Interview Q&A](#15-mock-interview-qa)
16. [Quick Reference Card](#16-quick-reference-card)

---

## 1. Core Concepts

### What is "Agentic AI"?

- **Autonomous AI systems** that can plan, reason, use tools, and execute multi-step tasks
- Unlike traditional ML models (request → response), agents operate in **loops**: observe → think → act → observe
- Examples: GitHub Copilot Workspace, AWS CodeWhisperer Agent, Cursor Agent, AutoGPT

### Why Observability is Different for Agents

| Traditional App Observability | Agentic AI Observability |
|------------------------------|--------------------------|
| Request/response latency | Multi-turn conversation traces |
| Error rates, HTTP codes | Tool call failures, reasoning failures |
| Resource utilization | Token consumption, cost per task |
| Linear call graphs | Non-deterministic execution paths |
| Known failure modes | Hallucinations, infinite loops, goal drift |

### Key Metrics for Agentic Systems

- **Token usage** (input/output per turn, cumulative)
- **Tool call success rate** (which tools fail, why)
- **Task completion rate** (did agent achieve the goal?)
- **Latency per step** vs **end-to-end latency**
- **Cost attribution** (per user, per task type)
- **Reasoning quality** (hard to measure, often human-in-the-loop)

---

## 2. MCP (Model Context Protocol)

MCP is an **open standard** (by Anthropic) for connecting AI agents to external tools/data sources.

### Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           MCP HOST                                       │
│  (Cursor, Claude Desktop, VSCode Extension, Custom App)                 │
│                                                                          │
│   ┌─────────────┐    ┌─────────────┐    ┌─────────────┐                │
│   │ MCP Client  │    │ MCP Client  │    │ MCP Client  │                │
│   └──────┬──────┘    └──────┬──────┘    └──────┬──────┘                │
└──────────┼──────────────────┼──────────────────┼────────────────────────┘
           │ stdio/SSE        │ stdio/SSE        │ stdio/SSE
           ▼                  ▼                  ▼
    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
    │ MCP Server  │    │ MCP Server  │    │ MCP Server  │
    │ (GitHub)    │    │ (Postgres)  │    │ (Slack)     │
    └─────────────┘    └─────────────┘    └─────────────┘
```

### Three Core Primitives

#### 1. Tools (Model-controlled)

```json
{
  "name": "query_database",
  "description": "Execute a read-only SQL query",
  "inputSchema": {
    "type": "object",
    "properties": {
      "query": { "type": "string" }
    },
    "required": ["query"]
  }
}
```

- Agent decides **when** to call
- Server executes and returns result
- **Observability hook**: instrument every tool invocation

#### 2. Resources (Application-controlled)

```json
{
  "uri": "file:///logs/app.log",
  "name": "Application Logs",
  "mimeType": "text/plain"
}
```

- Contextual data the host can inject
- Think: file contents, database schemas, documentation
- **Observability hook**: track which resources are fetched and token cost

#### 3. Prompts (User-controlled)

```json
{
  "name": "debug_error",
  "description": "Help debug an error",
  "arguments": [
    { "name": "error_message", "required": true }
  ]
}
```

### Message Flow (What to Trace)

| Message Type | What to Capture |
|--------------|-----------------|
| `tools/call` request | tool name, arguments, timestamp |
| `tools/call` response | result size, latency, error if any |
| `resources/read` | URI, bytes fetched, cache hit/miss |
| `notifications/progress` | task progress for long operations |

### Interview Talking Point

> "MCP standardizes how agents talk to external systems. From an observability perspective, this is a gift—every tool call goes through the same interface. I'd instrument at the MCP client level to capture all outbound tool calls, and at the server level to capture execution details."

---

## 3. AI Observability Enhancement

### The Problem Space

Traditional observability (metrics, logs, traces) was designed for **deterministic systems**. AI/LLM systems break these assumptions.

### Enhancement Area 1: Trace Enhancement

```
┌─────────────────────────────────────────────────────────────────┐
│                    ENHANCED AGENT TRACE                         │
├─────────────────────────────────────────────────────────────────┤
│ [Session Span]                                                  │
│   ├── session_id, user_id, tenant_id                           │
│   ├── task_description (semantic)                               │
│   ├── outcome: success | partial | failure                      │
│   │                                                             │
│   ├── [Turn 1 Span]                                             │
│   │     ├── [Context Assembly]                                  │
│   │     │     ├── tokens_from_history: 1200                     │
│   │     │     ├── tokens_from_tools: 800                        │
│   │     │     └── context_window_utilization: 45%               │
│   │     │                                                       │
│   │     ├── [LLM Inference]                                     │
│   │     │     ├── model: claude-3-opus                          │
│   │     │     ├── input_tokens: 2000                            │
│   │     │     ├── output_tokens: 350                            │
│   │     │     ├── latency_ms: 1200                              │
│   │     │     ├── cost_usd: 0.045                               │
│   │     │     └── thinking_tokens: 500 (if applicable)          │
│   │     │                                                       │
│   │     ├── [Tool Decision]                                     │
│   │     │     ├── tool_selected: "query_database"               │
│   │     │     └── alternatives_considered: [...]                │
│   │     │                                                       │
│   │     └── [Tool Execution]                                    │
│   │           ├── tool: query_database                          │
│   │           ├── latency_ms: 150                               │
│   │           └── status: success                               │
│   │                                                             │
│   └── [Turn 2 Span] ...                                         │
└─────────────────────────────────────────────────────────────────┘
```

### Enhancement Area 2: AI-Specific Metrics

```yaml
# Token Economics
agent_tokens_total{direction="input|output", model="...", tenant="..."}
agent_cost_usd_total{model="...", tenant="...", task_type="..."}
agent_context_window_utilization{model="..."}  # gauge 0-1

# Agent Behavior
agent_turns_per_session{task_type="..."}  # histogram
agent_tool_calls_per_session{tool="...", status="success|failure"}
agent_loop_detected_total{reason="same_tool|no_progress|max_turns"}

# Quality Signals
agent_task_completion_rate{task_type="...", outcome="success|partial|failure"}
agent_user_feedback_score{task_type="..."}  # if HITL
agent_guardrail_triggered_total{guardrail="...", action="block|warn"}

# Efficiency
agent_tokens_per_successful_task{task_type="..."}  # lower is better
agent_retry_rate{tool="...", reason="..."}
```

### Enhancement Area 3: Evaluation Integration

```
┌─────────────────────────────────────────────────────────────────┐
│                  EVAL-INTEGRATED OBSERVABILITY                  │
│                                                                 │
│   Agent Output ──┬──► Trace Storage                            │
│                  │                                              │
│                  ├──► Real-time Eval Pipeline                   │
│                  │      ├── LLM-as-Judge (async)               │
│                  │      ├── Deterministic checks               │
│                  │      └── Embedding similarity               │
│                  │                                              │
│                  └──► Eval Results ──► Metrics                  │
│                                         │                       │
│                                         ▼                       │
│                              ┌──────────────────┐               │
│                              │ Quality Dashboard│               │
│                              │ • Correctness %  │               │
│                              │ • Hallucination% │               │
│                              │ • Relevance score│               │
│                              └──────────────────┘               │
└─────────────────────────────────────────────────────────────────┘
```

### Enhancement Area 4: AI-Aware Anomaly Detection

| Anomaly | Detection Method |
|---------|------------------|
| Runaway agent | Turn count > 50 OR tokens > 100k in single session |
| Cost spike | Tenant daily spend > 3x rolling 7-day average |
| Tool degradation | Tool p95 latency > 2x baseline |
| Loop detection | Same tool called > 10x with similar args |
| Goal drift | Embedding distance from original task > threshold |

### Enhancement Area 5: Cost Observability

```python
# Cost Attribution Model
cost_per_session = (
    llm_input_tokens * INPUT_PRICE_PER_1K / 1000 +
    llm_output_tokens * OUTPUT_PRICE_PER_1K / 1000 +
    tool_calls * TOOL_OVERHEAD_COST +
    compute_time_seconds * COMPUTE_COST_PER_SEC
)

# Stored as metric with tenant label
metrics.gauge("agent_session_cost_usd", cost_per_session, 
              labels={"tenant_id": tenant_id, "agent_type": agent_type})
```

### Enhancement Area 6: Trace Replay for Debugging

```
Store for each turn:
  ├── Full input context (what agent saw)
  ├── Model response (raw)
  ├── Tool calls + results
  ├── System state at decision point
  └── Timestamps + latencies

Replay capabilities:
  ├── "Show me exactly what the agent saw at turn 3"
  ├── "What was the context window at decision point?"
  ├── "Re-run this turn with current model" (A/B debug)
  └── "Compare reasoning: old vs new prompt"
```

### Enhancement Roadmap

```
Phase 1: Foundation
├── Instrument all LLM calls (tokens, latency, cost)
├── Instrument all tool calls (MCP or custom)
├── Basic dashboards: cost, latency, error rates
└── Simple alerts: cost thresholds, error spikes

Phase 2: AI-Specific
├── Add semantic outcome tracking
├── Context window utilization metrics
├── Loop and goal drift detection
├── Cost attribution by tenant/task
└── Trace replay capability

Phase 3: Quality Integration
├── Async eval pipeline (LLM-as-judge)
├── Quality metrics in dashboards
├── Correlation: quality vs cost vs latency
└── Regression detection on prompt/model changes

Phase 4: Predictive
├── Anomaly detection (ML-based)
├── Cost forecasting
├── Capacity planning based on usage patterns
└── Auto-scaling based on quality signals
```

---

## 4. Environment Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        AGENT PLATFORM ENVIRONMENTS                          │
├──────────────┬──────────────┬───────────────────────────────────────────────┤
│ Environment  │ Cloud        │ Components                                    │
├──────────────┼──────────────┼───────────────────────────────────────────────┤
│ Dev          │ AWS (EKS)    │ Agent Core, Temporal, Registry, Memory DB    │
│ Integration  │ AWS + Azure  │ + ServiceNow, Salesforce, SAP connectors     │
│ Staging      │ AWS + Azure  │ Full stack mirror, security scan, load test  │
│ Production   │ AWS (primary)│ Live workloads, business users               │
│ DR           │ Azure        │ Temporal cluster, Registry replica, WorkOS   │
└──────────────┴──────────────┴───────────────────────────────────────────────┘

Access Matrix:
├── Dev:         Developer role
├── Integration: Developer + Operator
├── Staging:     Operator role only
├── Production:  Operator + Admin; change approval required
└── DR:          Admin only; break-glass access
```

---

## 5. Kubernetes / EKS Deep Dive

### EKS Architecture for AI Agents

```
┌─────────────────────────────────────────────────────────────────┐
│                     EKS AGENT PLATFORM                          │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                 Control Plane (AWS Managed)              │   │
│  │  API Server │ etcd │ Controller Manager │ Scheduler     │   │
│  └─────────────────────────────────────────────────────────┘   │
│                              │                                  │
│  ┌───────────────────────────┴───────────────────────────┐     │
│  │                    Data Plane                          │     │
│  │                                                        │     │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐ │     │
│  │  │ Node Group   │  │ Node Group   │  │ Node Group   │ │     │
│  │  │ (Agents)     │  │ (Temporal)   │  │ (Connectors) │ │     │
│  │  │ c6i.2xlarge  │  │ m6i.xlarge   │  │ t3.large     │ │     │
│  │  │ Spot + OD    │  │ On-Demand    │  │ On-Demand    │ │     │
│  │  └──────────────┘  └──────────────┘  └──────────────┘ │     │
│  │                                                        │     │
│  │  ┌──────────────┐                                      │     │
│  │  │ Fargate      │  (Batch jobs, ephemeral tasks)      │     │
│  │  └──────────────┘                                      │     │
│  └────────────────────────────────────────────────────────┘     │
│                                                                  │
│  Add-ons: ALB Controller, EBS CSI, VPC CNI, CoreDNS, Karpenter  │
└─────────────────────────────────────────────────────────────────┘
```

### Karpenter Provisioner for Agent Workloads

```yaml
apiVersion: karpenter.sh/v1alpha5
kind: Provisioner
metadata:
  name: agent-workloads
spec:
  requirements:
    - key: karpenter.sh/capacity-type
      operator: In
      values: ["spot", "on-demand"]
    - key: node.kubernetes.io/instance-type
      operator: In
      values: ["c6i.xlarge", "c6i.2xlarge", "c6i.4xlarge"]
  limits:
    resources:
      cpu: 1000
      memory: 2000Gi
  ttlSecondsAfterEmpty: 30
  ttlSecondsUntilExpired: 604800  # 7 days
  consolidation:
    enabled: true
```

### Zero-Downtime Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: agent-core
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    spec:
      containers:
      - name: agent
        image: agent-core:v2.1.0
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 15"]
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
      terminationGracePeriodSeconds: 60
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: agent-core-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: agent-core
```

---

## 6. Temporal Workflow Orchestration

### Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    TEMPORAL ARCHITECTURE                        │
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │                 Temporal Cluster                         │  │
│   │  ┌──────────┐  ┌──────────┐  ┌──────────┐              │  │
│   │  │ Frontend │  │ History  │  │ Matching │              │  │
│   │  │ Service  │  │ Service  │  │ Service  │              │  │
│   │  └──────────┘  └──────────┘  └──────────┘              │  │
│   │                      │                                   │  │
│   │              ┌───────┴───────┐                          │  │
│   │              │  Persistence  │                          │  │
│   │              │  (PostgreSQL) │                          │  │
│   │              └───────────────┘                          │  │
│   └─────────────────────────────────────────────────────────┘  │
│                           │                                     │
│          ┌────────────────┼────────────────┐                   │
│          ▼                ▼                ▼                   │
│   ┌────────────┐   ┌────────────┐   ┌────────────┐            │
│   │  Worker 1  │   │  Worker 2  │   │  Worker N  │            │
│   │  (Agents)  │   │  (Agents)  │   │  (Agents)  │            │
│   └────────────┘   └────────────┘   └────────────┘            │
└─────────────────────────────────────────────────────────────────┘
```

### Why Temporal for Agents

| Feature | Benefit for Agents |
|---------|-------------------|
| **Durable execution** | Agent can fail mid-task, resume from exact point |
| **Built-in retries** | LLM API failures handled automatically |
| **Visibility** | Full history of agent decisions |
| **Timeouts** | Prevent runaway agents |
| **Versioning** | Deploy new agent logic without breaking in-flight |

### Multi-Region DR Strategy

```
┌─────────────────────────────────────────────────────────────────┐
│              TEMPORAL MULTI-REGION DR STRATEGY                  │
│                                                                 │
│   AWS (Primary)                    Azure (DR)                   │
│   ┌─────────────────────┐         ┌─────────────────────┐      │
│   │ Temporal Cluster    │         │ Temporal Cluster    │      │
│   │ ┌─────────────────┐ │         │ ┌─────────────────┐ │      │
│   │ │ Frontend (3)    │ │         │ │ Frontend (2)    │ │      │
│   │ │ History (3)     │ │ ──────► │ │ History (2)     │ │      │
│   │ │ Matching (3)    │ │  Async  │ │ Matching (2)    │ │      │
│   │ └─────────────────┘ │  Repli- │ └─────────────────┘ │      │
│   │         │           │  cation │         │           │      │
│   │         ▼           │         │         ▼           │      │
│   │ ┌─────────────────┐ │         │ ┌─────────────────┐ │      │
│   │ │ Aurora PostgreSQL│ │ ──────►│ │ Azure PostgreSQL│ │      │
│   │ └─────────────────┘ │         │ └─────────────────┘ │      │
│   └─────────────────────┘         └─────────────────────┘      │
└─────────────────────────────────────────────────────────────────┘
```

### Key Commands

```bash
# List workflows
tctl workflow list

# Describe task queue
tctl taskqueue describe -tq agent-tasks

# Show workflow history
tctl workflow show -w <workflow-id>

# Terminate a stuck workflow
tctl workflow terminate -w <workflow-id> -r "Manual termination"
```

---

## 7. Agent Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                      AGENT ARCHITECTURE                         │
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │                     AGENT CORE                           │  │
│   │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐  │  │
│   │  │   Planner   │  │   Memory    │  │  Tool Executor  │  │  │
│   │  │ (Reasoning) │  │ (Context)   │  │  (Actions)      │  │  │
│   │  └──────┬──────┘  └──────┬──────┘  └────────┬────────┘  │  │
│   │         │                │                   │           │  │
│   │         └────────────────┼───────────────────┘           │  │
│   │                          │                                │  │
│   │                    ┌─────┴─────┐                         │  │
│   │                    │    LLM    │                         │  │
│   │                    │  (Brain)  │                         │  │
│   │                    └───────────┘                         │  │
│   └─────────────────────────────────────────────────────────┘  │
│                              │                                  │
│          ┌───────────────────┼───────────────────┐             │
│          ▼                   ▼                   ▼             │
│   ┌────────────┐      ┌────────────┐      ┌────────────┐      │
│   │ ServiceNow │      │ Salesforce │      │    SAP     │      │
│   │ Connector  │      │ Connector  │      │ Connector  │      │
│   └────────────┘      └────────────┘      └────────────┘      │
└─────────────────────────────────────────────────────────────────┘
```

### Components

| Component | Purpose | Implementation |
|-----------|---------|----------------|
| **Planner** | Breaks down tasks, decides next action | ReAct pattern |
| **Memory** | Short-term (conversation), Long-term (vector DB) | Memory DB (Redis) |
| **Tool Executor** | Calls external systems | Enterprise connectors |
| **Registry** | Stores agent definitions, versions, configs | Agent Registry |
| **Orchestrator** | Manages agent lifecycle | Temporal |

---

## 8. Enterprise Connector Security

```
┌─────────────────────────────────────────────────────────────────┐
│              ENTERPRISE CONNECTOR SECURITY                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. CREDENTIAL MANAGEMENT                                       │
│     ├── AWS Secrets Manager / Azure Key Vault                  │
│     ├── Rotation: automated, 90-day max                        │
│     ├── Access: IRSA → only connector pods can read            │
│     └── No credentials in container images or env vars         │
│                                                                 │
│  2. NETWORK SECURITY                                            │
│     ├── Private endpoints (AWS PrivateLink, Azure Private Link)│
│     ├── No public internet egress for production               │
│     ├── Network policies: connector → only specific endpoints  │
│     └── mTLS between services                                   │
│                                                                 │
│  3. AUTHORIZATION                                               │
│     ├── OAuth 2.0 with scoped permissions                      │
│     ├── Principle of least privilege per connector             │
│     └── Audit log all connector calls                          │
│                                                                 │
│  4. INPUT VALIDATION                                            │
│     ├── Sanitize all agent-generated queries                   │
│     ├── Prevent prompt injection → SQL injection chain         │
│     └── Rate limiting per agent session                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Secure Connector Pattern

```python
class ServiceNowConnector:
    def __init__(self):
        # Credentials from Secrets Manager, not env vars
        self.credentials = self._load_from_secrets_manager()
        
    def query_incidents(self, query: str, agent_context: AgentContext):
        # 1. Validate and sanitize
        sanitized = self._sanitize_query(query)
        
        # 2. Check permissions
        if not agent_context.has_permission("servicenow:read"):
            raise AuthorizationError("Agent lacks ServiceNow read permission")
        
        # 3. Rate limit
        if not self._check_rate_limit(agent_context.session_id):
            raise RateLimitError("Too many requests")
        
        # 4. Audit log
        self._audit_log(agent_context, "query_incidents", sanitized)
        
        # 5. Execute with timeout
        return self._execute_with_timeout(sanitized, timeout=30)
```

---

## 9. CI/CD Pipeline

```
┌─────────────────────────────────────────────────────────────────┐
│                    CI/CD PIPELINE                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PR Created                                                     │
│      │                                                          │
│      ▼                                                          │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ STAGE 1: Build & Unit Test                              │   │
│  │ ├── Lint (ruff, mypy)                                   │   │
│  │ ├── Unit tests (pytest)                                 │   │
│  │ ├── Build Docker image                                  │   │
│  │ └── Scan image (Trivy, Snyk)                           │   │
│  └─────────────────────────────────────────────────────────┘   │
│      │                                                          │
│      ▼                                                          │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ STAGE 2: Integration Test (Dev Environment)             │   │
│  │ ├── Deploy to dev EKS                                   │   │
│  │ ├── Agent sandbox tests                                 │   │
│  │ └── Connector integration tests                         │   │
│  └─────────────────────────────────────────────────────────┘   │
│      │                                                          │
│      ▼  (Merge to main)                                        │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ STAGE 3: Staging Validation                             │   │
│  │ ├── Deploy to staging (AWS + Azure)                     │   │
│  │ ├── Security scan (SAST, DAST)                         │   │
│  │ ├── Load testing (k6, Locust)                          │   │
│  │ └── Agent evaluation suite                              │   │
│  └─────────────────────────────────────────────────────────┘   │
│      │                                                          │
│      ▼  (Manual approval)                                       │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ STAGE 4: Production Deployment                          │   │
│  │ ├── Change request (ServiceNow integration)             │   │
│  │ ├── Canary deployment (10% → 50% → 100%)               │   │
│  │ └── Automated rollback on error rate spike             │   │
│  └─────────────────────────────────────────────────────────┘   │
│      │                                                          │
│      ▼  (Async)                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ STAGE 5: DR Sync                                        │   │
│  │ └── Replicate to Azure DR environment                   │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### GitHub Actions Example

```yaml
name: Agent CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Lint & Type Check
        run: |
          pip install ruff mypy
          ruff check .
          mypy src/
      
      - name: Unit Tests
        run: pytest tests/unit --cov=src --cov-report=xml
      
      - name: Build Image
        run: docker build -t agent-core:${{ github.sha }} .
      
      - name: Security Scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: agent-core:${{ github.sha }}
          severity: 'CRITICAL,HIGH'
          exit-code: '1'

  deploy-production:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment: 
      name: production
      # Manual approval required
    steps:
      - name: Canary Deploy
        run: |
          kubectl argo rollouts set image agent-core \
            agent=agent-core:${{ github.sha }}
```

---

## 10. Security Considerations

### AI Agent Security Threats

| Threat | Description | Mitigation |
|--------|-------------|------------|
| **Prompt Injection** | Malicious input causes agent to bypass controls | Input sanitization, system prompt defense |
| **Data Exfiltration** | Agent leaks sensitive data via tool calls | Output filtering, DLP integration |
| **Privilege Escalation** | Agent gains access beyond intended scope | Least privilege, scoped credentials |
| **Denial of Service** | Runaway agent consumes resources | Token limits, timeouts, rate limiting |
| **Supply Chain** | Compromised model, dependencies, or tools | SBOM, image scanning, model verification |
| **Tool Abuse** | Agent uses legitimate tools for unintended purposes | Guardrails, human-in-the-loop |

### Security Controls Matrix

| Control | Implementation |
|---------|----------------|
| Authentication | WorkOS, OIDC, mTLS |
| Authorization | RBAC, attribute-based policies |
| Input validation | Guardrails library, regex filters |
| Output filtering | PII detection, secret scanning |
| Audit logging | Every agent action logged |
| Network isolation | VPC, private endpoints, network policies |
| Secrets management | AWS Secrets Manager, no hardcoded creds |
| Runtime protection | Falco, runtime security policies |

---

## 11. Observability Stack

```
┌─────────────────────────────────────────────────────────────────┐
│                 AGENT OBSERVABILITY STACK                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  METRICS (Prometheus/CloudWatch)                                │
│  ├── agent_task_duration_seconds                               │
│  ├── agent_tokens_total{direction="in|out"}                    │
│  ├── agent_tool_calls_total{tool="...",status="..."}           │
│  ├── agent_cost_usd_total{tenant="..."}                        │
│  └── temporal_workflow_status{status="..."}                    │
│                                                                 │
│  TRACES (OpenTelemetry → Jaeger/X-Ray)                         │
│  ├── Full agent session trace                                  │
│  ├── Each LLM call as span                                     │
│  ├── Each tool call as span                                    │
│  └── Temporal workflow correlation                             │
│                                                                 │
│  LOGS (Fluent Bit → OpenSearch/CloudWatch)                     │
│  ├── Structured JSON logs                                      │
│  ├── Agent reasoning (sanitized)                               │
│  ├── Tool call inputs/outputs                                  │
│  └── Error details                                             │
│                                                                 │
│  DASHBOARDS                                                     │
│  ├── Agent health overview                                     │
│  ├── Cost per tenant                                           │
│  ├── Tool success rates                                        │
│  └── Temporal workflow metrics                                 │
│                                                                 │
│  ALERTS                                                         │
│  ├── Error rate > 5%                                           │
│  ├── P95 latency > 30s                                         │
│  ├── Cost spike (>3x baseline)                                 │
│  └── Temporal task queue backlog                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Observability Tools for AI

| Tool | Use Case |
|------|----------|
| **LangSmith** | LangChain tracing platform |
| **Arize Phoenix** | Open-source LLM observability |
| **Datadog LLM Observability** | Enterprise monitoring |
| **Helicone** | Proxy-based LLM observability |
| **OpenTelemetry** | Extend for LLM/agent spans |

---

## 12. AWS Services Reference

### Compute Layer (Where Agents Run)

| Service | Kill/Termination Mechanism | Use Case |
|---------|---------------------------|----------|
| **ECS (Fargate/EC2)** | `StopTask` API, `stopTimeout` config | Containerized agents |
| **Lambda** | Timeout (max 15min), no manual kill | Short-lived agent tasks |
| **EKS** | Pod termination, `terminationGracePeriodSeconds` | K8s-based agents |
| **Step Functions** | `StopExecution` API | Orchestrated agent workflows |

### Data Storage

| Data | Service | Why |
|------|---------|-----|
| Raw traces/logs | **S3** | Cheap, durable, Athena queryable |
| Real-time logs | **CloudWatch Logs** / **OpenSearch** | Live search, dashboards |
| Metrics | **CloudWatch Metrics** / **Timestream** | Native integration |
| Embeddings | **OpenSearch** / **pgvector** | Vector search |
| Session state | **DynamoDB** | Fast, TTL support |
| Model artifacts | **S3** | SageMaker integration |

### Agent Termination Pattern

```python
import boto3
from botocore.config import Config

config = Config(retries={'max_attempts': 3, 'mode': 'exponential_backoff'})
ecs = boto3.client('ecs', config=config)

def terminate_agent(cluster, task_arn, max_wait=60):
    # Step 1: Send stop signal
    ecs.stop_task(cluster=cluster, task=task_arn, reason="User termination")
    
    # Step 2: Wait for task to stop
    waiter = ecs.get_waiter('tasks_stopped')
    try:
        waiter.wait(
            cluster=cluster,
            tasks=[task_arn],
            WaiterConfig={'Delay': 5, 'MaxAttempts': max_wait // 5}
        )
        return True
    except WaiterError:
        return False
```

---

## 13. Incident Response

### Troubleshooting Timeline

```
INCIDENT: Agents timing out
──────────────────────────

T+0: Alert fires - P95 latency > 30s
│
├── IMMEDIATE (First 5 minutes)
│   ├── Check: Is it all agents or specific type?
│   │   └── kubectl get pods -l app=agent-core
│   ├── Check: Temporal task queue depth
│   │   └── tctl taskqueue describe -tq agent-tasks
│   ├── Check: Recent deployments?
│   │   └── kubectl rollout history deployment/agent-core
│   └── Check: Downstream dependencies
│       └── ServiceNow, Salesforce, SAP connector health
│
├── TRIAGE (Next 10 minutes)
│   ├── If Temporal queue backlog: Scale workers
│   │   └── kubectl scale deployment/agent-workers --replicas=10
│   ├── If connector timeout: Check connector metrics
│   ├── If LLM latency: Check Bedrock/OpenAI status
│   └── If memory pressure: Check Memory DB metrics
│
├── MITIGATION
│   ├── If recent deployment: Rollback
│   │   └── kubectl rollout undo deployment/agent-core
│   ├── If external dependency: Enable circuit breaker
│   └── If capacity issue: Scale + page SRE
│
└── POST-INCIDENT
    ├── RCA document
    ├── Timeline of events
    └── Action items to prevent recurrence
```

---

## 14. DR Failover Procedures

```
DR FAILOVER RUNBOOK
───────────────────

PRE-REQUISITES (Continuously verified)
├── Azure Temporal cluster: warm standby
├── Registry replica: synced within 5 minutes
├── WorkOS: backup auth configured
├── DNS: Route 53 health checks active
└── Runbook: tested monthly

EXECUTION STEPS
│
├── STEP 1: Confirm AWS is down (not just monitoring issue)
│   └── Check from multiple sources, cross-reference
│
├── STEP 2: Initiate DNS failover
│   └── Route 53 automatically routes to Azure (if health check)
│   └── Manual: Update DNS TTL was pre-lowered to 60s
│
├── STEP 3: Promote Azure databases
│   └── PostgreSQL: Promote geo-replica to primary
│   └── Redis: Failover to Azure replica
│
├── STEP 4: Verify Temporal cluster
│   └── Workers should reconnect automatically
│   └── In-flight workflows resume from checkpoint
│
├── STEP 5: Verify connectors
│   └── ServiceNow, Salesforce, SAP reachable from Azure
│
├── STEP 6: Smoke tests
│   └── Run agent health check suite
│
├── STEP 7: Notify stakeholders
│   └── Status page update
│
FAILBACK (When AWS recovers)
├── Sync data back to AWS
├── Gradual traffic shift (10% → 50% → 100%)
└── Full validation before completing
```

---

## 15. Mock Interview Q&A

### Technical Questions

**Q1: Your agent pods are running on EKS across multiple AZs. One AZ goes down. What happens?**

> Kubernetes scheduler detects unhealthy nodes and reschedules pods to healthy AZs. PodDisruptionBudget ensures minimum availability. Karpenter provisions new nodes if needed. Temporal workflows continue from last checkpoint on the surviving workers.

**Q2: An agent workflow has been running for 2 hours and seems stuck. How do you investigate?**

> Use `tctl workflow show -w <id>` to see history. Check which activity is pending. Look at worker logs for that activity type. Options: send a signal, terminate with reason, or let it timeout. Temporal's visibility UI shows the full event history.

**Q3: A security audit found agents could query any Salesforce object. How do you fix it?**

> Implement least privilege: create scoped OAuth connected apps with specific object permissions. Add input validation at the connector level. Implement code review requirements for connector permission changes. Add automated security testing in CI.

**Q4: How would you handle secrets rotation without downtime?**

> Use AWS Secrets Manager with automatic rotation. Application reads credentials with caching and grace period. On rotation event, invalidate cache. Dual-credential strategy: new credential is valid before old is revoked. Test rotation in staging first.

**Q5: What's the difference between Deployment and StatefulSet for agents?**

> Deployment: stateless, interchangeable pods, rolling updates. Use for agent workers. StatefulSet: stable network identity, ordered deployment, persistent storage. Use for Temporal cluster nodes or Memory DB if self-hosted.

### Scenario Questions

**Q6: Production incident - Memory DB connection refused at 3 AM. What do you do?**

> Assess scope first - all pods or specific? Check Memory DB status (CloudWatch/Azure Monitor). Possible causes: cluster failure, network issue, credential expiry, max connections. If Redis cluster down, check failover status. If network, check security groups/NSGs. If credentials, trigger rotation. Escalate to SRE if not resolved in 15 minutes.

**Q7: Developers want production data for testing. Security says no. How do you handle?**

> Understand both perspectives. Propose alternatives: anonymized data pipelines, data masking tools, synthetic data generation. Staging environment with realistic but safe data. If production access truly needed, implement audit logging, time-limited access, approval workflow.

### Rapid Fire

**Q8: Rate limiting for agents per tenant?**
> Token bucket algorithm in Redis, keyed by tenant_id. Check before each LLM call. Return 429 if exceeded.

**Q9: Rollback strategy for failed canary?**
> Argo Rollouts automatic rollback on error rate threshold. Metrics-based: if error_rate > 5% for 5 minutes, abort and rollback.

**Q10: How to know if you need more Temporal workers?**
> Monitor task queue schedule-to-start latency. If consistently > 1s, add workers. Also watch worker CPU utilization and workflow completion rate.

---

## 16. Quick Reference Card

```
┌────────────────────────────────────────────────────────────────┐
│              DEVOPS TECH LEAD - QUICK REFERENCE                │
├────────────────────────────────────────────────────────────────┤
│ YOUR STACK                                                     │
│ • Compute: EKS (AWS), AKS (Azure DR)                          │
│ • Orchestration: Temporal                                      │
│ • Storage: Memory DB (Redis), Registry, PostgreSQL            │
│ • Auth: WorkOS                                                 │
│ • Connectors: ServiceNow, Salesforce, SAP                     │
├────────────────────────────────────────────────────────────────┤
│ KEY KUBECTL COMMANDS                                           │
│ • kubectl get pods -n agents -w                               │
│ • kubectl logs -f deployment/agent-core                       │
│ • kubectl rollout undo deployment/agent-core                  │
│ • kubectl scale deployment/agent-workers --replicas=N         │
├────────────────────────────────────────────────────────────────┤
│ TEMPORAL COMMANDS                                              │
│ • tctl workflow list                                          │
│ • tctl taskqueue describe -tq agent-tasks                     │
│ • tctl workflow show -w <workflow-id>                         │
├────────────────────────────────────────────────────────────────┤
│ ENVIRONMENTS & ACCESS                                          │
│ • Dev: Developer role                                         │
│ • Integration: Developer + Operator                           │
│ • Staging: Operator only                                      │
│ • Production: Operator + Admin, change approval               │
│ • DR: Admin only, break-glass                                 │
├────────────────────────────────────────────────────────────────┤
│ KEY METRICS TO MONITOR                                         │
│ • agent_tokens_total (cost)                                   │
│ • agent_task_completion_rate (quality)                        │
│ • temporal_task_queue_depth (capacity)                        │
│ • connector_error_rate (reliability)                          │
├────────────────────────────────────────────────────────────────┤
│ YOUR DIFFERENTIATORS                                           │
│ • Multi-cloud experience (AWS + Azure)                        │
│ • Enterprise connector security                               │
│ • Temporal expertise for agent orchestration                  │
│ • Production DR experience                                    │
│ • AWS DevOps Agent + n8n background                           │
└────────────────────────────────────────────────────────────────┘
```

---

## Interview Sound Bites

### On AI Observability

> "Traditional observability assumes deterministic systems. AI observability needs to handle non-determinism, measure semantic quality, and attribute costs at a granular level."

### On Your Approach

> "I'd approach this from an SRE mindset—define SLOs that make sense for AI (task completion rate, cost per task, quality scores), build the instrumentation to measure them, and create feedback loops."

### On MCP

> "MCP gives us a consistent integration layer. From an observability standpoint, we can instrument tool calls uniformly—capture latency, success/failure, input/output sizes—regardless of what the agent is calling."

### On Agent Termination

> "I'd implement a tiered approach: SIGTERM with timeout, retry, then SIGKILL. In containers, the final backstop is pod deletion. For prevention, I design agents with cooperative cancellation—checking a termination flag between turns."

---

## Resources

- [Concepts-Revision GitHub](https://github.com/athada/Concepts-Revision) - Interview prep notes
- [Temporal Documentation](https://docs.temporal.io/)
- [EKS Best Practices](https://aws.github.io/aws-eks-best-practices/)
- [MCP Specification](https://modelcontextprotocol.io/)

---
