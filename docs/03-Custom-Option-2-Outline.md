# Custom Option 2 – Outline: Type C GenAI Solution for NextGen PLM

**Context:** GE HealthCare NextGen PLM – AI in SDLC  
**Solution type:** Type C – Custom (GenAI Orchestrator + Tools, Agents, Knowledge + external integrations)  
**Date:** February 2026  
**Status:** Planning / PROVISIONAL. No code changes implied by this outline.

---

## 1. Purpose

This document outlines a **custom-built GenAI solution** (Option 2) that can deliver SDLC orchestration and agent capabilities similar to the WisdomNext reference model, without depending on WisdomNext or a single commercial agent platform. The architecture follows the **Type C** pattern from the strategic view: front-end application, GenAI Orchestrator, and explicit connections to Tools, Agents, and Knowledge, with external integrations.

---

## 2. Strategic Rationale for Custom Option 2

- **Independence:** No lock-in to WisdomNext or a single vendor’s agent suite; ability to swap models, agents, and tools.
- **Control:** Full ownership of prompts, agent logic, data flows, and integration with JIRA, Windchill, and other ALM/PLM tools.
- **Fit:** Tailored to GE HealthCare’s SDLC, roles (Product Owner, Solution Architect), and Windchill/OOTB fitment process.
- **Extensibility:** Add or replace agents (e.g., new analysis agents, compliance agents) without depending on a platform roadmap.
- **Cost and risk:** Higher build and operate cost than a ready-made platform; requires clear scope and governance.

---

## 3. High-Level Architecture (Type C)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     Front-end application(s)                             │
│  (e.g. portal for triggering runs, reviewing outputs, human-in-the-loop) │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                      GenAI Orchestrator                                  │
│  • Pipeline definition (sequence of agents)                             │
│  • Input/output routing between agents                                  │
│  • Human-in-the-loop checkpoints (Product Owner, Solution Architect)     │
│  • Error handling, retries, audit trail                                 │
└─────────────────────────────────────────────────────────────────────────┘
                    │                │                │
                    ▼                ▼                ▼
        ┌───────────────┐  ┌───────────────┐  ┌───────────────┐
        │    Tools      │  │    Agents     │  │   Knowledge   │
        │ (connectors,  │  │ (Industry     │  │ (standards,   │
        │  APIs, file   │  │  Align, User  │  │  templates,   │
        │  I/O, etc.)   │  │  Story, …)    │  │  RAG sources) │
        └───────────────┘  └───────────────┘  └───────────────┘
                    │                │                │
                    └────────────────┼────────────────┘
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    External integrations                                 │
│  JIRA | Windchill | ALM tools | Repos | Document stores | etc.          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 4. Component Outline

### 4.1 Front-end application(s)

- **Purpose:** Trigger pipeline runs, view status, review agent outputs, and perform human-in-the-loop actions (approve/reject, edit, comment).
- **Possible capabilities:**
  - Start a run from a “list of user needs” or from JIRA (e.g., epic or project).
  - Display outputs per agent (alignment summary, user stories, functional spec, test cases, design doc, pseudocode).
  - Implement review gates: Product Owner (user stories + test cases), Solution Architect (functional spec + design doc).
  - Push approved artifacts back to JIRA (or other ALM) and update OOTB fitment fields where applicable.
- **Tech options (for later):** Web app (e.g. React, .NET), or extension of existing GE HealthCare portals; SSO and access control aligned with enterprise standards.

### 4.2 GenAI Orchestrator

- **Purpose:** Define and execute the SDLC agent pipeline; route data between agents and tools; enforce checkpoints and auditability.
- **Responsibilities:**
  - **Pipeline definition:** Configurable sequence of steps (e.g. Industry Align → User Story → Windchill Align → FuncSpec → TestCaseGen → DesignDoc → PseudoCode), with optional branching or conditional steps.
  - **Input/output contract:** Standardized payload per step (e.g. requirement set, user stories, functional spec document) so agents and tools can be swapped.
  - **Human-in-the-loop:** Pause at defined steps; pass context to front end; resume when review is complete (approved/rejected/edited).
  - **Integration calls:** Invoke Tools layer for JIRA read/write, Windchill API (if available), file read/write, and Knowledge retrieval.
  - **Observability:** Logging, trace IDs, and (optionally) metrics for run duration, token usage, and failure rates.
- **Tech options (for later):** Orchestration engine (e.g. workflow runtime, or lightweight service that calls agents and tools in sequence); could be cloud-hosted (e.g. Azure, AWS) with secure access to integrations.

### 4.3 Tools (integration and utilities)

- **Purpose:** Abstract all external systems and standard operations so agents and orchestrator do not embed integration logic.
- **Suggested tools:**
  - **JIRA tool:** Read issues, requirements, user stories; create/update issues; update custom fields (e.g. OOTB fitment).
  - **Windchill tool:** If APIs exist – read solution metadata, fitment info; support “Windchill Align” agent (e.g. mapping user stories to OOTB capabilities).
  - **ALM / other:** Connectors to other ALM or project tools if in scope.
  - **File/document I/O:** Read project files, templates; write generated artifacts (specs, test cases, design doc, pseudocode) in agreed formats (e.g. Markdown, XLS, repo-friendly structure).
  - **Export/import:** Push finalized artifacts to Confluence, SharePoint, or repos as per GE HealthCare standards.
- **Security:** Credentials and secrets in secure store; least-privilege access; audit of sensitive operations.

### 4.4 Agents

- **Purpose:** Implement the same logical roles as in the WisdomNext reference model; each agent consumes inputs from the orchestrator and produces structured output for the next step or for human review.
- **Suggested agents (aligned with reference):**

| Agent | Role | Input (conceptual) | Output (conceptual) |
|-------|------|--------------------|---------------------|
| Industry Align | Requirement alignment specialist | Requirements from JIRA / user needs | Requirement, Meets Industry Practice, Rationale, Alignment Actions |
| User Story | Agile product manager | Requirements + alignment summary | Epics, user stories (Actor, Description, Prerequisites, Steps, Business rules, Expected output, Migration/Integration needs) |
| Windchill Align | Solution/PLM alignment | User stories | Fitment assessment; updates to OOTB fitment fields (via JIRA tool) |
| FuncSpec | Business analyst | User stories + fitment | Functional specification document (consistent format) |
| TestCaseGen | QA expert | User stories / functional spec | Test cases in agreed file format |
| DesignDoc | Software architect | Functional spec + context | System overview, HLD, LLD, design decisions (e.g. Markdown) |
| PseudoCode | Software engineer | User stories (e.g. those needing customization) | Structured pseudocode by epic/feature |

- **Implementation approach (for later):** Each agent can be implemented as a service that uses:
  - **LLM/GenAI:** For generation and analysis (prompts, model selection, temperature, guardrails).
  - **Knowledge:** Retrieval of standards, templates, or examples (e.g. RAG over internal docs) where applicable.
  - **Orchestrator:** Invokes agents via a standard interface (e.g. REST or internal message contract); agents do not call JIRA or Windchill directly—they receive data from and return data to the orchestrator, which uses the Tools layer.

### 4.5 Knowledge

- **Purpose:** Improve consistency and quality of agent outputs; reduce drift; support industry alignment and standard formats.
- **Possible contents:**
  - **Industry standards and best practices:** For Industry Align agent (FR gap analysis).
  - **Templates:** Functional spec, design doc (HLD/LLD), test case layout, pseudocode structure.
  - **Internal playbooks:** GE HealthCare SDLC, Windchill usage, naming and documentation conventions.
  - **RAG (optional):** Vector store over approved design docs, past user stories, or test cases for retrieval-augmented generation.
- **Governance:** Curated and versioned; access control; periodic review to avoid stale or non-compliant content.

### 4.6 External integrations (summary)

- **JIRA:** Source of user needs/requirements; store user stories, test case links, OOTB fitment fields; optionally design doc links or references.
- **Windchill:** Solution and OOTB capability metadata for fitment (if APIs available); otherwise manual or semi-manual input into the Windchill Align step.
- **ALM / repos / document stores:** As needed for publishing specs, design docs, test cases, or pseudocode (e.g. Confluence, SharePoint, Git).

---

## 5. Pipeline Flow (Aligned with High-Level SDLC)

1. **Trigger:** User or system provides “list of user needs” or selects JIRA project/epic; orchestrator loads requirements (via JIRA tool).
2. **Industry Align:** Agent produces alignment summary; output stored and passed forward.
3. **User Story:** Agent produces epics and user stories; output stored and passed forward.
4. **Windchill Align:** Agent (with Windchill tool) produces fitment; orchestrator updates JIRA OOTB fitment fields via JIRA tool.
5. **FuncSpec:** Agent produces functional spec document; output stored.
6. **TestCaseGen:** Agent produces test cases; output stored (and optionally uploaded to JIRA via tool).
7. **Human-in-the-loop (Product Owner):** Orchestrator pauses; front end presents user stories and test cases for review; on approval, pipeline continues.
8. **DesignDoc:** Agent produces design document (HLD/LLD); output stored.
9. **Human-in-the-loop (Solution Architect):** Orchestrator pauses; front end presents functional spec and design doc for review; on approval, pipeline continues.
10. **PseudoCode:** Agent produces pseudocode for user stories requiring customization; output stored.
11. **Update JIRA:** Orchestrator uses JIRA tool to update issues with final links, status, or references as per process.
12. **Export (optional):** Push artifacts to Confluence, repos, or document stores via Tools.

---

## 6. Governance, Security, and Compliance (Outline)

- **Access control:** Only authorized roles (e.g. Product Owner, Solution Architect, release managers) can trigger runs or perform approvals; front end and orchestrator enforce roles.
- **Data:** Requirements and generated artifacts are sensitive; data in transit and at rest encrypted; processing in approved regions/tenants.
- **Audit:** Orchestrator logs runs, agent inputs/outputs (or references), and approval events for compliance and debugging.
- **Responsible AI:** Prompt and model guardrails; review of agent outputs before they are written to JIRA or published; optional human review for high-impact artifacts.
- **Vendor neutrality:** Orchestrator and agent contracts should allow swapping LLM providers (e.g. OpenAI, Azure OpenAI, or others) without changing pipeline structure.

---

## 7. Phasing (Conceptual)

- **Phase 1 – Foundation:** Orchestrator + one or two agents (e.g. User Story, FuncSpec) + JIRA tool; minimal front end for trigger and review; no Windchill automation.
- **Phase 2 – Full pipeline:** Add remaining agents (Industry Align, Windchill Align, TestCaseGen, DesignDoc, PseudoCode); implement both human-in-the-loop gates; JIRA updates end-to-end.
- **Phase 3 – Knowledge and optimization:** Introduce Knowledge layer (templates, standards, optional RAG); tune prompts and models; add observability and FinOps-style controls.
- **Phase 4 – Scale and extend:** Additional integrations (Confluence, repos); more agents or variants (e.g. compliance, localization); reuse for other PLM or SDLC streams.

---

## 8. Risks and Mitigations (Outline)

| Risk | Mitigation |
|------|------------|
| Windchill API limitations | Define Windchill Align scope (e.g. manual upload of fitment matrix, or limited API); use JIRA as system of record for fitment fields. |
| Agent quality / consistency | Prompt engineering, few-shot examples, Knowledge templates; human-in-the-loop at defined gates. |
| Cost of build and run | Clear scope for Phase 1; reuse existing auth and portal where possible; monitor token and infra cost from day one. |
| Dependency on one LLM vendor | Abstract model calls behind an interface; support multiple back ends (e.g. Azure OpenAI, OpenAI, or others) via configuration. |

---

## 9. Success Criteria (Conceptual)

- Pipeline runs end-to-end from user needs to JIRA updates and generated artifacts (functional spec, test cases, design doc, pseudocode).
- Product Owner and Solution Architect review gates are enforced and auditable.
- JIRA (and, where applicable, Windchill) reflect the outputs of the pipeline in line with GE HealthCare process.
- Documentation and run history support compliance and troubleshooting.

---

## 10. Using MCP (Model Context Protocol) in Custom Option 2

**Yes – MCP can be implemented** in Custom Option 2. It fits naturally as the standard way the Orchestrator (and optionally agents) discover and call the Tools layer, and optionally how Knowledge is exposed.

### 10.1 What is MCP?

**Model Context Protocol (MCP)** is an open protocol that lets LLM applications access **tools**, **resources**, and **prompts** from external servers in a standardized way. Servers expose capabilities (e.g. “get JIRA issues”, “update OOTB fitment”); clients discover and invoke them via the protocol. MCP is increasingly adopted (e.g. Cursor, Claude, Copilot Studio) for AI–tool integration.

### 10.2 Where MCP Fits in the Architecture

| Layer | Role of MCP |
|-------|----------------|
| **Tools** | Each tool (JIRA, Windchill, file I/O, export) is implemented as or wrapped by an **MCP server**. The server exposes discrete **tools** (e.g. `jira_get_requirements`, `jira_update_ootb_fitment`, `windchill_get_fitment_metadata`, `files_write_artifact`) with a name, description, and JSON Schema for parameters. |
| **Orchestrator** | Uses an **MCP client** to discover (`tools/list`) and invoke (`tools/call`) these tools. No need for custom integration code per system; add a new capability by standing up a new MCP server and connecting it. |
| **Knowledge** | Optionally expose Knowledge as an MCP server (e.g. **resources** for document/template retrieval, or tools like `knowledge_search_standards`, `get_template`). Agents or orchestrator then pull context via MCP. |
| **Agents** | Agents can stay as they are (orchestrator passes data in/out). Alternatively, agents could use an MCP client to call tools/resources directly if the design prefers agent‑driven tool use. |

So: **Tools (and optionally Knowledge) become MCP servers; Orchestrator uses MCP client(s) to call them.**

### 10.3 Benefits of Using MCP

- **Standardized interface:** One protocol for all tools; new integrations (Confluence, SharePoint, new ALM) = new MCP server(s), orchestrator unchanged.
- **Discovery:** Orchestrator can list available tools at runtime; good for dynamic or multi-tenant setups.
- **Ecosystem:** Reuse existing MCP servers where they exist (e.g. filesystem, generic HTTP); build custom MCP servers only for JIRA, Windchill, and GE HealthCare‑specific tools.
- **Separation of concerns:** Each MCP server owns credentials and logic for one system; orchestrator stays agnostic of implementation details.
- **Future‑proofing:** As MCP adoption grows, more tooling and patterns become available; Cursor/IDE and other clients can potentially connect to the same servers for developer workflows.

### 10.4 What Would Be Implemented as MCP Servers (Conceptual)

- **JIRA MCP server:** Tools such as `get_issues`, `get_requirements`, `create_or_update_issue`, `update_custom_fields` (e.g. OOTB fitment). Input/output via MCP tool schema.
- **Windchill MCP server** (if APIs allow): Tools such as `get_solution_metadata`, `get_ootb_capabilities`, `evaluate_fitment`. If no API, a thin server that wraps manual/semi‑manual inputs and still exposes a consistent tool interface.
- **File/document MCP server:** Tools such as `read_file`, `write_artifact`, `list_templates`, `export_to_path`. Covers spec, test case, design doc, pseudocode artifacts.
- **Knowledge MCP server (optional):** Resources or tools for `search_standards`, `get_template`, `get_playbook` to support Industry Align, FuncSpec, DesignDoc, etc.

Agents remain separate services (prompt + LLM); they receive inputs from and return outputs to the orchestrator. The orchestrator is the component that calls MCP tools (and optionally passes retrieved context to agents).

### 10.5 Trust and Safety (MCP Alignment)

MCP emphasizes human oversight for tool execution. In Custom Option 2 this aligns well with the existing design:

- **Human-in-the-loop gates** (Product Owner, Solution Architect) already ensure approvals before high‑impact writes (e.g. JIRA updates, published artifacts).
- The **front end** can show when and which tools were invoked (e.g. “JIRA updated with OOTB fitment”) and can require explicit confirmation for sensitive operations if desired.
- **Access control** stays at the orchestrator and front end; MCP servers run with least‑privilege credentials for their respective systems.

### 10.6 Summary

Implementing **MCP servers** for the Tools layer (and optionally Knowledge) in Custom Option 2 is **feasible and recommended** for a clean, standard, and extensible integration story. The orchestrator uses an MCP client to call these servers; no code changes to the high-level pipeline or agent roles are required. Implementation details (transport, auth, exact tool schemas) are for a later technical design phase.

---

## 11. Next Steps (No Code Yet)

1. **Stakeholder alignment:** Confirm scope of Custom Option 2 vs Option 1 (WisdomNext) and vs Copilot Studio.
2. **Integration discovery:** Confirm JIRA APIs and custom fields (OOTB fitment); Windchill API or manual process for fitment.
3. **Architecture deep-dive:** Choose technology stack for Orchestrator, Agents, Tools (including MCP server approach), and Front end (when moving to implementation).
4. **Phase 1 scope:** Agree on first agent subset, first integration (e.g. JIRA only), and minimal front end for pilot; decide whether Phase 1 uses MCP for the JIRA tool.
5. **Governance and security:** Align with GE HealthCare security and data policies; define ownership and support model.

---

**Document control**  
- Strictly Private and Confidential – PROVISIONAL, for planning purposes only.  
- This outline does not authorize or specify code changes; it is for planning and decision-making only.
