## **Project Summary**

This Hospital Readmissions Analysis Dashboard evaluates hospital performance based on excess readmission ratios, a key metric used by the Centers for Medicare & Medicaid Services (CMS) to improve hospital care quality.

In October 2012, CMS introduced the Hospital Readmissions Reduction Program (HRRP), which penalizes hospitals with excess readmissions for specific conditions. The program aims to reduce preventable readmissions by incentivizing hospitals to improve discharge planning, patient follow-up, and overall quality of care.

This project analyzes excess readmission ratios across different hospital sizes, conditions, and states to identify trends, outliers, and opportunities for reducing penalties and improving patient outcomes.

---

## **Objectives of the Analysis**

- Understand the impact of the CMS Hospital Readmissions Reduction Program (HRRP) and how hospitals are penalized for excess readmissions.
- Identify hospitals with the highest excess readmission ratios and assess their impact on Medicare reimbursement.
- Compare readmission performance by state and medical condition to see where hospitals struggle the most.
- Examine hospital size as a factor in readmission rates and determine if smaller hospitals are disproportionately affected.
- Provide data-driven recommendations to improve patient outcomes and reduce excess readmissions.

---

## **Key Insights**

### 1. Hospitals with the Highest Readmission Rates

- Some hospitals have significantly higher than expected readmission rates, indicating possible care quality issues, lack of follow-ups, or complex patient cases.

### 2. State-Level Readmission Performance

- Some states consistently show higher-than-average readmission rates, possibly due to regional healthcare disparities, hospital policies, or population health factors.

### 3. Best-Performing Hospitals

- Certain hospitals outperform expectations, with readmission rates significantly lower than predicted, suggesting effective patient management programs.

### 4. Worst 5 Hospitals by Condition

- Specific hospitals for each condition (e.g., Heart Failure, COPD) have excessive readmissions, identifying areas for targeted improvements.

### 5. Best vs. Worst Hospitals by Count

- Hospitals categorized into Above Benchmark (worst-performing) and Below Benchmark (best-performing) provide a quick reference to how many hospitals exceed vs. meet expectations.

### **6. Readmission Rates by State/Hospital Size**

- Smaller hospitals (0-99 discharges) have the highest readmission rates (1.03), while very large hospitals (1000+ discharges) perform best (0.97).
- Hospitals with missing discharge data ("Unknown") behave similarly to large hospitals, suggesting missing data may belong to well-resourced facilities.
- CMS targets six specific conditions in HRRP:
    - Heart Attack (AMI)
    - Heart Failure (HF)
    - Pneumonia
    - Chronic Obstructive Pulmonary Disease (COPD)
    - Hip/Knee Replacement (THA/TKA)
    - Coronary Artery Bypass Graft Surgery (CABG)
- Hospitals with high readmission ratios for these conditions face financial penalties.
- Smaller hospitals (0-99 discharges) have the highest readmission rates (1.03), making them more vulnerable to penalties.
- Very large hospitals (1000+ discharges) perform best (0.97), likely due to better post-discharge care programs.

---

## **Data Analysis with SQL**

### **1. The Top 10 Hospitals with the Highest Readmission Rates**

```sql
SELECT TOP 10 Facility_Name, State, Excess_Ratio
FROM readmissions
WHERE Excess_Ratio IS NOT NULL
ORDER BY Excess_Ratio DESC;
```

Finds hospitals with the highest excess readmission ratios, helping identify facilities that may need improvement.

---

### **2. State-Level Readmission Ratios**

```sql
SELECT
    State,
    Measure_Name,
    ROUND(AVG(Excess_Readmission_Ratio), 2) AS Avg_Readmission_Ratio
FROM readmissions
WHERE Excess_Readmission_Ratio IS NOT NULL
GROUP BY State, Measure_Name
ORDER BY Avg_Readmission_Ratio DESC;

```

Compares readmission performance across states to find regions with higher-than-expected readmission issues.

---

### **3. Evaluating High-Performing Hospitals (Best Improvement Margins)**

```sql
SELECT TOP 10
    Facility_Name,
    State,
    Predicted_Rate,
    Expected_Rate,
    ROUND(Expected_Rate - Predicted_Rate, 2) AS Improvement_Margin,
    ROUND(Excess_ratio,2) AS Excess_Ratio
FROM readmissions
WHERE Predicted_Rate < Expected_Rate
ORDER BY Improvement_Margin DESC;

```

Finds hospitals that performed significantly better than expected, highlighting facilities that excel in reducing readmissions.

---

### **4. Worst 5 Hospitals Per Condition**

```sql
WITH National_Avg AS (
    SELECT AVG(Excess_Ratio) AS Avg_National_Readmission
    FROM readmissions
    WHERE Excess_Ratio IS NOT NULL
),
Ranked_Hospitals AS (
    SELECT
        h.Facility_Name,
        h.State,
        h.Measure,
        h.Excess_Ratio,
        RANK() OVER (PARTITION BY h.Measure ORDER BY h.Excess_Ratio DESC) AS Rank
    FROM readmissions h
    JOIN National_Avg n
        ON h.Excess_Ratio > n.Avg_National_Readmission * 1.10
)
SELECT Facility_Name, State, Measure, Excess_Ratio
FROM Ranked_Hospitals
WHERE Rank <= 5
ORDER BY Measure, Rank;

```

Ranks the worst-performing hospitals for each condition, pinpointing where intervention is needed most.

---

### **5. Best vs. Worst Hospitals by Count**

```sql
WITH National_Avg AS (
    SELECT AVG(Excess_Ratio) AS Avg_National_Readmission
    FROM readmissions
    WHERE Excess_Ratio IS NOT NULL
)
SELECT
    CASE
        WHEN h.Excess_Ratio > n.Avg_National_Readmission * 1.10 THEN 'Above Benchmark'  -- Worst-Performing
        WHEN h.Excess_Ratio < n.Avg_National_Readmission * 0.90 THEN 'Below Benchmark'  -- Best-Performing
    END AS Performance_Category,
    COUNT(*) AS Total_Hospitals
FROM readmissions h
JOIN National_Avg n ON h.Excess_Ratio IS NOT NULL
GROUP BY
    CASE
        WHEN h.Excess_Ratio > n.Avg_National_Readmission * 1.10 THEN 'Above Benchmark'
        WHEN h.Excess_Ratio < n.Avg_National_Readmission * 0.90 THEN 'Below Benchmark'
    END;

```

Categorizes hospitals into best- and worst-performing groups for a quick high-level performance view.

---

### **6. Readmission Performance by Hospital Size**

```sql
SELECT
    CASE
        WHEN num_discharges IS NULL THEN 'Unknown'
        WHEN num_discharges < 100 THEN 'Small (0-99)'
        WHEN num_discharges BETWEEN 100 AND 499 THEN 'Medium (100-499)'
        WHEN num_discharges BETWEEN 500 AND 999 THEN 'Large (500-999)'
        ELSE 'Very Large (1000+)'
    END AS Hospital_Size_Category,
    COUNT(*) AS Total_Hospitals,
    ROUND(AVG(Excess_Ratio), 2) AS Avg_Readmission_Ratio
FROM readmissions
WHERE Excess_Ratio IS NOT NULL
GROUP BY
    CASE
        WHEN num_discharges IS NULL THEN 'Unknown'
        WHEN num_discharges < 100 THEN 'Small (0-99)'
        WHEN num_discharges BETWEEN 100 AND 499 THEN 'Medium (100-499)'
        WHEN num_discharges BETWEEN 500 AND 999 THEN 'Large (500-999)'
        ELSE 'Very Large (1000+)'
    END
ORDER BY Avg_Readmission_Ratio DESC;

```

Shows the impact of hospital size on readmission rates, revealing that smaller hospitals have higher readmission rates.

---

## **Conclusion and Recommendations**

- Hospitals with high readmission rates should analyze discharge planning and post-care follow-ups.
- States with consistently high readmission rates should assess regional healthcare policies.
- Smaller hospitals may need additional support to reduce preventable readmissions.
- Larger hospitals generally perform better, suggesting centralized resources help patient outcomes.
- Hospitals should focus on reducing readmissions for HRRP-targeted conditions to avoid CMS penalties.
- Smaller hospitals (0-99 discharges) may need additional support to implement effective post-discharge care strategies.
- Hospitals with high excess readmission ratios may face Medicare payment reductions, making it critical to invest in patient follow-ups and transitional care programs.
- Healthcare policymakers can use this data to assess whether HRRP disproportionately impacts certain hospital sizes or geographic regions.

---

**(Insert Tableau Screenshot & Link Here)**

---

## **Data Source**

https://data.cms.gov/provider-data/dataset/9n3s-kdb3#overview
