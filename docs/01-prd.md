# Product Requirements Document — Orbi 3D Quoting & Operations System

## 1. Overview

**Project name:** Orbi 3D — Automated Quoting, Inventory & Production Pipeline
**Owner:** Giovana Santiago Lima
**Business:** Orbi 3D, a real custom 3D-printing shop (print-to-order parts and products)
**Purpose:** A production-grade, end-to-end data science system that turns a free-text order description plus an uploaded STL file into an automatic price quote, while managing filament inventory and the production queue. Built as a data science portfolio piece targeting trainee program applications, but designed to be genuinely usable by the business.

## 2. Problem Statement

Today, order intake at Orbi 3D is manual: a customer's request (via WhatsApp or in person) is described informally by the owner or her business partner, and pricing/time estimates are produced by hand or through the slicer software. This is slow, inconsistent, and gives no structured record of demand, material consumption, or production capacity. There is no systematic link between an incoming order, the material it will consume, and the shop's current production backlog.

## 3. Goals

- Estimate print weight and time directly from STL geometry, without requiring a full slicer run.
- Extract structured order attributes (color, part type, urgency, desired material) from a free-text description using a locally-run LLM.
- Generate an automatic, defensible price quote from estimated weight/time and order attributes.
- Track filament inventory and automatically project stock depletion based on accepted orders.
- Maintain a production queue with due-date-based sequencing and provide an estimated delivery date to the customer at quote time.
- Automate recurring operational tasks (restock alerts, queue recalculation, dashboard refresh) without manual intervention.
- Serve as a rigorous, well-documented data science portfolio artifact: every model must be validated against a clear baseline and evaluated with an explicit metric.

## 4. Non-Goals

- No dimensional data warehouse / star schema — this system uses a direct operational schema, not a BI-oriented model.
- No conversion-probability / demand-elasticity modeling in v1 — the business does not yet have reliable quoted-vs-accepted history for this.
- No cloud-hosted LLM inference — extraction runs on a local open-source model (Ollama) for cost and data-privacy reasons.
- No multi-tenant support — single shop, single team (owner + business partner) usage.
- No customer-facing self-service portal in v1 — the form is operated by the owner/partner, not directly by end customers.

## 5. Users

- **Primary user:** Giovana (owner) and her business partner, who log incoming orders.
- **Secondary "user":** the system itself, via scheduled automation jobs.

## 6. Core User Flow

1. Owner/partner receives an order from a customer (via WhatsApp, in person, etc.).
2. Owner/partner opens the web form and enters a short free-text description of the order, plus uploads the STL file for the part.
3. The system:
   a. Sends the text description to a local LLM, which returns structured fields (part type, color, desired material, urgency).
   b. Extracts geometric features from the STL and predicts estimated weight and print time using a trained regression model.
   c. Feeds the estimated weight/time and structured fields into the pricing model to produce a suggested price.
   d. Checks current filament inventory and production queue to estimate a delivery date.
4. The form displays the suggested quote (price + estimated delivery date) for the owner to confirm or adjust.
5. On acceptance, the order is persisted; inventory is debited (projected), and the order enters the production queue.
6. Background jobs monitor projected inventory and trigger restock alerts; the queue is resequenced as new orders arrive.

## 7. Functional Requirements

### 7.1 Order Intake
- FR1: The system must accept a free-text description (Portuguese) and an STL file upload via a single web form.
- FR2: The system must return structured extraction output within a bounded time (target: under 10 seconds on local hardware).
- FR3: If LLM extraction fails or returns invalid/incomplete fields, the form must allow manual correction before submission.

### 7.2 Geometry-Based Estimation
- FR4: The system must extract geometric features from any valid STL file (volume, surface area, bounding box dimensions, triangle count, and derived ratios).
- FR5: The system must predict weight (grams) and print time (minutes) from these features using a trained regression model.
- FR6: The system must expose, for each prediction, a comparison against a physics-based baseline estimate (volume × material density, and a simple deposition-rate time estimate).

### 7.3 Pricing
- FR7: The system must compute a suggested price from: material cost (estimated weight × price/kg by material), machine-time cost (estimated time × cost-per-hour), and a configurable failure/scrap margin.
- FR8: Pricing parameters (cost per kg by material, cost per machine-hour, scrap margin) must be stored as configuration, not hard-coded, so they can be updated as real business costs change.

### 7.4 Inventory
- FR9: The system must maintain a filament inventory table (material type, color, quantity in grams, reorder point, supplier lead time).
- FR10: On order acceptance, the system must debit the estimated weight from a "projected stock" figure, distinct from physically counted stock.
- FR11: The system must trigger a restock alert when projected stock for any material/color falls below its reorder point, including a suggested reorder quantity.

### 7.5 Production Queue
- FR12: The system must maintain a production queue with order id, estimated print time, promised due date, and status (queued, printing, done, delivered).
- FR13: The system must sequence the queue using an earliest-due-date (EDD) heuristic and recompute sequencing whenever an order is added or its status changes.
- FR14: The system must return an estimated delivery date to the requester at quote time, based on current queue load.

### 7.6 Automation
- FR15: A scheduled job must run at least daily to: refresh demand/consumption forecasts, re-check inventory thresholds, and refresh the operational dashboard.
- FR16: Restock alerts must be delivered via a configurable channel (email or log, at minimum, for v1).

### 7.7 Reporting
- FR17: A dashboard (Streamlit or equivalent) must show: current queue, projected vs. physical inventory, and recent quote/order volume.

## 8. Non-Functional Requirements

- **Reproducibility:** every model must be trainable from a documented notebook with a fixed random seed and versioned dataset snapshot.
- **Evaluation rigor:** every predictive model must report performance against both a train/test split and k-fold cross-validation (given the small dataset size, ~50–200 labeled examples), and against the physics-based baseline.
- **Privacy:** order text and STL files stay local; no customer data is sent to third-party APIs.
- **Deployability:** the FastAPI service and database must be deployable to a managed host (e.g., Render/Railway); the local LLM component must be clearly documented as a self-hosted/on-premise dependency, with a documented fallback if unavailable.

## 9. Success Metrics

- Weight/time prediction: model outperforms the physics-based baseline on held-out data (metric: MAE or MAPE, cross-validated).
- LLM extraction: ≥ 90% field-level accuracy against a manually labeled evaluation set (~20–30 examples).
- Operational: end-to-end quote generation (text + STL → price + delivery date) completes without manual intervention for a valid input.
- Portfolio outcome: system is deployed and publicly viewable (with anonymized/synthetic data if needed for the public demo), with full documentation of modeling decisions and validation results.

## 10. Risks & Open Questions

- Dataset size (50–200 labeled STL examples) may limit model complexity — mitigated by using regularized models and cross-validation rather than deep learning.
- Local LLM extraction accuracy on informal Portuguese text is unverified until tested against the labeled evaluation set.
- Real cost inputs (cost per kg, cost per machine-hour) need to be sourced from the business before the pricing model can be calibrated.
- Deployment of the local LLM component alongside a cloud-hosted API needs an architecture decision (fully local deployment vs. hybrid).

## 11. Milestones

| Phase | Weeks | Deliverable |
|---|---|---|
| 1. Data foundation | 1–2 | Cleaned dataset of STL + real weight/time; physics-based baseline calculated |
| 2. Geometry model | 3–4 | Regularized regression model with cross-validation, benchmarked against baseline |
| 3. Local LLM extraction | 5 | Ollama-based extraction pipeline, evaluated against a labeled set |
| 4. Pricing & business logic | 6 | Costing model, inventory and queue logic |
| 5. Integration | 7 | End-to-end FastAPI + form, full pipeline working locally |
| 6. Deployment & documentation | 8+ | Live deployment, portfolio-ready README and write-up |
