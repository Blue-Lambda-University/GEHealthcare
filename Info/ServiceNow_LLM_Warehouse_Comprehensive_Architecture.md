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

------------------------------------------------------------------------

# ServiceNow + LLM Warehouse Integration Design

## With OTM / xCarrier / MSCA Integration

------------------------------------------------------------------------

# 1. Warehouse Systems Integration

## A. High-Value Signals to Pull

### OTM (Transportation Management)

-   Shipment backlog count (by site)
-   Late shipment / SLA-at-risk count
-   Failed dispatch / tender events
-   Integration error rate
-   Throughput vs baseline (last 30--60 min)

### xCarrier (Shipping Execution)

-   Label generation failure rate
-   Carrier rate/shop failures
-   Print queue depth / error rate
-   Station-level error bursts

### MSCA (Warehouse Execution / RF Scanning)

-   Scanner sync failures
-   Login failures
-   Transaction queue depth
-   RF connectivity errors
-   Pick/pack/ship transaction latency
-   Users impacted per station/zone

------------------------------------------------------------------------

# 2. Warehouse Health Snapshot (Normalized Schema)

Example normalized object:

-   site_id
-   warehouse_type (services / other)
-   ops_state (running / degraded / halted)
-   stations_impacted_count
-   backlog_delta_per_hour
-   failed_tx_rate
-   top_error_codes
-   workaround_present (true/false)
-   workaround_quality (stable / partial / risky)
-   last_healthy_timestamp

This object is attached to each incident.

------------------------------------------------------------------------

# 3. ServiceNow Data Model Options

## Option 1 -- Custom Incident Fields (Fastest Start)

Add fields: - u_warehouse_site - u_warehouse_flag - u_ops_state -
u_stations_impacted - u_backlog_risk - u_ai_priority_recommendation -
u_ai_confidence

Attach JSON snapshot in work notes or attachment.

## Option 2 -- Dedicated Table (Scalable)

Create table: `u_warehouse_health_snapshot`

Benefits: - Better reporting - Better pattern detection - Cleaner
long-term architecture

------------------------------------------------------------------------

# 4. Decision Logic Architecture

## Deterministic Policy Rules

1.  Services Warehouse → Minimum P2
2.  Operations halted OR station threshold exceeded → P1
3.  P2 (warehouse) → Start 2-hour monitor
4.  Security/compliance risk → Force P1

## LLM Responsibilities

-   Summarize incident + snapshot
-   Recommend priority (within policy limits)
-   Ask missing information questions
-   Draft communication messages
-   Provide justification

------------------------------------------------------------------------

# 5. ServiceNow Workflows

## Flow 1 -- Warehouse Incident Intake

Trigger: - Incident created - Category = Warehouse/Logistics - Caller
group = Warehouse Ops

Steps: 1. Identify warehouse site 2. Pull health snapshot 3. Apply
policy rules 4. Call LLM 5. Update incident fields 6. Notify resolver
groups

------------------------------------------------------------------------

## Flow 2 -- 2-Hour Warehouse Monitor

Trigger: - Monitoring deadline reached - Scheduled check every 15 min

Steps: 1. Refresh health snapshot 2. Detect impact growth 3. Escalate to
P1 if needed 4. Post automated update

------------------------------------------------------------------------

## Flow 3 -- P1 Major Incident Automation

Trigger: - Priority = P1

Steps: - Open bridge - Notify on-call - Notify leadership - Start update
cadence - Maintain executive summary

------------------------------------------------------------------------

# 6. Example Operational Thresholds (Starter Defaults)

-   Ops halted if:
    -   Shipments = 0 for 15 min OR
    -   Failure rate \> 80% for 10 min
-   Stations threshold for P1:
    -   = 3 stations impacted OR

    -   = 25% of active stations
-   Backlog risk:
    -   1.5× baseline throughput

These thresholds should be tuned using historical warehouse data.

------------------------------------------------------------------------

# Conclusion

This integration model:

-   Leverages existing ServiceNow modules
-   Integrates real-time warehouse signals
-   Enforces deterministic warehouse policies
-   Uses LLM safely for summarization and reasoning
-   Scales from assistive AI to intelligent orchestration
