# Securing a Multi-Tenant Banking-as-a-Service / Lending Platform — Reference Architecture

**Fictional reference architecture — a personal learning exercise. Not a real system, and not affiliated with or descriptive of any employer or product.**

A worked reference architecture for a **multi-tenant BaaS and embedded-lending platform** — fintechs build on it; their customers borrow, pay, and hold cards — secured end to end: system design, threat model, control map, and a GRC overlay covering the finance regulatory stack (**PCI DSS, GLBA, AML/KYC, fair lending, and money-movement controls**).

> **Fictional by design.** The platform, fintechs, customers, and data below are invented for this exercise. It is not a description of any production environment. Nothing here is legal, regulatory, or financial advice — the regulatory sections flag questions to resolve with compliance, financial-crime, and legal teams, not settled answers. Every finding is framed as a reusable pattern.

## Why this project exists

Most security portfolios list controls. This one **designs the system first, then shows where it breaks** — because the interesting failures in a financial platform are architectural and regulatory, not configuration typos. Five failure classes carry weight:

1. **Tenant isolation** — cross-fintech exposure of customer accounts, balances, and funds; the part every multi-tenant SaaS gets wrong at least once.
2. **AI-specific risk** — prompt injection, cross-fintech context bleed, and **financial data leaving your trust boundary to a third-party model**.
3. **Money movement** — an assistant with agentic reach + a successful injection = **unauthorized transfers**. This is theft, not a data leak. Unique to this domain.
4. **Decision integrity & financial crime** — AI-assisted credit decisions that are unfair or unexplainable (fair lending), and an LLM wrongly trusted as the control of record for KYC/sanctions screening.
5. **Detection & cloud posture** — including the failure modes that produce *no alert and no error*.

The through-line: **every technical finding maps back to privacy, compliance, financial-crime, and operational-risk impact.** A control that isn't tied to an outcome is just a checkbox.

## The system in one paragraph

A Banking-as-a-Service platform sold to fintechs. Each **tenant** is a fintech; its **customers** are borrowers and cardholders who interact through a consumer channel and never log into the platform directly. The platform provides accounts, payments, card issuing, and lending primitives on a shared core. An **LLM sits in the path** for customer support, KYC document processing, underwriting *assistance*, and dispute/fraud triage — and can suggest actions, some of which move money. **Money moves only through an authorization gate (maker-checker / dual control); the LLM never holds unilateral money-movement authority.** It runs on a standard cloud stack: compute, shared Postgres, cache, object storage, a PCI token vault, secrets manager, and observability.

That component set is deliberately realistic — it is exactly where AI security, financial-data law, and money-movement risk collide.

## Repo map

| File | What it is |
|---|---|
| [`docs/01-architecture.md`](docs/01-architecture.md) | System design, components, trust boundaries (incl. the money-movement boundary), data flows |
| [`assets/architecture.svg`](assets/architecture.svg) | The architecture diagram (six trust boundaries + the money-movement gate marked) |
| [`docs/02-threat-model.md`](docs/02-threat-model.md) | STRIDE-per-boundary + the AI, money-movement, and financial-crime threat surface |
| [`docs/03-control-map.md`](docs/03-control-map.md) | Threat → control → residual risk, with the design rationale |
| [`docs/04-grc-overlay.md`](docs/04-grc-overlay.md) | Data-flow / ROPA sketch + the PCI DSS / GLBA / AML / fair-lending analysis |
| [`risk-register.md`](risk-register.md) | The register as an on-page table (renders on GitHub) |
| [`risk-register.xlsx`](risk-register.xlsx) | Formula-driven version: inherent → residual, CIA, conditional-formatted bands, editable scoring |
| [`LICENSE`](LICENSE) | MIT — reuse freely |

**Suggested reading order:** architecture → threat model → control map → GRC overlay. Each also stands alone.

## The three findings worth your time

If you read nothing else:

- **RLS is not tenant isolation while the app connects as a privileged role.** Row-Level Security policies evaluate correctly and still enforce *nothing* when the connection is the table owner or a `BYPASSRLS` role. The enforcement mechanism is the connection role, not the policy — and here a miss means one fintech reading another fintech's customer accounts and balances. ([control map → TI-01](docs/03-control-map.md#ti-01))
- **Never send a PAN to an LLM.** Sending cardholder data across the model boundary drags the provider into **PCI DSS** scope and, in practice, breaches it. Cardholder data is tokenized in a vault and **never leaves the boundary in the clear** — the finance analog of "de-identify before egress." ([control map → AI-03](docs/03-control-map.md#ai-03))
- **Agentic + injection = theft.** An injected instruction that reaches a money-movement tool is fraud, not a data leak. Money moves only through a dual-control authorization gate the LLM cannot bypass; the model can *suggest*, a deterministic policy or human *authorizes*. ([threat model → AI-05](docs/02-threat-model.md#ai-05))

## What this demonstrates

System design · multi-tenant data isolation · LLM/application security · **money-movement and transaction-integrity controls** · cloud detection engineering · threat modelling (STRIDE) · **financial-data governance across PCI DSS, GLBA, AML/KYC, and fair lending** · formula-driven risk quantification.

## License / reuse

Personal portfolio work, released under the [MIT License](LICENSE) — reuse the patterns freely; there's nothing proprietary here. (Set your name in the `LICENSE` copyright line before publishing.)
