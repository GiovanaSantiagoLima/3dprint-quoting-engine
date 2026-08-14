# Data Contract — Orbi 3D Quoting & Operations System

## 1. Purpose

This document defines the structure, source, ownership, and quality expectations for every dataset and data flow used in the Orbi 3D system. It governs the interface between data producers (the business's historical records, the STL files, the user-facing form) and data consumers (the ML models, the pricing engine, the inventory/queue services).

## 2. Data Sources

| Source | Description | Format | Owner |
|---|---|---|---|
| Historical orders | 50–200 past orders with real weight, print time, material, and final price | Spreadsheet (to be exported to CSV) | Orbi 3D (business owner) |
| STL files | 3D model files corresponding to historical and new orders | `.stl` binary/ASCII files | Orbi 3D (business owner) |
| Order intake form | Free-text description + STL upload, submitted per new order | Web form (FastAPI) | Application (runtime) |
| Filament inventory | Current stock by material/color | Application database (seeded manually, updated automatically) | Orbi 3D (business owner) / Application |
| Pricing configuration | Cost per kg by material, cost per machine-hour, scrap margin | Config table/file | Orbi 3D (business owner) |

## 3. Core Entities & Schemas

### 3.1 `orders` (historical + operational)

| Field | Type | Required | Description |
|---|---|---|---|
| `order_id` | string (UUID) | Yes | Unique identifier |
| `description_text` | string | Yes | Free-text order description as entered |
| `stl_file_path` | string | Yes | Path/reference to the associated STL file |
| `part_type` | string | No (LLM-extracted) | Category of part (e.g., bracket, figurine, case) |
| `color` | string | No (LLM-extracted) | Requested color |
| `material_requested` | string | No (LLM-extracted) | Requested material (PLA, PETG, ABS, etc.) |
| `urgency` | enum [`low`,`normal`,`high`] | No (LLM-extracted) | Requested urgency |
| `estimated_weight_g` | float | Yes (model output) | Predicted weight in grams |
| `estimated_time_min` | float | Yes (model output) | Predicted print time in minutes |
| `baseline_weight_g` | float | Yes (model output) | Physics-based baseline weight |
| `baseline_time_min` | float | Yes (model output) | Physics-based baseline time |
| `actual_weight_g` | float | Only for historical/labeled data | Ground-truth weight, when known |
| `actual_time_min` | float | Only for historical/labeled data | Ground-truth print time, when known |
| `quoted_price` | float | Yes (model output) | System-generated price |
| `final_price` | float | No | Actual price charged, if different from quote |
| `status` | enum [`quoted`,`accepted`,`queued`,`printing`,`done`,`delivered`,`cancelled`] | Yes | Order lifecycle state |
| `due_date` | date | Yes (after acceptance) | Promised delivery date |
| `created_at` | timestamp | Yes | Order creation time |

**Constraints:**
- `estimated_weight_g` and `estimated_time_min` must be non-negative.
- `status` transitions must follow the defined lifecycle (no skipping from `quoted` directly to `delivered`).
- Every accepted order (`status != quoted`, `status != cancelled`) must have a non-null `due_date`.

### 3.2 `stl_features` (derived table, one row per STL)

| Field | Type | Required | Description |
|---|---|---|---|
| `stl_file_path` | string | Yes | Reference to source file |
| `volume_mm3` | float | Yes | Mesh volume |
| `surface_area_mm2` | float | Yes | Mesh surface area |
| `bbox_x_mm`, `bbox_y_mm`, `bbox_z_mm` | float | Yes | Bounding box dimensions |
| `triangle_count` | int | Yes | Number of mesh faces |
| `surface_to_volume_ratio` | float | Yes (derived) | Complexity proxy |
| `height_to_base_ratio` | float | Yes (derived) | Print-stability proxy |

**Quality rule:** any STL that fails to load or produces a non-manifold mesh must be flagged and excluded from model training, not silently imputed.

### 3.3 `filament_inventory`

| Field | Type | Required | Description |
|---|---|---|---|
| `material` | string | Yes | Material type (PLA, PETG, ABS, etc.) |
| `color` | string | Yes | Color |
| `physical_stock_g` | float | Yes | Manually counted physical stock |
| `projected_stock_g` | float | Yes | Physical stock minus weight committed to accepted/queued orders |
| `reorder_point_g` | float | Yes | Threshold that triggers a restock alert |
| `supplier_lead_time_days` | int | Yes | Expected replenishment lead time |
| `last_updated_at` | timestamp | Yes | Last update timestamp |

### 3.4 `pricing_config`

| Field | Type | Required | Description |
|---|---|---|---|
| `material` | string | Yes | Material type |
| `cost_per_kg` | float | Yes | Material cost |
| `cost_per_machine_hour` | float | Yes | Machine-time cost (depreciation + energy) |
| `scrap_margin_pct` | float | Yes | Expected failure/scrap rate, applied as a cost multiplier |
| `updated_at` | timestamp | Yes | Last update timestamp |

### 3.5 `llm_extraction_eval` (evaluation set, not used at runtime)

| Field | Type | Required | Description |
|---|---|---|---|
| `description_text` | string | Yes | Sample free-text input |
| `expected_part_type` | string | Yes | Manually labeled ground truth |
| `expected_color` | string | Yes | Manually labeled ground truth |
| `expected_material` | string | Yes | Manually labeled ground truth |
| `expected_urgency` | string | Yes | Manually labeled ground truth |

**Size requirement:** minimum 20–30 manually labeled examples, covering a representative range of description styles actually used by the owner/partner.

## 4. Data Quality Expectations

- Historical orders used for model training must have both `actual_weight_g` and `actual_time_min` populated — records missing either are excluded from training and logged as excluded, with a count reported in the model notebook.
- STL files must be validated (loadable, manifold) before being used for feature extraction; invalid files are excluded and logged.
- Pricing configuration values must be reviewed and confirmed by the business owner before the pricing model is calibrated — they are not to be estimated by the model itself.
- No customer-identifying information is required or stored beyond what is operationally necessary (this system does not need customer names/contacts for the modeling scope of this project; if added later for operational use, it must be handled separately from any published/portfolio dataset).

## 5. Data Handling for the Public Portfolio Version

Since this system uses real business data, the publicly deployed/demoed version must not expose real customer descriptions, real pricing, or proprietary cost configuration. Before publishing:
- Replace `pricing_config` values with illustrative placeholders.
- Anonymize or synthetically regenerate `description_text` fields used in any public demo or written case study.
- The trained models (weights, coefficients) may be shown/discussed, but underlying real order text should not be published verbatim.

## 6. Versioning

- Each training run of the geometry model and each dataset snapshot used must be tagged (e.g., `data_v1`, `data_v2`) so that model results in the documentation notebook are traceable to a specific dataset state.
- Changes to this contract (schema changes, new required fields) must be reflected here before being consumed by any model or service.
