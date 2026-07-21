# 02 — Threat Model

*Fictional reference system. Threats carry stable IDs; the [control map](03-control-map.md) and [risk register](../risk-register.xlsx) reference them.*

Organised four ways: **STRIDE per trust boundary**, an **AI threat surface**, a **decision-integrity & financial-crime** surface unique to finance, and **detection & posture**. A consolidated index sits at the end.

---

## Part A — STRIDE per trust boundary

Trust boundaries are defined in [architecture §3](01-architecture.md#3-trust-boundaries).

### TB-1 · Internet → Webhook

| STRIDE | Threat | ID |
|---|---|---|
| Spoofing | Forged inbound events / **forged payment-network webhook faking a settlement** | TI-06 |
| Tampering | Replayed or modified webhook payloads (double-credit) | TI-06 |
| DoS | Message flood exhausts the ingestion/LLM pipeline | AI-06 |
| Elevation | Malicious message content that later reaches the LLM as instructions | AI-02 |

### TB-2 · App → Postgres (isolation-critical)

| STRIDE | Threat | ID |
|---|---|---|
| Info disclosure | Cross-fintech read of customer accounts/balances because the app connects as owner/`BYPASSRLS` | **TI-01** |
| Tampering | Cross-fintech write (e.g. altering another fintech's records) | **TI-01** |
| Repudiation | No per-tenant audit trail to attribute a cross-fintech action | DET-03 |
| Elevation | Forged `tenant_id` selects another fintech's customers | TI-01 |

### TB-3 · App → Third-party LLM

| STRIDE | Threat | ID |
|---|---|---|
| Info disclosure | Financial PII / **cardholder data** in the prompt leaves your boundary (PCI DSS / GLBA) | **AI-03** |
| Tampering | Model returns manipulated output that drives a wrong action or decision | AI-05 / AI-08 |
| Repudiation | No record of what data was sent to the provider (fails ROPA + PCI scope review) | AI-03 |

### TB-4 · Fintech A ↔ Fintech B (shared infrastructure)

| STRIDE | Threat | ID |
|---|---|---|
| Info disclosure | RAG retrieval without a `tenant_id` filter surfaces another fintech's customer data | **AI-04** |
| Info disclosure | Object-storage path/pre-signed-URL scoping error exposes another fintech's KYC docs | TI-04 |
| Info disclosure | Unauthenticated / un-namespaced cache serves another fintech's data | TI-05 |
| DoS | One fintech's load degrades all (co-location blast radius) | TI-03 |

### TB-5 · App → Cloud control plane (IAM)

| STRIDE | Threat | ID |
|---|---|---|
| Elevation | Over-broad IAM role lets an app compromise escalate to infrastructure | DET-04 |
| Info disclosure | Root-account or console access outside break-glass | DET-01 |
| Repudiation | Control-plane actions not captured in the audit trail | DET-03 |

### TB-6 · Orchestration → Money-movement / payment rails

| STRIDE | Threat | ID |
|---|---|---|
| Tampering / Elevation | An assistant action (post-injection) executes an **unauthorized transfer** | **AI-05** |
| Repudiation | A money movement can't be attributed or reconstructed | DET-03 |
| Tampering | Missing idempotency lets a retried/replayed action move money twice | TI-06 / AI-05 |

---

## Part B — AI threat surface

### AI-01 — Direct prompt injection
A customer crafts a message that overrides the system prompt. Impact scales with what the assistant can *do* (see AI-05) and *reveal* (another customer's financial data).

### AI-02 — Indirect / second-order prompt injection
Malicious instructions **planted in ingested content** — a prior message, an uploaded KYC document, a field summarised later — that execute when the model processes them in a different, possibly higher-privilege context. The most dangerous variant here is one whose payload targets the money-movement path.

### AI-03 — Financial-data / cardholder-data exfiltration across the provider boundary *(flagship)*
Every prompt containing financial PII or **cardholder data (PAN)** that crosses TB-3 is a disclosure to a third-party processor. For a PAN this is worse than a privacy incident: it **drags the provider into PCI DSS scope** and, in practice, breaches it. Financial PII also engages GLBA and cross-border transfer rules. Full dual-regime analysis in the [GRC overlay](04-grc-overlay.md); highest-scored data-disclosure risk in the [register](../risk-register.xlsx).

### AI-04 — Cross-fintech context bleed (the AI twin of TI-01)
A shared vector store without a `tenant_id` filter returns Fintech A's customer embeddings for Fintech B's query. Same failure shape as RLS-vs-superuser: the isolation control exists conceptually but the enforcing filter is missing at the point of use.

### AI-05 — Agentic action → unauthorized money movement *(flagship — theft, not disclosure)*
The finance-defining threat. If the assistant can reach a money-movement capability, a successful AI-01/AI-02 injection becomes an **irreversible transfer of funds** — theft. This is why the architecture forces every transfer through the deterministic money-movement gate ([§4b](01-architecture.md#4b--the-llm-can-suggest-money-movement-it-can-never-authorize-it)): the model *suggests*, the gate *authorizes*. It is one of the two highest-consequence risks in the register.

### AI-06 — Token flood / cost DoS
Unbounded LLM generations as a **financial denial-of-service**. Per-tenant token budgets, rate limits upstream of the model, and a fail-safe ceiling.

### AI-07 — System-prompt leakage / jailbreak
Extraction or guardrail-bypass. Leaked prompts reveal internal logic and any (mistaken) reliance on prompt-level guardrails for money-movement or screening decisions.

---

## Part C — Decision-integrity & financial-crime surface (finance-specific)

### AI-08 — Unfair, biased, or unexplainable AI-assisted credit decision *(differentiator)*
An LLM used in underwriting can produce a decision that is **discriminatory, inaccurate, or unexplainable**. Consequences: fair-lending liability (ECOA / Reg B in the US; equality and FCA rules in the UK), an inability to issue a compliant **adverse-action** explanation, and restrictions on **solely-automated decisions with legal or significant effect** (UK GDPR Art. 22). Controls: human-in-the-loop for credit decisions, adverse-action explainability, and bias testing — the assistant *assists*, a human *decides*.

### AI-09 — LLM trusted as the control of record for KYC / AML *(differentiator)*
Using the LLM as the system of record for **sanctions screening or suspicious-activity detection** invites **false negatives** — a missed sanctioned party or a missed SAR is a serious regulatory failure (BSA/FinCEN in the US; MLR/FCA in the UK). Control: deterministic screening remains the control of record; the LLM only assists document extraction and alert triage, with human review of alerts.

---

## Part D — Detection & posture threats

### DET-01 — Root / privileged login outside break-glass
The canonical "should never happen quietly" event.

### DET-02 — Silent detection failure
A metric filter whose pattern **doesn't match the actual log format** matches nothing — *no alert and no error*. Detection correctness must be **positively tested**, not assumed from deployment.

### DET-03 — Missing / inconsistent log forwarding
No logs → no downstream detection, no reconstruction of a cross-fintech action or a disputed money movement, and a **broken financial audit trail** (SOX-relevant).

### DET-04 — IAM over-privilege
Broad roles turn an app-level compromise into an infrastructure one. Least-privilege with an explicit region lock bounds the blast radius.

### DET-05 — No AI-specific / money-movement detection
Classic detection misses AI and fraud abuse: anomalous token spend (AI-06), injection signatures (AI-01/02), cross-fintech retrieval (AI-04), and **anomalous money-movement patterns** (AI-05) each need their own signals.

---

## Consolidated threat index

| ID | Threat | Boundary | Class |
|---|---|---|---|
| **TI-01** | RLS bypassed via privileged connection role / forged tenant claim | TB-2 | Tenant isolation |
| TI-03 | Co-location blast radius | TB-4 | Tenant isolation |
| TI-04 | Object-storage cross-fintech path scoping | TB-4 | Tenant isolation |
| TI-05 | Unauthenticated / un-namespaced cache | TB-4 | Tenant isolation |
| TI-06 | Forged / replayed webhook (incl. payment webhooks) | TB-1/TB-6 | Integrity |
| **AI-03** | Financial-/cardholder-data exfiltration across provider boundary | TB-3 | AI / financial-data |
| **AI-05** | Agentic action → unauthorized money movement | TB-6 | AI / money movement |
| **AI-02** | Indirect (second-order) prompt injection | TB-1→TB-6 | AI |
| **AI-04** | Cross-fintech vector/context bleed | TB-4 | AI / isolation |
| AI-01 | Direct prompt injection | TB-1 | AI |
| AI-06 | Token flood / cost DoS | TB-1 | AI / availability |
| AI-07 | System-prompt leakage / jailbreak | TB-1 | AI |
| **AI-08** | Unfair / unexplainable AI-assisted credit decision | TB-3 | Decision integrity |
| **AI-09** | LLM as control-of-record for KYC/AML → missed screening | TB-3 | Financial crime |
| DET-01 | Root/privileged login | TB-5 | Detection |
| **DET-02** | Silent metric-filter failure | TB-5 | Detection |
| DET-03 | Missing log forwarding | TB-2/TB-5/TB-6 | Detection |
| DET-04 | IAM over-privilege | TB-5 | Posture |
| DET-05 | No AI-specific / money-movement detection | TB-3/TB-6 | Detection |

**Bold = the highest-signal findings**, carried into the [control map](03-control-map.md) with full rationale and quantified in the [register](../risk-register.xlsx).
