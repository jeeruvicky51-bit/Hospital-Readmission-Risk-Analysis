# Hospital-Readmission-Risk-Analysis
Analyzed 101,766 hospital encounters using MySQL to identify 30-day readmission risks. Applied data cleaning, CTEs, window functions, joins, and statistical testing to uncover high-risk patient groups and recommend targeted follow-up strategies.

## 🎯 The Business Problem

Hospitals are financially penalized when patients are readmitted within 30 days of discharge, and every avoidable return signals a gap in discharge planning. Care teams have limited follow-up capacity, so they need to know **who to prioritize**.

This project analyzes diabetic patient encounters from 130 U.S. hospitals (1999-2008) to answer:

1. Which patients are most likely to be readmitted within 30 days?
2. Which diagnoses, discharge settings and care patterns are linked to higher risk?
3. What simple, actionable rule could a care team use to target follow-up?

## ⚡ Headline Results

| 69,973 | 9.0% | 3.6x | 3.8x |
|:---:|:---:|:---:|:---:|
| patients analyzed | overall 30-day readmission rate | risk for patients with 4+ prior inpatient visits (29.3% vs 8.1%) | risk when discharged to rehab vs home (26.3% vs 6.9%) |

> **💡 Key insight:** Two signals available at discharge, **2+ prior inpatient stays** or **transfer to rehab / SNF / another inpatient setting**, flag only **20.7% of patients** yet capture **36.7% of all 30-day readmissions** (15.9% readmission rate, 1.8x the baseline).

## 📊 Key Findings

### 1. Prior inpatient history is the strongest predictor
Readmission climbs steadily with every prior inpatient stay in the previous year (chi-square p < 0.001).

<p align="center"><img src="images/prior.png" width="650" alt="Readmission rate by prior inpatient visits"></p>

### 2. Discharge destination matters
Patients transferred to another care setting return far more often than those discharged home (6.9%).

<p align="center"><img src="images/discharge.png" width="650" alt="Readmission rate by discharge disposition"></p>

### 3. Diagnosis category: injury and circulatory lead
Using ICD-9 range bucketing and a correlated subquery, only categories **above the 9.0% population average** were surfaced.

<p align="center"><img src="images/diag.png" width="420" alt="Readmission rate by primary diagnosis"></p>

### 4. Process signals are real but smaller
| Signal | Result |
|---|---|
| Medication burden | 7.3% (≤10 drugs) → 9.2% (11-20) → **10.5% (21+)** |
| Medication changed during stay | 9.4% vs 8.6% when unchanged |
| A1C tested during stay | 8.4% vs 9.1% when untested (p = 0.012) |
| Age | Peaks at **10.8%** for ages 80-90 |

> ⚠️ These are unadjusted, observational rates, so they show association, not causation.

## 🧹 Data Cleaning (all in SQL)

Real-world data is messy and the cleaning decisions change the answer:

| Step | What I did | Why |
|---|---|---|
| 1 | Converted `'?'` placeholders to `NULL` with `NULLIF` | Unknowns silently corrupt counts and averages |
| 2 | Profiled missingness: weight **96.9%**, specialty 49.1%, payer 39.6% | Weight was unusable and excluded from analysis |
| 3 | Deduplicated to **first encounter per patient** (self-join on `MIN(encounter_id)`) | 101,766 encounters belong to 71,518 patients; repeat visits are not independent |
| 4 | Removed patients who died or went to hospice (discharge codes 11, 13, 14, 19, 20, 21, 26) | They cannot be readmitted, so keeping them understates risk |
| 5 | Built lookup tables for discharge and admission IDs | Numeric codes become readable business labels |
| 6 | Engineered `age_midpoint` | Enables ordered age analysis |

<p align="center"><img src="images/funnel.png" width="620" alt="Data funnel from raw encounters to analytic cohort"></p>

## 🛠️ SQL Techniques Showcased

| Technique | Applied to |
|---|---|
| **CTEs** | Reusable `patient_risk_base` cohort with a 30-day readmission flag |
| **Window functions** `RANK() OVER (PARTITION BY ...)` | Ranked medication burden within each age group |
| **`NTILE(4)`** | Risk quartiles by prior inpatient count |
| **CASE-based tiering** | Low / Medium / High medication and diagnosis-complexity tiers |
| **Correlated subquery + `HAVING`** | Isolated diagnosis categories above the population average |
| **Self-join + aggregation** | Patient-level deduplication |
| **Multi-table joins** | Decoded IDs via mapping tables |

<details>
<summary><b>Example: diagnosis categories with above-average readmission</b></summary>

```sql
WITH diag_categorized AS (
  SELECT encounter_id,
    CASE
      WHEN diag_1 LIKE '250%' THEN 'Diabetes'
      WHEN CAST(LEFT(diag_1, 3) AS UNSIGNED) BETWEEN 390 AND 459 THEN 'Circulatory'
      WHEN CAST(LEFT(diag_1, 3) AS UNSIGNED) BETWEEN 460 AND 519 THEN 'Respiratory'
      WHEN CAST(LEFT(diag_1, 3) AS UNSIGNED) BETWEEN 520 AND 579 THEN 'Digestive'
      WHEN CAST(LEFT(diag_1, 3) AS UNSIGNED) BETWEEN 580 AND 629 THEN 'Genitourinary'
      WHEN CAST(LEFT(diag_1, 3) AS UNSIGNED) BETWEEN 800 AND 999 THEN 'Injury'
      ELSE 'Other'
    END AS diagnosis_category,
    CASE WHEN readmitted = '<30' THEN 1 ELSE 0 END AS is_readmitted_30
  FROM diabetic_data_dedup
  WHERE diag_1 IS NOT NULL
)
SELECT diagnosis_category,
       COUNT(*) AS total_encounters,
       ROUND(AVG(is_readmitted_30) * 100, 1) AS readmission_rate_pct
FROM diag_categorized
GROUP BY diagnosis_category
HAVING AVG(is_readmitted_30) > (SELECT AVG(is_readmitted_30) FROM diag_categorized)
ORDER BY readmission_rate_pct DESC;
```
</details>

## ✅ Recommendations

1. **Flag high-risk patients at discharge:** 2+ prior inpatient stays or transfer to rehab / SNF / another inpatient setting triggers a priority follow-up call within 48-72 hours.
2. **Strengthen facility handoffs:** standardize discharge summaries and medication reconciliation; audit facilities with the highest return rates.
3. **Extend care coordination** for circulatory and injury diagnoses and for patients on 21+ medications (pharmacist review).
4. **Add A1C testing and medication-change follow-up** to the discharge checklist.

## 🔍 Limitations

- Observational data: associations, not causal effects; rates are unadjusted.
- Data covers 1999-2008, so thresholds should be revalidated on current data.
- One encounter per patient avoids bias but cannot show how risk changes over time.
- Diabetic population only, so findings may not generalize to all patients.

## 🚀 Next Steps

- [ ] Build a logistic regression / gradient boosting risk score in Python and compare with the rule-based tiers
- [ ] Publish an interactive Power BI dashboard (drill-down by diagnosis, age, discharge setting)
- [ ] Estimate cost-benefit of targeted follow-up calls

## 📁 Repository Structure

```
Hospital-Readmission-Risk-Analysis/
├── README.md
├── sql/
│   └── hospital_readmission_analysis.sql   # full cleaning + analysis script
├── images/                                 # charts used in this README
└── report/
    └── Hospital_Readmission_Risk_Analysis_Report.pdf
```

## ▶️ How to Reproduce

1. Download the dataset from the [UCI repository](https://archive.ics.uci.edu/dataset/296/diabetes+130-us+hospitals+for+years+1999-2008) (CC BY 4.0).
2. In MySQL Workbench, create the database and import `diabetic_data.csv` as `diabetic_data_raw` and `IDS_mapping.csv` as `ids_mapping_raw`.
3. Run `sql/hospital_readmission_analysis.sql` top to bottom.

## 👤 About Me

**Vicky Jeeru**: Data Analyst & Power BI Developer, Hyderabad, India

[![LinkedIn](https://img.shields.io/badge/LinkedIn-0A66C2?style=flat&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/vicky-jeeru-257a18281/)
[![Portfolio](https://img.shields.io/badge/Portfolio-13A89E?style=flat&logo=googlechrome&logoColor=white)](https://vickyjeeru.my.canva.site/)
[![Email](https://img.shields.io/badge/Email-D14836?style=flat&logo=gmail&logoColor=white)](mailto:jeeruvicky51@gmail.com)

*Open to Data Analyst and BI roles. If this project was useful, a ⭐ is appreciated.*

**Dataset citation:** Strack et al., "Impact of HbA1c Measurement on Hospital Readmission Rates," *BioMed Research International*, 2014. UCI Machine Learning Repository, CC BY 4.0.
