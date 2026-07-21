# 03 — Control Map

*Fictional reference system. Each control maps to a [threat ID](02-threat-model.md) and states the **residual** risk honestly — no control drives residual to zero. Regulatory points flag questions for compliance/financial-crime/legal, not settled answers.*

Format: **threat → control(s) → residual**. The anchor findings get full design rationale; the rest are tabulated.

---

## Anchor controls

### TI-01
**RLS bypassed via privileged connection role / forged tenant claim** — *the central isolation finding.*

**Controls:**
1. Application connects as a **dedicated non-privileged role** — not the table owner, no `BYPASSRLS`. This is the actual enforcement mechanism.
2. **`FORCE ROW LEVEL SECURITY`** on every tenant table.
3. `tenant_id` from a **trusted, request-scoped claim** set per transaction — never a client-supplied parameter.
4. A CI check that fails the build if the app's effective role is owner or holds `BYPASSRLS`.
5. Negative test: a query under Fintech A's context returns zero of Fintech B's customer rows.

**Why the policy alone is not the control:** RLS policies evaluate correctly and enforce nothing when the connection role is privileged. The role switch is the enforcement mechanism; the policy is only its expression.

**Residual:** Low likelihood, but impact stays severe — cross-fintech exposure of accounts and balances is a reportable breach and a trust-destroying event. A migration that recreates a table without re-applying `FORCE`, or a new service reusing an over-privileged connection, silently reopens it — hence the CI check (control 4).

### AI-03
**Financial-/cardholder-data exfiltration across the LLM boundary** — *flagship. A technical control and a scope-management control at once.*

**Controls:**
1. **Cardholder data (PAN) is never sent to the LLM.** It lives only in the PCI token vault; the app and the model handle tokens. This keeps the LLM provider **out of PCI DSS scope** — the only defensible posture, since a general LLM provider is not a validated PCI service provider for cardholder data.
2. **Minimise and redact other financial PII** before egress; send the minimum the task needs.
3. **Contractual + region controls** for anything identifiable that must be sent: processor terms, no-training guarantee, pinned region (GLBA Safeguards expectations; cross-border transfer mechanism).
4. **Log what is sent** to the provider — for the ROPA, a GLBA/records view, and a PCI scope review.

**Why:** you cannot un-send data, and a single PAN in a prompt can invalidate a PCI posture. The control is *keep cardholder data out of scope entirely* plus minimise-and-contract for the rest — as much a scope-management control as a technical one. Detailed in the [GRC overlay](04-grc-overlay.md).

**Residual:** Medium–high, honestly. Tokenization removes cardholder-data exposure, but residual confidentiality risk remains whenever any identifiable financial data is sent, and free-text redaction is imperfect. One of the two highest residuals by design.

### AI-05
**Agentic action → unauthorized money movement** — *flagship. Theft, not disclosure.*

**Controls:**
1. **The LLM holds no money-movement capability.** It emits a *proposed* action; execution is impossible without the gate.
2. **Deterministic money-movement gate** enforcing maker-checker / dual control, per-tenant limits, and **idempotency** (so a replayed action can't move money twice).
3. **Out-of-band authorization** for high-value or irreversible transfers — a second authority the model cannot satisfy on its own.
4. **Segregation of duties** across request, approval, and execution (also a SOX-style control).
5. **Anomaly detection** on money-movement patterns feeds alerting (ties to DET-05).

**Why:** this is what turns a successful injection from theft into a rejected suggestion. The gate is deterministic precisely because a probabilistic model must never be the thing that authorizes an irreversible transfer.

**Residual:** Medium. The gate substantially reduces likelihood, but money movement is irreversible and never fully de-risked — insider misuse and gate-logic flaws remain, which is why anomaly detection (control 5) and segregation of duties (control 4) exist.

### DET-02
**Silent detection failure.**

**Controls:**
1. **Positively test every metric filter** against a real sample of the current log format before trusting it.
2. Assert the log format at the forwarding layer (structured/JSON).
3. A scheduled **synthetic canary event** that should trip each alert — if it doesn't fire, the detection is broken and *that* alarms.
4. Alert on **absence of expected log volume**, catching DET-03 in the same mechanism.

**Why:** a deployed-but-non-matching filter produces no alert and no error. Deployment status is not detection status. The only proof a detection works is a known event firing it.

**Residual:** Low with canaries; without them, effectively unbounded because the failure is invisible by construction.

---

## Control map — remainder

| Threat | Control(s) | Residual |
|---|---|---|
| **TI-03** — co-location blast radius | Separate fintechs/trust levels onto separate compute; if co-located, document as accepted risk + compensating monitoring | High — funds + financial data make this costly to accept; revisit the trade-off |
| **TI-04** — object-storage cross-fintech paths | Tenant-scoped prefixes; single-object signed URLs, short TTL; deny-by-default bucket policy | Low |
| **TI-05** — cache isolation | Require cache auth; namespace keys by `tenant_id`; never share cross-fintech keys | Low |
| **TI-06** — forged/replayed webhooks | Verify signature on every event; reject missing/invalid; replay window + **idempotency keys** (critical for payment webhooks) | Low |
| **AI-01** — direct prompt injection | Treat customer input as data, not instruction; least-privilege assistant; no secrets/other-customer data in context | Medium — injection isn't fully solvable; limit blast radius |
| **AI-02** — indirect/second-order injection | Data-not-instruction stance for retrieved content; sandbox output from actions (esp. the money-movement path); flag instruction-like patterns | Medium — hardest to fully close; the honest control is limiting what a successful injection can reach — above all, funds |
| **AI-04** — cross-fintech context bleed | Mandatory `tenant_id` filter at retrieval (wrapper-enforced); per-fintech namespaces; retrieval test | Low–medium — namespacing reduces it structurally |
| **AI-06** — token flood / cost DoS | Per-tenant token budgets and rate limits upstream of the model; spend-anomaly alert; fail-safe ceiling | Low–medium |
| **AI-07** — prompt leakage / jailbreak | Assume the system prompt is public; keep nothing sensitive in it; never let prompt-level text be the guardrail for money movement or screening | Medium — treat as "will leak" |
| **AI-08** — unfair/unexplainable credit decision | Human-in-the-loop for credit decisions; adverse-action explainability; bias testing; scope/claims discipline (assist, don't decide); Art. 22 safeguards | Medium — model-risk and fairness are managed, not eliminated |
| **AI-09** — LLM as KYC/AML control-of-record | Deterministic sanctions/KYC screening stays the system of record; LLM assists extraction/triage only; human review of alerts; audit | Medium — false-negative risk persists; keep the LLM off the control path |
| **DET-01** — root/privileged login | Metric filter on the audit trail → alarm; break-glass procedure | Low |
| **DET-03** — missing log forwarding | Forward all app logs in an asserted format; alert on log-volume absence; treat the audit trail as a financial record | Low |
| **DET-04** — IAM over-privilege | Least-privilege scoped policy; explicit region lock; periodic access review | Low–medium |
| **DET-05** — no AI / money-movement detection | Signals for anomalous token spend, injection patterns, cross-fintech retrieval, and **anomalous transfers** | Medium — emerging discipline; partial coverage is honest |

---

## The residual-risk stance

Four principles run through the residual column:

1. **No control drives residual to zero** — AI-03 (data boundary) and AI-05 (money movement) retain the highest residuals by design.
2. **The enforcing mechanism must live at the point of use** — a filter in a wrapper, `FORCE RLS` on the table, a claim set per transaction, a **deterministic gate on money movement** — not in review discipline or convention, which drift.
3. **Some controls are regulatory or scope controls, not just technical** — keeping cardholder data out of LLM scope, an Art. 22 safeguard, a deterministic AML system of record. In a financial product, GRC is part of the architecture.
4. **Detection status ≠ deployment status** — proven only by a known event firing the alert.

Quantified likelihood/impact and residual scores for every ID are in the [risk register](../risk-register.xlsx).
