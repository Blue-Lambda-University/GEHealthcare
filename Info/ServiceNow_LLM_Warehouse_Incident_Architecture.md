# ServiceNow + LLM Integration Architecture

## Warehouse Incident Management

------------------------------------------------------------------------

# 1. Architecture Overview

## Core Principle

ServiceNow remains the **System of Record**.\
The LLM acts as a **decision-support and orchestration layer** with
strong guardrails and auditability.

------------------------------------------------------------------------

## A. ServiceNow (System of Record)

-   Incident Table (`incident`)
-   Major Incident module
-   CMDB / Service Mapping
-   Knowledge Base / Runbooks
-   Assignment Groups & On-call schedules

------------------------------------------------------------------------

## B. Integration Layer (Middleware Recommended)

-   API Gateway (authentication, rate limiting, logging)
-   Orchestrator service:
    -   Pulls ticket + context
    -   Applies policy engine
    -   Calls LLM
    -   Writes results back to ServiceNow
-   Optional: Event queue for retry and decoupling

------------------------------------------------------------------------

## C. Policy & Rules Engine (Deterministic)

Hard-coded warehouse policies:

-   Services Warehouse → Minimum Priority = P2
-   Operations halted or multiple stations blocked → P1
-   2-hour monitoring rule for warehouse P2

This ensures safety and overrides LLM improvisation.

------------------------------------------------------------------------

## D. LLM Layer

-   Structured JSON output
-   Confidence scoring
-   Missing information detection
-   Optional RAG for internal SOP grounding

Example JSON Output:

``` json
{
  "location": "Warehouse - Chicago",
  "impact_scope": "1 shipping station",
  "operations_status": "degraded but running",
  "priority_recommendation": "P2",
  "justification": "Warehouse policy: minor issue auto P2",
  "confidence": "High"
}
```

------------------------------------------------------------------------

## E. Action Layer

Automations allowed:

-   Update incident fields
-   Add work notes
-   Draft communications
-   Start monitoring timer
-   Notify assignment groups

Restricted actions:

-   No production restarts
-   No inventory data changes
-   No automatic downgrade of P1 without approval

------------------------------------------------------------------------

## F. Governance & Observability

-   Full audit logging
-   PHI/PII redaction
-   Access control
-   Model performance tracking
-   Escalation review process

------------------------------------------------------------------------

# 2. Decision Flow Diagram

``` mermaid
flowchart TD
  A[Incident Created] --> B[Enrich Context]
  B --> C{Services Warehouse?}
  C -- Yes --> D[Minimum P2]
  C -- No --> E[Standard ITIL Scoring]

  D --> F{Operations Halted?}
  F -- Yes --> G[P1 - Major Incident Workflow]
  F -- No --> H[P2 - Start 2hr Monitor]

  H --> I{Resolved in 2 hrs?}
  I -- Yes --> J[Close / Downshift]
  I -- No --> K{Impact Growing?}
  K -- Yes --> G
  K -- No --> L[Continue P2 Monitoring]
```

------------------------------------------------------------------------

# 3. Implementation Roadmap

------------------------------------------------------------------------

## Phase 1 -- Copilot (Assistive Mode)

### Capabilities

-   AI suggests priority (human approves)
-   Warehouse policy reminders
-   Draft communication templates
-   Missing information prompts

### Automation Level

-   Writes work notes only
-   No automatic priority changes

### Success Metrics

-   Faster triage
-   Improved classification accuracy

------------------------------------------------------------------------

## Phase 2 -- Guarded Autopilot

### Capabilities

-   Auto-set P2 for warehouse incidents
-   Auto-start 2-hour monitoring
-   Auto-notify correct resolver groups
-   Escalation recommendation to P1

### Controls

-   Approval required for Major Incident declaration
-   Structured output validation

### Success Metrics

-   SLA breaches reduced
-   Escalation latency reduced

------------------------------------------------------------------------

## Phase 3 -- Major Incident Agent

### Capabilities

-   Cross-incident pattern detection
-   Throughput/backlog metric analysis
-   Automated incident bridge creation
-   Post-incident report draft generation

### Governance

-   Strict tool allowlist
-   Mandatory evidence citation
-   Executive oversight

### Success Metrics

-   MTTR reduced
-   Improved postmortem quality
-   Reduced repeat incidents

------------------------------------------------------------------------

# Conclusion

This architecture ensures:

-   Policy-aligned warehouse incident handling
-   Reduced manual triage burden
-   Controlled AI automation
-   Strong compliance and audit readiness
-   Scalable maturity from assistive AI to agentic orchestration
