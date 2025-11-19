# Ensuring Dynamic Access Control Using AI & Blockchain

This repository accompanies my 2025 cybersecurity research internship on **dynamic access control** using **Boosted-3R ABAC**, **federated learning**, **blockchain auditability**, and **lightweight anomaly detection**.

The repo is meant as a **high-level, code-free overview** of the system design, experiments, and lessons learned. It does *not* contain production code or proprietary datasets.

---

## 1. Project overview

Modern organizations generate large volumes of access logs across multiple departments and applications. Policies need to:

- Adapt to new combinations of user and resource attributes.
- Preserve privacy across organizational silos.
- Provide **auditable provenance** of rules and model updates.

This project builds on the **Boosted-3R** line of work for Attribute-Based Access Control (ABAC), which combines:

1. **Rule mining** (association rules) to cover the bulk of requests.
2. **A classifier** (CatBoost) to handle uncovered cases.
3. **Federated learning + permissioned blockchain** to coordinate updates across clients without sharing raw logs.

My internship work had two main goals:

1. **Reproduce** an on-chain Boosted-3R + federated pipeline and validate its behavior.
2. **Study lightweight anomaly screening** with Isolation Forest at two locations in the pipeline.

---

## 2. System at a glance

At a high level, the system has four main pieces:

1. **On-demand training (Boosted-3R):**
   - Stage 1: Association-rule mining on access logs.
   - Stage 2: Reliability filtering (keep high-confidence rules).
   - Stage 3: Redundancy pruning (keep a concise set of distinct rules).

2. **Federated CatBoost learner:**
   - Each department trains a local CatBoost model only on requests not covered by rules.
   - A federated server aggregates model updates to form a **global model**.

3. **Permissioned blockchain coordination:**
   - A smart contract records **hashes** of local and global models.
   - Only registered clients can submit updates.
   - The chain acts as an **audit log** for model evolution and participation.

4. **Decision engine:**
   - At runtime, requests are processed **rule-first, model-second**:
     - If a rule matches → use the rule’s decision.
     - Otherwise → query the federated CatBoost model.

[Picture: High-level architecture diagram showing clients with local logs and Boosted-3R mining, a federated server, and a permissioned blockchain recording model hashes and coordinating rounds.]

---

## 3. What I implemented

This project is based on **reproduction + extensions**:

### 3.1 Baseline reproduction (on-chain Boosted-3R)

I implemented and validated an on-chain Boosted-3R + federated pipeline across multiple clients:

- Reproduced the **stage-by-stage behavior** of the reference framework:
  - Rules alone cover roughly **70%** of requests at **mid-90s** accuracy.
  - After federated CatBoost, **coverage reaches 100%** while accuracy remains around **94–95%** on held-out logs.
- Integrated a **permissioned blockchain** (Quorum-style) where:
  - Clients submit hashes of their local CatBoost models plus metadata (e.g., sample counts).
  - The aggregator records hashes of global models and emits events so clients can synchronize.
- Kept the **heavy learning off-chain**; only deterministic hashes and minimal coordination data are written to the chain.

[Picture: Workflow diagram of the on-demand training (Boosted-3R), federated rounds, and smart contract interactions.]

---

### 3.2 Try #1 – Pre-mining Isolation Forest (training-time filter)

**Motivation.** The baseline assumes training logs are “clean.” In practice, logs may contain noise, misconfigurations, or rare-but-legitimate events. I first explored an **Isolation Forest pre-filter** that lightly trims suspicious rows before rule mining.

**Design (high-level):**

- Train an Isolation Forest **per department/silo** on local logs.
- Remove a small fraction of records flagged as outliers.
- Run Boosted-3R + CatBoost on the filtered logs; runtime behavior (rule → model) is unchanged.

**Key observations (summarized):**

- The filter removes only about **2–3%** of training rows.
- Overall accuracy remains **essentially unchanged**.
- Rule coverage stays **stable** on a dense synthetic university dataset.
- On a larger, sparser enterprise-style dataset, coverage drops **slightly**, consistent with occasionally trimming rare-but-legitimate patterns.

Interpretation: pre-mining Isolation Forest works as a **light efficiency/denoising step**—it can reduce training cost and sometimes clarify supports, but carries a small coverage risk on sparse logs.

[Picture: On-demand training pipeline showing Isolation Forest in front of the Boosted-3R stages, with arrows to rule mining and CatBoost.]

---

### 3.3 Try #2 – Decision-engine Isolation Forest (runtime gate)

To preserve **training coverage**, I then moved Isolation Forest from training time into the **decision engine**, as a runtime gate in front of rules + CatBoost:

- Train IF on logs that represent “normal” granted accesses.
- At runtime:
  1. The IF gate scores each incoming request.
  2. Clearly normal requests proceed to rules/CatBoost.
  3. Strongly anomalous requests can be flagged or held for review.

For evaluation, I injected a small fraction of **synthetic anomalies** into test streams (e.g., unusual role–resource pairs or hierarchy inconsistencies) and measured detection quality.

**Pilot result (high-level):**

- The runtime gate detects only a modest fraction of injected anomalies and raises many false alarms.
- Conclusion: in its **current unsupervised point-anomaly form**, IF is **not sufficient** as a primary runtime detector. It needs:
  - At least some supervision or weak labels,
  - Richer features (relations, history, temporal context),
  - Or combination with a supervised verifier.

[Picture: Decision-engine diagram showing an incoming request passing through an anomaly gate (IF) before the rule + CatBoost evaluation.]

---

## 4. Takeaways & future directions

From the internship work:

- **Boosted-3R + federated learning + blockchain** can be combined into a practical, auditable pipeline where rules and models are coordinated across sites without sharing raw logs.
- **Pre-mining Isolation Forest** is a reasonable **efficiency tool**, trimming a small fraction of rows with stable accuracy, but can slightly reduce coverage on sparse logs.
- As a **runtime defender**, an unsupervised IF gate is a useful probe but not enough on its own; it highlights the need to move from “rarity” to **risk-aligned detection**.

Future directions I outline in the report include:

- Two-stage screening (fast anomaly score + supervised verifier).
- Semi-/self-supervised anomaly detectors in a federated setting.
- Cross-silo sharing of **privacy-preserving anomaly signatures**.
- Better feature engineering for access logs (graphs, temporal patterns, session context).
- On-chain governance and telemetry to track how thresholds and models evolve over time.

---

## 5. Repository contents

This repository is currently **documentation-focused**. It is intended to host:

- A **project overview** (this README).
- My **internship report** (PDF) summarizing the system and experiments.
- **Slides/figures** illustrating the architecture, decision engine, and IF integration.
- (Planned) A small **public demo** on synthetic data or open datasets, once it can be released safely.

There is **no production code** or proprietary data in this repository.

---

## 6. Audience

This repo may be useful if you are:

- Interested in **ABAC policy mining** and Boosted-3R.
- Exploring **federated learning + blockchain** for access control.
- Studying **anomaly detection** in high-cardinality access logs.
- Evaluating trade-offs between **coverage, accuracy, and efficiency** in policy mining.

---

## 7. Contact

If you’d like to discuss the project, feel free to reach out:

- **Author:** Lucy (Luping) Jing  
- **Email:** luping2@ualberta.ca  
- **Affiliation at time of project:** Cybersecurity research intern, supervised by Prof. Nora Boulahia Cuppens (Polytechnique Montréal)

