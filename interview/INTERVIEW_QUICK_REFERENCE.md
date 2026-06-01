# Interview Quick Reference Guide
## DevOps Tech Lead - AI Agent Platform

---

## SECTION 1: Kubernetes / EKS Deep Dive

*Reference: Deployment/Kubernetes*

### Q1: Explain EKS architecture for running AI agents at scale

**Answer:**

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

**Key points to mention:**
- Managed node groups for predictable workloads (Temporal workers)
- Karpenter for dynamic scaling of agent workloads
- Spot instances for dev/integration, On-Demand for production
- Pod Disruption Budgets for Temporal to prevent split-brain
- IRSA (IAM Roles for Service Accounts) for least-privilege access

---

### Q2: How would you handle node scaling for bursty agent workloads?

**Answer:**

```yaml
# Karpenter Provisioner for Agent Workloads
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
  
  # Consolidation for cost optimization
  consolidation:
    enabled: true
```

**Talking points:**
- Karpenter over Cluster Autoscaler for faster scaling (seconds vs minutes)
- Mixed instance types for better spot availability
- Consolidation to bin-pack and reduce costs
- Separate provisioners for different workload types

---

### Q3: How do you implement zero-downtime deployments for agents?

**Answer:**

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
        
        # Graceful shutdown for in-flight agent tasks
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 15"]
        
        # Health checks
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

## SECTION 2: Temporal (Workflow Orchestration)

### Q4: Explain Temporal architecture and why it's used for agents

**Answer:**

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
│   │              │  (PostgreSQL/ │                          │  │
│   │              │   Cassandra)  │                          │  │
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

**Why Temporal for Agents:**

| Feature | Benefit for Agents |
|---------|-------------------|
| **Durable execution** | Agent can fail mid-task, resume from exact point |
| **Built-in retries** | LLM API failures handled automatically |
| **Visibility** | Full history of agent decisions |
| **Timeouts** | Prevent runaway agents |
| **Versioning** | Deploy new agent logic without breaking in-flight |

---

### Q5: How would you handle Temporal cluster DR between AWS and Azure?

**Answer:**

```
┌─────────────────────────────────────────────────────────────────┐
│              TEMPORAL MULTI-REGION DR STRATEGY                  │
│                                                                 │
│   AWS (Primary)                    Azure (DR)                   │
│   ┌─────────────────────┐         ┌─────────────────────┐      │
│   │ Temporal Cluster    │         │ Temporal Cluster    │      │
│   │ ┌─────────────────┐ │         │ ┌─────────────────┐ │      │
│   │ │ Frontend (3)    │ │         │ │ Frontend (2)    │ │      │
│   │ │ History (3)     │ │         │ │ History (2)     │ │      │
│   │ │ Matching (3)    │ │ ──────► │ │ Matching (2)    │ │      │
│   │ └─────────────────┘ │  Async  │ └─────────────────┘ │      │
│   │         │           │  Repli- │         │           │      │
│   │         ▼           │  cation │         ▼           │      │
│   │ ┌─────────────────┐ │         │ ┌─────────────────┐ │      │
│   │ │ Aurora PostgreSQL│ │ ──────►│ │ Azure PostgreSQL│ │      │
│   │ │ (Multi-AZ)      │ │         │ │ (Geo-replica)   │ │      │
│   │ └─────────────────┘ │         │ └─────────────────┘ │      │
│   └─────────────────────┘         └─────────────────────┘      │
│                                                                 │
│   Failover Strategy:                                            │
│   1. DNS failover (Route 53 health checks)                     │
│   2. Promote Azure replica to primary                          │
│   3. Workers reconnect via new endpoint                        │
│   4. In-flight workflows resume from last checkpoint           │
└─────────────────────────────────────────────────────────────────┘
```

**Key points:**
- Temporal's event sourcing means state is in the persistence layer
- Workers are stateless—just point them at new cluster
- RPO depends on replication lag (typically seconds)
- RTO depends on DNS TTL and promotion time

---

## SECTION 3: GenAI / Agents

*Reference: GenAI/Agents*

### Q6: What are the key components of an Agent architecture?

**Answer:**

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

**Components explained:**

| Component | Purpose | Your Stack |
|-----------|---------|------------|
| **Planner** | Breaks down tasks, decides next action | ReAct pattern |
| **Memory** | Short-term (conversation), Long-term (vector DB) | Memory DB |
| **Tool Executor** | Calls external systems | Enterprise connectors |
| **Registry** | Stores agent definitions, versions, configs | Agent Registry |
| **Orchestrator** | Manages agent lifecycle | Temporal |

---

### Q7: How do you secure enterprise connectors (ServiceNow, Salesforce, SAP)?

**Answer:**

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
│     ├── ServiceNow: read-only for queries, scoped write        │
│     └── Audit log all connector calls                          │
│                                                                 │
│  4. INPUT VALIDATION                                            │
│     ├── Sanitize all agent-generated queries                   │
│     ├── Prevent prompt injection → SQL injection chain         │
│     └── Rate limiting per agent session                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Code example - Secure connector pattern:**

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

## SECTION 4: Docker & CI/CD

*Reference: Deployment/Docker, Deployment/GitHub-Actions*

### Q8: Walk me through your CI/CD pipeline for agent deployments

**Answer:**

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
│  │ ├── Agent evaluation suite                              │   │
│  │ └── Cross-agent testing                                 │   │
│  └─────────────────────────────────────────────────────────┘   │
│      │                                                          │
│      ▼  (Manual approval)                                       │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ STAGE 4: Production Deployment                          │   │
│  │ ├── Change request (ServiceNow integration)             │   │
│  │ ├── Canary deployment (10% → 50% → 100%)               │   │
│  │ ├── Automated rollback on error rate spike             │   │
│  │ └── Post-deployment smoke tests                         │   │
│  └─────────────────────────────────────────────────────────┘   │
│      │                                                          │
│      ▼  (Async)                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ STAGE 5: DR Sync                                        │   │
│  │ └── Replicate to Azure DR environment                   │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**GitHub Actions example:**

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

  deploy-staging:
    needs: build-test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - name: Deploy to Staging
        run: |
          kubectl set image deployment/agent-core \
            agent=agent-core:${{ github.sha }}
          kubectl rollout status deployment/agent-core

  deploy-production:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment: 
      name: production
      # Manual approval required
    steps:
      - name: Canary Deploy
        run: |
          # Argo Rollouts canary deployment
          kubectl argo rollouts set image agent-core \
            agent=agent-core:${{ github.sha }}
```

---

## SECTION 5: Security

*Reference: GenAI/Security*

### Q9: What are the key security concerns for AI agents in enterprise?

**Answer:**

```
┌─────────────────────────────────────────────────────────────────┐
│                 AI AGENT SECURITY THREATS                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. PROMPT INJECTION                                            │
│     └── Malicious input causes agent to bypass controls        │
│     └── Mitigation: Input sanitization, system prompt defense  │
│                                                                 │
│  2. DATA EXFILTRATION                                           │
│     └── Agent leaks sensitive data via tool calls              │
│     └── Mitigation: Output filtering, DLP integration          │
│                                                                 │
│  3. PRIVILEGE ESCALATION                                        │
│     └── Agent gains access beyond intended scope               │
│     └── Mitigation: Least privilege, scoped credentials        │
│                                                                 │
│  4. DENIAL OF SERVICE                                           │
│     └── Runaway agent consumes resources                       │
│     └── Mitigation: Token limits, timeouts, rate limiting      │
│                                                                 │
│  5. SUPPLY CHAIN                                                │
│     └── Compromised model, dependencies, or tools              │
│     └── Mitigation: SBOM, image scanning, model verification   │
│                                                                 │
│  6. TOOL ABUSE                                                  │
│     └── Agent uses legitimate tools for unintended purposes    │
│     └── Mitigation: Guardrails, human-in-the-loop              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Security controls matrix:**

| Control | Implementation |
|---------|----------------|
| **Authentication** | WorkOS (your stack), OIDC, mTLS |
| **Authorization** | RBAC, attribute-based policies |
| **Input validation** | Guardrails library, regex filters |
| **Output filtering** | PII detection, secret scanning |
| **Audit logging** | Every agent action logged |
| **Network isolation** | VPC, private endpoints, network policies |
| **Secrets management** | AWS Secrets Manager, no hardcoded creds |
| **Runtime protection** | Falco, runtime security policies |

---

## SECTION 6: Observability

### Q10: How do you monitor agents in production?

**Answer:**

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

---

## SECTION 7: Scenario Questions

### Q11: Production incident - Agents are timing out. Walk me through your troubleshooting.

**Answer:**

```
INCIDENT RESPONSE TIMELINE
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
│   │   └── Rate limiting? Auth failure? Network issue?
│   ├── If LLM latency: Check Bedrock/OpenAI status
│   │   └── Fallback to secondary model if configured
│   └── If memory pressure: Check Memory DB metrics
│       └── Redis cluster health, eviction rates
│
├── MITIGATION
│   ├── If recent deployment: Rollback
│   │   └── kubectl rollout undo deployment/agent-core
│   ├── If external dependency: Enable circuit breaker
│   │   └── Return cached/degraded response
│   └── If capacity issue: Scale + page SRE
│
└── POST-INCIDENT
    ├── RCA document
    ├── Timeline of events
    ├── What detection/monitoring was missing?
    └── Action items to prevent recurrence
```

---

### Q12: How would you handle a DR failover to Azure?

**Answer:**

```
DR FAILOVER RUNBOOK
───────────────────

PRE-REQUISITES (Continuously verified)
├── Azure Temporal cluster: warm standby
├── Registry replica: synced within 5 minutes
├── WorkOS: backup auth configured
├── DNS: Route 53 health checks active
└── Runbook: tested monthly

FAILOVER TRIGGER
├── AWS region outage (confirmed via status page + monitoring)
├── OR: Manual decision by incident commander

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
│   └── May need different credentials (check Vault)
│
├── STEP 6: Smoke tests
│   └── Run agent health check suite
│   └── Test one connector each
│
├── STEP 7: Notify stakeholders
│   └── Status page update
│   └── Internal comms
│
FAILBACK (When AWS recovers)
├── Sync data back to AWS
├── Gradual traffic shift (10% → 50% → 100%)
└── Full validation before completing
```

---

## Quick Reference Card

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
│ YOUR DIFFERENTIATORS                                           │
│ • Multi-cloud experience (AWS + Azure)                        │
│ • Enterprise connector security                               │
│ • Temporal expertise for agent orchestration                  │
│ • Production DR experience                                    │
└────────────────────────────────────────────────────────────────┘
```

---
