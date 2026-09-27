# AAI Construction Hindrance Analytics Dashboard

## Project Overview

The **AAI Construction Hindrance Analytics Dashboard** is an end-to-end data analytics project developed from a construction hindrance register encountered during an internship workflow associated with the **Airports Authority of India (AAI)**.

The project converts operational hindrance records into an interactive Power BI dashboard for monitoring:

- Project hindrance volume
- Gross, overlapping, net and effective hindrance duration
- Root-cause categories
- Affected work components
- Hindrance trends over time
- Impact/severity distribution

The project demonstrates a complete analytics workflow using **Microsoft Excel, PostgreSQL and Power BI/DAX**.

---

## Business Problem

Construction projects can experience delays because of factors such as design and drawing approvals, forest/statutory clearances, electrical dependencies, material availability, weather, government restrictions, elections and holidays, legal issues, and other project-interface constraints.

A raw hindrance register is useful for record keeping, but it is difficult to use directly for management-level analysis.

The objective of this project was to convert the operational register into a structured analytical model and provide a dashboard that answers:

> **How much disruption occurred, when did it occur, what caused it, which work components were affected, and how severe were the recorded hindrances?**

---

## Data Source

The project is based on an **AAI construction hindrance register** containing fields such as:

- Hindrance ID / record identifier
- Nature of hindrance
- Start date
- End date
- Period of hindrance
- Overlapping period
- Net period of hindrance
- Weightage
- Effective hindrance
- Remarks and supporting remarks
- Project component
- Hindrance type
- Impact level
- Source-page reference

### Data note

The source register references a sequence of approximately 98 records, while the cleaned CSV used in the current implementation contains **97 populated records**. One serial number was not present in the supplied source extract and was not fabricated.

The dashboard therefore uses the validated **97-record cleaned dataset** currently loaded into PostgreSQL.

---

# Technology Stack

| Tool | Purpose |
|---|---|
| **Microsoft Excel** | Data cleaning, standardization, mapping and validation |
| **PostgreSQL** | Relational storage, SQL analysis and validation |
| **Power BI** | Data modelling, interactive visualization and dashboarding |
| **DAX** | KPI and analytical measure development |

---

# Project Workflow

```text
AAI Hindrance Register
        │
        ▼
Microsoft Excel
        │
        ├── Raw data capture
        ├── Data cleaning
        ├── Standardization
        ├── Hindrance-type mapping
        ├── Project mapping
        ├── Impact classification
        └── Data-quality checks
        │
        ▼
CSV Export
        │
        ▼
PostgreSQL
        │
        ├── project
        ├── hindrance_type
        ├── hindrance
        └── date
        │
        ├── SQL validation
        ├── SQL aggregation
        ├── Root-cause analysis
        └── Time/component analysis
        │
        ▼
Power BI
        │
        ├── Data model
        ├── DAX measures
        ├── Slicers
        └── Interactive dashboard
```

---

# Excel Data Preparation

The Excel stage was used as the primary data-cleaning and preparation layer.

## Raw Data

The `Raw_Data` sheet preserves the information from the source register.

## Cleaned Hindrance

The `Cleaned_Hindrance` sheet standardizes fields and adds analytical attributes such as:

- `Project_ID`
- `Project_Component`
- `Hindrance_ID`
- `Type_ID`
- `Impact_Level`

It also preserves the source-derived duration fields:

- `Period of Hindrance`
- `Overlapping Period`
- `Net Period of Hindrance`
- `Weightage`
- `Effective Hindrance`

## Lookup Tables

Additional Excel sheets were created for:

- `Hindrance_Type`
- `Project`
- `Date`

These tables were subsequently represented in PostgreSQL and Power BI.

---

# Hindrance Calculation Logic

The dashboard follows the duration logic available in the cleaned dataset:

```text
Gross Hindrance Period
          │
          ▼
Less: Overlapping Period
          │
          ▼
Net Hindrance Period
          │
          ▼
Apply Weightage
          │
          ▼
Effective Hindrance
```

Conceptually:

```text
Net Hindrance
= Period of Hindrance - Overlapping Period
```

and the intended weighted relationship is:

```text
Effective Hindrance
≈ Net Hindrance × Weightage
```

The source-derived `Effective Hindrance` values were preserved and independently checked during SQL validation rather than being silently overwritten.

---

# PostgreSQL Database

The project uses a simple relational PostgreSQL database:

```text
AAI_Hindrance_DB
│
├── project
├── hindrance_type
├── hindrance
└── date
```

## Main Table: `hindrance`

The central analytical table contains:

```text
record_id
original_sno
nature_of_hindrance
start_date
end_date
period_of_hindrance
overlapping_period
net_period_of_hindrance
weightage
effective_hindrance
remarks
remarks_ii
source_page
project_id
project_component
hindrance_id
type_id
impact_level
```

## Relationships

```text
project.project_id
        │
        │ 1 : *
        ▼
hindrance.project_id


hindrance_type.type_id
        │
        │ 1 : *
        ▼
hindrance.type_id


date.date_value
        │
        │ 1 : *
        ▼
hindrance.start_date
```

This creates a simple relational/star-style analytical model without unnecessary data-warehouse complexity.

---

# SQL Analysis

PostgreSQL was used for both validation and analytical querying.

### Total hindrance

```sql
SELECT COUNT(DISTINCT hindrance_id) AS total_hindrances
FROM hindrance;
```

### Total effective hindrance

```sql
SELECT
    ROUND(SUM(effective_hindrance), 2) AS total_effective_hindrance
FROM hindrance;
```

### Hindrance by category

```sql
SELECT
    ht.hindrance_category,
    COUNT(h.hindrance_id) AS hindrance_count,
    ROUND(SUM(h.effective_hindrance), 2) AS effective_hindrance
FROM hindrance h
JOIN hindrance_type ht
    ON h.type_id = ht.type_id
GROUP BY ht.hindrance_category
ORDER BY effective_hindrance DESC;
```

### Top hindrance types

```sql
SELECT
    h.hindrance_id,
    ht.standardized_hindrance_name,
    ht.hindrance_category,
    h.effective_hindrance
FROM hindrance h
JOIN hindrance_type ht
    ON h.type_id = ht.type_id
ORDER BY h.effective_hindrance DESC
LIMIT 10;
```

### Yearly analysis

```sql
SELECT
    EXTRACT(YEAR FROM start_date)::INT AS year,
    COUNT(*) AS hindrance_count,
    ROUND(SUM(effective_hindrance), 2) AS effective_hindrance
FROM hindrance
GROUP BY EXTRACT(YEAR FROM start_date)
ORDER BY year;
```

---

# Power BI Data Model

The PostgreSQL tables were imported into Power BI using the **Import** connection mode.

The model contains four tables:

```text
              project
                  │
                  │
                  ▼
              hindrance
              /                    /                     ▼           ▼
 hindrance_type        date
```

The relationships are:

- `project[project_id]` → `hindrance[project_id]`
- `hindrance_type[type_id]` → `hindrance[type_id]`
- `date[date_value]` → `hindrance[start_date]`

The active date relationship uses **hindrance start date** for the primary time-series analysis.

---

# DAX Measures

The dashboard uses measures to make the KPIs dynamic and responsive to slicers.

## Total Hindrances

```DAX
Total Hindrances =
DISTINCTCOUNT(
    'public hindrance'[hindrance_id]
)
```

## Gross Hindrance Days

```DAX
Gross Hindrance Days =
SUM(
    'public hindrance'[period_of_hindrance]
)
```

## Overlapping Hindrance Days

```DAX
Overlapping Hindrance Days =
SUM(
    'public hindrance'[overlapping_period]
)
```

## Net Hindrance Days

```DAX
Net Hindrance Days =
SUM(
    'public hindrance'[net_period_of_hindrance]
)
```

## Effective Hindrance Days

```DAX
Effective Hindrance Days =
SUM(
    'public hindrance'[effective_hindrance]
)
```

## Average Effective Hindrance

```DAX
Average Effective Hindrance =
AVERAGE(
    'public hindrance'[effective_hindrance]
)
```

## Critical Hindrances

```DAX
Critical Hindrances =
CALCULATE(
    [Total Hindrances],
    'public hindrance'[impact_level] = "Critical"
)
```

## Overlap %

```DAX
Overlap % =
DIVIDE(
    [Overlapping Hindrance Days],
    [Gross Hindrance Days],
    0
)
```

## Effective Hindrance Ratio

```DAX
Effective Hindrance Ratio =
DIVIDE(
    [Effective Hindrance Days],
    [Net Hindrance Days],
    0
)
```

---

# Dashboard

## AAI Construction Hindrance Analytics

The dashboard is designed as a **single-page executive overview** combining KPI monitoring and root-cause analysis.

### KPI Cards

The dashboard tracks:

1. **Total Hindrances**
2. **Gross Hindrance Days**
3. **Overlapping Hindrance Days**
4. **Net Hindrance Days**
5. **Effective Hindrance Days**
6. **Average Effective Hindrance**

### Interactive Slicers

Users can dynamically filter the dashboard using:

- **Year**
- **Hindrance Category**
- **Project Component**
- **Impact Level**

### Visualizations

#### Effective Hindrance by Category
A horizontal bar chart showing the contribution of analytical hindrance categories to effective hindrance.

#### Effective Hindrance by Work Component
A horizontal bar chart showing which project/work components are associated with greater effective hindrance.

#### Effective Hindrance Trend
A time-series line chart showing effective hindrance by month.

#### Hindrance Events by Impact Level
A donut chart showing the distribution of hindrance records across:

- Critical
- High
- Medium
- Low

---

# Dashboard Questions

The dashboard is designed to answer four management-level questions:

### How much disruption occurred?

Answered through:

- Total Hindrances
- Gross Hindrance
- Net Hindrance
- Effective Hindrance

### Why did disruption occur?

Answered through:

- Hindrance Category
- Hindrance Type

### What was affected?

Answered through:

- Project Component

### When and how severe was it?

Answered through:

- Effective Hindrance Trend
- Impact Level

---

# Data Validation

Validation was performed across the Excel and PostgreSQL stages before dashboard development.

Checks included:

- Record count
- Duplicate hindrance IDs
- Missing identifiers
- Date consistency
- Project-ID mapping
- Hindrance-type mapping
- Net-period reconciliation
- Effective-hindrance reconciliation
- Impact-level consistency
- Final Excel-to-SQL control totals

The objective was to ensure that dashboard figures could be traced back to the cleaned operational dataset.

---

# Important Data Interpretation Note

The source dataset contains hindrance-duration information, but it does **not directly contain actual revenue-loss or monetary-loss figures**.

Therefore, the current dashboard does not present a fabricated "actual revenue loss" metric.

A future extension could introduce a clearly labelled **scenario-based financial impact** measure using an externally provided daily project-cost assumption.

---

# Key Analytics Concepts Demonstrated

This project demonstrates practical experience with:

- Excel data cleaning
- Data standardization
- XLOOKUP-based mapping
- PostgreSQL table design
- CSV data import
- SQL joins
- SQL aggregations
- CTEs
- Window functions
- Date-based SQL analysis
- Power BI data modelling
- One-to-many relationships
- DAX measures
- KPI design
- Interactive slicers
- Root-cause analysis
- Time-series analysis
- Data-quality validation
- Dashboard design

---

# Suggested Repository Structure

```text
AAI-Hindrance-Analytics/
│
├── README.md
│
├── data/
│   ├── Raw_Data.csv
│   ├── Cleaned_Hindrance.csv
│   ├── Hindrance_Type.csv
│   ├── Project.csv
│   └── Date.csv
│
├── sql/
│   ├── create_tables.sql
│   ├── data_validation.sql
│   └── analysis_queries.sql
│
├── powerbi/
│   └── AAI_Hindrance_Analytics.pbix
│
└── screenshots/
    └── dashboard.png
```

---

# Project Outcome

The project transforms a construction hindrance register from a record-oriented dataset into an interactive analytics solution.

The final workflow connects:

**Excel data preparation → PostgreSQL analysis → Power BI modelling → DAX KPI development → Interactive dashboarding**

The result is a portfolio project demonstrating how operational project records can be structured and analysed to support **delay monitoring, root-cause analysis, work-impact assessment and project performance reporting**.
