# Risk Register

*Fictional reference architecture — not legal, regulatory, or financial advice. Scores are illustrative.*

This is an on-page mirror of the formula-driven register in [`risk-register.xlsx`](risk-register.xlsx). The spreadsheet is the source of truth — it recomputes inherent and residual scores from the likelihood/impact inputs, with conditional-formatted bands and dropdown validation. Full control detail for each ID is in [`docs/03-control-map.md`](docs/03-control-map.md).

**Scoring model:** Inherent = Likelihood × Impact (before controls). Residual = Likelihood × Impact re-scored after the mapped control(s). Both on 1–5 scales, so scores run 1–25. Bands: Low 1–4 · Medium 5–9 · High 10–15 · Critical 16–25. Risk reduction = (Inherent − Residual) / Inherent.

| ID | Risk | Class | TB | CIA | Inherent | Residual | ↓ Risk | Owner |
|----|------|-------|----|-----|----------|----------|--------|-------|
| **TI-01** | RLS bypassed: app connects as owner/BYPASSRLS role, or tenant_id is client-supplied -> cross-fintech access to accounts/balances | Tenant isolation | TB-2 | C·I | 3×5=15 (High) | 1×5=5 (Medium) | 67% | Platform Engineering |
| **TI-03** | Co-location blast radius: multiple fintechs' isolation boundaries share one failure domain | Tenant isolation | TB-4 | C·I·A | 3×5=15 (High) | 2×5=10 (High) | 33% | Leadership |
| **TI-04** | Object-storage cross-fintech access to KYC documents via path/pre-signed-URL scoping error | Tenant isolation | TB-4 | C | 2×5=10 (High) | 1×4=4 (Low) | 60% | Platform Engineering |
| **TI-05** | Cache serves another fintech's data (unauthenticated or un-namespaced) | Tenant isolation | TB-4 | C | 2×4=8 (Medium) | 1×3=3 (Low) | 62% | Platform Engineering |
| **TI-06** | Forged or replayed webhook events (incl. payment-network webhooks faking a settlement) | Integrity | TB-1 | I | 3×4=12 (High) | 1×4=4 (Low) | 67% | Platform Engineering |
| **AI-01** | Direct prompt injection overrides system instructions | AI | TB-1 | C·I | 4×3=12 (High) | 3×2=6 (Medium) | 50% | Platform Engineering |
| **AI-02** | Indirect / second-order injection via ingested content processed later | AI | TB-1/TB-6 | C·I | 3×4=12 (High) | 2×4=8 (Medium) | 33% | Platform Engineering |
| **AI-03** | Financial-/cardholder-data exfiltration across the third-party LLM boundary | AI / financial-data | TB-3 | C | 4×5=20 (Critical) | 3×4=12 (High) | 40% | DPO / Legal |
| **AI-04** | Cross-fintech context bleed via shared vector store (missing tenant filter) | AI / isolation | TB-4 | C | 3×5=15 (High) | 1×5=5 (Medium) | 67% | Platform Engineering |
| **AI-05** | Agentic action -> unauthorized money movement (theft) after injection | AI / money movement | TB-6 | I·A | 4×5=20 (Critical) | 2×5=10 (High) | 50% | Platform Engineering |
| **AI-06** | Token flood / cost DoS from unbounded generations | AI / availability | TB-1 | A | 3×3=9 (Medium) | 2×2=4 (Low) | 56% | Platform Engineering |
| **AI-07** | System-prompt leakage / jailbreak weakens guardrails | AI | TB-1 | C | 4×2=8 (Medium) | 3×2=6 (Medium) | 25% | Security |
| **AI-08** | Unfair, biased, or unexplainable AI-assisted credit decision | Decision integrity | TB-3 | I | 3×4=12 (High) | 2×4=8 (Medium) | 33% | Compliance / Financial Crime |
| **AI-09** | LLM trusted as control-of-record for KYC/AML -> missed sanctions / suspicious activity | Financial crime | TB-3 | I | 3×5=15 (High) | 2×4=8 (Medium) | 47% | Compliance / Financial Crime |
| **DET-01** | Root / privileged login outside break-glass | Detection | TB-5 | C·I·A | 2×5=10 (High) | 1×4=4 (Low) | 60% | Platform Engineering |
| **DET-02** | Silent detection failure: metric filter pattern does not match log format | Detection | TB-5 | C·I·A | 3×4=12 (High) | 1×4=4 (Low) | 67% | Security |
| **DET-03** | Missing / inconsistent log forwarding -> broken audit trail (SOX-relevant) | Detection | TB-2/TB-6 | C·I | 3×4=12 (High) | 1×4=4 (Low) | 67% | Platform Engineering |
| **DET-04** | IAM over-privilege enables app-compromise -> infra escalation | Posture | TB-5 | C·I·A | 2×4=8 (Medium) | 1×3=3 (Low) | 62% | Platform Engineering |
| **DET-05** | No AI / money-movement detection (token spend, injection, cross-fintech retrieval, anomalous transfers) | Detection | TB-6 | C·A | 3×4=12 (High) | 2×3=6 (Medium) | 50% | Security |

## Key control per risk

| ID | Key control(s) |
|----|----------------|
| **TI-01** | Non-privileged connection role + FORCE RLS + trusted per-tx tenant claim + CI role check |
| **TI-03** | Separate fintechs/trust levels onto separate compute; if co-located, accept explicitly + compensating monitoring |
| **TI-04** | Tenant-scoped prefixes; single-object signed URLs, short TTL; deny-by-default bucket policy |
| **TI-05** | Require cache auth; namespace keys by tenant_id; never share cross-fintech keys |
| **TI-06** | Verify signature on every event; replay window + idempotency keys (critical for payment webhooks) |
| **AI-01** | Treat customer input as data not instruction; least-privilege assistant; no secrets/other-customer data in context |
| **AI-02** | Data-not-instruction stance for retrieved content; sandbox output from actions (esp. money-movement path) |
| **AI-03** | PAN never sent to LLM (tokenize in PCI vault); minimise/redact other financial PII; processor terms + region pin; log what is sent |
| **AI-04** | Mandatory tenant_id filter at retrieval (wrapper-enforced); per-fintech namespaces; retrieval test |
| **AI-05** | LLM holds no money-movement capability; deterministic gate (maker-checker, limits, idempotency); out-of-band auth; SoD; anomaly detection |
| **AI-06** | Per-tenant token budgets + rate limits upstream of model; spend-anomaly alert; fail-safe ceiling |
| **AI-07** | Assume prompt is public; keep nothing sensitive in it; never let prompt text guard money movement or screening |
| **AI-08** | Human-in-the-loop credit decisions; adverse-action explainability; bias testing; assist-not-decide; Art.22 safeguards |
| **AI-09** | Deterministic screening stays system of record; LLM assists extraction/triage only; human review of alerts; audit |
| **DET-01** | Metric filter on audit trail -> alarm on root/privileged login; documented break-glass |
| **DET-02** | Positively test filters vs real log sample; assert log format; scheduled canary; alert on log absence |
| **DET-03** | Forward all app logs in asserted format; alert on log-volume absence; treat audit trail as a financial record |
| **DET-04** | Least-privilege policy scoped to needed actions; explicit region lock; periodic access review |
| **DET-05** | Signals for anomalous token spend, injection patterns, cross-fintech retrieval, and anomalous money movement |

## Owner roles

- **Security** — Self-authored / detection-design items (no production access required).
- **Platform Engineering** — Controls requiring production changes; executed by the systems owner.
- **DPO / Legal** — Data-protection and PCI/GLBA data-handling decisions (contracts, transfers, minimisation).
- **Compliance / Financial Crime** — AML/KYC, fair lending, credit decisioning, and money-movement governance.
- **Leadership** — Accepted-risk / trade-off decisions (e.g. co-location cost).

> Draft — validate impact bands and residuals against a real environment, and validate AI-03 (data boundary), AI-05 (money movement), AI-08 (fair lending), and AI-09 (AML/KYC) with compliance, financial-crime, and legal before treating as final. No control drives residual to zero; AI-03 and AI-05 retain the highest residuals by design.
