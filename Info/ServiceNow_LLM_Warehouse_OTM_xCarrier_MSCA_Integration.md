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
