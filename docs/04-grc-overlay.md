# 04 — GRC Overlay

*Fictional reference system. This is where the technical findings become privacy, compliance, financial-crime, and operational-risk statements. **Nothing here is legal, regulatory, or financial advice** — it structures the questions a compliance, financial-crime, and legal team would resolve, and flags where the answer depends on facts not fixed in a reference exercise.*

## 1. Why a financial platform forces a heavier GRC overlay

Three things raise the stakes above a generic SaaS:

1. **The data is regulated several ways at once.** Cardholder data (PCI DSS), non-public personal financial information (GLBA in the US; UK GDPR + FCA rules in the UK), and KYC/AML records all live here, each with its own regime.
2. **A third-party LLM is in the path.** The moment financial data crosses [TB-3](01-architecture.md#3-trust-boundaries) into a model you don't control, you have a sub-processor disclosure — and a single PAN in a prompt can invalidate a PCI posture.
3. **The system moves money.** A control failure isn't only a disclosure; it can be **theft**, and money movement is irreversible. That pulls in transaction-integrity, segregation-of-duties, and SOX-style concerns generic SaaS never has.

So [AI-03](02-threat-model.md#ai-03) (data boundary) and [AI-05](02-threat-model.md#ai-05) (money movement) are the twin centres of gravity, and [AI-08](02-threat-model.md#ai-08)/[AI-09](02-threat-model.md#ai-09) add fair-lending and financial-crime dimensions.

## 2. The regulatory stack, mapped to the architecture

The same platform may serve UK/EU and US fintechs and must satisfy whichever regimes apply — often several at once.

| Regime | What it governs here | The load-bearing control |
|---|---|---|
| **PCI DSS** | Cardholder data (PAN) | **Never send a PAN to the LLM.** Tokenize in the vault; keep the LLM and most of the system out of scope ([AI-03](03-control-map.md#ai-03)) |
| **GLBA (US)** / **UK GDPR + FCA (UK)** | Non-public personal financial information / personal data | Minimise + redact before egress; Safeguards-Rule-style security program; lawful basis; transfer mechanism |
| **AML / KYC** (BSA·FinCEN / MLR·FCA; sanctions OFAC·OFSI) | Identity verification, sanctions screening, suspicious-activity reporting | Deterministic screening is the **control of record**; the LLM assists only ([AI-09](03-control-map.md#ai-09)) |
| **Fair lending** (ECOA·Reg B / Equality Act·FCA; UK GDPR Art. 22) | Credit decisions and adverse action | Human-in-the-loop credit decisions; adverse-action explainability; bias testing ([AI-08](03-control-map.md#ai-08)) |
| **SOX-style ICFR** (if in scope) | Integrity of financial records and money movement | Segregation of duties; maker-checker on transfers; a defensible audit trail ([AI-05](03-control-map.md#ai-05), [DET-03](03-control-map.md#det-03)) |

**The load-bearing consequence:** cardholder data is kept out of LLM scope *by construction* (tokenization), and money movement is gated *deterministically*. Those two architectural facts are what let the compliance story hold — GRC here is design, not paperwork bolted on afterward.

## 3. Data flow — where financial data and funds travel

| Stage | Data / value | Boundary crossed | GRC note |
|---|---|---|---|
| Inbound message | Customer identifiers, volunteered financial detail | Internet → Webhook (TB-1) | Collection point; GLBA/GDPR basis |
| Payment webhook | Settlement/network events | Internet → Webhook (TB-1) | Integrity-critical; forge/replay = fraud ([TI-06](02-threat-model.md#part-a--stride-per-trust-boundary)) |
| Persistence | Accounts, balances, application data | App → Postgres (TB-2) | Isolation = the confidentiality control for a financial obligation |
| Card data | PAN | App → **Token vault (PCI)** | Tokenized here; **never travels to the LLM** |
| Retrieval (RAG) | Tenant/customer context | Fintech boundary (TB-4) | Bleed ([AI-04](02-threat-model.md#ai-04)) = cross-fintech disclosure |
| **LLM call** | **Financial PII (tokenized/minimised)** | **App → Third-party LLM (TB-3)** | **Sub-processor disclosure — PCI scope + GLBA/transfer gate** |
| KYC screening | Identity, sanctions match | internal | Deterministic control of record ([AI-09](02-threat-model.md#ai-09)) |
| Credit decision | Underwriting output | internal | Fair-lending + Art. 22 review ([AI-08](02-threat-model.md#ai-08)) |
| **Money movement** | **Funds** | **Orchestration → rails (TB-6)** | **Gate: maker-checker, limits, idempotency, SoD** |

The two lines that define the product: the **LLM call** (financial data leaving your boundary) and the **money movement** (irreversible transfer of funds). Every regime above hangs off one or both.

## 4. ROPA / records sketch (Article 30 style)

Draft — validate with compliance, financial-crime, and legal before treating as final.

| Field | Entry |
|---|---|
| **Processing activity** | AI-assisted customer support, KYC document handling, underwriting assistance, and dispute/fraud triage |
| **Controller / processor role** | Platform = processor for fintech controllers; LLM provider = sub-processor; card networks/banks = separate controllers/rails |
| **Data categories** | Customer identifiers; account/transaction data; **cardholder data (PAN — tokenized)**; KYC/identity documents; credit/underwriting data; derived risk scores |
| **Data subjects** | Fintechs' customers (borrowers, cardholders, account holders) |
| **Purposes** | Support, KYC/onboarding, underwriting assistance, fraud/AML triage, payments |
| **Recipients** | LLM provider (sub-processor); cloud provider; payment networks/banking rails |
| **Special handling** | PAN under **PCI DSS** (kept out of LLM scope); AML/KYC records under BSA/MLR; sanctions data |
| **International transfers** | Pin provider/processing region; document transfer mechanism if data leaves the jurisdiction |
| **Retention** | Account/transaction records per financial-records rules; **prompt/response logs** need their own retention + minimisation decision; AML records per statutory periods |
| **Security measures** | Tenant isolation (TI-01), tokenization (AI-03), deterministic money-movement gate (AI-05), least-privilege IAM, audit logging, deterministic AML screening (AI-09) |
| **Lawful basis** | Set by the fintech-controller; platform records and honours it |

**Honest labelling:** a *sketch to structure the conversation with compliance and financial crime*, not a completed record. Overclaiming completeness on a financial-services compliance artifact is the failure mode to avoid.

## 5. Automated decisions, fair lending, and financial crime

Two finance-specific governance points a security-only review would miss:

- **Credit decisioning is not a place for an autonomous LLM.** Solely-automated decisions with a legal or significant effect are restricted under UK GDPR Art. 22; fair-lending regimes require a compliant **adverse-action** explanation and freedom from prohibited-basis discrimination. The assistant is kept to *assist-and-explain*, a human decides, and bias testing is part of model risk ([AI-08](03-control-map.md#ai-08)).
- **AML/KYC screening must stay deterministic.** A probabilistic model as the control of record invites false negatives — a missed sanctioned party or unfiled SAR is a serious regulatory failure. The LLM assists document extraction and alert triage; the screening system of record is deterministic and auditable ([AI-09](03-control-map.md#ai-09)).

## 6. Mapping to a recognised framework

The controls line up with ISO 27001 Annex A themes — access control and tenant isolation (A.5/A.8), cryptography and secrets, logging and monitoring, **supplier relationships** (the LLM provider as a managed sub-processor; the card networks and banking partners as regulated third parties), and secure development (the CI checks enforcing isolation at the point of use). For a financial platform this also intersects **PCI DSS** control objectives, **GLBA Safeguards Rule**-style program requirements, and, where applicable, **SOX ICFR** through segregation of duties and the money-movement audit trail. The register is structured so its entries can be cross-referenced to whichever framework a given fintech-controller — or the platform itself — is audited against.
