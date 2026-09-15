# Data Engineering Technical Test — Insights & Assumptions

## 1. Overview

This document records the assumptions, decisions, and insights made while building
the ETL pipeline for HDB resale flat prices (Jan 2012 – Dec 2016).

## 2. Data Sources

| File | Period | Source |
|------|--------|--------|
| Resale Flat Prices (Based on Approval Date), 2000 – Feb 2012 | 2000 – Feb 2012 | data.gov.sg |
| Resale Flat Prices (Based on Registration Date), Mar 2012 – Dec 2014 | Mar 2012 – Dec 2014 | data.gov.sg |
| Resale Flat Prices (Based on Registration Date), Jan 2015 – Dec 2016 | Jan 2015 – Dec 2016 | data.gov.sg |

**Assumption:** Only rows with `month` between `2012-01` and `2016-12` are kept,
as per the assignment scope.

## 3. Assumptions

### 3.1 Remaining Lease

- HDB lease is assumed to be **99 years**.
- Lease start date is assumed to be **1 January of `lease_commence_date`**,
  since the dataset only provides the year.
- Reference date: **September 2026** (current month at time of writing).
- Remaining lease is rounded down to **years and months**.

### 3.2 Composite Key

- The composite key is **all columns except `resale_price`**.
- Derived columns (`remaining_lease_computed`, `town_status`) are
  **excluded** from the composite key, as they are computed, not source fields.
- Where duplicates exist, the record with the **higher `resale_price`** is kept
  as the reference.

### 3.3 Authoritative Set

- The **January 2012 dataset** is used as the authoritative set for validating
  `month`, `town`, `flat_type`, `flat_model`, and `storey_range`.

### 3.4 Anomaly Detection

- Method: **Segmented IQR** by `(town, flat_type)`, k = 1.5.
- Rationale: Resale prices vary significantly by town and flat type. A global
  threshold would misclassify premium models (e.g., `TERRACE`, `MAISONETTE`)
  as anomalies.
- Groups with fewer than 10 records are skipped to ensure stable quartiles.

## 4. Insights

### 4.1 Dataset Size

- Combined raw rows: X
- After deduplication: X
- Quarantined rows: X
- Final cleaned rows: X

### 4.2 Validation Findings

| Column | Invalid Values Found | Treatment |
|--------|----------------------|-----------|
| `town` | `LIM CHU KANG` | Flagged as new town (post-2012) |
| `flat_type` | `MULTI GENERATION` | Flagged as new category |
| `flat_model` | Case variants (e.g., `2-room` vs `2-ROOM`) | Standardised to canonical form |

### 4.3 Anomaly Detection Results

- Total anomalies flagged: 1,357 (1.47% of dataset)
- Method: Segmented IQR by `(town, flat_type)`, k = 1.5
- These rows are moved to the **Quarantined** output.

### 4.4 Remaining Lease Distribution

- Minimum remaining lease: X years
- Maximum remaining lease: X years
- Median remaining lease: X years

## 5. Trade-offs

| Decision | Alternative | Why Chosen |
|----------|-------------|------------|
| Segmented IQR | Global IQR | Handles premium models correctly |
| Keep higher price on duplicate | Keep lower price | Assignment specifies higher |
| Flag new values instead of dropping | Drop | Preserves data for downstream analysis |
| Compute remaining lease uniformly | Use existing column | Consistency across all files |

## 6. Limitations

- `lease_commence_date` only provides the year, so remaining lease may be off
  by up to 11 months.
- Anomaly detection is heuristic — flagged rows are not necessarily errors.
- The authoritative set (Jan 2012) may not capture all valid values in later years.

## 7. Output Structure

| Output Group | Description | Location |
|--------------|-------------|----------|
| Raw | Original files as-is | `output/raw/` |
| Cleaned | Passed all quality checks | `output/cleaned/` |
| Transformed | After transformation rules | `output/transformed/` |
| Quarantined | Failed validation, duplicates, anomalies | `output/quarantined/` |
| Hashed | Cleaned data with hashed identifier | `output/hashed/` |

---

*End of document.*