# Phase 2b: Carbon Estimation Pipeline Architecture

**Date**: 19 August 2026
**Author Role**: Principal ML/Remote Sensing Systems Architect
**Scope**: Architecture for the carbon stock and emissions-reduction estimation pipeline, bridging raw D-MRV data ingestion (Phase 2) and VVB verification (Phase 4).
**Baseline**:
- D-MRV Ingestion & Storage: [phase2-core-architecture.md](file:///c:/carbon-credit-marketplace/architecture/phase2-core-architecture.md) (§2.3, §6 Problem 3)
- VVB Verification Workflow: [phase4-marketplace-workflows.md](file:///c:/carbon-credit-marketplace/architecture/phase4-marketplace-workflows.md) (Workflow 2)
- Risk Audit: [phase6-risk-audit-and-roadmap.md](file:///c:/carbon-credit-marketplace/architecture/phase6-risk-audit-and-roadmap.md)

---

## 1. Executive Context

The Phase 2 architecture defined the **D-MRV Ingestion Service** to gather, normalize, and store raw multi-modal data (satellite tiles, IoT telemetry). However, raw NDVI values or cookstove temperature logs are not carbon credits.

The **Carbon Estimation Pipeline** (CEP) is the translation layer. It consumes raw and derived data from Object Storage/TimescaleDB and produces a statistically bounded, auditable estimate of carbon stock or emissions reduced (`confirmed volume: N tCO₂e`). This estimate forms the quantitative basis for the Phase 4 VVB Verification Workflow. The pipeline is designed around a core principle: **conservative crediting** driven by quantified uncertainty, not black-box point estimates.

---

## 2. AFOLU Estimation: Build vs. Buy Evaluation

The most complex pipeline on the platform is the AFOLU (Agriculture, Forestry, and Other Land Use) biomass estimation model, required for ARR and REDD+ methodologies. This requires converting 2D satellite imagery into 3D biomass estimates.

We evaluated two approaches for providing this capability to the platform:

| Dimension | Option A: Build In-House | Option B: Buy/Partner (Commercial dMRV Provider) |
|:---|:---|:---|
| **Description** | Build custom Computer Vision (CV) pipelines and allometric models processing raw Sentinel-2/PlanetScope imagery internally. | Integrate via API with specialized dMRV providers (e.g., Pachama, Sylvera, Renoster) who already operate verified biomass models. |
| **Cost Profile** | High initial CAPEX (ML engineering salaries, ground-truth data acquisition, GPU compute). | High OPEX (per-project or per-hectare API licensing fees). |
| **Time-to-First-Estimate** | 12–18 months for a production-grade model capable of passing VVB scrutiny. | 1–3 months (standard API integration). |
| **Model Maturity** | Low initially. Requires years of iterative calibration across different biomes. | High. Providers have multi-year head starts and biome-specific calibrations. |
| **Vendor Lock-In** | None. Complete data sovereignty. | High. The platform becomes reliant on the provider's proprietary models and uptime. |
| **Strategic Capability** | Platform owns the core intellectual property of carbon estimation. | Platform remains an aggregator/marketplace, conceding estimation IP to third parties. |

### 2.1 Recommendation: Buy / Partner for MVP

**Recommendation**: We must proceed with **Option B (Buy/Partner)** for the initial 24-month roadmap.

**Rationale**: Building a production-grade, VVB-auditable remote sensing biomass model is not a standard software engineering task; it is a multi-year applied research problem (detailed in §3). Attempting to build this in-house for the MVP introduces an unacceptable risk to the Phase 6 roadmap timeline (Jul 2027 MVP launch). The platform will ingest raw D-MRV data (Phase 2), route it to a commercial dMRV partner API, and ingest their estimation outputs.

---

## 3. Technical Risk Assessment: The AFOLU CV Layer

> [!WARNING]
> **Open Research Risk**
> If the business overrides the recommendation in §2.1 and mandates an in-house build, the AFOLU CV layer must be treated as a high-risk applied science project, not a routine integration.

### 3.1 Why Estimating Biomass from Space is Genuinely Hard

Extracting canopy height and Above Ground Biomass (AGB) from 2D optical (Sentinel-2) and Synthetic Aperture Radar (Sentinel-1) data is an inherently underdetermined problem. 

1. **The 3D from 2D Problem**: Optical imagery provides top-of-canopy reflectance. SAR provides some structural penetration but suffers from signal saturation in dense mature forests (e.g., Amazon, Congo basins). Neither directly measures volume.
2. **Ground-Truth Scarcity**: To train a CV model (CNN/ViT) to predict canopy height, it requires massive amounts of co-located ground-truth data. This requires ingesting and aligning sparse spaceborne LiDAR (GEDI, ICESat-2) and localized airborne LiDAR campaigns. Aligning 10m-resolution Sentinel-2 pixels with 25m GEDI footprints while correcting for temporal gaps and geolocation errors requires specialized geospatial expertise.
3. **Biome Variability**: A model trained on Pacific Northwest conifers will fail catastrophically on Indonesian mangroves or Brazilian tropical dry forests. An in-house model would require continuous re-calibration for every new geographic region the marketplace supports.
4. **Atmospheric and Seasonal Noise**: Cloud masking, atmospheric correction (Bottom of Atmosphere reflectance), and phenological variations (dry season leaf-off vs. wet season) introduce massive noise that simple ML models fail to generalize across.

### 3.2 Staffing Implications & Phase 6 Reconciliation

The Phase 6 roadmap (Section "Role Staffing Timeline") currently allocates **1x "D-MRV / Geospatial Engineer"** starting in Q1 2027.

- **Under the Buy/Partner Recommendation**: This staffing level is **adequate**. One engineer can manage the API integrations with commercial dMRV providers, handle the spatial boundary validations (GeoJSON topology), and manage the Phase 2 raw data storage pipeline.
- **Under a Build-In-House Override**: This staffing level is **critically inadequate**. Building the CV pipeline described above requires, at minimum, a dedicated team of 1x Remote Sensing Scientist (PhD level), 2x ML Engineers (specializing in geospatial CV), and 1x Geospatial Data Engineer. 
- **Flag**: If the business elects to build in-house, the Phase 6 roadmap and budget must be immediately amended to reflect a 4-headcount D-MRV team.

---

## 4. Estimation Approaches for Non-AFOLU Projects

Unlike AFOLU, other project types do not require complex CV pipelines and can be managed by standard engineering teams.

### 4.1 Soil Organic Carbon (SOC)
* **Approach**: Biogeochemical Modeling parameterized by ML Regression.
* **Mechanism**: Process environmental covariates (NDVI, soil moisture index, topography) via standard ML regression (Random Forests) mapped against sparse ground-truth soil core samples. Feed these parameters into established biogeochemical models (RothC, DayCent).

### 4.2 Cookstoves
* **Approach**: Statistical Extrapolation & Stratification (Not ML).
* **Mechanism**: Ingest continuous temperature telemetry from a statistically representative sample of the deployed fleet. Calculate exact usage hours. Extrapolate to the full fleet using standard frequentist statistics, stratifying by user demographics.

### 4.3 Biochar & Enhanced Rock Weathering (ERW) / DAC
* **Approach**: Deterministic Mass-Balance / Metering.
* **Mechanism**: Direct programmatic calculation based on laboratory-certified carbon content and scale weights, or direct ingestion from mass flow controllers (DAC). Zero ML required.

---

## 5. Human-in-the-Loop: VVB Integration (Workflow 2)

The Carbon Estimation Pipeline does not issue credits; it prepares a quantitative brief for the VVB. To prevent the "black box rubber-stamp" risk, the pipeline outputs probabilistic distributions (Confidence Intervals), requiring the VVB to review the uncertainty bounds.

### 5.1 Integration into Phase 4, Workflow 2
Before the developer requests VVB assignment, the pipeline generates a `VerificationBrief`. The VVB reviews this brief via the platform UI. **The VVB can override the pipeline's volume.** If they identify field errors or disagree with the stratification, they submit the verification decision as `APPROVED with ADJUSTED volume`, providing structured rationale.

### 5.2 `VerificationBrief` JSON Schema

This schema defines the payload presented to the VVB, enforcing the requirement for uncertainty bounds and conservative crediting.

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "VerificationBrief",
  "type": "object",
  "required": ["brief_id", "project_id", "monitoring_period", "estimation"],
  "properties": {
    "brief_id": { "type": "string", "format": "uuid" },
    "project_id": { "type": "string", "format": "uuid" },
    "monitoring_period": {
      "type": "object",
      "properties": {
        "start_date": { "type": "string", "format": "date" },
        "end_date": { "type": "string", "format": "date" }
      }
    },
    "estimation": {
      "type": "object",
      "required": ["mean_tco2e", "confidence_interval_90", "conservative_deduction", "recommended_volume"],
      "properties": {
        "mean_tco2e": { "type": "number", "description": "The raw mean estimate" },
        "confidence_interval_90": {
          "type": "object",
          "properties": {
            "lower_bound": { "type": "number" },
            "upper_bound": { "type": "number" }
          }
        },
        "conservative_deduction": {
          "type": "number",
          "description": "Volume deducted due to uncertainty (mean - lower_bound)"
        },
        "recommended_volume": {
          "type": "number",
          "description": "The final recommended crediting volume (lower_bound)"
        }
      }
    },
    "anomalies": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "anomaly_type": { "enum": ["SENSOR_DEVIATION", "PROVENANCE_FAILURE", "PLOT_VARIANCE"] },
          "description": { "type": "string" },
          "severity": { "enum": ["WARNING", "CRITICAL"] }
        }
      }
    }
  }
}
```

---

## 6. Model Versioning and Auditability

A carbon credit's legitimacy may be challenged years after issuance. The platform must be able to deterministically reproduce the exact tCO₂e calculation.

### 6.1 `EstimationExecutionGraph` JSON Schema

Every time the pipeline executes, it generates an immutable Estimation Execution Graph (EEG). The EEG is stored in Object Storage, and its content hash is appended to the Phase 2 `CREDIT_EVENT` (type=`ISSUED`).

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "EstimationExecutionGraph",
  "type": "object",
  "required": ["execution_id", "timestamp", "pipeline_version", "inputs", "outputs"],
  "properties": {
    "execution_id": { "type": "string", "format": "uuid" },
    "timestamp": { "type": "string", "format": "date-time" },
    "pipeline_version": {
      "type": "object",
      "properties": {
        "git_commit_hash": { "type": "string" },
        "docker_image_digest": { "type": "string" },
        "model_weights_uri": { "type": "string", "description": "S3/MLflow URI for exact model weights used" }
      }
    },
    "inputs": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "data_type": { "enum": ["SATELLITE_IMAGERY", "IOT_TELEMETRY", "FIELD_PLOTS"] },
          "source_uri": { "type": "string" },
          "sha256_hash": { "type": "string", "description": "Cryptographic hash of the exact input dataset" }
        }
      }
    },
    "outputs": {
      "type": "object",
      "properties": {
        "verification_brief_id": { "type": "string", "format": "uuid" }
      }
    }
  }
}
```

---

## 7. Security & Risk Mitigation (Phase 6 Alignment)

This architecture explicitly addresses the adversarial risks identified in the Phase 6 Risk Audit.

### 7.1 Mitigating OR-1: Sensor Data Poisoning
**Phase 6 Finding**: Compromised IoT sensors inflating telemetry.
**CEP Mitigation**: 
- The pipeline executes unsupervised anomaly detection (e.g., Isolation Forests) across the time-series IoT data before statistical extrapolation. It flags sensors with >3σ deviations from their historical baselines. These flags are surfaced in the `VerificationBrief` as `SENSOR_DEVIATION` anomalies, forcing VVB manual review.

### 7.2 Mitigating OR-3: Satellite Imagery Substitution
**Phase 6 Finding**: Man-in-the-middle attacks injecting fraudulent satellite imagery.
**CEP Mitigation**: 
- The pipeline enforces a strict pre-flight check. Before passing any Sentinel-2 or PlanetScope imagery to the CV model (or external dMRV provider), the pipeline validates the provider-signed metadata manifest (e.g., Copernicus signatures). If the signature is invalid, a `PROVENANCE_FAILURE` critical anomaly is generated and execution aborts.

---

## Executive Summary

**Date**: 19 August 2026
**Purpose**: Define the logic, validation, and infrastructure for converting raw D-MRV data into VVB-auditable carbon volume estimates.

**Key Decisions**:
1. **Buy over Build**: Explicit recommendation to partner with commercial dMRV providers for AFOLU estimation during the MVP phase due to extreme technical complexity and timeline risks.
2. **Conservative Crediting**: The pipeline outputs probability distributions (confidence intervals), not point estimates, automatically applying uncertainty deductions via the `VerificationBrief`.
3. **VVB Sovereignty**: The pipeline informs, but does not replace, the VVB. VVBs must explicitly review uncertainty bounds and can override pipeline volumes with captured rationale.
4. **Absolute Reproducibility**: Every estimate generates an immutable `EstimationExecutionGraph` (EEG) capturing data hashes, model weights, and code versions.

**Phase 6 Reconciliations**:
- **Staffing**: The allocated 1x Geospatial Engineer in Q1 2027 is adequate *only* for the recommended Buy/Partner strategy.
- **OR-1 (Sensor Poisoning)**: Addressed via Isolation Forest anomaly detection.
- **OR-3 (Imagery Substitution)**: Addressed via mandatory cryptographic provenance verification before processing.
