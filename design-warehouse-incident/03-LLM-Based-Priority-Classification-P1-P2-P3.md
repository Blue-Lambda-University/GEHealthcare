# LLM-Based Incident Priority Classification (P1 / P2 / P3)

**Goal:** Design how the **LLM** can classify incident priority (P1, P2, P3) instead of—or in addition to—purely rule-based classification.  
**Constraint:** No code yet; design and options only.

---

## 1. Why Use the LLM for Classification

- **Nuance:** Free-text descriptions, work notes, and logs often contain context that strict rules miss (e.g. “second station starting to fail,” “workaround works but only for one user”).
- **Consistency:** LLM can apply the same policy language (e.g. from Incident Priority Definitions) across many incident phrasings.
- **Explainability:** LLM can output both **priority** and **justification**, which helps audit and user trust.
- **Evolving policy:** Policy text can be updated in prompts or RAG without rewriting rule code.

Trade-off: rules are deterministic and auditable; LLM can drift or hallucinate. So we need **guardrails** and **human-in-the-loop** where appropriate.

---

## 2. What the LLM Needs to Classify

The LLM should receive everything that defines priority per the Incident Priority Definitions:

- **Impact:** Scope (how many users/stations/systems), business criticality, upstream/downstream impact.
- **Urgency:** Time sensitivity, SLA/deadline at risk, shipping delays.
- **Business function affected:** e.g. logistics, warehouse, order processing.
- **Security/compliance risk:** Yes/no; regulatory or reputational exposure.
- **Location/context:** Services warehouse? Which site? (drives warehouse exception rules.)
- **Operations status:** Running / degraded / halted; stations impacted count.
- **Workaround:** Present? Quality (stable / partial / risky)? Scalable?
- **Raw context:** Incident description, work notes, warehouse health snapshot, **log excerpts** (from ServiceNow + logs doc).

So: the **enriched payload** (ServiceNow + logs) we already designed, **plus** a clear **policy summary** (or RAG-retrieved Incident Priority Definitions), so the LLM “knows” the criteria for P1 vs P2 vs P3 and warehouse exceptions.

---

## 3. Structured Output from the LLM

To use the LLM’s answer in automation, we need **structured** output, not free text only.

### 3.1 Suggested Schema (Conceptual)

- **priority:** `"P1"` | `"P2"` | `"P3"`
- **confidence:** `"High"` | `"Medium"` | `"Low"` (so we can route low-confidence to human)
- **justification:** Short explanation (1–3 sentences) citing policy and evidence (e.g. “Services warehouse, one station impaired → P2 per policy; escalate to P1 if not resolved in 2h.”)
- **applied_rules_or_criteria:** Optional list (e.g. “Warehouse minimum P2,” “No workaround,” “Operations halted”) for audit.
- **suggested_actions:** Optional (e.g. “Start 2-hour monitor,” “Notify on-call”).
- **missing_information:** Optional (e.g. “Unclear if workaround is scalable; recommend asking caller.”)

Implementation: ask the LLM to respond in **JSON** that conforms to this schema (e.g. via prompt + response format, or a structured-output / tool-call pattern). Validate the JSON and the `priority` enum before using it.

---

## 4. Three Ways to Use the LLM for P1/P2/P3

### Option A — LLM as primary classifier (human in the loop)

- **Flow:** Enriched context (ServiceNow + logs) + policy summary → **LLM** → outputs P1/P2/P3 + justification + confidence.
- **Usage:** System **suggests** this priority; **human** (e.g. agent, manager) **approves or overrides** before the incident is saved. No automatic priority change without approval.
- **Guardrails:** Low confidence → force human review. P1 or downgrade from P1 → require explicit approval. Full audit (input snapshot, model, output, who approved).
- **Pros:** LLM handles nuance; human keeps final say.  
- **Cons:** Every ticket may need a human click for high-stakes environments.

### Option B — LLM + rules: LLM suggests, rules can override

- **Flow:** Enriched context → **LLM** → suggests P1/P2/P3. **Policy engine** runs on **structured** data (site, ops_state, stations_impacted, etc.) and produces **rule-based** priority. **Comparator** logic:
  - If **rule says P1** (e.g. operations halted) → **always P1** (rules override LLM).
  - If **LLM says P1** but rules don’t → **escalate to human** or treat as P2 until confirmed.
  - Otherwise use **LLM suggestion** (or the stricter of the two), with optional human review for P1 or downgrades.
- **Guardrails:** Rules act as **safety floor/ceiling** (e.g. “services warehouse never below P2”); LLM cannot under-classify past the rule. High-stakes (P1, downgrade) can still require human approval.
- **Pros:** Combines LLM nuance with rule-based safety.  
- **Cons:** Two systems to maintain; need clear precedence rules.

### Option C — LLM only for “edge” cases; rules for the rest

- **Flow:** **Rules** run first. If rules have **enough structured data** and **unambiguous** result (e.g. services warehouse + one station → P2; ops halted → P1), use it. If **ambiguous** (e.g. missing ops_state, or rule says “unknown”), **call LLM** with enriched context and get P1/P2/P3 + justification; use that as **suggested** priority (with human approval for P1 or downgrade).
- **Guardrails:** LLM used only when rules cannot decide; human approval for any P1 or downgrade from P1.
- **Pros:** Fewer LLM calls; rules handle clear cases; LLM handles edge cases.  
- **Cons:** Need a clear definition of “ambiguous.”

---

## 5. Making the LLM Classification Reliable

### 5.1 Give the LLM the policy explicitly

- **Prompt** (or RAG): Include a concise summary of **Incident Priority Definitions**: when P3→P2, when P2→P1, when workaround allows downgrade, **GSPO warehouse exceptions** (any minor incident in services warehouse = P2; 2h monitor; P1 when operations halted or multiple stations). So the LLM is “reading the same playbook” as the rules.

### 5.2 Provide evidence in the payload

- Include **warehouse health snapshot**, **log excerpts**, **affected CI**, and **site** in the payload so the LLM can cite them in the justification (e.g. “Logs show scanner sync failures at site X; one station impaired.”). Redact PHI/PII as needed.

### 5.3 Structured output and validation

- Require **JSON** with `priority` (P1|P2|P3), `confidence`, `justification`. **Validate** after the call: reject if priority is not in enum or if JSON is malformed; then apply guardrails (e.g. don’t auto-apply P1 if confidence is Low).

### 5.4 Confidence and human escalation

- If **confidence = Low** or **missing_information** is non-empty for critical fields → **do not** auto-apply; send to **human** with LLM suggestion and justification. Optional: if LLM says P1 and rules didn’t → always human confirm.

### 5.5 Audit and tuning

- **Log:** incident ID, payload summary (no PHI), model, full LLM output, which priority was applied (and by whom if human overrode). Use for **audit** and to **tune** prompts or rules (e.g. if LLM often disagrees with rules in a specific scenario, adjust policy or add a rule).

---

## 6. Summary: How to Have the LLM Classify P1/P2/P3

| Aspect | Recommendation |
|--------|----------------|
| **Input** | Enriched context (ServiceNow + logs) + explicit policy summary (Incident Priority Definitions, including warehouse exceptions). |
| **Output** | Structured: `priority` (P1/P2/P3), `confidence`, `justification`, optional `applied_criteria`, `missing_information`, `suggested_actions`. |
| **Usage** | Choose one: (A) LLM suggests, human always approves; (B) LLM suggests, rules override and can force human for P1; (C) Rules first, LLM only for ambiguous cases. |
| **Safety** | Rules can enforce floor/ceiling (e.g. warehouse min P2; ops halted → P1). No auto-downgrade from P1 without approval. Low confidence → human review. |
| **Audit** | Log input summary, model, output, and final priority (and overrides). |

With this, the **LLM** can classify incident priority (P1, P2, P3) in a controlled, auditable way while still respecting policy and optional rule-based guardrails. No code changes in this doc—design only.
