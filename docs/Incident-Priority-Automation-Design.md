# Automating P1 / P2 / P3 Incident Priority Classification

**Context:** GE HealthCare – Incident Priority Definitions (including GSPO warehouse exceptions)  
**Goal:** Think through how to automate the classification of incidents into P1, P2, or P3.  
**Status:** Design / options only. No code changes implied.  
**Related:** The **Info** folder (`../Info/`) contains detailed ServiceNow + LLM warehouse incident architecture, OTM/xCarrier/MSCA integration, and implementation phases; this doc summarizes and extends that with the Incident Priority Definitions.

---

## 1. What Needs to Be Automated

From the Incident Priority Definitions document, classification depends on:

- **Four parameters:** Impact (scope, criticality, upstream/downstream), Urgency (time sensitivity, SLAs, shipping delays), Business Function Affected, Security/Compliance Risk.
- **Upgrade rules:** When P3→P2 (e.g. impact grows, workaround not scalable) and when P2→P1 (e.g. complete outage, no workaround, operations halted, regulatory/VIP).
- **Workaround rules:** When a workaround justifies downgrading (stable, documented, scalable, impact reduced) vs when it does not (complex, manual overhead, not scalable, security/compliance risk).
- **GSPO warehouse exceptions:** In a services warehouse, any minor incident = P2 immediately; monitor 2 hours, then escalate to P1 if not resolved. Examples: scanner not syncing, label printer, OTM/xCarrier/MSCA slow, integration lag, device login, printer delays, one shipping station impaired. P1 when operations halted or multiple stations/entire warehouse blocked.

Automation can cover:

1. **Initial classification** when an incident is created or first updated.
2. **Re-classification** when new information arrives (e.g. workaround found, impact grows).
3. **Escalation over time** (e.g. “P2 in warehouse → if not resolved in 2 hours, suggest or trigger P1”).

---

## 2. Inputs the Automation Would Need

To classify an incident, the system needs data. The more structured, the easier to automate.

### 2.1 Ideally Available (Structured)

| Input | Purpose | Example |
|-------|--------|--------|
| **Location / site** | Warehouse exception rules | "Services warehouse – Building A" vs "Corporate IT" |
| **Affected system / application** | Match to “automatic P2” list | Scanner, label printer, OTM, xCarrier, MSCA, integration, device login, printer, shipping station |
| **Scope (users / stations)** | Impact | "One station" vs "Multiple stations" vs "Entire warehouse" |
| **Operations status** | P1 trigger | "Operations halted" vs "Degraded" vs "Running" |
| **Workaround present?** | Downgrade / upgrade logic | Yes/No |
| **Workaround quality** | Downgrade allowed? | Stable & documented? Scalable? Security/compliance risk? |
| **Business function** | Impact parameter | Logistics, order processing, warehouse, etc. |
| **SLA / deadline at risk?** | Urgency | Yes/No, date |
| **Security or compliance flag** | Parameter + P2→P1 | Yes/No |
| **VIP / executive impacted?** | P2→P1 | Yes/No |
| **Incident age / last update** | Escalation (“monitor 2 hours”) | Created time, last updated |

### 2.2 Often Unstructured (Description, Notes)

- Free-text **summary** and **description**.
- **Updates** (“workaround in place,” “second station now down”).

Automation can use:

- **Structured fields only** (rule-based), or  
- **Structured + unstructured** (rules plus NLP/LLM to infer missing fields or suggest priority from text).

---

## 3. Automation Approaches

### 3.1 Rule-Based Engine (Structured Data)

**Idea:** Encode the document’s rules as explicit IF/THEN logic.

**Examples:**

- IF `location` = services warehouse AND (`affected_system` IN [scanner, label printer, OTM, xCarrier, MSCA, …] OR `scope` = one station) → **P2**.
- IF `operations_halted` = true OR `scope` = entire warehouse / multiple stations → **P1**.
- IF `workaround` = yes AND `workaround_stable` = yes AND `workaround_scalable` = yes AND no security risk → allow **downgrade** (e.g. P1→P2 or P2→P3) depending on remaining impact.
- IF `workaround` = yes AND (`workaround_scalable` = no OR `manual_overhead` = high) → do **not** downgrade.

**Pros:** Clear, auditable, no model training; aligns exactly with the written policy.  
**Cons:** Depends on good structured data; weak when critical fields are missing or only in free text.

**Best for:** Ticketing systems that already capture location, affected system, scope, operations status, workaround (and ideally workaround quality).

---

### 3.2 LLM / GenAI-Assisted Classification (Unstructured + Structured)

**Idea:** Use an LLM to interpret the incident description (and any structured fields) and suggest P1/P2/P3 with a short rationale, using the Incident Priority Definitions as context (e.g. in the prompt or via RAG).

**Flow:**

- Input: incident title, description, any structured fields (location, system, scope, etc.).
- Prompt (or retrieved context): summary of the priority parameters, upgrade/downgrade rules, and GSPO warehouse exceptions.
- Output: suggested priority (P1/P2/P3) + rationale (e.g. “Services warehouse + one station impaired → P2 per policy; escalate to P1 if not resolved in 2h”).

**Pros:** Works when data is in free text; can handle partial or messy input; can explain why.  
**Cons:** Needs guardrails (e.g. don’t auto-downgrade on “workaround” unless criteria are clearly met); should be **suggested** priority for human confirmation in sensitive cases.

**Best for:** Incidents submitted as narrative; enriching or validating rule-engine output when fields are missing.

---

### 3.3 Hybrid (Recommended Direction)

- **Rule engine** runs first on structured data (and any fields inferred from text, if you add a small NLP step).
- **Rules produce:** P1, P2, P3, or “Insufficient data.”
- **If “Insufficient data” or high uncertainty:** call LLM with incident text + rules summary to suggest priority and rationale.
- **Human:** sees suggested priority + rationale; confirms or overrides. Overrides can be used later to tune rules or prompts.

Escalation over time (e.g. “P2 in warehouse, 2 hours, not resolved”) can be a **scheduled job or workflow**: re-run rules (and optionally LLM) and either suggest P1 or auto-escalate per policy.

---

## 4. Where Automation Could Live

| Option | Description |
|-------|--------------|
| **Inside incident/ticketing tool** | e.g. ServiceNow, JIRA Service Management. On create/update, run a script or call an API that runs the rule engine (and optionally LLM). Write suggested priority (and rationale) into a custom field; workflow can require approval for P1 or for downgrades. |
| **Separate microservice/API** | Classification service (rules + optional LLM) exposed as API. Incident tool calls it when a ticket is created or updated; service returns suggested P1/P2/P3 + rationale. |
| **Scheduled “escalation” job** | Job runs every X minutes: fetch open P2 incidents in “services warehouse” older than 2 hours with no resolution; re-classify or set “suggested P1” and notify. |

---

## 5. GSPO Warehouse Rules – Explicit Automation Logic

To make the document’s warehouse rules machine-ready:

1. **Tag incidents with location type:** e.g. `location_type` = "services_warehouse" (from site list or from description).
2. **Tag affected system:** from list (scanner, label printer, OTM, xCarrier, MSCA, integration, device login, printer, shipping station) or from free text (LLM or keyword match).
3. **Initial classification:**
   - IF `location_type` = services_warehouse AND (affected_system IN automatic_P2_list OR scope = one_station) → **P2**, set `escalation_check_at` = created_time + 2 hours.
   - IF operations_halted OR scope IN [multiple_stations, entire_warehouse] → **P1**.
   - ELSE apply general Impact/Urgency/Business Function/Security rules (rule engine or LLM).
4. **Escalation job:** For incidents with `escalation_check_at` in the past and still open: re-check; if not resolved and impact unchanged or higher → suggest or set **P1** and notify.

---

## 6. Human-in-the-Loop and Audit

- **Suggested priority:** Automation proposes P1/P2/P3; agent or manager **confirms or overrides**. Overrides (especially downgrades) can require a short comment.
- **Audit:** Log for each incident: suggested priority, rationale (rule path or LLM summary), final priority, who confirmed/overrode. Supports compliance and tuning.
- **Sensitive cases:** For “regulatory or reputational risk” or “VIP impacted,” policy may require human classification or mandatory human sign-off on automation suggestion.

---

## 7. Practical Next Steps (No Code Yet)

1. **List your incident tool and data model:** Which system (ServiceNow, JIRA, etc.) and which fields already exist (location, affected system, scope, workaround, etc.)?
2. **Decide scope of automation:** Initial classification only, or also re-classification and time-based escalation?
3. **Choose first approach:** Rule-only (if structured data is good), LLM-only (if most data is free text), or hybrid.
4. **Encode warehouse rules first:** They are the most explicit (automatic P2, 2h escalation, P1 when halted). Then add general P3/P2/P1 and workaround rules.
5. **Design for override and audit:** Suggested priority + rationale; human confirms; log everything.

---

## 8. ServiceNow + LLM Architecture (from Info)

The **Info** folder contains a concrete architecture for warehouse incident management using **ServiceNow** and an **LLM**. The following aligns P1/P2/P3 automation with that design.

### 8.1 Core Principle

- **ServiceNow** = **System of Record** (incident table, Major Incident module, CMDB, Knowledge, assignment groups).
- **LLM** = **Decision-support and orchestration layer** with guardrails and auditability; it does **not** override deterministic policy.

### 8.2 Integration Layer (Middleware)

- **API Gateway:** Authentication, rate limiting, logging.
- **Orchestrator service:** Pulls ticket + context → applies **policy engine** (rules) → calls **LLM** → writes results back to ServiceNow.
- Optional: event queue for retry and decoupling.

### 8.3 Policy & Rules Engine (Deterministic – Runs First)

Hard-coded warehouse policies (same as Incident Priority Definitions): **Services warehouse** → Minimum P2; **Operations halted** or **multiple stations blocked** → P1; **P2 (warehouse)** → Start 2-hour monitoring; **Security/compliance risk** → Force P1. These rules **override** LLM output.

### 8.4 LLM Layer (After Policy)

Structured JSON output (priority recommendation, justification, confidence), missing-information detection, optional RAG. Example (from Info): `location`, `impact_scope`, `operations_status`, `priority_recommendation`, `justification`, `confidence`.

### 8.5 Action Layer – Allowed vs Restricted

**Allowed:** Update incident fields, work notes, draft comms, start 2hr timer, notify groups. **Restricted:** No production restarts, no inventory changes, **no automatic downgrade of P1 without approval.**

### 8.6 Governance

Full audit logging, PHI/PII redaction, access control, model performance tracking, escalation review process.

---

## 9. Warehouse Health Snapshot and OTM / xCarrier / MSCA (from Info)

**Signals to pull:** OTM (backlog, late shipments, failed dispatch, integration errors, throughput); xCarrier (label failures, print queue, station errors); MSCA (scanner sync, login failures, queue depth, RF errors, latency, users impacted). **Health snapshot schema:** `site_id`, `warehouse_type`, `ops_state`, `stations_impacted_count`, `backlog_delta_per_hour`, `failed_tx_rate`, `top_error_codes`, `workaround_present`, `workaround_quality`, `last_healthy_timestamp`. Attach to each incident.

---

## 10. ServiceNow Data Model (from Info)

**Option 1 – Custom incident fields:** `u_warehouse_site`, `u_warehouse_flag`, `u_ops_state`, `u_stations_impacted`, `u_backlog_risk`, `u_ai_priority_recommendation`, `u_ai_confidence`. **Option 2 – Dedicated table:** `u_warehouse_health_snapshot` for reporting and pattern detection.

---

## 11. Decision Flow and Implementation Phases (from Info)

**Flow:** Incident Created → Enrich Context → Services Warehouse? Yes → Min P2 → Operations Halted? Yes → P1; No → P2, 2hr Monitor → Resolved in 2h? No → Impact Growing? Yes → P1. **Phase 1 (Copilot):** AI suggests priority (human approves), work notes only, no auto priority. **Phase 2 (Guarded Autopilot):** Auto P2 warehouse, 2hr monitor, notify groups; P1 requires approval. **Phase 3 (Major Incident Agent):** Pattern detection, bridge creation, post-incident drafts; strict governance.

---

## 12. Operational Thresholds and References (from Info)

**Thresholds (tune with data):** Ops halted = shipments 0 for 15 min OR failure rate > 80% for 10 min. P1 stations = ≥ 3 or ≥ 25% active. Backlog risk = e.g. 1.5× baseline. **References:** Info/ServiceNow_LLM_Warehouse_Incident_Architecture.md, Info/ServiceNow_LLM_Warehouse_OTM_xCarrier_MSCA_Integration.md, Info/ServiceNow_LLM_Warehouse_Comprehensive_Architecture.md.

---

## 13. Summary

- **Automation is feasible** by encoding the Incident Priority Definitions as rules and/or using an LLM for free text; **hybrid (rules + LLM)** plus **human confirmation** and **audit** is recommended.
- **GSPO warehouse rules** are the most automatable: services warehouse + minor/one station → P2; operations halted / multiple stations → P1; 2-hour monitor → escalation.
- **Info folder** adds the **ServiceNow + LLM** design: ServiceNow as SoR, orchestrator + policy engine + LLM, warehouse health snapshot from OTM/xCarrier/MSCA, custom fields or dedicated table, three phases (Copilot → Guarded Autopilot → Major Incident Agent), and allowed/restricted actions (no auto-downgrade of P1 without approval).

Next step: choose phase (e.g. Phase 1 – work notes only) and align the exact rule set and LLM prompt with the Incident Priority Definitions and ServiceNow data model above.
