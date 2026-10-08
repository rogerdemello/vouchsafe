<div align="center">

# Vouchsafe

**Explainable voucher classification for Indian GST books, powered by open-weight small language models that run fully offline.**

![Track](https://img.shields.io/badge/Track%204-VYOM%2B%20Voucher%20Classification-1971c2)
![Model](https://img.shields.io/badge/Primary%20model-Qwen3.5--9B%20open--weight-f08c00)
![Offline](https://img.shields.io/badge/Runs-100%25%20offline-2f9e44)
![License](https://img.shields.io/badge/License-Apache--2.0-6741d9)
![Stage](https://img.shields.io/badge/Stage-Qualifier%20proposal-e03131)

Hacktober Fest Open Source AI Hackathon, organised by Elevate · Qualifier round submission

</div>

| | |
|---|---|
| **Problem statement** | **No. 4** · VYOM+ Intelligent Voucher Classification Using Open-Source LLMs |
| **Team** | Vouchsafe (solo) |
| **Member** | Roger Demello · design, models, backend, evaluation and UI · GitHub [@rogerdemello](https://github.com/rogerdemello) |
| **Repository status** | Qualifier round: this README is the entire repository, as the rules require. No code, datasets, notebooks or binaries. Implementation happens at the final hackathon. |

> **The idea in three lines**
>
> 1. **Don't ask a model to pick from 27 look-alike labels.** Ask it six accounting questions it can answer from evidence, then derive the label from a deterministic decision table. Every confusing pair named in the brief is separated by one of those six questions.
> 2. **Resolve "whose books are these?" before anything else.** Purchase vs Sales, Debit Note vs Credit Note and Receipt Note vs Delivery Note are questions of perspective, not keywords.
> 3. **Spend reasoning only where it pays.** A calibrated fast tier, distilled from the open SLM's own labels, answers the clear-cut rows. An SLM agent with tools reasons over the hard ones. Every prediction ships with a confidence score, its reasoning and the fields it relied on.

---

## Contents

1. [Project Name](#1-project-name)
2. [Problem Statement](#2-problem-statement)
3. [Project Overview](#3-project-overview)
4. [Proposed Solution](#4-proposed-solution)
5. [Objectives](#5-objectives)
6. [Target Users / Use Case](#6-target-users--use-case)
7. [Open-Source AI Technology Selected](#7-open-source-ai-technology-selected)
8. [Why This Technology Was Selected](#8-why-this-technology-was-selected)
9. [AI's Role in the System](#9-ais-role-in-the-system)
10. [System Architecture](#10-system-architecture)
11. [Component-Level Architecture](#11-component-level-architecture)
12. [Data / Information Flow](#12-data--information-flow)
13. [Agentic Workflow](#13-agentic-workflow)
14. [Technology Stack](#14-technology-stack)
15. [Expected Features](#15-expected-features)
16. [Implementation Approach](#16-implementation-approach)
17. [Expected Final Output](#17-expected-final-output)
18. [Future Scope / Scalability](#18-future-scope--scalability)
19. [Open-Source Dependencies / Components](#19-open-source-dependencies--components)
20. [Expected Challenges and Mitigation](#20-expected-challenges-and-mitigation)

---

## 1. Project Name

**Vouchsafe** (repository: `vouchsafe`)

To *vouchsafe* is to grant something with assurance. The name carries the promise of the system: a voucher type you can trust, delivered together with the evidence that justifies it.

---

## 2. Problem Statement

Every business event in an Indian SME's books is recorded as a **voucher**, and the voucher type is the first decision in the entry. It decides which ledgers move, whether stock changes, and where the transaction lands in GST reporting (outward supplies in GSTR-1, input tax credit in GSTR-3B). VYOM+ already turns invoices into structured rows. The missing step is the bridge from a structured row to the right voucher, which today is still done by an accountant reading each row by hand.

**The task:** given an Excel file of transactions with the voucher-type column removed, assign every row one of **27 voucher categories**, using an open-weight model as the primary intelligence, robustly under missing and ambiguous data, with a reproducible evaluation on unseen records.

Why this is harder than it looks:

| Difficulty | What it looks like in real data |
|---|---|
| **Same columns, opposite meaning** | A purchase invoice and a sales invoice have identical fields. Which one it is depends only on whether the books owner is the buyer or the seller. |
| **Keywords mislead** | "Return" in a narration does not make a return voucher. "Advance" appears in ordinary payments. "Transfer" can be a Contra or a Stock Journal. |
| **Value vs quantity** | A Receipt Note and a Purchase both bring goods in, but only one moves ledger value. The same holds for Rejection Out vs Debit Note. |
| **Look-alike pairs** | The brief names seven confusable groups: Purchase/Sales, Purchase Return/Sales Return, Payroll/others, Contra/Payment/Receipt, Journal/Purchase/Sales, inventory movement/Purchase/Sales, Import/Export/domestic. |
| **Messy, incomplete rows** | Headers vary between exports ("Party A/c", "Vendor", "Bill From"), fields go missing, amounts arrive as `1,00,000.00 Dr`, narrations mix Hindi and English. |
| **Long tail** | 27 classes, several of them rare (Physical Stock, Job Work orders, Attendance). Overall accuracy hides failure on rare classes. Macro-F1 does not. |
| **No labels** | The dataset arrives without voucher types, so there is nothing to train a supervised model on out of the box. |

A wrong voucher type is not cosmetic. It posts to the wrong ledgers, misstates tax liability or input tax credit, and leaves stock registers out of step, which surfaces months later as reconciliation work.

---

## 3. Project Overview

Vouchsafe is a **local-first classification service** with three faces: a **CLI** for batch runs and evaluation, a **REST API** for VYOM+ integration, and a **review console** for accountants.

It reads any transaction workbook, works out whose books it is looking at, turns each row into an **evidence card**, and classifies it through a **two-tier cascade**: a fast calibrated model for clear-cut rows and an **open-SLM reasoning agent** for everything else. Every output carries a voucher type, a calibrated confidence, the six-axis reasoning behind it, the source fields it relied on, and a review flag.

<p align="center">
  <img src="https://github.com/user-attachments/assets/7b8881b7-3663-4dc2-8a39-731df4125330" alt="System overview" width="100%">
</p>

---

## 4. Proposed Solution

Vouchsafe rests on six design ideas. Each one targets a specific difficulty from Section 2.

### 4.1 Voucher Algebra: 27 labels as answers to six questions

The 27 categories are not 27 unrelated things. Each is a point on a small set of accounting axes, the same questions an experienced accountant asks without thinking:

| Axis | The question it answers | Values |
|---|---|---|
| **Stage** | What is this document in the transaction lifecycle? | `ORDER` · `MOVEMENT` · `BILL` · `RETURN` · `SETTLEMENT` · `ADJUSTMENT` · `STOCK` · `HR` |
| **Direction** | Which way do goods or money flow, seen from the books owner? | `IN` · `OUT` · `INTERNAL` · `NONE` |
| **Counterparty** | Who is on the other side? | `SUPPLIER` · `CUSTOMER` · `JOB_WORK_PARTY` · `EMPLOYEE` · `OWN_ACCOUNT` · `OTHER_EXTERNAL` · `NONE` |
| **Value** | Does the entry move ledger value, or only quantity or commitment? | `FINANCIAL` · `QUANTITY` |
| **Border** | Does the transaction cross India's border? | `DOMESTIC` · `CROSS_BORDER` |
| **Qualifier** | The last distinction a few stages need | `GOODS`/`SERVICE` for bills · `AGAINST_BILL`/`ADVANCE` for settlements · `TRANSFER`/`COUNT` for stock |

The SLM answers the six axes from evidence. A **deterministic decision table** maps the answers to a label. The model also gives its own direct label, so every row gets **two independent readings**. When they agree, confidence is high. When they disagree, the agent knows exactly which axis to re-examine.

<p align="center">
  <img src="https://github.com/user-attachments/assets/75cba022-1771-4440-a89b-75e67605b969" alt="Voucher Algebra decision tree for all 27 voucher types" width="858">
</p>

<details>
<summary><b>Full axis signature of all 27 voucher types</b> (click to expand)</summary>

<br/>

| # | Voucher type | Stage | Direction | Counterparty | Value | Border | Qualifier |
|---|---|---|---|---|---|---|---|
| 1 | Purchase | BILL | IN | SUPPLIER | FINANCIAL | DOMESTIC | GOODS |
| 2 | Expense | BILL | IN | SUPPLIER | FINANCIAL | DOMESTIC | SERVICE |
| 3 | Import | BILL | IN | SUPPLIER | FINANCIAL | CROSS_BORDER | any |
| 4 | Sales | BILL | OUT | CUSTOMER | FINANCIAL | DOMESTIC | any |
| 5 | Export | BILL | OUT | CUSTOMER | FINANCIAL | CROSS_BORDER | any |
| 6 | Purchase Return / Debit Note | RETURN | OUT | SUPPLIER | FINANCIAL | any | any |
| 7 | Sales Return / Credit Note | RETURN | IN | CUSTOMER | FINANCIAL | any | any |
| 8 | Rejection Out | RETURN | OUT | SUPPLIER | QUANTITY | any | any |
| 9 | Rejection In | RETURN | IN | CUSTOMER | QUANTITY | any | any |
| 10 | Purchase Order | ORDER | IN | SUPPLIER | QUANTITY | any | any |
| 11 | Sales Order | ORDER | OUT | CUSTOMER | QUANTITY | any | any |
| 12 | Job Work Out Order | ORDER | OUT | JOB_WORK_PARTY | QUANTITY | any | any |
| 13 | Job Work In Order | ORDER | IN | JOB_WORK_PARTY | QUANTITY | any | any |
| 14 | Receipt Note | MOVEMENT | IN | SUPPLIER | QUANTITY | any | any |
| 15 | Delivery Note | MOVEMENT | OUT | CUSTOMER | QUANTITY | any | any |
| 16 | Material Out | MOVEMENT | OUT | JOB_WORK_PARTY | QUANTITY | any | any |
| 17 | Material In | MOVEMENT | IN | JOB_WORK_PARTY | QUANTITY | any | any |
| 18 | Stock Journal | STOCK | INTERNAL | NONE | QUANTITY | any | TRANSFER |
| 19 | Physical Stock | STOCK | INTERNAL | NONE | QUANTITY | any | COUNT |
| 20 | Payment | SETTLEMENT | OUT | external | FINANCIAL | any | AGAINST_BILL |
| 21 | Receipt | SETTLEMENT | IN | external | FINANCIAL | any | AGAINST_BILL |
| 22 | Advance / Prepayment | SETTLEMENT | IN or OUT | external | FINANCIAL | any | ADVANCE |
| 23 | Contra | SETTLEMENT | INTERNAL | OWN_ACCOUNT | FINANCIAL | any | any |
| 24 | Journal | ADJUSTMENT | NONE | any | FINANCIAL | any | any |
| 25 | Salary / Payroll | HR | OUT | EMPLOYEE | FINANCIAL | any | any |
| 26 | Attendance | HR | NONE | EMPLOYEE | QUANTITY | any | any |
| 27 | Other / Miscellaneous | no signature matches, or the row carries no usable evidence | | | | | |

The table is linted at start-up: **no axis signature may match two labels**. That property is a unit test, so the mapping cannot drift silently.

</details>

Why this works better than a flat 27-way prompt:

- **Narrow questions suit small models.** "Is the books owner the buyer here?" has direct evidence. "Which of 27 labels is this?" does not. The gain is measured, not assumed: the flat prompt is kept as a baseline in every evaluation.
- **Built-in uncertainty signal.** Disagreement between the direct label and the axis-derived label costs nothing to detect and points to the exact axis in doubt.
- **The explanation comes free.** The six answers *are* the explanation, in the accountant's own vocabulary.
- **Conventions are data, not weights.** If VYOM+ books a services bill as Purchase rather than Expense, one row of the decision table changes. Nothing is retrained.

### 4.2 Each look-alike pair is settled by one decisive axis

| Look-alike pair | Decisive axis | Evidence Vouchsafe reads |
|---|---|---|
| Purchase vs Sales | Direction | books owner's role in the row (buyer or seller), which party columns are filled, document title, position of the owner's own GSTIN |
| Purchase Return vs Sales Return | Direction | who sends goods back, whether the original-invoice reference links to a purchase or a sale (transaction graph), debit-note vs credit-note wording |
| Salary / Payroll vs other | Counterparty | employee ID, pay period, Basic / HRA / PF / ESI / TDS / net-pay columns, absence of GSTIN and HSN |
| Contra vs Payment / Receipt | Counterparty | both sides are the owner's own cash or bank ledgers, deposit / withdrawal / inter-bank transfer, no external party |
| Journal vs Purchase / Sales | Stage | no goods and no cash movement, narration about depreciation, provision, accrual, write-off or tax set-off |
| Inventory movement vs Purchase / Sales | Value | quantities without taxable value or GST, challan or GRN number instead of an invoice number |
| Import / Export vs domestic | Border | foreign currency and exchange rate, port code, bill of entry or shipping bill, IEC, LUT or zero-rated supply |
| *Also handled:* Purchase vs Expense | Qualifier | HSN code (goods) vs SAC code starting `99` (services), stock item vs overhead narration (rent, AMC, power, freight) |
| *Also handled:* Payment vs Advance | Qualifier | does a matching bill exist earlier in the transaction graph, or only an order? |
| *Also handled:* Debit Note vs Rejection Out | Value | tax and amount present vs quantity-only with a QC-rejection status |
| *Also handled:* Delivery Note vs Material Out | Counterparty | customer vs job worker, job-work challan fields (Section 143, CGST Act) |
| *Also handled:* Stock Journal vs Physical Stock | Qualifier | source and destination godowns or consumption and production vs book quantity, counted quantity and variance |

### 4.3 Perspective resolution: whose books are these?

Half of the confusable pairs flip on a single fact: is the reporting entity the buyer or the seller? Vouchsafe settles that **once per dataset**, before any row is classified.

<p align="center">
  <img src="https://github.com/user-attachments/assets/04f75b52-b78d-4459-9e56-5a607296ec12" alt="Perspective resolution flow" width="700">
</p>

GSTIN structure helps here: the first two digits are the state code and the next ten are the PAN. Two GSTINs sharing a PAN belong to one legal entity, which marks branch transfers rather than true purchases or sales. State codes also tell intra-state supplies (CGST + SGST) from inter-state ones (IGST), which the verifier cross-checks against the tax columns.

### 4.4 Evidence cards, not raw rows

Each row is converted into a typed **evidence card**: about 40 deterministic signals, the counterparty profile, the narration, links to related rows, and an explicit list of **missing** fields. Missing data is a signal in itself. Orders have no invoice number, attendance rows have no amounts, and Receipt Notes have quantities but no tax. The SLM reasons over the card and must cite the fields it used.

### 4.5 A confidence-gated cascade

A **fast tier** (k-nearest exemplars plus a LightGBM model over the evidence signals and narration embeddings) produces calibrated probabilities in milliseconds. A row goes straight to the verifier only if all of these hold:

- calibrated confidence and top-1 vs top-2 margin are above thresholds,
- perspective confidence is high,
- the verifier finds no hard-constraint violation.

Everything else goes to the SLM agent. The threshold is not hand-tuned. It is chosen on the gold dev set so that **the fast tier only answers where it is at least as precise as the SLM would be**. Setting it to 1.0 sends every row to the SLM, a single dial between maximum accuracy and maximum speed.

### 4.6 A transaction graph across rows

A workbook is not a bag of independent rows. Purchase Order → Receipt Note → Purchase → Payment form chains, and credit notes point at original invoices. Vouchsafe links rows by order, invoice, challan and GRN references, and by party + amount + date windows. The agent can query the graph. That is how it tells an **Advance** (money before any bill) from a **Payment** (money against a bill), and which way a return flows.

### 4.7 Honest uncertainty

- **"Other / Miscellaneous" is a real class, not a dumping ground.** It is predicted only when the evidence positively fits no category or the row is empty. A low-confidence row still gets its best label, plus `needs_review: true`. Dumping uncertain rows into "Other" would trade recall for nothing.
- Confidence is **calibrated** on the gold dev split, so the score means something.
- Every prediction is written to an **audit log** with the route taken, model version and prompt hash.

---

## 5. Objectives

1. **Classify every row** of an unseen workbook into exactly one of the 27 categories and emit machine-readable output (JSON and XLSX).
2. **Maximize macro-F1** (primary metric) and per-category F1, with explicit reporting on the confusable pairs from the brief.
3. **Explain every prediction** with calibrated confidence, the six-axis answers and the source fields relied on.
4. **Degrade gracefully** on missing, renamed or ambiguous fields: never crash, never silently default.
5. **Keep the primary intelligence open-weight and local.** Zero proprietary API calls, runs offline on a laptop.
6. **Make evaluation one command** and reproducible: pinned weights, fixed seeds, deterministic decoding.
7. **Keep a human in the loop:** a review queue ordered by uncertainty, and corrections that improve the system immediately.

**Success criteria for the final**

| Criterion | How it is measured | Bar |
|---|---|---|
| Valid output | Schema validation of every row | 100%, guaranteed by grammar-constrained decoding |
| Determinism | Two full runs, diff the outputs | Identical |
| Beats baselines | Macro-F1 on the gold test split vs keyword rules and a flat zero-shot SLM prompt | Higher than both, overall and on the named confusable pairs |
| Calibration | Expected calibration error and a reliability plot | Reported |
| Efficiency | Rows per second, p50 / p95 latency per route, peak memory, CPU-only and GPU | Reported |
| Robustness | Macro-F1 under synthetic perturbations, contrast-set flip accuracy | Reported |

---

## 6. Target Users / Use Case

| User | Pain today | What Vouchsafe gives them |
|---|---|---|
| **VYOM+ platform** | Extraction produces clean rows, but voucher creation still needs a human decision | An API that returns voucher type + confidence, so confident rows post automatically and uncertain rows go to review |
| **SME accountants** | Bulk exports from billing and ERP tools are classified row by row | Batch classification, then review of only the flagged rows |
| **CA firms and bookkeeping teams** | Many clients, each with a different export format | Schema-agnostic ingestion and a per-client books-owner setting |
| **Auditors** | Need to know *why* an entry was booked a certain way | Evidence-cited explanations and a complete audit log |

**Use case.** An accountant at a CA firm receives a client's 2,000-row transaction export with no voucher types. They drop it into the review console. Vouchsafe reports how it understood each column and whose books it is looking at, classifies every row, and sorts the review queue by uncertainty. The accountant checks the 5 to 10 percent of rows that are flagged, corrects two, and those corrections immediately guide every similar row in the file. The result exports as a clean, voucher-typed sheet ready for posting.

---

## 7. Open-Source AI Technology Selected

| Role | Selected | License |
|---|---|---|
| **Primary reasoning model** | **Qwen3.5-9B** (open-weight SLM) | Apache-2.0 |
| Speed variant for CPU-only runs | Qwen3.5-4B / Qwen3.5-2B | Apache-2.0 |
| Bake-off contenders and fallback | Gemma 4 E4B and Gemma 4 26B A4B, Phi-4-mini, Llama 3.1 8B | Apache-2.0, MIT, Llama 3.1 Community |
| Embedding model | multilingual-e5-small | MIT |
| Local inference runtime | llama.cpp `llama-server` (CPU and laptop GPU, GGUF quantized) | MIT |
| GPU inference runtime | vLLM (continuous batching, prefix caching) | Apache-2.0 |
| Structured output | JSON-schema grammar decoding in llama.cpp, xgrammar in vLLM | MIT, Apache-2.0 |
| Agent orchestration | LangGraph | MIT |
| Fast tier and calibration | LightGBM, scikit-learn | MIT, BSD-3-Clause |
| Exemplar memory | FAISS | MIT |
| Distillation (stretch goal) | Unsloth + PEFT + TRL (QLoRA) | Apache-2.0 |

Both runtimes expose the same OpenAI-compatible HTTP interface, so the model is swappable through configuration alone.

**Reference hardware:** a 16 GB RAM laptop running CPU-only (Qwen3.5-4B, 4-bit), and a single consumer GPU for the 9B model and the teacher pass.

---

## 8. Why This Technology Was Selected

**Why open-weight and local, not a hosted API**

- **Privacy.** Transaction exports carry GSTINs, PANs, bank details and salaries. Payroll data is personal data under India's Digital Personal Data Protection Act, 2023. Keeping inference on-premises removes the question entirely.
- **Cost.** Zero marginal cost per row, which matters when a firm classifies millions of rows a year.
- **Reproducibility.** Pinned weights plus deterministic decoding give the same output for the same input, every time. A hosted model can change under you.
- **Adaptability.** Open weights can be distilled and fine-tuned on a firm's own conventions.
- The brief rules out a proprietary API as the primary engine anyway. Vouchsafe is designed to need none at all.

**Why Qwen3.5-9B as the primary model**

- Strong reasoning for its size, and it runs on a single consumer GPU or quantized on CPU.
- **Broad multilingual coverage** (201 languages), which matters for narrations that mix Hindi and English.
- Apache-2.0, with a family of smaller sizes (4B, 2B, 0.8B) that share a tokenizer and behavior, so the same prompts serve as a distillation target.

**Why a small model is enough.** The pipeline does the heavy lifting. Axis decomposition turns a 27-way judgement into six narrow questions, evidence cards hand the model pre-computed facts, and arithmetic or checksum work never reaches the model at all. What remains (reading narrations, weighing conflicting evidence, mapping odd headers) is squarely within an SLM's strengths.

**The model is chosen by measurement, not reputation.** At the final, Qwen3.5-9B, Qwen3.5-4B, Gemma 4 E4B, Gemma 4 26B A4B and Phi-4-mini are run on the gold dev set with identical prompts. The winner is picked by macro-F1, then latency, then memory, and its weights are pinned by hash.

**Alternatives considered**

| Alternative | Why it is not the primary approach |
|---|---|
| Proprietary LLM API | Not allowed as the primary engine. Sends financial PII off-premises, costs per row, not reproducible. |
| Keyword rules only | Breaks on perspective and noisy narrations. Kept as a baseline and as weak labeling functions. |
| Supervised ML only | There are no labels to train on. Used only as the distilled fast tier. |
| Flat 27-way LLM prompt | Look-alike labels blur together and there is no internal consistency check. Kept as a baseline. |
| 70B-class model | Will not run on a laptop, slow, and unnecessary once the problem is decomposed. |

**Why LangGraph, LightGBM and FAISS.** LangGraph gives an explicit, bounded state machine with inspectable traces rather than an open-ended agent loop. Boosted trees are the fastest accurate learner on tabular signals and give probabilities that calibrate well. FAISS keeps exemplar retrieval in-process with no extra service to run.

---

## 9. AI's Role in the System

**Where AI does the work, and why rules alone fail there**

| Task | AI component | Why deterministic code is not enough |
|---|---|---|
| Axis reasoning on uncertain rows | **Open SLM (primary)** | Requires weighing conflicting, partial evidence |
| Labels for the fast tier | **Open SLM as teacher** | No labels are provided |
| Understanding narrations | Open SLM + embeddings | Free text, abbreviations, Hindi-English mix ("adv agst PO 113", "sal Sep") |
| Mapping unknown columns | Embeddings, then SLM fallback | Every export names its headers differently |
| Explanations | Open SLM | A natural-language rationale that cites fields |
| Similar-case retrieval | Embeddings + FAISS | Precedent from past and corrected rows |
| Fast classification | LightGBM student, distilled from the SLM | Speed on clear-cut rows |

**Where AI is deliberately not used:** tax arithmetic, GSTIN checksum and state-code checks, date ordering, Indian number-format parsing, and the axis-to-label mapping. The principle is simple: **deterministic code wherever correctness is checkable, the model wherever judgement is needed.**

**The open SLM is the primary intelligence.** Every label in the system originates in its judgement, either directly (escalated rows) or through the fast tier, which is trained on the SLM's own labels. With the gate threshold set to 1.0, every single row is classified by the SLM.

---

## 10. System Architecture

<p align="center">
  <img src="https://github.com/user-attachments/assets/2d5d83b6-2834-4de4-8367-bf3519427f2b" alt="System architecture" width="1037">
</p>

**Deployment view.** One `docker compose up`, no internet required after the model weights are pulled.

<p align="center">
  <img src="https://github.com/user-attachments/assets/bf28c28f-40a8-4622-93e9-8ee95b58f9b2" alt="Deployment view" width="100%">
</p>

---

## 11. Component-Level Architecture

| # | Component | Responsibility | Input → Output | Built with |
|---|---|---|---|---|
| C1 | **Workbook Loader** | Read every sheet, detect the real header row, unmerge cells, drop subtotal and blank rows | `.xlsx` / `.csv` → raw table | pandas, openpyxl |
| C2 | **Schema Mapper** | Map arbitrary headers onto a canonical schema of about 45 fields: synonym dictionary, then fuzzy match, then embedding similarity, then SLM fallback for leftovers. Writes a mapping report. | raw table → canonical rows | RapidFuzz, e5 embeddings, SLM |
| C3 | **Normalizer** | Parse Indian number formats (`1,00,000`), `Dr`/`Cr` suffixes, bracketed negatives, Excel date serials, currency codes. Validate GSTIN checksums, extract PAN and state code. | canonical rows → typed rows + data-quality flags | pandas, Pydantic |
| C4 | **Perspective Resolver** | Decide whose books the dataset represents and each row's role (Section 4.3) | typed rows → `books_of`, `role_in_row`, confidence | RapidFuzz, rules |
| C5 | **Evidence Extractor** | Compute about 40 typed signals plus a missing-field mask per row | typed rows → evidence cards | Python, Pydantic |
| C6 | **Transaction Graph** | Link rows by order, invoice, challan and GRN references and by party + amount + date window | typed rows → graph of document chains | NetworkX |
| C7 | **Exemplar Memory** | Store labeled exemplars (rulebook examples, gold rows, reviewer corrections) for nearest-neighbour lookup | evidence card → k similar labeled rows | FAISS, e5 embeddings |
| C8 | **Fast Tier** | Predict a calibrated label distribution from signals, narration embeddings and the kNN vote | evidence card → probabilities | LightGBM, scikit-learn |
| C9 | **Confidence Gate** | Accept or escalate using confidence, margin, perspective confidence and verifier pre-checks | probabilities → route | Python |
| C10 | **Reasoning Agent** | Six-axis reasoning with tools, consistency loop and self-consistency vote (Section 13) | evidence card → axes, label, cited evidence | LangGraph, open SLM |
| C11 | **Verifier** | Apply hard constraints, axis-to-label consistency and cross-row consistency, then calibrate confidence | candidate label → final label or re-route | Python, decision table |
| C12 | **Output Writer** | Emit JSON, an XLSX with added columns, and the audit log | final labels → files | pandas, openpyxl |
| C13 | **Evaluation Harness** | Metrics, confusion matrix, ablations, stress suites, latency profile | predictions + truth → report | scikit-learn, MLflow |
| C14 | **Review Console** | Uncertainty-ordered queue, evidence view, one-click correction into memory | user actions → exemplars | Streamlit |

**The evidence signals (C5)**

| Group | Example signals |
|---|---|
| Perspective and party | `role_in_row`, counterparty GSTIN present and valid, same-PAN related entity, counterparty looks like a bank or cash ledger, counterparty looks like an employee |
| Document | title keywords (tax invoice, purchase order, delivery challan, GRN, debit note, credit note, bill of entry, shipping bill), invoice number present, order reference present, original-invoice reference present, challan or GRN number present |
| Value and tax | taxable value present or zero, GST type (IGST, CGST + SGST, none, zero-rated), tax arithmetic consistent, discount, freight, total sign, debit and credit columns |
| Goods | line items present, quantity and its sign, HSN vs SAC (`99xx`), unit of measure, godown fields, book vs counted quantity |
| Money | payment mode (cash, bank, UPI, cheque, NEFT), own accounts on both sides, amount matches an earlier bill in the graph, payment dated before any linked bill |
| Cross-border | currency other than INR, exchange rate, port code, bill of entry or shipping bill number, IEC code, LUT reference, foreign address |
| Payroll and HR | employee ID, pay period, Basic / HRA / PF / ESI / TDS / net pay, days present, leave, overtime hours |
| Returns and job work | return reason, negative quantity, QC rejected status, job-work process, ITC-04 reference, principal or job-worker fields |
| Text and quality | narration embedding, multilingual keyword hits, missing-field mask, column-mapping confidence |

**An evidence card** (illustrative, fictional data)

```json
{
  "row_id": 418,
  "perspective": { "books_of": "Acme Components Pvt Ltd", "role_in_row": "buyer", "confidence": 0.97 },
  "signals": {
    "doc_title": "Tax Invoice",
    "has_invoice_no": true,
    "has_order_ref": true,
    "taxable_value": 48200.0,
    "gst_type": "CGST+SGST",
    "tax_math_ok": true,
    "code_type": "SAC",
    "code": "9987",
    "currency": "INR",
    "port_code": null,
    "counterparty_kind": "registered_supplier",
    "own_accounts_both_sides": false,
    "employee_fields": false
  },
  "narration": "AMC charges for CNC machine, Q3",
  "linked_rows": [ { "row_id": 112, "relation": "order_ref", "label": "Purchase Order" } ],
  "missing": ["delivery_note_no", "payment_mode"]
}
```

---

## 12. Data / Information Flow

**Runtime flow, with the data artifact each stage produces**

<p align="center">
  <img src="https://github.com/user-attachments/assets/9a51e7d7-fe9e-491c-b18d-6a06aa3f91cf" alt="Runtime data flow" width="390">
</p>

Only the evidence card, the narration and linked-row summaries reach the model, never the whole workbook. That keeps prompts short (a few hundred tokens per row) and keeps unrelated personal data out of the model's context.

**Learning flow: bootstrapping without labels**

The dataset arrives unlabeled, so Vouchsafe manufactures its own training signal and keeps a small, hand-labeled gold set strictly for measurement.

<p align="center">
  <img src="https://github.com/user-attachments/assets/e5e5a371-9053-487f-b3b0-6c475c188cc4" alt="Learning flow: bootstrapping without labels" width="100%">
</p>

Synthetic data is generated from the same decision table the classifier uses, so scoring the system on it would be circular. It is used only for rare-class coverage and robustness testing. **Headline numbers come from the hand-labeled gold test split, drawn from the real dataset.**

---

## 13. Agentic Workflow

Only escalated rows enter the agent. It is a **bounded LangGraph state machine**, not an open-ended loop: at most two reasoning rounds and four tool calls per row, and every step is traced into the audit log.

<p align="center">
  <img src="https://github.com/user-attachments/assets/caa9a5c7-d981-4fb9-8fcd-cdcbf35b607a" alt="Agentic workflow for escalated rows" width="691">
</p>

**Tools available to the agent** (all read-only and deterministic)

| Tool | Returns | Typically resolves |
|---|---|---|
| `rulebook(label_a, label_b)` | Decision card for the two competing labels: definitions, required evidence, counter-evidence, GST references | Any axis conflict |
| `similar_cases(row, k)` | Nearest labeled exemplars with their axes | Precedent for unusual rows |
| `linked_documents(row)` | Rows connected by order, invoice, challan or GRN references, or by party + amount + date | Payment vs Advance, return direction, Receipt Note vs Purchase |
| `tax_check(row)` | Intra- vs inter-state consistency from GSTIN state codes, rate × taxable value, zero-rating | Import / Export, Journal vs bill |
| `party_profile(party)` | How this counterparty's other rows were classified, GSTIN / PAN facts, bank-like or employee-like | Contra, Payroll, Purchase vs Expense |

**Prompt contract.** (1) A fixed system prefix with role, axis definitions and output schema, cached across rows by the runtime's prefix cache. (2) Rulebook cards for the fast tier's top candidate labels. (3) Three nearest exemplars. (4) The evidence card, inside delimiters and marked as data. (5) Output constrained by a JSON schema whose label and axis fields are enums, so an invalid or invented label is impossible.

**Guardrails.** Narration text is treated as data, never as instructions. No tool can write or reach the network. Decoding is temperature 0 with a fixed seed, except during the self-consistency vote, which uses fixed seeds too.

---

## 14. Technology Stack

| Layer | Choice |
|---|---|
| Language and packaging | Python 3.11, `uv` lockfile |
| Data handling | pandas, openpyxl |
| Matching and validation | RapidFuzz, Pydantic v2 |
| Embeddings and retrieval | sentence-transformers, multilingual-e5-small, FAISS |
| Open SLM | Qwen3.5-9B / 4B (Gemma 4 and Phi-4-mini in the bake-off) |
| Inference | llama.cpp `llama-server` (CPU, GGUF 4-bit), vLLM (GPU) |
| Structured output | JSON-schema grammar decoding, xgrammar |
| Agent orchestration | LangGraph |
| Fast tier | LightGBM, scikit-learn calibration |
| Transaction graph | NetworkX |
| API | FastAPI + Uvicorn |
| Review console | Streamlit |
| Synthetic data | Faker (`en_IN` locale) with valid GSTIN checksums and tax arithmetic |
| Experiment tracking | MLflow |
| Fine-tuning (stretch) | Unsloth, PEFT, TRL (QLoRA) |
| Testing | pytest, Hypothesis (property and metamorphic tests) |
| Deployment | Docker Compose, fully offline |

---

## 15. Expected Features

| Feature | Priority |
|---|---|
| Upload `.xlsx` or `.csv`, multi-sheet, any column naming | Must |
| Automatic column mapping with a human-readable mapping report | Must |
| Books-owner detection with manual override | Must |
| 27-way classification with confidence, six-axis reasoning, cited evidence and explanation | Must |
| Schema-guaranteed JSON output and an XLSX with added prediction columns | Must |
| One-command reproducible evaluation report | Must |
| Fast tier + confidence gate (speed and efficiency) | Should |
| Transaction graph and agent tools for cross-row reasoning | Should |
| Review console: uncertainty-ordered queue, evidence view, one-click corrections into memory | Should |
| REST API for VYOM+ integration | Should |
| Audit log of every decision (route, model hash, prompt hash) | Should |
| QLoRA-distilled smaller SLM | Stretch |
| Model bake-off dashboard (Qwen3.5 vs Gemma 4 vs Phi) | Stretch |
| Accounting-software XML voucher export | Stretch |

---

## 16. Implementation Approach

The plan is ordered so that **a complete, working classifier exists by Phase 2**. Every later phase adds accuracy, speed or product polish on top of something that already runs end to end.

<p align="center">
  <img src="https://github.com/user-attachments/assets/56ee6166-bf48-4e6e-b4f2-f6ab094fd946" alt="Implementation phases" width="100%">
</p>

| Phase | Deliverable | Demo-able checkpoint |
|---|---|---|
| **P0** Pre-event (no code) | Model weights downloaded and quantized, the 27-card voucher rulebook and the annotation guide written as documents | Environment ready |
| **P1** Ingestion | Loader, schema mapper, normalizer, perspective resolver | Any workbook becomes clean typed rows with a mapping report |
| **P2** SLM classifier (**MVP**) | Evidence extractor, axis prompt with grammar-constrained output, decision table | Every row gets a valid label, axes and explanation |
| **P3** Evaluation | Gold set from the real data, evaluation harness, keyword and flat-prompt baselines | First scorecard |
| **P4** Cascade | Teacher labeling, LightGBM student, calibration, gate | Same accuracy at a fraction of the SLM calls |
| **P5** Agent + graph | LangGraph loop, tools, transaction graph, verifier, self-consistency | Better scores on the confusable pairs |
| **P6** Product | FastAPI, Streamlit review console, Docker Compose | Full demo |
| **P7** Stretch | QLoRA distillation, model bake-off, XML export | Extra efficiency and integration |

**Evaluation methodology**

- **Gold set.** A few hundred rows sampled from the provided dataset, stratified by predicted family and enriched with rows the teacher disagreed on, labeled with the written annotation guide. A random tenth is re-labeled blind later to measure label consistency. Dev and test splits are grouped by party and invoice to prevent leakage. The test split is touched only for final numbers.
- **Metrics.** Accuracy, macro-F1 (primary), weighted F1, per-class precision, recall and F1, confusion matrix, accuracy on each named confusable pair, expected calibration error, rows per second, p50 / p95 latency per route, peak RAM and VRAM, share of rows escalated to the SLM.
- **Baselines and ablations.** Majority class, keyword rules, flat zero-shot SLM prompt, fast tier alone, then the full system with one component removed at a time (perspective resolver, axis decomposition, transaction graph, exemplar retrieval, verifier), plus model size (2B / 4B / 9B) and quantization (4-bit / 8-bit) sweeps.
- **Stress suites.** Synthetic perturbations (10, 25 and 50 percent of fields dropped, renamed headers, Hindi-English narrations, typos), **contrast sets** (minimal edits that must flip the label, such as swapping buyer and seller or adding a port code), and **metamorphic tests** (shuffling column order or row order must not change any prediction).
- **One command.** `vouchsafe eval` produces `report.html` and `metrics.json` from pinned weights and fixed seeds.

| Configuration | Macro-F1 | Confusable-pair accuracy | Rows / s (CPU) | Escalated to SLM |
|---|---|---|---|---|
| Keyword rules | measured at final | measured at final | measured at final | 0% |
| Flat zero-shot SLM | measured at final | measured at final | measured at final | 100% |
| Vouchsafe, SLM on every row | measured at final | measured at final | measured at final | 100% |
| Vouchsafe, cascade | measured at final | measured at final | measured at final | measured at final |

---

## 17. Expected Final Output

A working, offline system with three entry points:

```bash
vouchsafe classify transactions.xlsx --out predictions.json --xlsx predictions.xlsx
vouchsafe eval --pred predictions.json --truth labels.xlsx --report report.html
docker compose up    # API on :8000, review console on :8501
```

**Per-row output** (extends the minimum format in the brief)

```json
{
  "row_id": 418,
  "invoice_number": "INV-2026-1042",
  "voucher_type": "Expense",
  "confidence": 0.91,
  "axes": {
    "stage": "BILL",
    "direction": "IN",
    "counterparty": "SUPPLIER",
    "value": "FINANCIAL",
    "border": "DOMESTIC",
    "qualifier": "SERVICE"
  },
  "evidence": ["perspective.role_in_row", "code_type", "code", "narration"],
  "explanation": "Inward tax invoice to the books owner for a maintenance service (SAC 9987), not stock-in-trade, so it is an expense bill rather than a Purchase.",
  "route": "reasoning_agent",
  "needs_review": false
}
```

**Files produced**

| File | Contents |
|---|---|
| `predictions.json` | One object per row, as above |
| `predictions.xlsx` | The original sheet with `voucher_type`, `confidence`, `needs_review` and `explanation` columns added |
| `mapping_report.json` | How every input column was interpreted, with confidence |
| `report.html` / `metrics.json` | Full evaluation: metrics, confusion matrix, per-class table, calibration plot, latency profile |
| `audit.jsonl` | One line per decision: route, model hash, prompt hash, tool calls |

**What the final demo shows**

1. `docker compose up` on a laptop with networking switched off.
2. Upload the organizers' workbook. The mapping report and the detected books owner appear.
3. The results grid with voucher type, confidence and route, filtered to rows that need review.
4. Open a row: evidence card, six-axis answers, linked documents, similar past cases.
5. Correct one label and watch similar rows update through exemplar memory.
6. Run `vouchsafe eval` and walk through the scorecard and ablations.

---

## 18. Future Scope / Scalability

<p align="center">
  <img src="https://github.com/user-attachments/assets/783b0685-5474-4837-8aa1-2ea97740628e" alt="Scalability architecture" width="100%">
</p>

**Scaling.** The API is stateless, so it scales horizontally. The fast tier runs on cheap CPU workers, and only escalations reach the GPU pool, where vLLM batches requests and reuses the shared prompt prefix. Per-tenant exemplar memory lets each client firm's conventions apply without retraining.

**Roadmap**

- **End-to-end pipeline.** Plug in upstream invoice extraction so a scanned bill goes from document to fields to voucher type to a posted entry.
- **Ledger suggestion.** Propose the debit and credit ledgers, not just the voucher type.
- **GST return mapping.** Map each voucher to the right GSTR-1 table and flag input tax credit risks.
- **Anomaly detection.** Duplicate invoices, tax-math mismatches, payments without bills.
- **Continual learning.** Corrections take effect immediately through memory, with periodic re-distillation into the student models.
- **Edge deployment.** Qwen3.5-2B or 0.8B for fully on-device use by small accountants.
- **Other tax regimes.** Swap the rulebook to support VAT-style systems elsewhere.

---

## 19. Open-Source Dependencies / Components

| Component | License | Role in Vouchsafe | Why it is needed |
|---|---|---|---|
| Qwen3.5-9B / 4B / 2B | Apache-2.0 | Primary reasoning SLM, teacher, speed variants | Open-weight reasoning that runs locally |
| Gemma 4 E4B / 26B A4B | Apache-2.0 | Bake-off contender and fallback | The model choice is decided by measurement |
| Phi-4-mini | MIT | Bake-off contender | Small-model baseline |
| multilingual-e5-small | MIT | Embeddings for headers, narrations and exemplars | Multilingual and fast on CPU |
| llama.cpp | MIT | CPU and laptop inference, JSON-schema grammars, prompt caching | Runs quantized models without a GPU |
| vLLM | Apache-2.0 | GPU inference with batching, prefix caching, structured outputs | Throughput for the teacher pass and at scale |
| LangGraph | MIT | Bounded agent state machine | Inspectable, deterministic orchestration |
| LightGBM | MIT | Fast tier | Fast, accurate learner on tabular signals |
| scikit-learn | BSD-3-Clause | Calibration, metrics, splits | Standard evaluation tooling |
| sentence-transformers | Apache-2.0 | Embedding inference | Simple, batched embedding API |
| FAISS | MIT | Exemplar memory | In-process nearest-neighbour search |
| pandas | BSD-3-Clause | Tabular processing | Core data handling |
| openpyxl | MIT | Excel reading | Merged cells, multiple sheets |
| RapidFuzz | MIT | Fuzzy header and party matching | Fast string similarity |
| Pydantic | MIT | Schemas for evidence cards and outputs | Validation at every boundary |
| NetworkX | BSD-3-Clause | Transaction graph | Document-chain queries |
| FastAPI, Uvicorn | MIT, BSD-3-Clause | REST API | Integration surface for VYOM+ |
| Streamlit | Apache-2.0 | Review console | Fast UI for reviewers |
| Faker | MIT | Synthetic transactions | Rare-class coverage and stress tests |
| MLflow | Apache-2.0 | Experiment tracking | Reproducible ablations |
| Unsloth, PEFT, TRL | Apache-2.0 | QLoRA distillation (stretch) | Smaller, faster student SLM |
| pytest, Hypothesis | MIT, MPL-2.0 | Unit, property and metamorphic tests | Guards the decision table and parsers |
| Docker Compose | Apache-2.0 | One-command offline deployment | Reproducible demo environment |
| uv | MIT / Apache-2.0 | Locked environment | Reproducible installs |

The final implementation will be released under **Apache-2.0**.

---

## 20. Expected Challenges and Mitigation

| Challenge | Impact | Mitigation |
|---|---|---|
| **No labeled training data** | Nothing to train or measure against | Rulebook labeling functions + SLM teacher with self-consistency + a hand-labeled gold set + synthetic data for rare classes |
| **Unknown columns in the hidden dataset** | Signals silently go missing | Layered schema mapper with SLM fallback, a mapping report, and a canonical schema where every field is optional |
| **Whose books is it?** | Purchase/Sales and both return types flip | Three-strategy perspective resolver, manual override, confidence carried into the gate |
| **Rare classes** | Good accuracy can hide zero recall on rare classes | Macro-F1 as the primary metric, synthetic oversampling, class weights, rulebook exemplars for every class |
| **Look-alike categories** | Systematic confusions between pairs | Axis decomposition, pairwise rulebook cards, contrast sets in evaluation |
| **Invalid or invented labels** | Unparseable output | Grammar-constrained decoding against an enum of the 27 labels, plus the verifier |
| **Non-deterministic outputs** | Inconsistent runs | Temperature 0, fixed seeds, pinned weights, consistency check in the evaluation |
| **Slow inference on CPU** | Poor speed scores | Cascade, 4-bit quantization, prompt-prefix caching, short structured outputs |
| **Our labeling conventions differ from VYOM+'s** | Correct reasoning scored as wrong | Conventions live in an editable rulebook and decision table, not in model weights. Per-class reporting makes any mismatch visible and quick to fix. |
| **Instructions hidden in narration text** | Model manipulated through the data | Narrations are delimited and marked as data, tools are read-only, output is schema-constrained |
| **New model architecture not yet supported by a runtime** | Model will not load | Fallback models with mature runtime support (Gemma 4 E4B, Phi-4-mini) behind the same interface |
| **Solo team bandwidth** | Scope risk | Phased plan with a complete MVP by Phase 2 and stretch goals clearly separated |
| **Sensitive financial and payroll data** | Privacy exposure | Fully offline, no telemetry, minimal context per prompt, hashed identifiers in logs |

---

<div align="center">

**Vouchsafe** · Hacktober Fest Open Source AI Hackathon, Elevate · Problem Statement 4

*Qualifier repository: README.md only. Implementation follows at the final hackathon.*

</div>
