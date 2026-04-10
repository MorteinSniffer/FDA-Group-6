# Digital vs. Analog Pre-Sleep Activity: Nocturnal Cardiac Response & Sleep Onset Latency

**Course:** JBM170
**Institution:** Eindhoven University of Technology (TU/e)  
**Study type:** 14-day within-subject field study  
**Data collection period:** 14 consecutive nights per participant  
**Authors:** Clement Arul, Slav Papazov, Asmita Chakraborty, Utsav Swami, Povilas Masiulionis  

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Research Question & Hypotheses](#2-research-question--hypotheses)
3. [Repository Structure](#3-repository-structure)
4. [Data Sources & Schema](#4-data-sources--schema)
5. [How to Run the Pipeline](#5-how-to-run-the-pipeline)
6. [Pipeline Architecture (R³ Framework)](#6-pipeline-architecture-r3-framework)
7. [FAIR Principles Compliance](#7-fair-principles-compliance)
8. [Ethics & GDPR](#8-ethics--gdpr)
9. [Key Results Summary](#9-key-results-summary)
10. [Limitations](#10-limitations)
11. [Dependencies](#11-dependencies)
12. [Citation](#12-citation)

---

## 1. Project Overview

This repository contains all analytical code, data-processing pipelines, and visualization scripts for a within-subject field study examining how **pre-sleep activity type** (screen-based "digital" vs. non-screen "analog") affects:

- **Nocturnal Cardiac Response (NCR):** mean heart rate and HR dipping magnitude measured via 1-minute PPG sampling on a Xiaomi Smart Band 10.
- **Sleep Onset Latency (SOL):** time elapsed from device cessation to first recorded Deep and REM sleep epochs.
- **Subjective Vitality:** next-morning energy ratings on a 1–10 scale (adapted from the Ryan & Frederick Subjective Vitality Scale).

The study fuses three data streams — continuous heart rate time series, discrete sleep-stage timestamps, and manual morning vitality logs — into a single per-night analytical record. All processing follows the **R³ (Reliable, Robust, Reproducible)** framework and the **FAIR Data Principles** (Wilkinson et al., 2016).

---

## 2. Research Question & Hypotheses

> *"To what extent does pre-sleep activity type affect nightly cardiac response and sleep onset latency in individual wellness-trackers, compared between screen-based and non-screen activity preceding sleep?"*

| Hypothesis | Prediction | Outcome |
|---|---|---|
| **H₁ — Cardiac Response** | Analog wind-down → lower mean HR and greater HR dipping magnitude | *Partial support* — dipping trend p = .057 (r = .294); mean HR non-significant (p = .956) |
| **H₂ — Sleep Onset Latency** | Analog wind-down → shorter latency to first Deep/REM epoch | *Rejected on methodological grounds* — accelerometer artefact inflated analog latency |
| **H₃ — Subjective Vitality** | Physiological differences will appear in next-morning vitality scores | *Fully supported* — analog median 7.0 vs digital 4.5 (p = .015, r = .369) |

---

## 3. Repository Structure

```
project-root/
│
├── FDA_Code_v1.ipynb          # Main analytical notebook (run cells in order)
│
├── data/
│   ├── raw/
│   │   ├── hr_data/           # Per-user CSV exports from Xiaomi Mi Fitness app
│   │   │                      # Format: Uid, timestamp_utc, bpm
│   │   ├── sleep_stages/      # Per-user JSON exports (sleep stage arrays)
│   │   │                      # SLEEP_STATE_MAP: {2: 'deep', 3: 'light', 4: 'REM', 5: 'Wake up'}
│   │   └── manual_logs/       # Per-user manual transaction CSVs
│   │                          # Columns: night_id, activity_type, intervention_start_utc,
│   │                          #          vitality_score, notes (caffeine/alcohol/exercise flags)
│   │
│   ├── processed/
│   │   ├── Fused_Dataset.csv  # Output of Cell 1: merged HR + sleep stages + manual logs
│   │   └── Nightly_Analysis_Summary.csv  # Output of Cell 5: one row per night, 16 columns
│   │
│   └── metadata/
│       └── data_dictionary.md # Column definitions, units, encoding keys (see Section 4)
│
├── outputs/
│   ├── viz_primary_outcome.png      # Violin plot — deep sleep latency by condition
│   ├── viz_boxplots.png             # Side-by-side boxplots, all 5 outcome variables
│   ├── viz_hr_trajectory.png        # Group-mean HR at 1-minute resolution (±1 SD bands)
│   ├── viz_spearman_correlations.png # Spearman scatterplots (latency → vitality; dipping → vitality)
│   └── viz_per_user.png             # Per-user slope charts, all 5 variables
│
└── README.md                  # This file
```

---

## 4. Data Sources & Schema

### 4.1 Heart Rate Data (`hr_data/`)

Exported from the Xiaomi Mi Fitness app as CSV. Sampled at **1-minute intervals** via PPG.

| Column | Type | Unit | Description |
|---|---|---|---|
| `Uid` | string | — | Anonymized participant identifier (e.g., `8196418023`) |
| `timestamp_utc` | datetime | UTC | ISO 8601 timestamp of measurement |
| `bpm` | integer | beats/min | Instantaneous heart rate |

**Quality filter applied in pipeline:** values outside `[35, 180]` BPM are discarded as physiologically implausible during sleep (de Zambotti et al., 2019).

### 4.2 Sleep Stage Data (`sleep_stages/`)

Exported as JSON arrays from the Xiaomi Mi Fitness app. Proprietary multi-stage algorithm using PPG + tri-axial accelerometry.

```
SLEEP_STATE_MAP = {2: 'deep', 3: 'light', 4: 'REM', 5: 'Wake up'}
```

Each record contains: `Uid`, `night_id` (derived from wake-up date), `stage`, `start_utc`, `end_utc`, `duration_min`.

### 4.3 Manual Transaction Logs (`manual_logs/`)

Logged each morning via structured CSV to prevent retrospective bias.

| Column | Type | Encoding | Description |
|---|---|---|---|
| `night_id` | string | `YYYY-MM-DD` (wake date) | Composite key with `Uid` |
| `activity_type` | integer | `0 = Digital`, `1 = Analog` | Pre-sleep condition |
| `intervention_start_utc` | datetime | UTC | Timestamp of device cessation |
| `vitality_score` | integer | 1–10 | Morning subjective energy (Ryan & Frederick SVS, state version) |
| `notes` | string | free text | Contextual outlier flags (caffeine, alcohol, exercise) |

### 4.4 Nightly Analysis Summary (`Nightly_Analysis_Summary.csv`)

One row per night — the primary input for all statistical tests. 16 retained columns:

`Uid`, `night_id`, `activity_type`, `deep_sleep_latency_min`, `rem_sleep_latency_min`, `light_sleep_latency_min`, `sleep_efficiency`, `sleep_deep_duration`, `sleep_rem_duration`, `sleep_score`, `vitality_score`, `hr_dipping_magnitude`, `nocturnal_mean_bpm`, `mean_bpm_deep`, `mean_bpm_light`, `mean_bpm_REM`

**`hr_dipping_magnitude`** = mean BPM of first 60 minutes of sleep minus overnight minimum BPM. Used as a proxy for autonomic recovery speed (Ben-Dov et al., 2007).

---

## 5. How to Run the Pipeline

Run cells in the notebook **in order**. Each section depends on the output of the previous one.

```
Cell 1  → Data Cleaning + Merging       → produces Fused_Dataset.csv
Cell 3  → Data Quality Audit            → prints audit report, no file output
Cell 5  → Data Summarization            → produces Nightly_Analysis_Summary.csv
Cell 7  → Statistical Modeling          → Shapiro-Wilk + Mann-Whitney U results printed
Cell 9  → Visualizations (boxplots)     → produces viz_boxplots.png, viz_primary_outcome.png
Cell 10 → GLM Analysis                  → model summary printed
Cell 11 → Spearman Correlations         → produces viz_spearman_correlations.png
```

**Input files required before running:**

Place the following in the paths shown in Section 3:
- Raw HR CSVs for each Uid under `data/raw/hr_data/`
- Sleep stage JSONs for each Uid under `data/raw/sleep_stages/`
- Manual log CSV(s) under `data/raw/manual_logs/`

---

## 6. Pipeline Architecture (R³ Framework)

The dataflow follows the **Reliable, Robust, Reproducible (R³)** framework documented at each transformation step:

```
Xiaomi Smart Band 10  →  Xiaomi Mi Fitness app  →  CSV/JSON export
       ↓
Manual transaction log (device cessation UTC, activity condition, vitality)
       ↓
Cell 1: Merge on composite key [Uid + night_id]
  • UTC normalization across all timestamps (prevents clock-drift SOL artefacts)
  • HR filter: discard BPM < 35 or > 180
  • Deduplication of identical timestamp rows
  • Validity check: deep_sleep_duration > 0 required for stage latency calculation
       ↓
Fused_Dataset.csv  (24,854 HR observations × 29 variables, 58 nights, 4 users)
       ↓
Cell 5: Aggregation to one row per night
  • Preserves: latencies, stage durations, HR metrics, vitality scores
  • Mitigates oversampling bias from minute-level data
       ↓
Nightly_Analysis_Summary.csv  (58 rows × 16 columns)
       ↓
Cell 7: Non-parametric statistics
  • Shapiro-Wilk normality tests → all key outcomes non-normal
  • Mann-Whitney U (digital n=32 vs analog n=26), effect size r = 1 - 2U/(n₁n₂)
  • Spearman rank correlations
  • GLM: Vitality ~ Condition + sleep_deep_duration
```

**Why UTC normalization matters:** even a 5–10 minute clock drift between a smartphone and wearable can artificially inflate or deflate Sleep Onset Latency. All timestamps are converted to UTC before any latency calculation.

---

## 7. FAIR Principles Compliance

This project is designed to meet the FAIR Guiding Principles (Wilkinson et al., *Scientific Data*, 2016, DOI: 10.1038/sdata.2016.18).

### Findable (F)

- **F1 — Persistent identifiers:** Each participant is assigned a unique numeric `Uid` (e.g., `8196418023`) used consistently across all three data streams and all output files.
- **F2 — Rich metadata:** The `SLEEP_STATE_MAP` dictionary explicitly documents the encoding of all proprietary sleep stage integers (`{2: 'deep', 3: 'light', 4: 'REM', 5: 'Wake up'}`), transforming raw sensor outputs into human- and machine-readable labels.
- **F3 — Identifier linkage:** Every processed record includes both `Uid` and `night_id` to unambiguously link rows across HR, sleep stage, and vitality datasets.
- **F4 — Searchable structure:** Tabular `.csv` outputs use standardized column names matching physiological convention (e.g., `nocturnal_mean_bpm`, `hr_dipping_magnitude`) to support indexing by external tools.

### Accessible (A)

- **A1 — Open protocols:** All data is stored in `.csv` (tabular) and `.json` (sensor output) formats accessible via standard open-source Python libraries (`pandas`, `json`). No proprietary reader is required.
- **A1.1 — Universal formats:** CSV and JSON are universally implementable without licensing restrictions.
- **A2 — Metadata persistence:** The `data_dictionary.md` and this README document the schema independently of the data files themselves, so metadata remains interpretable even if raw sensor exports are unavailable.

### Interoperable (I)

- **I1 — Shared language:** Python is used throughout as a formal, widely adopted language for data science. All analytical steps can be reproduced in any standard Python environment.
- **I2 — Community-standard metrics:** Raw sensor values are mapped to community-standard physiological metrics — heart rate in BPM, sleep duration in minutes, latencies in minutes — enabling comparison with published wearable validation studies (Lee et al., 2019; Zambotti et al., 2025).
- **I3 — Cross-dataset references:** The composite key `[Uid, night_id]` provides qualified links between the HR time series, sleep stage records, and vitality logs, enabling programmatic joins without ambiguity.

### Reusable (R)

- **R1 — Rich attribute documentation:** The nightly summary retains 16 attributes covering behavioral condition, cardiac metrics, sleep architecture, and subjective well-being, providing a plurality of variables for future reanalysis or meta-analytic integration.
- **R1.1 — License:** Data and code are released under **CC BY 4.0** for reuse with attribution. All third-party libraries are open-source (pandas, scipy, statsmodels, matplotlib, seaborn).
- **R1.2 — Provenance (audit trail):** Every transformation is documented in the notebook cells with inline comments:
  - HR filter bounds: `HR_MIN = 35`, `HR_MAX = 180`
  - Stage map: `SLEEP_STATE_MAP = {2: 'deep', 3: 'light', 4: 'REM', 5: 'Wake up'}`
  - Activity encoding: `0 = Digital`, `1 = Analog`
  - Aggregation logic: `.groupby(['Uid', 'night_id']).first()` for per-night collapse
- **R1.3 — Community standards:** HR dipping magnitude follows the cardiovascular research convention (Ben-Dov et al., 2007; Frontiers, 2021). Vitality is operationalized per the Subjective Vitality Scale (Ryan & Frederick, 1997; Bostic et al., 2000).

---

## 8. Ethics & GDPR

This study was reviewed under the **TU/e Code of Scientific Conduct (2019)** and the **Netherlands Code of Conduct for Research Integrity (2018)**, and classified as low-risk by the TU/e Ethical Review Board.

- **Anonymization:** Real participant names are never stored. All records use the `Uid` numeric identifier throughout.
- **Informed consent:** Participation was strictly voluntary. All data contributors are co-authors of the study.
- **GDPR/AVG compliance:** No personally identifiable information (PII) is present in any exported file. The mapping between real identities and `Uid` values is held only by the participants themselves.
- **Data minimization:** Only the variables necessary to test H₁–H₃ were retained in `Nightly_Analysis_Summary.csv`. Raw HR time series are kept separately and are not required for reproducing the main statistical results.
- **Transparency:** All analytical decisions — including the rejection of H₂ on methodological grounds rather than suppressing the result — are documented in both this README and the accompanying report.

---

## 9. Key Results Summary

| Metric | Digital Median | Analog Median | U | p | Effect r | Decision |
|---|---|---|---|---|---|---|
| Deep Sleep Latency (min) | 14.0 | 77.0 | 203.0 | .0009*** | .512 (Large) | H₂ rejected — sensor artefact |
| Vitality Score (1–10) | 4.5 | 7.0 | 262.5 | .0146* | .369 (Medium) | H₃ fully supported |
| HR Dipping Magnitude (BPM) | 10.7 | 13.5 | 293.5 | .0565† | .294 (Small-Medium) | H₁ partial — trend only |
| Nocturnal Mean HR (BPM) | 59.7 | 57.6 | 412.0 | .956 | .010 (Negligible) | H₁ not supported |
| Deep Sleep Duration (min) | 125.0 | 129.0 | 311.5 | .104 | .251 (Small) | Not significant |

*p < .05, ***p < .001, †p < .10 trend*

**GLM result:** `Vitality ~ Condition + sleep_deep_duration` → condition coefficient +0.999 (p = .024); deep sleep duration p = .477. Analog wind-down predicts ~1 point higher vitality **independently of how much deep sleep was obtained**.

---

## 10. Limitations

1. **Sample size:** n = 4 users, 58 nights. Results are not generalizable beyond healthy young adults.
2. **Latency paradox:** The Xiaomi Smart Band uses accelerometry for sleep onset detection. Stationary analog activities (reading, meditation) were misclassified as sleep, inflating analog SOL. Negative latency values (device logged sleep before cessation timestamp) confirm this artefact.
3. **HR sampling density:** 1-minute intervals capture macro dipping trends but are insufficient for HRV metrics (RMSSD, HF power) that would provide more sensitive parasympathetic markers.
4. **Expectancy bias:** Participants were aware of the digital/analog hypothesis, which may have introduced a placebo effect on vitality self-reports.
5. **Naturalistic design:** No strict counterbalancing. Condition assignment may be confounded with weekday stress cycles or other environmental factors.

---

## 11. Dependencies

```
Python >= 3.9

pandas
numpy
scipy
statsmodels
matplotlib
seaborn
json (stdlib)
os (stdlib)
```

Install all at once:

```bash
pip install pandas numpy scipy statsmodels matplotlib seaborn
```

No additional data download is required — all input files are produced by the pipeline itself or provided by participants.

---

## 12. Citation

**Dataset & code:**  
Arul, C., Papazov, S., Chakraborty, A., Swami, U., & Masiulionis, P. (2025). *Digital vs. Analog Pre-Sleep Activity: Nocturnal Cardiac Response & Sleep Onset Latency — Data and Analysis Pipeline.* Eindhoven University of Technology.

**FAIR Principles reference:**  
Wilkinson, M. D. et al. (2016). The FAIR Guiding Principles for scientific data management and stewardship. *Scientific Data*, 3, 160018. https://doi.org/10.1038/sdata.2016.18

**Vitality scale:**  
Ryan, R. M., & Frederick, C. (1997). On energy, personality, and health: Subjective vitality as a dynamic reflection of well-being. *Journal of Personality*, 65(3), 529–565.

**HR dipping convention:**  
Ben-Dov, I. Z. et al. (2007). Blunted heart rate dip during sleep and all-cause mortality. *Archives of Internal Medicine*, 167(19), 2116–2121.
