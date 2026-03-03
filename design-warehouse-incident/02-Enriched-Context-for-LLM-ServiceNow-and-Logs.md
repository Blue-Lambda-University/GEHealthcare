# Enriched Context for LLM: ServiceNow + Logs

**Idea:** Collect **all relevant information** from ServiceNow plus **logs** (and other sources), package it, and **feed that enriched payload to the LLM** so it has full context for summarization, priority recommendation within policy, justification, and missing-information detection.

**Design principle unchanged:** Deterministic **policy rules** still run first and set P1/P2/P3 bounds; the LLM works **within** those bounds and uses the enriched context to reason and explain.

---

## 1. What to Collect and Feed to the LLM

### A. From ServiceNow

| Source | What to pull | Purpose for LLM |
|--------|----------------|------------------|
| **Incident** | Short description, full description, state, priority, category, subcategory, assignment group, assigned to, caller, opened/updated time, work notes (recent), closure notes | Core incident narrative and history. |
| **CMDB / Service mapping** | Affected CI(s), business service, related applications (e.g. OTM, xCarrier, MSCA), site/location | What is impacted and where. |
| **Related records** | Linked incidents, problems, changes; same CI or same site | Pattern: recurring? related outage? |
| **Assignment & on-call** | Resolver group, on-call schedule, escalation history | Who is handling it; LLM can reference in drafts. |
| **Knowledge / Runbooks** | Articles or runbooks linked to category, CI, or keyword-matched | Ground LLM in approved procedures; suggest next steps. |
| **Warehouse-specific fields** | u_warehouse_site, u_warehouse_flag, u_ops_state, u_stations_impacted, u_backlog_risk, u_ai_priority_recommendation, u_ai_confidence | Already part of our model; include in payload. |
| **Warehouse health snapshot** | Full normalized object (site_id, ops_state, stations_impacted_count, backlog_delta, failed_tx_rate, top_error_codes, workaround_present, workaround_quality, etc.) | Real-time warehouse state; LLM summarizes and ties to incident. |

So “all info from ServiceNow” = incident + CMDB/CI + related tickets + assignment/knowledge + warehouse fields and health snapshot.

#### Where the Knowledge Base fits (detailed)

The **ServiceNow Knowledge Base** (and runbooks) is included in three places:

1. **What we pull (source):** Row **"Knowledge / Runbooks"** in the table above – articles or runbooks linked to the incident’s **category**, **subcategory**, **affected CI**, or **keyword-matched** from the incident description (e.g. scanner, label printer, OTM).
2. **When we pull it (flow):** In **step 2** (How It Fits in the Flow): when the orchestrator pulls ServiceNow context, it **also** retrieves relevant Knowledge articles – by category, by CI, or by search on short_description/description – and attaches them to the payload.
3. **How we feed it to the LLM (payload):** In the **payload shape** (section 3), the **`knowledge_snippets`** field: an array of `{ title, snippet }` (or `{ kb_number, title, snippet, url }`) so the LLM can ground its summary, next steps, and draft comms in **approved procedures**.

**Retrieval options:** (a) **By linkage** – pull articles linked to the incident or affected CI (up to N articles, snippet length limited). (b) **By category / CI** – query Knowledge for articles tagged with incident category (e.g. Warehouse/Logistics) or affected app (OTM, xCarrier, MSCA). (c) **By search** – keyword/full-text search on short_description/description; cap articles and total length for token budget.

**How the LLM uses it:** The prompt tells the LLM to use knowledge snippets to suggest **next steps** or **runbook steps** that match approved procedures, improve **draft communications** (e.g. refer to KB article X for scanner reset), and avoid contradicting internal SOPs. Optionally use the same knowledge for **RAG** (retrieve at query time) in addition to pre-fetching.

**Summary:** Knowledge base is part of the ServiceNow **pull** step and appears in the payload as **`knowledge_snippets`** so the LLM grounds output in approved procedures and suggests relevant runbook steps.

---

### B. Logs (and similar)

| Source | What to pull | Purpose for LLM |
|--------|----------------|------------------|
| **Application logs** | Errors, warnings, exceptions for the affected app (OTM, xCarrier, MSCA) in the relevant time window (e.g. 30–60 min before incident opened) | Correlate “what broke” with incident description. |
| **Integration logs** | Failed API calls, timeouts, retries between warehouse systems | Explain integration lag or sync failures. |
| **Infrastructure / platform logs** | Server, container, or middleware errors for the CIs linked to the incident | Distinguish app vs infra. |
| **Audit / access logs** | Login failures, permission errors (e.g. for “device login issue”) | Support security/compliance and login-failure incidents. |

Logs should be **scoped** (e.g. by site, CI, time window, log level) and **size-limited** (e.g. last N lines or token budget) so the payload stays within LLM limits and cost.

### C. Optional: Other signals

- **Monitoring / APM:** Alerts or metrics (e.g. response time, error rate) for the same time window.
- **Recent changes:** Changes that touched the affected CI(s) in the last 24–48 hours (from ServiceNow Change or from a change feed).

---

## 2. How It Fits in the Flow

1. **Incident created or updated** (e.g. Warehouse/Logistics, Warehouse Ops).
2. **Orchestrator** (or integration layer):
   - Pulls **ServiceNow context** (incident + CMDB + related + assignment + knowledge + warehouse fields + health snapshot).
   - Pulls **logs** (and optionally other signals) for the same time window and scope (site, CI, app).
3. **Policy engine** runs on **structured data** (site, ops_state, stations_impacted, security flag, etc.) and sets **deterministic** P1/P2/P3 (or min priority and 2hr monitor).
4. **Enriched payload** is built: structured fields + incident text + work notes + health snapshot + **log excerpts** (and optional monitoring/changes).
5. **LLM** receives:
   - Policy outcome (e.g. “Minimum P2 per warehouse policy” or “P1 – operations halted”).
   - Enriched context (ServiceNow + logs).
   - Instruction: summarize, recommend priority **within policy**, justify, ask for missing info, draft comms.
6. **LLM output** (summary, justification, suggested next steps, draft message) is written back to ServiceNow (work notes, fields) or shown to the agent; **priority** remains the one set by the **rules** unless a human overrides.

So we **do** feed “all info from ServiceNow plus logs” to the LLM; we **don’t** let the LLM override the rule-based incident level.

---

## 3. Payload Shape (Conceptual)

A single JSON or structured blob sent to the LLM could look like:

- **incident:** { number, short_description, description, state, priority, category, caller, opened_at, updated_at, work_notes_snippet, assignment_group, assigned_to, … }
- **affected_ci:** { name, business_service, applications[], site }
- **related:** [ { type, number, state, short_description } ] (e.g. last 5)
- **warehouse_health_snapshot:** { site_id, warehouse_type, ops_state, stations_impacted_count, … }
- **policy_result:** { applied_priority, reason, rule_triggered }
- **logs:** { source: "OTM" | "xCarrier" | "MSCA" | "integration" | "infra", time_window, excerpt (last N lines or summarized), error_codes }
- **knowledge_snippets:** [ { title, snippet } ] (optional, from runbooks/knowledge)
- **token_budget_note:** e.g. "Log excerpt truncated to 2000 chars; full logs in attachment."

The LLM prompt would say: “Using the incident, warehouse snapshot, and log excerpts below, and given the policy outcome already applied, provide a summary, justification, and draft communication.”

---

## 4. Log Collection: Practical Notes

- **Where logs live:** App servers, log aggregation (e.g. Splunk, Elastic, Datadog, cloud watch), or SIEM. The **orchestrator** or a dedicated **log service** calls the right API or query.
- **Scoping:** Filter by **time** (e.g. 1 hour before incident opened), **site/CI/app**, and **level** (e.g. ERROR, WARN) to keep volume down.
- **Size and safety:** Truncate or summarize if needed; redact PHI/PII if logs contain it; avoid streaming huge raw logs into the LLM in one go.
- **Failure handling:** If log fetch fails or times out, still send ServiceNow + health snapshot so the LLM can work with what’s available; optionally add “Logs unavailable: &lt;reason&gt;” so the LLM doesn’t invent log content.

---

## 5. Benefits

- **Richer context:** LLM sees the full picture (ticket + systems + logs) and can give a better summary and justification.
- **Better correlation:** Log errors and incident description can be aligned in one place.
- **Missing information:** LLM can point out gaps (e.g. “No log excerpt for MSCA in the last 30 min; consider pulling MSCA logs for site X”).
- **Draft quality:** Comms and work notes can reference specific errors or time windows from logs.
- **Still safe:** Priority stays rule-based; LLM only interprets and explains.

---

## 6. Summary

- **Yes:** Collect **all relevant info from ServiceNow** (incident, CMDB, related, assignment, knowledge, warehouse fields, health snapshot) **plus logs** (app, integration, infra, scoped and truncated).
- **Feed that enriched payload to the LLM** so it can summarize, justify, ask for missing info, and draft comms **within** the rule-derived incident level.
- **Rules** remain the single source of truth for **P1/P2/P3**; the LLM uses the enriched context to support humans and automation, not to override policy.

This document can be updated as we define exact payload schemas, log sources, and token budgets.
