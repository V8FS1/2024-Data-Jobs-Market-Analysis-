# Full Project Documentation
## Data Jobs Dashboard 2024

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Project Objective](#2-project-objective)
3. [Dataset Overview](#3-dataset-overview)
4. [Data Understanding](#4-data-understanding)
5. [Data Cleaning Steps](#5-data-cleaning-steps)
6. [Power Query Transformations — Table by Table](#6-power-query-transformations--table-by-table)
7. [Advanced Editor & M Code Discussion](#7-advanced-editor--m-code-discussion)
8. [Tables Created Outside Power Query](#8-tables-created-outside-power-query)
9. [Data Model Design](#9-data-model-design)
10. [DAX Measures](#10-dax-measures)
11. [Dashboard Design](#11-dashboard-design)
12. [Page-by-Page Explanation](#12-page-by-page-explanation)
13. [Key Findings](#13-key-findings)
14. [Challenges, Assumptions & Limitations](#14-challenges-assumptions--limitations)
15. [Lessons Learned](#15-lessons-learned)
16. [Appendix](#16-appendix)

---

## 1. Introduction

This document provides a complete technical record of the **Data Jobs Dashboard 2024** — a Power BI project built on approximately 479,000 global data job postings. It covers every stage of the project lifecycle: data ingestion, cleaning, transformation, modeling, DAX development, and dashboard design.

The documentation serves two purposes. First, it is a precise technical reference that explains what was built and how. Second, it functions as a portfolio record that demonstrates the analytical and technical skills applied at each stage — from handling messy raw data to communicating findings through a professional interactive dashboard.

The project was built entirely in Power BI Desktop using Power Query (M language) for ETL, DAX for measures, and the Power BI modeling and reporting layers for data structure and visualization.

---

## 2. Project Objective

The primary goal was to answer practical questions about the 2024 data job market that are useful to job seekers, hiring managers, and anyone tracking the data profession:

- Which data roles are most in demand globally, and where?
- What skills do employers consistently require — and how do they vary by role?
- How do salaries compare across roles and employment types?
- How remote-friendly is the data job market?
- Which companies are hiring the most, and on which platforms?
- How complete and reliable is the underlying data?

A secondary and equally important goal was to build this as a portfolio-grade project — one that demonstrates the ability to handle real-world data challenges, design a clean relational model, write analytical DAX, and deliver a polished, navigable dashboard.

---

## 3. Dataset Overview

### Source
Multiple CSV files loaded via Power Query's `Folder.Files()` function from a local directory:

```
C:\Users\Faisa\OneDrive\Documents\PowerBI\star_schema_files\
```

Files included: `job_postings_fact.csv`, `company_dim.csv`, `skills_job_dim.csv`, and a skills raw source (`Skills_Raw` — origin uncertain; likely a separate CSV or manually constructed table).

### Scale
- **~479,000 unique job postings** — the fact grain of the model
- **1.5M+ skill observations** in `Skills_Job_Dim` due to expansion of multi-value skill cells
- **98,000+ unique hiring companies**
- **198 distinct skills** (after cleaning)
- **10 standardized job role categories**
- **Global scope — 2024**

### Original Schema (`job_postings_fact.csv` — 16 columns on load)

| Column | Type | Notes |
|---|---|---|
| `job_id` | Integer | Primary key |
| `company_id` | Integer | Foreign key to company data |
| `job_title_short` | Text | Standardized role category |
| `job_title` | Text | Raw full job title (removed in final model) |
| `job_location` | Text | City/region of posting |
| `job_via` | Text | Source platform (e.g., "via LinkedIn") |
| `job_schedule_type` | Text | Employment type — multi-value (e.g., "Full-time, Part-time") |
| `job_work_from_home` | Logical | Remote flag |
| `search_location` | Text | Search query location used to find the posting (removed) |
| `job_posted_date` | DateTime | Posting date + time combined |
| `job_no_degree_mention` | Logical | Whether degree requirement was mentioned |
| `job_health_insurance` | Logical | Whether health insurance was mentioned |
| `job_country` | Text | Country of the job posting |
| `salary_rate` | Text | "hour", "year", or blank |
| `salary_year_avg` | Integer | Average yearly salary (many nulls) |
| `salary_hour_avg` | Number | Average hourly salary (many nulls) |

---

## 4. Data Understanding

Before any transformation, the following quality issues were identified in the source data:

| Issue | Column(s) Affected | Impact |
|---|---|---|
| Incorrect data types on load | All columns | Prevents date filtering, numeric calculations |
| Datetime stored as single value | `job_posted_date` | Cannot separate date from time for date table join |
| Multi-value cells | `job_schedule_type`, `job_skills` | Charts count incorrectly; filtering breaks |
| JSON-formatted skill strings | `job_skills` | Unusable for categorical analysis without parsing |
| ~94% null salary data | `salary_year_avg`, `salary_hour_avg` | Many postings have only one of the two salary types |
| Inconsistent skill naming | Skills source | 70+ variants for the same skill (e.g., "Power Bi", "Powerbi", "power BI") |
| Mislabeled country values | `job_country` | "Sudan" incorrectly applied to US-based postings |
| Prefixed source names | `job_via` | "via LinkedIn", "via Indeed" — prefix adds noise |
| Boolean flags | `job_no_degree_mention`, `job_health_insurance` | True/False renders poorly in chart labels |
| Irrelevant skills in dataset | Skills source | Tools like Zoom, Teams, Jira dilute analytical signal |

---

## 5. Data Cleaning Steps

### 5.1 Data Type Enforcement
All 16 columns were explicitly typed on load using `Table.TransformColumnTypes`. Key assignments: `job_id` and `company_id` as `Int64`; date fields as `datetime`; Boolean fields as `logical`; salary fields as `number` / `Int64`.

**Skill demonstrated:** Explicit type enforcement on ingestion prevents downstream calculation errors and is a professional ETL standard.

### 5.2 Datetime Column Split
The `job_posted_date` column contained both date and time in a single datetime value. A new date-only column was extracted using `Date.From([job_posted_date])`, the columns were reordered, and then renamed to create `Posted_Date` (date) and `Posted_Time` (datetime).

This is required to create a clean join between the fact table and the Date Table on a pure date key.

### 5.3 Job Source Text Cleaning
Two cleaning operations were applied to `job_via`:
1. The string `"via "` was stripped from the beginning of all values using `Table.ReplaceValue`
2. Any value containing "linkedin" (case-insensitive, using `Comparer.OrdinalIgnoreCase`) was standardized to `"LinkedIn"`

**Skill demonstrated:** Case-insensitive text matching and prefix normalization — production-quality text cleaning.

### 5.4 Salary Null-Filling (Bidirectional)
The dataset mixed hourly and yearly salary postings, with many nulls in the non-primary column:

- `salary_hour_adj` = `salary_hour_avg × 2,080` (hourly → yearly conversion)
- `Salary_Yearly_adj` = `salary_year_avg ÷ 2,080` (yearly → hourly conversion)
- `Yearly_Salary` = `salary_year_avg` if not null, else `salary_hour_adj`
- `Hourly_Salary` = `salary_hour_avg` if not null, else `Salary_Yearly_adj`

The 2,080-hour figure represents a standard 40-hour work week across 52 weeks. After null-filling, the intermediate adjustment columns (`salary_hour_adj`, `Salary_Yearly_adj`) were removed from the final table.

**Skill demonstrated:** Domain-aware feature engineering; two-pass null resolution; clean-up of intermediate columns.

### 5.5 Salary Classification Columns
Three new columns were created to classify salary availability:

- **`Salary_Rate_Original`** — conditional column: `"Yearly Original"` if `salary_year_avg ≠ null`; `"Hourly Original"` if `salary_hour_avg ≠ null`; `"Not Mentioned"` otherwise
  
- **`Salary_Mentioned`** — binary flag: `"Salary Disclosed"` / `"Salary Not Disclosed"` derived from `Salary_Rate_Original`. This column was created incorrectly in a first pass (duplicated from the wrong column), then removed and rebuilt correctly.

### 5.6 Salary Bucketing
`Salary_Hourly_Bucket` was created using a dynamic formula with `rangeSize = 10`, producing labels in the form `"0-10"`, `"10-20"`, `"20-30"` etc., enabling a histogram-style distribution visual on the Salary Insights page.

### 5.7 Boolean-to-Label Conversions
Two conditional columns replaced True/False source flags with dashboard-readable labels:
- `Degree` — `true` → `"Not Required"`, `false` → `"Degree Required"` (from `job_no_degree_mention`)
- `Health_Insurance` — `true` → `"Provided"`, `false` → `"Not Provided"` (from `job_health_insurance`)

**Skill demonstrated:** UX-aware data preparation — labels that render cleanly in charts without requiring visual-layer formatting.

### 5.8 Country Mislabeling Correction
While building country-based charts, Sudan appeared anomalously high in both hiring volume and salary rankings. Cross-referencing the `search_location` column confirmed that every job posting with `job_country = "Sudan"` had a US-based search location. All such rows were corrected to `"United States"` using `Table.ReplaceValue`.

**Skill demonstrated:** Country Data Correction - catching and correcting a source error through chart-driven investigation, not just a prior audit.

### 5.9 Company Name Standardization
The `company_name` column in `company_dim.csv` contained `"confidential"` as a value for undisclosed companies. These were replaced with `"Undisclosed"` to be more informative and consistent.

### 5.10 Column Removals
After normalization, the following columns were removed from `Job_Postings` to avoid redundancy and keep the fact table clean:
- `job_title` (long-form) — only `job_title_short` (→ `Job_Title`) retained
- `search_location` — used only for country validation
- `salary_rate` — replaced by `Salary_Rate_Original`
- `salary_year_avg`, `salary_hour_avg` — replaced by `Yearly_Salary`, `Hourly_Salary`
- `salary_hour_adj`, `Salary_Yearly_adj` — intermediate calculation columns

---

## 6. Power Query Transformations — Table by Table

### 6.1 Job_Postings
**Source:** `job_postings_fact.csv` via `Folder.Files()`

The most complex transformation pipeline in the project — approximately 20 named steps in the Advanced Editor. The full sequence covers type enforcement, datetime split, text cleaning (`job_via`), salary null-filling (4 new columns), salary classification (3 new columns), bucketing, Boolean-to-label conversions, country correction, column removal, and final renaming.

Final columns (19):
`Job_id`, `Company_id`, `Job_Title`, `Job_Location`, `Job_Via`, `job_schedule_type`, `Is_remote`, `Posted_Date`, `Posted_Time`, `No_Degree_Mentioned`, `Has_Health_Insurance`, `Country`, `Yearly_Salary`, `Hourly_Salary`, `Salary_Hourly_Bucket`, `Salary_Rate_Original`, `Salary_Mentioned`, `Degree`, `Health_Insurance`

---

### 6.2 Company_Dim
**Source:** `company_dim.csv` via `Folder.Files()`

Original file had 5 columns: `company_id`, `name`, `link`, `link_google`, `thumbnail`.

Transformations:
- `thumbnail`, `Google_Search` column removed
- Columns renamed: `name` → `Company_Name`, `link` → `Company_Website`, 
- `"confidential"` values in `Company_Name` replaced with `"Undisclosed"`

Final columns: `company_id`, `Company_Name`, `Company_Website`

---

### 6.3 Employment_Type_Dim
**Source:** Reference of `Job_Postings` (not a separate CSV)

The `job_schedule_type` column in the source data frequently contained multiple employment types in a single cell (e.g., *"Full-time, Part-time, and Internship"*), which would cause incorrect counts in any chart that filters by employment type.

Transformations:
1. Selected only `Job_id` and `job_schedule_type` columns
2. Replaced `", and"` → `","` and `" and"` → `","` to normalize the delimiter
3. Applied `Table.ExpandListColumn` with `Splitter.SplitTextByDelimiter(",")` to expand multi-value rows into individual rows (one employment type per row)
4. Applied `Text.Trim` to clean whitespace
5. Renamed `job_schedule_type` → `Employment_Type`
6. Filtered out any empty `Employment_Type` rows

Result: Every row in this table contains exactly one employment type. Because a single 
`Job_id` can have multiple rows here, this table sits on the **many side** of its 
relationship with `Job_Postings`. Bidirectional cross-filtering was implemented so that 
employment type selections propagate correctly back to the fact table and all connected 
dimensions.

Final columns: `Job_id`, `Employment_Type`

---

### 6.4 Skills_Job_Dim
**Source:** `skills_job_dim.csv` via `Folder.Files()`

This table acts as the bridge/junction table resolving the many-to-many relationship between job postings and skills. It explicitly expands the grain of the dataset, and mapping the '*' side of this bridge to the '1' side of `Job_Postings` is a deliberate design choice to resolve many-to-many relationships while preserving filter propagation.

Transformations:
1. Loaded from CSV (2 columns, Windows-1252 encoding)
2. Renamed `job_id` → `Job_id`, `skill_id` → `Skill_id`
3. Set both columns to `Int64` type
4. Performed an **inner join** with the `Skills` table on `Skill_id`
5. Expanded the join result and removed the joined `skill_id` column

The inner join serves as an automatic filter - only skill IDs that survived the cleaning process in the `Skills` table are retained in this bridge table. Skills that were marked for removal during cleaning are automatically excluded.

Final columns: `Job_id`, `Skill_id`

---

### 6.5 Skills
**Source:** `Skills_Raw` (origin uncertain — likely a CSV or manually constructed table)

This is the most complex transformation in the entire project. The raw skills data contained inconsistent casing, abbreviated names, duplicate entries, and a category taxonomy that was not analytically useful.

**Step 1 — Text normalization:**
Applied `Text.Trim` + `Text.Clean` to both `Skills` and `Type` columns, then `Text.Proper` to capitalize each word.

**Step 2 — Type column normalization:**
Added `Type Clean` column to standardize raw type values:
- `"Webframeworks"` → `"Web Frame Works"`
- `"Os"` → `"Operating System"`
- `"Analyst_Tools"` → `"Analyst Tools"`

**Step 3 — Skill name standardization (70+ rules):**
A `Standardized Skill` column was created using a long `if/else` chain covering 70+ individual corrections. Examples:
- `"Power Bi"` / `"Powerbi"` → `"Power BI"`
- `"Aws"` → `"AWS"`, `"Gcp"` → `"GCP"`, `"Sql"` → `"SQL"`
- `"Pytorch"` → `"PyTorch"`, `"Tensorflow"` → `"TensorFlow"`
- `"Github"` → `"GitHub"`, `"Npm"` → `"NPM"`
- `"Mongodb"` / `"Mongo"` → `"MongoDB"`

**Step 4 — Inline category list definitions:**
Seven named lists were defined directly inside the M query:

| List Name | Skills Included (examples) |
|---|---|
| `RemoveSkills` | Zoom, Teams, Slack, Jira, Notion, Word, PowerPoint, Watson, GDPR, Unity |
| `CloudDevOpsSkills` | AWS, GCP, Azure, Docker, Kubernetes, Git, GitHub, Terraform, Bash |
| `DatabaseSkills` | PostgreSQL, MySQL, MongoDB, Snowflake, BigQuery, Redshift, Oracle |
| `WebFrameworkSkills` | React, Django, Flask, Node.js, Angular, FastAPI, Vue, Next.js |
| `BIToolSkills` | Power BI, Tableau, Excel, Looker, Databricks, SAS, Jupyter, Alteryx |
| `DataScienceSkills` | PyTorch, TensorFlow, Pandas, NumPy, scikit-learn, Spark, Kafka |
| `OperatingSystemSkills` | Linux, Ubuntu, Windows, macOS, Red Hat, CentOS, Debian |

**Step 5 — Dual-priority category assignment:**
The `Category` column was assigned using list membership as the primary lookup, with `Type Clean` as the fallback:
- If a skill is in `RemoveSkills` → `"Remove"`
- Else if in a named list → that list's category name
- Else if `Type Clean` matches a known type → mapped category
- Else → `"Remove"`

**Step 6 — Filter and finalize:**
- `KeepSkill` boolean flag added (`[Category] <> "Remove"`)
- Table filtered to `KeepSkill = true`
- `KeepSkill`, original `Skills`, `Type`, and `Type Clean` columns removed
- `Standardized Skill` renamed to `Skills`
- `Table.Distinct` applied three times to handle edge-case duplicates:
  - `"Sql Server"` → `"SQL Server"` fix + dedup
  - `"No-Sql"` / `"Nosql"` → `"NoSQL"` fix + dedup

Final result: **198 distinct, correctly named, meaningfully categorized skills** across 7 categories.

Final columns: `skill_id`, `Skills`, `Category`

---

### 6.6 Top_5_Skills
**Source:** Reference of `Skills_job_dim`

This is a pre-aggregated analytical table built to power the top-skills-per-role table visual on the Skills & Roles page. It was designed as a summary table rather than a dynamic calculation because it requires a complex multi-step aggregation that is more efficient to pre-compute.

**Pipeline:**
1. Start with `Skills_job_dim`
2. **Left outer merge** with `Job_Postings` → expand to get `Job_Title`
3. **Left outer merge** with `Skills` → expand to get `Skills` name
4. **First Group By** on `Job_Title` + `Skills` → produces `Skill_Count` (frequency)
5. **Second Group By** on `Job_Title` → produces a nested `All_Rows` table per job title
6. **Filter** rows where `Job_Title = null`
7. **Custom column** — for each job title's nested table:
   - Sort by `Skill_Count` descending, then `Skills` ascending (deterministic tie-breaking)
   - `List.FirstN(..., 4)` — take the top 4 skill names
   - `Text.Combine(..., ", ")` — concatenate into a single comma-separated string
8. Remove `All_Rows` column
9. **Split** the concatenated string into 4 columns by delimiter `", "`
10. **Rename** columns to `Top Skill 1` through `Top Skill 4`

Final columns: `Job_Title`, `Top Skill 1`, `Top Skill 2`, `Top Skill 3`, `Top Skill 4`

---

## 7. Advanced Editor & M Code Discussion

### 7.1 Data Ingestion Pattern
All CSV files were loaded using `Folder.Files()`, which reads from a folder path rather than individual file references. This makes it straightforward to update the data source by replacing files in the folder without changing the query logic.

```m
Source = Folder.Files("C:\Users\Faisa\OneDrive\Documents\PowerBI\star_schema_files")
```

Each file was then selected from the `Source` table by its filename and imported via `Csv.Document`.

### 7.2 Inline List Definitions in M
One of the more sophisticated M code patterns in this project is the definition of category membership lists directly inside the `Skills` query. Rather than using a lookup table, the lists are defined as named M variables within the `let` block and then referenced in a downstream `Table.AddColumn` step. This keeps the categorization logic self-contained and readable.

### 7.3 Dual-Priority Category Logic
The `Category` assignment in the `Skills` table uses a two-tier logic:
- **Tier 1:** Explicit list membership (`List.Contains`) — covers known technology names that might be miscategorized by the source `Type` field
- **Tier 2:** Source `Type Clean` value — acts as a fallback for any skill not explicitly assigned in Tier 1

This pattern ensures that well-known tools (e.g., Power BI, which might appear under "Analyst Tools" in the source) receive the correct custom category regardless of the source classification.

### 7.4 Dynamic Salary Bucketing
The `Salary_Hourly_Bucket` formula uses a `rangeSize` variable set to `10`, making the bucket boundaries dynamic rather than hardcoded:

```m
each let
    rangeSize = 10,
    offset = 0,
    inclusive = false,
    rangeIndex = Number.RoundDown(([Hourly_Salary] - offset) / rangeSize)
in
    Text.From(rangeIndex * rangeSize + offset, "en-US") & "-" &
    Text.From((rangeIndex + 1) * rangeSize + offset - (if inclusive then 1 else 0), "en-US")
```

This produces labels `"0-10"`, `"10-20"`, `"20-30"` etc. and can be reconfigured simply by changing `rangeSize`.

### 7.5 ExpandListColumn Pattern for Multi-Value Normalization
Rather than the split-to-N-columns + unpivot approach described in earlier workflow notes, the final M code for `Employment_Type_Dim` uses the more elegant `Table.ExpandListColumn` with `Splitter.SplitTextByDelimiter`:

```m
Table.ExpandListColumn(
    Table.TransformColumns(#"Replaced Value", {
        {"job_schedule_type",
         Splitter.SplitTextByDelimiter(",", QuoteStyle.Csv),
         let itemType = (type nullable text) meta [Serialized.Text = true] in type {itemType}}
    }),
    "job_schedule_type"
)
```

This directly expands the multi-value column into separate rows without the intermediate wide-column step.

### 7.6 Top-N Extraction with Nested Tables
The `Top_4_Skills` table uses a compound M expression to rank and extract top skills from a nested table:

```m
Text.Combine(
    List.FirstN(
        Table.Column(
            Table.Sort(
                [All_Rows],
                {{"Skill_Count", Order.Descending}, {"Skills.Skills", Order.Ascending}}
            ),
            "Skills.Skills"
        ),
        5
    ),
    ", "
)
```

This sorts the nested per-job-title table, extracts the top 5 skill names as a list, and joins them into a single string. The secondary sort on `Skills.Skills` ascending ensures deterministic results when two skills share the same count.

### 7.7 Inner Join as Filter
The `Skills_Job_Dim` query performs an inner join against the cleaned `Skills` table:

```m
Table.NestedJoin(#"Changed Type", {"Skill_id"}, Skills, {"skill_id"}, "Skills", JoinKind.Inner)
```

This is not just a lookup — it acts as a filter. Any `Skill_id` in the bridge table that does not have a matching entry in the cleaned `Skills` dimension (i.e., skills that were removed during cleaning) is automatically excluded. The two tables stay in sync without a separate filter step.

### 7.8 Self-Correcting Pipeline (Salary_Mentioned)
The `Job_Postings` M code contains a visible correction mid-pipeline. `Salary_Mentioned` was first created incorrectly by duplicating the `Salary_Hourly_Bucket` column, then immediately removed and rebuilt correctly using conditional logic based on `Salary_Rate_Original`. This appears as three consecutive steps in the pipeline — a real-world example of iterative refinement visible in the query history.

---

## 8. Tables Created Outside Power Query

### 8.1 Date Table
A dedicated time dimension was created outside Power Query — either via DAX `CALENDAR()` or `CALENDARAUTO()` *(exact method unconfirmed)* — and formally marked as the Date Table in the Power BI model settings.

| Column | Purpose |
|---|---|
| `Date` | Primary key — joins to `Posted_Date` in `Job_Postings` |
| `Day Name` | Day of week label |
| `Day Number` | Numeric day (1–7) |
| `Month Name` | Month label |
| `Month Number` | Numeric month (1–12) |
| `Quarter` | Q1–Q4 |
| `Week Number` | ISO week number |
| `Year` | 4-digit year |

Marking the table as a Date Table disables Power BI's auto date/time feature and enables proper DAX time intelligence functions (`TOTALYTD`, `DATESYTD`, etc.).

**Skill demonstrated:** Understanding of the Date Table requirement in Power BI — a foundational best practice for any time-based analysis.

### 8.2 Data Quality Table
A static reference table created via Power BI's "Enter Data" feature. It stores pre-calculated data coverage figures for four key fields:

| field | status | percentage |
|---|---|---|
| Salary | Available | 6.35% |
| Salary | Missing | 93.65% |
| Health Insurance | Available | 12.83% |
| Health Insurance | Missing | 87.17% |
| Website Listed | Available | 35.55% |
| Website Listed | Missing | 64.45% |
| Degree Info | Available | 32.52% |
| Degree Info | Missing | 67.48% |

This table powers the stacked available/missing bar chart on the Data Quality page. The percentages may be hard-coded from external calculation or linked to DAX measures *(exact source unconfirmed)*.

**Skill demonstrated:** Knowing when to use a purpose-built static table rather than a complex dynamic calculation — a practical dashboard engineering decision.

### 8.3 Measure Tables
Two empty tables were created to serve as organizational containers for DAX measures:
- **`Skills Measures`** — holds 5 skills-focused measures
- **`Measures`** — holds 19 general KPI measures

Using dedicated measure tables is a professional Power BI practice that keeps the field list clean and makes measures easy to find and maintain.

---

## 9. Data Model Design

### 9.1 Schema Overview

The model follows a **star schema** structure with `Job_Postings` as the central fact table, 
surrounded by dimension tables, a bridge table for the many-to-many skills relationship, a 
dedicated Date Table, and supporting analytical and display tables. Categorical attributes 
are decoupled into dedicated dimensions, keeping the fact table clean and the model 
optimized for flexible cross-filtering.

```
                         ┌─────────────────┐
                         │   Date Table    │
                         │   (time dim)    │
                         └────────┬────────┘
                              1   │ Date → Posted_Date
                                  │ (single direction →)
    ┌──────────────────┐          │ *
    │   Company_Dim    │   1──────┴──────*    ┌──────────────────────┐
    │  (dimension)     │◄──────Job_Postings──►│ Employment_Type_Dim  │
    └──────────────────┘   *  (fact table) *  │  (detail/dim table)  │
      company_id → *          │    │          └──────────────────────┘
    (single direction →)      │ *  │ *            (bidirectional ◄►)
                              │    │
               ┌──────────────┘    └──────────────────┐
               │ *                                     │ *
               │ (bidirectional ◄►)                    │
    ┌──────────┴───────┐                   ┌───────────┴──────────┐
    │  Top_4_Skills    │                   │    Skills_Job_Dim    │
    │(summary table)   │                   │   (bridge M:M)       │
    └──────────────────┘                   └───────────┬──────────┘
      Job_Title → *                                    │ *
    (single direction →)                               │ (single direction →)
                                                       │ 1
                                            ┌──────────┴──────────┐
                                            │       Skills        │
                                            │    (dimension)      │
                                            └─────────────────────┘

Separate (no model relationships):
┌───────────────┐   ┌──────────────────┐
│  Data Quality │   │  Measure Tables  │
│ (static ref)  │   │ (DAX containers) │
└───────────────┘   └──────────────────┘
```

### 9.2 Table Roles

| Table | Role | Notes |
|---|---|---|
| `Job_Postings` | Fact | Central table; 19 columns; one row per job posting |
| `Company_Dim` | Dimension | Company attributes; joined on `Company_id` |
| `Employment_Type_Dim` | Dimension | Normalized from multi-value source column |
| `Skills` | Dimension | 198 cleaned skills with 7 categories |
| `Skills_Job_Dim` | Bridge (M:M) | Resolves many-to-many between jobs and skills |
| `Date Table` | Time dimension | Marked as Date Table; enables time intelligence |
| `Top_4_Skills` | Analytical summary | Pre-aggregated; powers the top skills table visual; extracts top 4 skills per job title |
| `Data Quality` | Static display | Powers the coverage chart on Data Quality page |
| `Skills Measures` | Measure container | 5 DAX measures; no data rows |
| `Measures` | Measure container | 19 DAX measures; no data rows |

### 9.3 Relationships

| From Table | Column | To Table | Column | Cardinality | Filter Direction | Notes |
|---|---|---|---|---|---|---|
| `Employment_Type_Dim` |`Job_id`| `Job_Postings` | `Job_id` | Many → One (* : 1) | **Bidirectional (◄►)** | Allows employment type filters to propagate both ways |
| `Skills_Job_Dim` | `Job_id` | `Job_Postings` | `Job_id` | Many → One (* : 1) | **Bidirectional (◄►)** | Bridge table requires bidirectional to enable skill-based filtering on the fact table |
| `Skills_Job_Dim` | `Skill_id` | `Skills` | `skill_id` | Many → One (* : 1) | Single (Skills → Skills_Job_Dim) | Standard dimension-to-bridge filter |
| `Company_Dim` | `company_id` | `Job_Postings` | `Company_id` | One → Many (1 : *) | Single (Company_Dim → Job_Postings) | Standard dimension-to-fact filter |
| `Date Table` | `Date` | `Job_Postings` | `Posted_Date` | One → Many (1 : *) | Single (Date Table → Job_Postings) | Time intelligence filter flow |
| `Top_4_Skills` | `Job_Title` | `Job_Postings` | `Job_Title` | Many → One (* : 1) | Single (Job_Postings → Top_4_Skills) | Summary table joined on job title for visual use |

**Notes on bidirectional filtering:**
Both `Employment_Type_Dim` and `Skills_Job_Dim` use bidirectional cross-filters. These tables sit on the `*` (many) side of their relationships with `Job_Postings`, meaning they contain expanded, non-unique rows. Bidirectional filtering allows slicer selections on Employment Type or Skill to propagate back through the bridge to filter `Job_Postings` and, by extension, all other connected dimensions — enabling correct KPI values when filtering by employment type or skill category.

Technical Note: Bidirectional filtering was implemented so that skill and employment selections in the detail/bridge tables can **Filter-Up** from the 'Many' side to the central `Job_Postings` hub. This preserves `Job_Postings` as the authoritative fact table while still allowing 
detail-level selections to drive the model.

---

## 10. DAX Measures

### 10.1 Skills Measures Table

| Measure | Description |
|---|---|
| `Skill Count` | Total number of skill mentions in current filter context (likely `COUNTROWS(Skills_Job_Dim)`) |
| `Skill Per Job` | Average skills per job posting (`[Skill Count] / [Job Count]`) |
| `Most Demanded Skill` | Returns the single skill with the highest count in current context (likely uses `TOPN` + `RANKX` pattern) |
| `Show Top Skill` | Display-optimized wrapper around `Most Demanded Skill` — powers the KPI card |
| `Skill Rank by Job Title` | Ranks skills within each job title by mention frequency — uses `RANKX` pattern |

### 10.2 General Measures Table

**Posting Volume:**

| Measure | Description |
|---|---|
| `Job Count` | `COUNTROWS(Job_Postings)` — total postings in filter context |

**Salary Metrics:**

| Measure | Description |
|---|---|
| `Median Yearly Salary` | `MEDIAN(Job_Postings[Yearly_Salary])` — median used over mean for skew resistance |
| `Median Hourly Salary` | `MEDIAN(Job_Postings[Hourly_Salary])` |
| `Jobs With Salary` | Count of postings where `Salary_Mentioned = "Salary Disclosed"` |
| `Jobs With Salary By Country` | Same, filtered to current country context |
| `Salary Coverage Rate Percentage` | `[Jobs With Salary] / [Job Count]` |
| `Jobs Without Salary Rate Percentage` | Inverse of coverage rate |
| `Salary Coverage By Employment Type Percentage` | `[Jobs With Salary]` within employment type context |
| `Salary Sample Size` | Count of rows used in salary median calculation |
| `US Salary Data Share Percentage` | Proportion of salary data originating from US postings |

**Benefits & Requirements:**

| Measure | Description |
|---|---|
| `Health Insurance Coverage Percentage` | % of postings where `Health_Insurance = "Provided"` |
| `Companies Offering Insurance` | Distinct count of companies offering health insurance |
| `No Degree Required` | Count of postings where `Degree = "Not Required"` |
| `Degree Waiver Rate` | `[No Degree Required] / [Job Count]` |
| `Company Waiving Degree` | Distinct count of companies with at least one no-degree posting |
| `Degree Info Coverage Percentage` | % of postings where degree info is not null |

**Company & Geography:**

| Measure | Description |
|---|---|
| `Remote Friendly Companies` | Distinct count of companies with `Is_remote = true` postings |
| `Companies With Website Listed Percentage` | % of companies with non-blank `Company_Website` |
| `Country Rank Excl US` | `RANKX` on job count or salary, filtered to exclude United States |

### 10.3 Key DAX Patterns Used

**Median over Mean:** Every salary measure uses `MEDIAN` rather than `CALCULATE(AVERAGE(...))`. Salary data is right-skewed by high-end roles; median is more representative of a typical posting.

**RANKX pattern:** Used in `Skill Rank by Job Title` and `Country Rank Excl US` to produce relative rankings within a filter context. This is an intermediate-to-advanced DAX pattern.

**Coverage rate pattern:** Multiple measures follow the structure `[subset count] / [total count]` to produce a % that updates dynamically with slicer selections. This pattern powers the salary coverage, degree, and insurance KPIs throughout the dashboard.

> **Note:** DAX formula expressions were not provided in the source files. The descriptions above are reconstructed from measure names and contextual evidence. Exact implementations may differ.

---

## 11. Dashboard Design

### 11.1 Design Philosophy
The dashboard follows a **journalistic layout philosophy**: every chart title is a question that the chart answers. Chart subtitles are not axis descriptions — they are plain-language insights drawn from the data. This approach was applied consistently across all five pages and is one of the strongest communication features of the dashboard.

Example from Market Overview:
- **Title:** *"When Were Companies Hiring the Most?"*
- **Subtitle:** *"Hiring peaked in February (55K) and collapsed in November (14K), likely reflecting year-end budget freezes."*

### 11.2 Visual Theme
A consistent warm brown color palette with rounded card containers and white typography is applied across all five pages. The currently active navigation button is highlighted in black/dark; inactive buttons are in brown outline. This creates clear wayfinding without distracting from the data.

### 11.3 Navigation Structure
A persistent five-button navigation bar runs across the top of every page. The active page button is visually distinct. This was built using **Power BI navigation buttons with page navigation actions** — confirming the use of buttons and page navigation features.

### 11.4 Slicer Design Strategy
Rather than using a single fixed slicer set across all pages, each page has its own contextually relevant slicer combination:

| Page | Slicers |
|---|---|
| Market Overview | Country, Employment Type, Job Source |
| Company Insights | Country, Degree, Health Insurance |
| Skills & Roles | Country, Employment Type, Skill Category, Role |
| Salary Insights | Country, Employment Type, Job Source |
| Data Quality | Country, Health Insurance, Degree |

This demonstrates awareness that the right filters depend on the analytical question the page is answering — not a one-size-fits-all approach.

### 11.5 Clear All Filters Button
Every page includes a "Clear all Filters" button that resets all slicers on that page simultaneously. This is a Power BI bookmark-driven feature and improves the user experience by giving users a reliable reset point.

### 11.6 Information Button (Salary Insights)
The Salary Insights page includes a unique ⓘ button with:
- A hover tooltip stating: *"Salary data is available for only ~30,000 of 480,000 postings (6.35%). For More Information CTRL+Click"*
- A CTRL+Click action that navigates to the Data Quality page

This is the only cross-page navigation element in the dashboard and creates an intentional editorial link between the salary analysis and its data quality context — a deliberate design decision.

---

## 12. Page-by-Page Explanation

### 12.1 Page 1 — Market Overview

**Purpose:** Entry point. Establishes scale, temporal patterns, geographic distribution, role demand, and benefit/degree norms at a glance.

**KPIs:** Total Job Postings (479K), Hiring Companies (98K), Unique Job Roles (10)

**Visuals:**
- **Line chart** — monthly job posting trend across 2024; February peak (55K), November trough (14K)
- **Horizontal bar chart** — top 10 countries by job count; US leads at 151K
- **100% stacked bar chart** — On-Site vs Remote split by employment type; shows Contractor and Temp Work as most remote-friendly relative to Full-time
- **Horizontal bar chart** — Most Demanded Roles; Data Engineer leads (129K), followed by Data Analyst (113K) and Data Scientist (98K)
- **Donut charts (×2)** — Degree Required (33%) vs Not Required (67%); Health Insurance Provided (13%) vs Not Provided (87%)

**Slicers:** Country, Employment Type, Job Source

**Key insight surfaced:** 67% of postings do not require a degree, and only 13% offer health insurance — painting a clear picture of the employer-side terms in this market.

---

### 12.2 Page 2 — Company Insights

**Purpose:** Employer-side analysis — who is hiring, on what platforms, with what policies, and from where.

**KPIs:** Total Companies (98K), Companies Offering Insurance (61K), Remote Friendly Companies (19K), Companies Waiving Degree (42K / 43% of all companies)

**Visuals:**
- **Horizontal bar chart** — top 10 hiring companies by posting count; Dice leads (3,635), followed by Listpro, Capital One, OpenClassrooms
- **Horizontal bar chart** — top 10 posting platforms; LinkedIn dominates (183K), 3.3× ahead of BeBee (52K)
- **Ring/donut chart** — degree waiver rate per company for the top 10 most flexible employers; top companies include Tata Consultancy Services (66%), Listpro, Tesla, Robert Half
- **Horizontal bar chart** — top 10 countries by unique company count; US leads (27.8K), India (9.8K), UK (9.2K)

**Slicers:** Country, Degree, Health Insurance

**Key insight surfaced:** 43% of hiring companies waive degree requirements — skills-first hiring is the norm, not the exception, in this dataset.

---

### 12.3 Page 3 — Skills & Roles

**Purpose:** Skills intelligence — what employers want, how demand varies by role, and how role complexity differs.

**KPIs:** Total Skill Mentions (2M), Unique Skills (198), Most Demanded Skill (Python — dynamic DAX card), Average Skill Per Job (4)

**Visuals:**
- **Horizontal bar chart** — top 10 skills by mention count; Python (244K) and SQL (240K) nearly tied at top; AWS (100K), Azure (94K), Tableau (74K) follow
- **Horizontal bar chart** — 7 skill categories by total mentions; Programming Languages leads (341K), then BI & Analytics Tools (214K), Cloud & DevOps (193K)
- **Table visual** — top 4 skills per job title; directly powered by the `Top_4_Skills` pre-aggregated table; shows Business Analyst → SQL, Excel, Tableau, Power BI; Data Engineer → SQL, Python, AWS, Azure; etc.
- **Column chart** — average skills required per role; Senior Data Engineer leads (6.7), Business Analyst lowest (2.5)

**Slicers:** Country, Employment Type, Skill Category, Role

**Key insight surfaced:** Senior technical roles (Senior Data Engineer at 6.7 skills) require nearly 3× the skill breadth of business-facing roles (Business Analyst at 2.5) — role complexity varies dramatically across the market.

---

### 12.4 Page 4 — Salary Insights

**Purpose:** Compensation intelligence, with transparent disclosure of data limitations.

**KPIs:** Jobs With Salary (30K), Salary Coverage Rate (6.35%), Median Hourly Salary ($53), Median Yearly Salary ($110K)

**Information button (ⓘ):** Tooltip reads: *"Salary data is available for only ~30,000 of 480,000 postings (6.35%). For More Information CTRL+Click"* — navigates to Data Quality page on CTRL+Click.

**Visuals:**
- **Horizontal bar chart** — median yearly salary by role; Machine Learning Engineer and Senior Data Scientist lead at $150K; Data Analyst lowest at $84K
- **Horizontal bar chart** — median yearly salary by employment type; Contractor ($115K) > Full-time ($110K); Internship lowest ($50K); annotated note acknowledges that benefits narrow the effective contractor premium
- **Column/histogram chart** — job count by hourly salary bucket; distribution peaks in the $41–60 range; directly powered by `Salary_Hourly_Bucket`
- **Line chart** — quarterly median yearly salary trend; Q1 $103K → Q2 $115K (peak) → Q3 $113K → Q4 $111K; annotated as possibly driven by mid-year tech hiring cycles

**Slicers:** Country, Employment Type, Job Source

**Key insight surfaced:** Only 6.35% of postings disclosed salary — all median figures are based on this subset. Salary figures should be read as indicative of the postings that chose to disclose, not the full market.

---

### 12.5 Page 5 — Data Quality

**Purpose:** Full dataset transparency — communicates coverage, completeness, and the geographic skew of the salary data.

**KPIs:** Total Job Postings (479K), Salary Coverage Rate (6.35%), Without Salary Rate (93.65%), Website Listed (35.55%)

**Visuals:**
- **Line chart** — monthly count of salary-disclosing postings through 2024; declined from 3.8K in February to 1.2K in December; annotated as possibly a data collection artifact rather than a real trend
- **Horizontal bar chart** — salary coverage % by employment type; Contractor (12.70%) and Part-time (10.40%) highest; Full-time lowest at 4.45%
- **Horizontal bar chart** — countries by salary disclosure count, **US excluded** (US accounts for 87.93% of salary data); Canada leads (478), India (466), UK (144)
- **Stacked available/missing bar chart** — coverage breakdown for 4 fields across the full 479K postings; powered by the manually created `Data Quality` table:
  - Website Listed: 35.55% available
  - Degree Info: 32.52% available
  - Health Insurance: 12.83% available
  - Salary: 6.35% available

**Slicers:** Country, Health Insurance, Degree

**Key insight surfaced:** The majority of the dataset's key fields are incompletely populated. Salary (93.65% missing) and health insurance (87.17% missing) have especially low coverage, which is why those metrics should be interpreted with caution throughout the dashboard.

---

## 13. Key Findings

The following findings are drawn from the dashboard visuals and represent what the data communicates at a global, unfiltered level.

### Role Demand
- **Data Engineer** is the most in-demand role with **129K postings**, 14% more than Data Analyst
- **Data Analyst (113K)** and **Data Scientist (98K)** follow as the second and third most posted roles
- The top 3 roles together represent ~70% of all postings across the 10 standardized categories

### Skills Market
- **Python and SQL** are the dominant individual skills, each appearing in over 240K postings and nearly equal in demand
- **AWS (100K) and Azure (94K)** are the leading cloud platforms — cloud skills are now effectively expected in data roles
- **Programming Languages (341K mentions)** is the largest skill category, followed by **BI & Analytics Tools (214K)**
- The average data job posting requires **4 skills**; Senior Data Engineers require 6.7 on average vs. 2.5 for Business Analysts

### Geography & Hiring Timing
- **The United States dominates** with 151K postings — more than India, UK, and France combined
- **Hiring peaked in February** (55K) and hit its lowest point in November (14K), consistent with annual budget cycles
- **LinkedIn accounts for 183K job postings** — by far the most important platform for job seekers to monitor

### Employer Behavior
- **43% of hiring companies waive degree requirements** — skills-first hiring is mainstream in this dataset
- Only **13% of postings mention health insurance** — a notably low benefits disclosure rate
- **19K companies** (out of 98K) offer remote-friendly roles

### Salary
- **Median yearly salary: $110K** across all roles; **median hourly: $53**
- **Machine Learning Engineers and Senior Data Scientists** lead at $150K median yearly salary
- **Contractors earn ~4.5% more** than full-time employees ($115K vs $110K), though benefits reduce the effective gap
- **Salary data is available for only 6.35% of postings** — all compensation figures are based on this subset

### Data Quality
- **Salary:** 6.35% coverage - the most significant data gap in the dataset
- **Health Insurance:** 12.83% coverage
- **Degree Info:** 32.52% coverage
- **Website Listed:** 35.55% coverage

---

## 14. Challenges, Assumptions & Limitations

### 14.1 Multi-Value Cell Normalization & Many-to-Many Resolution

**Challenge 1 — Multi-value employment type cells:**
The `job_schedule_type` column stored multiple employment types in a single row 
(e.g., *"Full-time, Part-time, and Internship"*), making it impossible to filter 
or count correctly by employment type. This is a one-to-many relationship — one 
job posting can have multiple valid employment types.

**Solution:** Created `Employment_Type_Dim` as a reference of `Job_Postings` using 
`ExpandListColumn` after normalizing all delimiters. Every row in this table contains 
exactly one employment type linked to its `Job_id`. Bidirectional cross-filtering was 
applied so that employment type selections propagate correctly back to the fact table 
and all connected dimensions.

**Challenge 2 — Many-to-many skills relationship:**
Each job posting can require multiple skills, and each skill can appear across many 
job postings — a true many-to-many relationship that Power BI cannot handle directly 
with a single relationship.

**Solution:** The `Skills_Job_Dim` bridge table resolves this by sitting between 
`Job_Postings` and `Skills`, with each row representing one job-skill pair. 
Bidirectional filtering on both sides of the bridge ensures that skill-based 
slicer selections correctly propagate to the fact table and all other dimensions.

### 14.2 JSON-Formatted Skills Column
**Challenge:** Skills were stored in a JSON-like format in the source `job_skills` column, making direct use in charts impossible.

**Solution:** Pre-processed `skills_job_dim.csv` into a bridge table format externally (the exact parsing step for the original JSON column is not documented in the available project files). The bridge table was then used in Power Query with an inner join to filter to only valid, cleaned skills.

### 14.3 Sparse Salary Data
**Challenge:** Only ~6.35% of postings disclosed salary information, making it statistically important to communicate this limitation clearly.

**Approach:** Salary coverage rate was surfaced as a KPI card on the Salary Insights page, the ⓘ tooltip explicitly states the limitation, and the entire Data Quality page provides a comprehensive breakdown. Median was used rather than mean to reduce sensitivity to outliers in the limited sample.

**Assumption:** The 30K postings that disclosed salary are assumed to be broadly representative of the categories they belong to — though this cannot be confirmed.

### 14.4 Sudan Country Mislabeling
**Challenge:** "Sudan" appeared anomalously high in country rankings, which was inconsistent with expected market data.

**Resolution:** Cross-referencing with `search_location` confirmed all Sudan-labeled postings had US-based search origins. Corrected to "United States." *(Note: the root cause - likely a data entry or scraping error in the source - is not fully explained.)*

### 14.5 Known Code Issue — Typo in Salary_Rate_Original
The M code for `Job_Postings` contains a typo in the conditional column step: the value `"Hourly Original"` is written as `"Hourly Orignial"` (transposed letters). This typo exists in the actual data values of the `Salary_Rate_Original` column. It does not affect salary calculations (which use `Yearly_Salary` / `Hourly_Salary` directly) but would affect any filter or visual that references the `Salary_Rate_Original` values directly.

### 14.6 Top_5_Skills Naming Inconsistency
The query is named `Top_5_Skills` and the M code extracts 5 skills per job title, but the final pipeline step removes `Top Skill 5`, leaving 4 skill columns. The table name and the actual output are mismatched. This appears to be an intentional late-stage design decision (keeping the table cleaner or reflecting available space in the visual), but the naming inconsistency is worth noting.

### 14.7 Skills_Raw Table Origin
The `Skills` table in Power Query references a source called `Skills_Raw`, but no M code was provided for this source table. Its origin — whether a CSV, a manually entered table, or a hidden query — is uncertain.


---

## 15. Lessons Learned

**Normalize early.** The multi-value cell problem in `job_schedule_type` and the JSON skills column were both architectural issues that needed to be resolved before any analytical work could begin. Building the dimension and bridge tables early made the downstream dashboard work significantly cleaner.

**Let charts surface data quality issues.** The Sudan mislabeling was not caught in a data audit — it was caught by noticing an anomaly in a country chart. Building dashboard visuals iteratively and critically reviewing them is part of the data cleaning process, not just the presentation layer.

**Median matters.** For skewed distributions like salary, using mean would have produced a misleading KPI. Choosing median is a small decision with meaningful analytical impact on every number shown on the Salary Insights page.

**Data transparency builds credibility.** Dedicating an entire page to data quality — and being explicit about the 6.35% salary coverage rate — makes the dashboard more credible, not less. Users who understand what the data cannot tell them trust the findings it can.

**Pre-aggregate when it adds value.** The `Top_4_Skills` table required complex M logic (nested tables, sorting, extracting top-N) that would be cumbersome to replicate in DAX. Computing it in Power Query and storing the result as a static table was the right engineering decision.

**Organize DAX from the start.** Separating measures into `Skills Measures` and `Measures` tables is a small structural decision that pays dividends when the field list gets large. It is much easier to maintain 24 measures when they are logically grouped.
---
## 16. Appendix

### A. Original Dataset Column Reference

| # | Original Column Name | Type | Kept? | Final Name |
|---|---|---|---|---|
| 1 | `job_id` | Integer | ✅ | `Job_id` |
| 2 | `company_id` | Integer | ✅ | `Company_id` |
| 3 | `job_title_short` | Text | ✅ | `Job_Title` |
| 4 | `job_title` | Text | ❌ Removed | — |
| 5 | `job_location` | Text | ✅ | `Job_Location` |
| 6 | `job_via` | Text | ✅ | `Job_Via` |
| 7 | `job_schedule_type` | Text | ✅ (also in dim) | `job_schedule_type` |
| 8 | `job_work_from_home` | Logical | ✅ | `Is_remote` |
| 9 | `search_location` | Text | ❌ Removed | — |
| 10 | `job_posted_date` | DateTime | ✅ Split | `Posted_Date` + `Posted_Time` |
| 11 | `job_no_degree_mention` | Logical | ✅ | `No_Degree_Mentioned` |
| 12 | `job_health_insurance` | Logical | ✅ | `Has_Health_Insurance` |
| 13 | `job_country` | Text | ✅ | `Country` |
| 14 | `salary_rate` | Text | ❌ Replaced | `Salary_Rate_Original` |
| 15 | `salary_year_avg` | Integer | ❌ Replaced | `Yearly_Salary` |
| 16 | `salary_hour_avg` | Number | ❌ Replaced | `Hourly_Salary` |

### B. Final Table Column Reference

**Job_Postings (Fact Table — 19 columns):**
`Job_id`, `Company_id`, `Job_Title`, `Job_Location`, `Job_Via`, `job_schedule_type`, `Is_remote`, `Posted_Date`, `Posted_Time`, `No_Degree_Mentioned`, `Has_Health_Insurance`, `Country`, `Yearly_Salary`, `Hourly_Salary`, `Salary_Hourly_Bucket`, `Salary_Rate_Original`, `Salary_Mentioned`, `Degree`, `Health_Insurance`

**Company_Dim:** `company_id`, `Company_Name`, `Company_Website`

**Employment_Type_Dim:** `Job_id`, `Employment_Type`

**Skills_Job_Dim:** `Job_id`, `Skill_id`

**Skills:** `skill_id`, `Skills`, `Category`

**Top_4_Skills:** `Job_Title`, `Top Skill 1`, `Top Skill 2`, `Top Skill 3`, `Top Skill 4`

**Date Table:** `Date`, `Day Name`, `Day Number`, `Month Name`, `Month Number`, `Quarter`, `Week Number`, `Year`

**Data Quality:** `field`, `status`, `percentage`

### C. Skill Category Reference

| Category | Example Skills |
|---|---|
| Programming Languages | Python, SQL, R, Java, JavaScript, Scala, Go, VBA |
| BI & Analytics Tools | Power BI, Tableau, Excel, Looker, SAS, Databricks, Jupyter |
| Cloud & DevOps | AWS, Azure, GCP, Docker, Kubernetes, Terraform, Git, Bash |
| Data Science Libraries | PyTorch, TensorFlow, Pandas, NumPy, scikit-learn, Spark |
| Databases & Warehouses | PostgreSQL, MySQL, Snowflake, BigQuery, MongoDB, Redshift |
| Web Frameworks | React, Django, Flask, Node.js, FastAPI, Angular |
| Operating Systems | Linux, Ubuntu, Windows, macOS, Red Hat |


## Conclusion & Scalability
This dashboard was architected for modularity, allowing for future integration of real-time job market APIs and advanced predictive analytics. Future iterations of this model could include a "Skill Gap Calculator" using DAX-driven comparison measures to help job seekers map their existing competencies against high-demand roles.