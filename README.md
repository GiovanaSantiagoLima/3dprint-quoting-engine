# 3D Print Quoting Engine — Automated Quoting, Inventory & Production Pipeline

An end-to-end data science system for a real custom 3D-printing shop. It turns a short free-text order description plus an uploaded STL file into an automatic price quote, tracks filament inventory, and manages the production queue — built as a portfolio project, but designed to be genuinely usable by the business.

## Why this project

Most portfolio projects simulate a generic e-commerce store with synthetic data. This one is built on a real business with real constraints: pricing depends on both material and scarce machine time, "inventory" means both raw filament *and* production capacity, and incoming orders arrive as informal, unstructured text. That combination makes it a genuine data engineering + ML + LLM problem, not a toy exercise.

## What it does

1. **Order intake** — the owner (or her business partner) types a short description of what the customer wants and uploads the part's STL file.
2. **LLM extraction** — a locally-run open-source LLM (via Ollama) parses the free-text description into structured fields: part type, color, requested material, urgency.
3. **Geometry-based estimation** — the STL file is processed to extract geometric features (volume, surface area, bounding box, mesh complexity), which feed a regression model trained on real historical prints to estimate **weight and print time** — without needing to run a slicer.
4. **Pricing** — estimated weight/time and order attributes feed a costing model (material cost + machine-time cost + scrap margin) to produce a suggested price.
5. **Inventory & queue** — accepted orders debit a *projected* filament stock (distinct from physically counted stock) and enter a production queue sequenced by due date, which is used to estimate delivery time for the customer.
6. **Automation** — scheduled jobs recheck inventory thresholds, trigger restock alerts, and refresh the operational dashboard without manual intervention.

## Why it's technically interesting

- **Geometry → weight/time estimation** is validated against a physics-based baseline (density × volume, deposition-rate time estimate), not just reported in isolation — the project shows whether the learned model actually beats a reasonable non-ML approach.
- With a small labeled dataset (50–200 historical prints), the project deliberately favors **regularized models and cross-validation** over deep learning, and documents that trade-off explicitly rather than overfitting to a flashy technique.
- The **LLM component runs locally** (Ollama), which keeps business data private and avoids per-request API cost — a deliberate architectural choice, evaluated against a manually labeled extraction accuracy set rather than assumed to work.
- The system distinguishes **physical stock vs. projected stock**, which is what actually drives a useful restock alert (warning before you run out, not when you already have).

## Architecture

```
Web form (FastAPI + Jinja2)
        │  description text + STL upload
        ▼
┌─────────────────────────────────────────┐
│  Processing pipeline                     │
│  text → local LLM → structured fields    │
│  STL  → geometry features → ML model     │
│         → estimated weight/time          │
│  fields + estimate → pricing model       │
└─────────────────────┬─────────────────────┘
                       ▼
        Database (orders, inventory, queue)
                       ▼
┌─────────────────────────────────────────┐
│  Automation (scheduled jobs)             │
│  restock alerts · queue resequencing ·   │
│  dashboard refresh                       │
└─────────────────────────────────────────┘
```

## Tech stack

- **Backend:** FastAPI, SQLAlchemy, APScheduler
- **ML:** scikit-learn / XGBoost (regularized regression), cross-validation
- **Geometry processing:** `trimesh` / `numpy-stl`
- **LLM:** Ollama, running an open-source instruction-tuned model locally
- **Database:** PostgreSQL / MySQL
- **Dashboard:** Streamlit
- **Deployment:** Render / Railway (API + database); LLM inference documented as a self-hosted/on-premise component

## Repository structure

```
3d-print-quoting-engine/
├── app/
│   ├── main.py              # FastAPI app
│   ├── routes/               # pedidos, dashboard
│   ├── models/                # geometry model, pricing model, LLM extraction
│   ├── services/              # inventory, queue, alerts
│   ├── scheduler.py           # automated jobs
│   └── db/                    # SQLAlchemy models, schema
├── notebooks/
│   ├── 01_exploracao_stl.ipynb
│   ├── 02_modelo_geometria.ipynb
│   ├── 03_baseline_fisico.ipynb
│   └── 04_precificacao.ipynb
├── templates/                 # order intake form
├── tests/
├── PRD.md
├── DATA_CONTRACT.md
├── requirements.txt
└── README.md
```

## Documentation

- [`PRD.md`](./PRD.md) — full product requirements: goals, functional/non-functional requirements, success metrics, milestones.
- [`DATA_CONTRACT.md`](./DATA_CONTRACT.md) — schemas, data sources, and quality expectations for every dataset used.

## Status

This project is under active development as part of a data science portfolio. Milestones and validation results are documented progressively in the `notebooks/` folder as each model is trained and evaluated.

## Note on data privacy

This system runs on real business data. Any real customer descriptions, real pricing, and proprietary cost configuration are excluded or anonymized from the public version of this repository and any accompanying case study, per the data handling policy in `DATA_CONTRACT.md`.
