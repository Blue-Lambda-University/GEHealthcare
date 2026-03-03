# NextGen PLM – AI in SDLC: Overview & Documentation

**Document type:** Planning / POV (Point of View)  
**Context:** GE HealthCare  
**Date:** February 2026  
**Status:** Strictly Private and Confidential – PROVISIONAL, for planning purposes only.

---

## 1. Purpose of This Document

This document captures the NextGen PLM initiative and the AI-in-SDLC (Software Development Life Cycle) point of view presented for GE HealthCare. It summarizes:

- Strategic scope (KDD – AI Decisions)
- High-level SDLC flow and tooling (JIRA, Windchill)
- Agentic flow and agent details (TCS WisdomNext as reference)
- Types of GenAI solutions and options (WisdomNext vs Custom)

It is intended for internal planning and alignment only.

---

## 2. KDD – AI Decisions (Scope)

**KDD Scope** defines the boundaries of the AI-in-SDLC discussion:

1. **Discussion on AI tools & methods**  
   Evaluation of available AI tools and methodologies relevant to SDLC and PLM.

2. **Discussion on available platforms and their pros and cons**  
   Assessment of platforms (e.g., embedded AI in Windchill, dedicated GenAI platforms like WisdomNext, custom builds) and their trade-offs.

3. **Possible AI intervention across various streams of end-to-end processes**  
   Identifying where AI can augment or automate steps from requirements through design, test, and implementation.

---

## 3. Types of GenAI Solutions (Strategic View)

Three solution types frame the options:

| Type | Name | Description | Examples |
|------|------|-------------|----------|
| **A** | **Embedded AI** | GenAI offered as a capability *within* an existing licensed third-party application; often requires additional licensing. | Windchill AI Parts Rationalization, Windchill AI Assistant |
| **B** | **Vertical AI** | Dedicated platform for GenAI, offered as a configurable low-code/no-code GenAI SaaS. Fully onboarded specifically for GenAI. | **WisdomNext** |
| **C** | **Custom** | Custom-built GenAI solution with explicit architecture: front-end application, GenAI Orchestrator, and connections to Tools, Agents, and Knowledge, with external integrations. | Custom tool (Option 2) |

---

## 4. High-Level SDLC Flow (Current / Target State)

The flow is **JIRA-centric** and integrates **Windchill** for PLM/solution fitment. Human-in-the-loop reviews are explicit.

### 4.1 Flow Summary

| Phase | Steps | Key systems | Roles |
|-------|--------|-------------|--------|
| **Requirements & user stories** | Create user needs/requirements in JIRA → Extract requirements → Expand requirements using best practices → Update in JIRA → Create user stories from requirements → Update in JIRA | JIRA | — |
| **Fitment & specs** | Windchill solution fitment to user stories; update OOTB fitment fields in JIRA → Create functional specs | JIRA, Windchill | — |
| **Test cases** | Develop test cases for user stories; upload to JIRA | JIRA | — |
| **Review (1)** | Human-in-the-loop review by **Product Owner** for user stories and test cases | JIRA | Product Owner |
| **Design** | Post approval → Generate design document | — | — |
| **Review (2)** | Human-in-the-loop review by **Solution Architect** for functional specs and design document | — | Solution Architect |
| **Implementation prep** | Develop pseudo code for user stories that require implementation via customization | — | — |
| **Update** | Update details in JIRA | JIRA | — |

### 4.2 Noteworthy aspects

- **OOTB fitment fields** in JIRA reflect how well user stories align with out-of-the-box Windchill capabilities.
- **Human-in-the-loop** is required at two gates: Product Owner (user stories + test cases), Solution Architect (functional specs + design).
- **Pseudo code** is scoped to user stories that need **customization** (not pure configuration).

---

## 5. Agentic Flow (TCS WisdomNext – Reference)

The following agentic flow is used as a **reference model** (Option 1 – WisdomNext). It shows how specialized agents can map onto the SDLC.

### 5.1 Inputs

- **List of user needs**
- **JIRA / other ALM tool** (as source of requirements and context)

### 5.2 Agent pipeline (sequential)

| Order | Agent | Primary output |
|-------|--------|----------------|
| 1 | Industry Align Agent | Alignment summary (requirement, industry practice, rationale, actions) |
| 2 | User Story Agent | User stories (epics, actors, steps, rules, etc.) |
| 3 | Windchill Align Agent | Windchill/solution fitment alignment |
| 4 | FuncSpec Agent | Functional specification document |
| 5 | TestCaseGen Agent | Test case document |
| 6 | DesignDoc Agent | High-level design document |
| 7 | PseudoCode Agent | Pseudo code |

---

## 6. Agent Details (Reference)

Below is a concise reference for each agent (purpose, role, key output).

### 6.1 Industry Align Agent

- **Purpose:** Review each item, perform Functional Requirement (FR) gap analysis by comparing requirements against recognized standards, and produce a structured summary aligned with industry best practices.
- **Agent role:** Requirement alignment specialist.
- **Key output:** Requirement, Meets Industry Practice, Rationale, Alignment Actions.

### 6.2 User Story Agent

- **Purpose:** Analyze detailed project files to create clear, well-structured epics and user stories.
- **Agent role:** Agile product manager.
- **Key output:** Actor, Brief Description, Prerequisites, Use case steps, Business Rules, Expected Output, Migration Needs, Integration Needs.

### 6.3 Windchill Align Agent

- **Purpose:** Align Windchill solution with user stories; support OOTB fitment and JIRA updates.
- **Agent role:** (As per flow: solution/PLM alignment.)
- **Key output:** Fitment assessment and updates to OOTB fitment fields in JIRA.

### 6.4 FuncSpec Agent (Functional Specification Agent)

- **Purpose:** Produce detailed, stakeholder-friendly documents that bridge technical and non-technical audiences, ensuring alignment on system features, user interactions, and requirements.
- **Agent role:** Business analyst.
- **Key output:** Well-organized documents in a consistent format.

### 6.5 TestCaseGen Agent (Test Case Generation Agent)

- **Purpose:** Produce test cases that thoroughly validate individual code units, covering various scenarios, edge cases, and potential failures.
- **Agent role:** Quality assurance expert.
- **Key output:** Test cases in the appropriate file format.

### 6.6 DesignDoc Agent (Design Document Agent)

- **Purpose:** Produce a clear, professional document in Markdown including both High-Level Design (HLD) and Low-Level Design (LLD) elements.
- **Agent role:** Software architect.
- **Key output:** System Overview, Architectural Design (HLD), Detailed Design (LLD), Design Decisions and Rationale.

### 6.7 PseudoCode Agent

- **Purpose:** Generate clean, readable pseudocode from the user stories derived from requirements.
- **Agent role:** Software engineer.
- **Key output:** Structured pseudocode with logical breakdowns, organized by epic/feature.

---

## 7. Options in Scope

| Option | Description |
|--------|-------------|
| **Option 1** | **WisdomNext** – Brief introduction, onboarding, and pricing. Uses the agentic flow and agents described above. |
| **Option 2** | **Custom tool** – Custom GenAI solution (Type C) with orchestration, agents, tools, knowledge, and external integrations. |

**Open question (from stakeholder discussion):** If WisdomNext is not available, can **Copilot Studio** be used to create similar agents and a workflow for SDLC orchestration for NextGen PLM?

---

## 8. Agenda (From Presentation)

1. KDD – AI Decisions  
2. High-level SDLC flow & AI agents  
3. Practical demo on TCS tool  
4. Brief introduction of WisdomNext (Option 1)  
5. Onboarding WisdomNext and pricing  
6. Custom tool (Option 2)  
7. Q&A  

---

## 9. References & Disclaimer

- **Confidentiality:** Strictly Private and Confidential – PROVISIONAL, for planning purposes only. Not to be copied, distributed, or reproduced without prior approval.
- **Copyright:** © 2024 GE HealthCare. GE is a trademark of General Electric Company used under trademark license.
- **Source:** AI in SDLC-POV presentation (NextGen PLM), Feb 2026; internal chat on SDLC orchestration and agents.
