# ServiceNow + LLM Warehouse Incident Design (Combined)

**Context:** GE HealthCare – Warehouse incident management and P1/P2/P3 classification  
**Purpose:** Single combined design document for the ServiceNow + LLM warehouse integration: base integration (OTM/xCarrier/MSCA), enriched context (ServiceNow + logs + knowledge base), and LLM-based priority classification.  
**Status:** Design only; no code.

---

## Table of contents

1. **Part I – Base integration:** Warehouse systems (OTM, xCarrier, MSCA), health snapshot, ServiceNow data model, decision logic, workflows, thresholds.  
2. **Part II – Enriched context for LLM:** What to collect from ServiceNow (incident, CMDB, related, knowledge base, warehouse snapshot) and logs; payload shape; where knowledge base fits; flow.  
3. **Part III – LLM-based priority classification:** How the LLM can classify P1/P2/P3; structured output; three usage options; guardrails and audit.  
4. **Related references.**

---

# Part I – ServiceNow + LLM Warehouse Integration (OTM / xCarrier / MSCA)

------------------------------------------------------------------------

## I.1 Warehouse Systems Integration

### High-Value Signals to Pull

**OTM (Transportation Management)**  
- Shipment backlog count (by site)  
- Late shipment / SLA-at-risk count  
- Failed dispatch / tender events  
- Integration error rate  
- Throughput vs baseline (last 30–60 min)

**xCarrier (Shipping Execution)**  
- Label generation failure rate  
- Carrier rate/shop failures  
- Print queue depth / error rate  
- Station-level error bursts  

**MSCA (Warehouse Execution / RF Scanning)**  
- Scanner sync failures  
- Login failures  
- Transaction queue depth  
- RF connectivity errors  
- Pick/pack/ship transaction latency  
- Users impacted per station/zone  

------------------------------------------------------------------------

## I.2 Warehouse Health Snapshot (Normalized Schema)

- site_id  
- warehouse_type (services / other)  
- ops_state (running / degraded / halted)  
- stations_impacted_count  
- backlog_delta_per_hour  
- failed_tx_rate  
- top_error_codes  
- workaround_present (true/false)  
- workaround_quality (stable / partial / risky)  
- last_healthy_timestamp  

This object is attached to each incident.

------------------------------------------------------------------------

## I.3 ServiceNow Data Model Options

**Option 1 – Custom Incident Fields (Fastest Start)**  
Add fields: u_warehouse_site, u_warehouse_flag, u_ops_state, u_stations_impacted, u_backlog_risk, u_ai_priority_recommendation, u_ai_confidence. Attach JSON snapshot in work notes or attachment.

**Option 2 – Dedicated Table (Scalable)**  
Create table: `u_warehouse_health_snapshot`. Benefits: better reporting, pattern detection, cleaner long-term architecture.

------------------------------------------------------------------------

## I.4 Decision Logic Architecture

**Deterministic Policy Rules**  
1. Services Warehouse → Minimum P2  
2. Operations halted OR station threshold exceeded → P1  
3. P2 (warehouse) → Start 2-hour monitor  
4. Security/compliance risk → Force P1  

**LLM Responsibilities**  
- Summarize incident + snapshot  
- Recommend priority (within policy limits)  
- Ask missing information questions  
- Draft communication messages  
- Provide justification  
- Use **knowledge base** (runbooks/articles) to suggest next steps and ground drafts in approved procedures (knowledge snippets included in payload; see Part II).  

------------------------------------------------------------------------

## I.5 ServiceNow Workflows

**Flow 1 – Warehouse Incident Intake**  
Trigger: Incident created; Category = Warehouse/Logistics; Caller group = Warehouse Ops.  
Steps: 1. Identify warehouse site 2. Pull health snapshot 3. Pull relevant knowledge base articles (by category, CI, or search) 4. Apply policy rules 5. Call LLM (with incident + snapshot + knowledge snippets + logs) 6. Update incident fields 7. Notify resolver groups  

**Flow 2 – 2-Hour Warehouse Monitor**  
Trigger: Monitoring deadline reached; scheduled check every 15 min.  
Steps: 1. Refresh health snapshot 2. Detect impact growth 3. Escalate to P1 if needed 4. Post automated update  

**Flow 3 – P1 Major Incident Automation**  
Trigger: Priority = P1.  
Steps: Open bridge; notify on-call; notify leadership; start update cadence; maintain executive summary  

------------------------------------------------------------------------

## I.6 Example Operational Thresholds (Starter Defaults)

- Ops halted if: Shipments = 0 for 15 min OR Failure rate > 80% for 10 min  
- Stations threshold for P1: ≥ 3 stations impacted OR ≥ 25% of active stations  
- Backlog risk: e.g. 1.5× baseline throughput  

Tune using historical warehouse data.

------------------------------------------------------------------------

# Part II – Enriched Context for LLM: ServiceNow + Logs + Knowledge Base

**Idea:** Collect all relevant information from ServiceNow plus logs (and optionally other sources), package it, and feed that enriched payload to the LLM.  
**Principle:** Policy rules still run first and set P1/P2/P3 bounds; the LLM works within those bounds.

------------------------------------------------------------------------

## II.1 What to Collect and Feed to the LLM

### A. From ServiceNow

| Source | What to pull | Purpose for LLM |
|--------|----------------|------------------|
| **Incident** | Short/full description, state, priority, category, subcategory, assignment group, assigned to, caller, opened/updated time, work notes (recent), closure notes | Core incident narrative and history. |
| **CMDB / Service mapping** | Affected CI(s), business service, related applications (OTM, xCarrier, MSCA), site/location | What is impacted and where. |
| **Related records** | Linked incidents, problems, changes; same CI or same site | Pattern: recurring? related outage? |
| **Assignment & on-call** | Resolver group, on-call schedule, escalation history | Who is handling it; LLM can reference in drafts. |
| **Knowledge / Runbooks** | Articles or runbooks linked to category, CI, or keyword-matched | Ground LLM in approved procedures; suggest next steps. |
| **Warehouse-specific fields** | u_warehouse_site, u_warehouse_flag, u_ops_state, u_stations_impacted, u_backlog_risk, u_ai_priority_recommendation, u_ai_confidence | Include in payload. |
| **Warehouse health snapshot** | Full normalized object (site_id, ops_state, stations_impacted_count, backlog_delta, failed_tx_rate, top_error_codes, workaround_present, workaround_quality, etc.) | Real-time warehouse state. |

So “all info from ServiceNow” = incident + CMDB/CI + related tickets + assignment + **knowledge base** + warehouse fields and health snapshot.

### Where the Knowledge Base fits (detailed)

1. **What we pull:** Articles or runbooks linked to the incident’s category, subcategory, affected CI, or keyword-matched from the incident description (e.g. scanner, label printer, OTM).  
2. **When we pull it:** In the orchestrator pull step: when pulling ServiceNow context, also retrieve relevant Knowledge articles (by category, CI, or search on short_description/description) and attach to the payload.  
3. **How we feed it to the LLM:** In the payload as **`knowledge_snippets`**: array of `{ title, snippet }` (or `{ kb_number, title, snippet, url }`) so the LLM can ground summary, next steps, and draft comms in approved procedures.  

**Retrieval options:** (a) By linkage – articles linked to incident or affected CI (cap N articles, snippet length). (b) By category/CI – articles tagged with incident category (e.g. Warehouse/Logistics) or affected app (OTM, xCarrier, MSCA). (c) By search – keyword/full-text on short_description/description; cap articles and total length for token budget.  

**How the LLM uses it:** Suggest next steps or runbook steps that match approved procedures, improve draft communications (e.g. refer to KB article X for scanner reset), avoid contradicting internal SOPs. Optionally use same knowledge for RAG in addition to pre-fetching.

### B. Logs (and similar)

| Source | What to pull | Purpose for LLM |
|--------|----------------|------------------|
| **Application logs** | Errors, warnings, exceptions for affected app (OTM, xCarrier, MSCA) in time window (e.g. 30–60 min before incident opened) | Correlate “what broke” with incident. |
| **Integration logs** | Failed API calls, timeouts, retries between warehouse systems | Explain integration lag or sync failures. |
| **Infrastructure / platform logs** | Server, container, or middleware errors for CIs linked to incident | Distinguish app vs infra. |
| **Audit / access logs** | Login failures, permission errors | Support “device login issue” type incidents. |

Scope logs by site, CI, time window, log level; size-limit (e.g. last N lines or token budget).

### C. Optional: Other signals

- Monitoring/APM: alerts or metrics for the same time window.  
- Recent changes: changes that touched affected CI(s) in last 24–48 hours.

------------------------------------------------------------------------

## II.2 How It Fits in the Flow

1. Incident created or updated (e.g. Warehouse/Logistics, Warehouse Ops).  
2. Orchestrator pulls ServiceNow context (incident + CMDB + related + assignment + **knowledge** + warehouse fields + health snapshot) and pulls logs (and optionally other signals) for the same time window and scope.  
3. Policy engine runs on structured data and sets deterministic P1/P2/P3 (or min priority and 2hr monitor).  
4. Enriched payload is built: structured fields + incident text + work notes + health snapshot + log excerpts + **knowledge_snippets**.  
5. LLM receives policy outcome + enriched context; instruction: summarize, recommend priority within policy, justify, ask for missing info, draft comms.  
6. LLM output is written back to ServiceNow; priority remains the one set by rules unless a human overrides.

------------------------------------------------------------------------

## II.3 Payload Shape (Conceptual)

- **incident:** { number, short_description, description, state, priority, category, caller, opened_at, updated_at, work_notes_snippet, assignment_group, assigned_to, … }  
- **affected_ci:** { name, business_service, applications[], site }  
- **related:** [ { type, number, state, short_description } ] (e.g. last 5)  
- **warehouse_health_snapshot:** { site_id, warehouse_type, ops_state, stations_impacted_count, … }  
- **policy_result:** { applied_priority, reason, rule_triggered }  
- **logs:** { source, time_window, excerpt, error_codes }  
- **knowledge_snippets:** [ { title, snippet } ] (from runbooks/knowledge)  
- **token_budget_note:** e.g. "Log excerpt truncated to 2000 chars; full logs in attachment."  

------------------------------------------------------------------------

## II.4 Log Collection: Practical Notes

- Where: app servers, log aggregation (Splunk, Elastic, Datadog, etc.) or SIEM; orchestrator or log service calls the API/query.  
- Scoping: filter by time (e.g. 1 hour before incident opened), site/CI/app, level (e.g. ERROR, WARN).  
- Size and safety: truncate or summarize; redact PHI/PII; avoid huge raw logs in one call.  
- Failure: if log fetch fails, still send ServiceNow + health snapshot; optionally add “Logs unavailable: &lt;reason&gt;” so the LLM does not invent log content.

------------------------------------------------------------------------

## II.5 Benefits

- Richer context: LLM sees full picture (ticket + systems + logs + knowledge).  
- Better correlation: log errors and incident description aligned.  
- Missing information: LLM can point out gaps.  
- Draft quality: comms can reference specific errors, time windows, and KB articles.  
- Safety: priority stays rule-based unless human overrides.

------------------------------------------------------------------------

# Part III – LLM-Based Incident Priority Classification (P1 / P2 / P3)

**Goal:** How the LLM can classify incident priority (P1, P2, P3) instead of or in addition to purely rule-based classification.  
**Constraint:** Design only; no code.

------------------------------------------------------------------------

## III.1 Why Use the LLM for Classification

- **Nuance:** Free-text, work notes, and logs contain context rules can miss.  
- **Consistency:** LLM can apply same policy language across many phrasings.  
- **Explainability:** Output both priority and justification for audit and trust.  
- **Evolving policy:** Update policy in prompts or RAG without rewriting rule code.  

Trade-off: rules are deterministic and auditable; LLM can drift or hallucinate → need guardrails and human-in-the-loop where appropriate.

------------------------------------------------------------------------

## III.2 What the LLM Needs to Classify

- Impact (scope, criticality, upstream/downstream)  
- Urgency (time sensitivity, SLA at risk, shipping delays)  
- Business function affected (e.g. logistics, warehouse)  
- Security/compliance risk  
- Location/context (services warehouse? site?)  
- Operations status (running / degraded / halted); stations impacted count  
- Workaround (present? quality? scalable?)  
- Raw context: incident description, work notes, warehouse health snapshot, log excerpts  

So: the **enriched payload** (Part II) **plus** a clear **policy summary** (or RAG-retrieved Incident Priority Definitions) so the LLM knows P1 vs P2 vs P3 and warehouse exceptions.

------------------------------------------------------------------------

## III.3 Structured Output from the LLM

- **priority:** "P1" | "P2" | "P3"  
- **confidence:** "High" | "Medium" | "Low"  
- **justification:** Short explanation citing policy and evidence  
- **applied_rules_or_criteria:** Optional list for audit  
- **suggested_actions:** Optional (e.g. Start 2-hour monitor, Notify on-call)  
- **missing_information:** Optional (e.g. Unclear if workaround is scalable)  

Ask the LLM for JSON conforming to this schema; validate JSON and priority enum before use.

------------------------------------------------------------------------

## III.4 Three Ways to Use the LLM for P1/P2/P3

**Option A – LLM as primary classifier (human in the loop)**  
Flow: Enriched context + policy summary → LLM → P1/P2/P3 + justification + confidence. System suggests; human approves or overrides before saving. Guardrails: low confidence → human review; P1 or downgrade from P1 → explicit approval; full audit.  

**Option B – LLM + rules: LLM suggests, rules can override**  
Flow: Enriched context → LLM suggests priority; policy engine runs on structured data. If rule says P1 → always P1. If LLM says P1 but rules don’t → escalate to human or treat as P2 until confirmed. Otherwise use LLM suggestion (or stricter), with optional human review for P1 or downgrades. Rules act as safety floor/ceiling (e.g. services warehouse never below P2).  

**Option C – LLM only for “edge” cases; rules for the rest**  
Flow: Rules run first. If rules have enough data and unambiguous result → use it. If ambiguous (e.g. missing ops_state) → call LLM with enriched context; use as suggested priority with human approval for P1 or downgrade. Fewer LLM calls; rules handle clear cases.

------------------------------------------------------------------------

## III.5 Making the LLM Classification Reliable

- **Policy in prompt (or RAG):** Include concise summary of Incident Priority Definitions and GSPO warehouse exceptions.  
- **Evidence in payload:** Warehouse snapshot, log excerpts, affected CI, site so LLM can cite them; redact PHI/PII.  
- **Structured output and validation:** Require JSON; validate; reject malformed or invalid priority; apply guardrails (e.g. don’t auto-apply P1 if confidence Low).  
- **Confidence and human escalation:** Low confidence or critical missing_information → do not auto-apply; send to human. Optional: if LLM says P1 and rules didn’t → always human confirm.  
- **Audit and tuning:** Log incident ID, payload summary (no PHI), model, full LLM output, final priority, who approved/overrode; use to tune prompts or rules.

------------------------------------------------------------------------

## III.6 Summary: LLM Classifying P1/P2/P3

| Aspect | Recommendation |
|--------|----------------|
| **Input** | Enriched context (ServiceNow + logs + knowledge) + explicit policy summary (Incident Priority Definitions, warehouse exceptions). |
| **Output** | Structured: priority (P1/P2/P3), confidence, justification, optional applied_criteria, missing_information, suggested_actions. |
| **Usage** | (A) LLM suggests, human always approves; (B) LLM suggests, rules override and can force human for P1; (C) Rules first, LLM only for ambiguous cases. |
| **Safety** | Rules can enforce floor/ceiling; no auto-downgrade from P1 without approval; low confidence → human review. |
| **Audit** | Log input summary, model, output, final priority, overrides. |

------------------------------------------------------------------------

# Related References

- **Incident Priority Definitions** – GSPO warehouse P1/P2/P3 rules and workaround logic (referenced by decision logic in Part I).  
- **docs/Incident-Priority-Automation-Design.md** – Automation design that references this integration and the Info folder.  
- **Info/** – Original architecture and integration docs.  

*This combined document merges content from README.md, 01-ServiceNow-LLM-Warehouse-OTM-xCarrier-MSCA-Integration.md, 02-Enriched-Context-for-LLM-ServiceNow-and-Logs.md, and 03-LLM-Based-Priority-Classification-P1-P2-P3.md. Design only; no code.*
