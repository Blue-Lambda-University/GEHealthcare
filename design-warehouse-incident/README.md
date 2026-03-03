# Design: ServiceNow + LLM Warehouse Incident (OTM / xCarrier / MSCA)

**Context:** GE HealthCare – Warehouse incident management and P1/P2/P3 classification  
**Purpose:** This folder is the **design workspace** for the ServiceNow + LLM warehouse integration. Design starts from the OTM/xCarrier/MSCA integration document and expands from here.

---

## Combined document (all-in-one)

- **[Warehouse-Incident-Design-Combined.md](Warehouse-Incident-Design-Combined.md)**  
  Single file containing the full design: Part I (base integration), Part II (enriched context: ServiceNow + logs + knowledge base), Part III (LLM-based P1/P2/P3 classification). Use this for a single read or share.

## Starting point (individual docs)

- **[01-ServiceNow-LLM-Warehouse-OTM-xCarrier-MSCA-Integration.md](01-ServiceNow-LLM-Warehouse-OTM-xCarrier-MSCA-Integration.md)**  
  Base design: warehouse systems integration (OTM, xCarrier, MSCA signals), health snapshot schema, ServiceNow data model options, decision logic, workflows, and operational thresholds.  
  *Source: copied from `Info/ServiceNow_LLM_Warehouse_OTM_xCarrier_MSCA_Integration.md`.*

---

## Folder structure (design only)

| Item | Description |
|------|-------------|
| **README.md** | This file – design workspace index. |
| **Warehouse-Incident-Design-Combined.md** | **All-in-one:** Part I (base integration) + Part II (enriched context + KB + logs) + Part III (LLM P1/P2/P3 classification). |
| **01-ServiceNow-LLM-Warehouse-OTM-xCarrier-MSCA-Integration.md** | Base integration design (signals, schema, data model, rules, workflows, thresholds). |
| **02-Enriched-Context-for-LLM-ServiceNow-and-Logs.md** | What to collect from ServiceNow (incident, CMDB, related, knowledge, warehouse snapshot) plus logs; how to package and feed to the LLM; rules still set P1/P2/P3. |
| **03-LLM-Based-Priority-Classification-P1-P2-P3.md** | How to have the LLM classify P1/P2/P3: inputs, structured output, three usage options (LLM primary with human approval; LLM + rules override; LLM for edge cases only), guardrails, audit. |
| **05-Architectural-Design-Warehouse-Incident.md** | **Architectural design** for the warehouse incident solution: system context, logical architecture (layers, components), component design, data flows, deployment view, security and governance, key design decisions (with Mermaid diagrams). Derived from the combined doc. |
| **06-MCP-and-RAG-Warehouse-Incident.md** | **MCP and RAG extension:** How Model Context Protocol (MCP) and Retrieval-Augmented Generation (RAG) fit in the warehouse incident solution; mapping adapters to MCP servers; explicit RAG for knowledge and policy. Companion HTML: **06-MCP-and-RAG-Warehouse-Incident.html**. |
| *Further design docs* | To be added as the design grows (e.g. detailed flows, API contracts, LLM prompt spec, data dictionary). |

---

## Related

- **Incident Priority Definitions** – GSPO warehouse P1/P2/P3 rules and workaround logic (referenced by the decision logic in 01).
- **docs/Incident-Priority-Automation-Design.md** – Automation design that references this integration and the Info folder.
- **Info/** – Original architecture and integration docs.

---

*No code in this folder; design and documentation only.*
