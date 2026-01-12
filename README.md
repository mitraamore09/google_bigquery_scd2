# Slowly Changing Dimension Type 2 in BigQuery (Google Cloud Platform)

[![BigQuery](https://img.shields.io/badge/BigQuery-4285F4?style=flat&logo=google-cloud&logoColor=white)](https://cloud.google.com/bigquery)
[![SQL](https://img.shields.io/badge/SQL-CC2927?style=flat&logo=microsoft-sql-server&logoColor=white)](https://en.wikipedia.org/wiki/SQL)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

> A production-ready implementation of Slowly Changing Dimension Type 2 (SCD2) in BigQuery, demonstrating how to track complete dimensional history using pure SQL.

## 📋 Table of Contents

- [Overview](#overview)
- [What Problem Does This Solve?](#what-problem-does-this-solve)
- [The Data](#the-data)
- [Key Concepts](#key-concepts)
- [Technical Implementation](#technical-implementation)
- [Project Structure](#project-structure)
- [Getting Started](#getting-started)
- [Challenges & Solutions](#challenges--solutions)
- [Validation Queries](#validation-queries)
- [Performance Considerations](#performance-considerations)
- [What You'll Learn](#what-youll-learn)
- [Contributing](#contributing)
- [License](#license)

## 🎯 Overview

Ever wondered how data warehouses track the complete history of changing data? When a product's quantity changes or a customer updates their address, you don't want to lose the old values—you need a time machine for your data.

This project implements **SCD Type 2**, the gold standard for dimensional history tracking. By the end, you'll have a dimension table that preserves every version that ever existed, complete with:
- Start and end dates for each version
- Active/inactive flags
- Complete audit trail
- Point-in-time query capabilities

## 🔍 What Problem Does This Solve?

### The Challenge

You receive product updates regularly. Some products are new, others have changed, and some haven't changed at all. Your system needs to:

1. **Identify new products** → Insert as active versions
2. **Detect changes** → Expire old versions, insert new ones
3. **Preserve history** → Never lose past states
4. **Enable time travel** → Query data as it existed on any date

### Real-World Example

**Toothpaste (Product_Id = 3) timeline:**

```
Dec 20, 2025: Quantity = 16 (initial state)
Dec 25, 2025: Quantity = 18 (update received)
```

**Without SCD2:** You'd overwrite 16 with 18 → History lost  
**With SCD2:** You keep both versions with date ranges → History preserved

```sql
-- Version 1 (expired)
Product_Id: 3, Quantity: 16, Modified_At: 2025-12-25, Is_Active: No

-- Version 2 (current)
Product_Id: 3, Quantity: 18, Modified_At: 9999-12-31, Is_Active: Yes
```

Now you can accurately answer: *"What was Toothpaste's quantity on December 22?"* → **16**

## 📊 The Data

This project tracks a retail product catalog with 7 products:

| Product_Id | Product_Name | Product_Quantity |
|------------|--------------|------------------|
| 1 | Detergent | 20 |
| 2 | Soap | 13 |
| 3 | Toothpaste | 16 |
| 4 | Coffee | 8 |
| 5 | Cable | 15 |
| 6 | Water | 40 |
| 7 | Toothpaste | 10 |

As these products receive updates (quantity changes, name corrections, restocks), the dimension table preserves complete history while maintaining a clean "current state" view.

## 💡 Key Concepts

### Slowly Changing Dimension (SCD)
A dimension table that tracks how attributes change over time. Several types exist (0, 1, 2, 3, 6), each handling changes differently.

### SCD Type 2 (This Project)
Every change creates a new row with its own date range. Previous versions are "expired" by setting end dates. **Most comprehensive** but increases table size.

### Key Fields

| Field | Purpose | Example |
|-------|---------|---------|
| `Product_Id` | Business key (can repeat) | 3 |
| `Product_Name` | Product attribute | "Toothpaste" |
| `Product_Quantity` | Tracked metric | 16 → 18 |
| `Modified_At` | Version end date | 9999-12-31 = current |
| `Is_Active` | Current version flag | "Yes" or "No" |

### The "9999-12-31" Convention
We use a far-future date (December 31, 9999) to mark active records. This is cleaner than NULL values and makes date range queries straightforward:

```sql
-- Point-in-time query pattern
WHERE created_at <= '2025-06-15' AND modified_at > '2025-06-15'
```

## 🔧 Technical Implementation

### Schema Design

**Raw Dataset (source):**
```sql
CREATE TABLE scd_demo.raw_product (
  Product_Id INT64,         -- Business key: 1-7
  Product_Name STRING,      -- Detergent, Soap, etc.
  Product_Quantity INT64    -- Current count
);
```

**Dimension Table (target):**
```sql
CREATE TABLE scd_demo.dim_product (
  Product_Id INT64,         -- Business key (repeats for versions)
  Product_Name STRING,
  Product_Quantity INT64,
  Modified_At DATE,         -- End date (9999-12-31 = current)
  Is_Active STRING          -- "Yes" or "No"
);
```

### The Core Algorithm

**Approach: Incremental Batch Processing**

Process one load date at a time, in chronological order:

```sql
FOR load_batch IN (SELECT DISTINCT load_date FROM staging ORDER BY load_date)
DO
  -- Step 1: Isolate today's changes
  CREATE TEMP TABLE stg_run AS
  SELECT * FROM staging WHERE load_date = load_batch.load_date;
  
  -- Step 2: Expire changed active rows
  UPDATE dim_product
  SET Modified_At = load_batch.load_date, 
      Is_Active = 'No'
  WHERE Is_Active = 'Yes'
    AND EXISTS (
      SELECT 1 FROM stg_run s
      WHERE s.Product_Id = dim_product.Product_Id
        AND (s.Product_Name != dim_product.Product_Name
         OR s.Product_Quantity != dim_product.Product_Quantity)
    );
  
  -- Step 3: Insert new active versions
  INSERT INTO dim_product
  SELECT 
    Product_Id,
    Product_Name,
    Product_Quantity,
    DATE('9999-12-31') AS Modified_At,
    'Yes' AS Is_Active
  FROM stg_run
  WHERE NOT EXISTS (
    SELECT 1 FROM dim_product d
    WHERE d.Product_Id = stg_run.Product_Id
      AND d.Is_Active = 'Yes'
      AND d.Product_Name = stg_run.Product_Name
      AND d.Product_Quantity = stg_run.Product_Quantity
  );
  
END FOR;
```

### Why Use a Loop?

**Temporal consistency.** SCD Type 2 requires processing changes in the order they occurred. Loading multiple dates simultaneously risks:
- Overlapping date ranges
- Incorrect active flags
- Lost or duplicated versions

The loop guarantees sequential processing, building accurate history step by step.

## 📁 Project Structure

```
slowly-changing-dimension/
│
├── data/
│   ├── raw_dataset.csv              # Initial 7 products
│   └── sample_changes.csv           # Simulated updates over time
│
├── setup/
│   ├── 01_create_dataset.sql        # BigQuery dataset creation
│   ├── 02_create_tables.sql         # DDL for raw and dim tables
│   └── 03_load_initial_data.sql     # Load your 7 products
│
├── scd2_implementation/
│   ├── approach_1_two_step.sql      # Separate UPDATE + INSERT
│   ├── approach_2_single_merge.sql  # UNION ALL technique
│   └── approach_3_replay_loop.sql   # ✅ Production solution
│
├── validation/
│   ├── timeline_queries.sql         # Version history per product
│   ├── active_snapshot.sql          # Current state only
│   ├── change_frequency.sql         # Audit: most volatile products
│   └── point_in_time.sql            # Historical state queries
│
├── docs/
│   ├── CHALLENGES.md                # Detailed problem-solving log
│   └── CONCEPTS.md                  # Deep dive into SCD theory
│
├── .gitignore
├── LICENSE
└── README.md
```

## 🚀 Getting Started

### Prerequisites

- Google Cloud Platform account
- BigQuery access (free tier/sandbox works)
- Basic SQL knowledge

### Quick Start (5 minutes)

#### 1️⃣ Create Dataset
```sql
CREATE SCHEMA `your-project.scd_demo`;
```

#### 2️⃣ Create Tables
```sql
-- Raw product table
CREATE TABLE scd_demo.raw_product (
  Product_Id INT64,
  Product_Name STRING,
  Product_Quantity INT64
);

-- Dimension table with SCD2 tracking
CREATE TABLE scd_demo.dim_product (
  Product_Id INT64,
  Product_Name STRING,
  Product_Quantity INT64,
  Modified_At DATE,
  Is_Active STRING
);
```

#### 3️⃣ Load Initial Data
```sql
INSERT INTO scd_demo.raw_product VALUES
  (1, 'Detergent', 20),
  (2, 'Soap', 13),
  (3, 'Toothpaste', 16),
  (4, 'Coffee', 8),
  (5, 'Cable', 15),
  (6, 'Water', 40),
  (7, 'Toothpaste', 10);
```

#### 4️⃣ Create Staging with Load Dates
```sql
CREATE TABLE scd_demo.staging_product (
  load_date DATE,
  Product_Id INT64,
  Product_Name STRING,
  Product_Quantity INT64
);

-- Simulate initial load
INSERT INTO scd_demo.staging_product
SELECT CURRENT_DATE() AS load_date, * FROM scd_demo.raw_product;
```

#### 5️⃣ Run SCD2 Pipeline
Execute `scd2_implementation/approach_3_replay_loop.sql`

#### 6️⃣ Simulate a Change
```sql
-- Day 2: Toothpaste quantity changes
INSERT INTO scd_demo.staging_product VALUES
  (DATE_ADD(CURRENT_DATE(), INTERVAL 1 DAY), 3, 'Toothpaste', 18);

-- Re-run the replay loop script
```

#### 7️⃣ Verify Results
```sql
SELECT 
  Product_Id,
  Product_Name,
  Product_Quantity,
  Modified_At,
  Is_Active
FROM scd_demo.dim_product
WHERE Product_Id = 3
ORDER BY Modified_At;
```

**Expected Output:**
```
Product_Id | Product_Name | Quantity | Modified_At | Is_Active
-----------|--------------|----------|-------------|----------
3          | Toothpaste   | 16       | 2025-12-26  | No
3          | Toothpaste   | 18       | 9999-12-31  | Yes
```

## 🐛 Challenges & Solutions

### Challenge 1: "No history being created"
**Symptom:** Dimension shows only latest snapshot, no versions  
**Root Cause:** Loading only current state doesn't create changes to track  
**Solution:** Process by `load_date` incrementally or replay in chronological order

### Challenge 2: "Incorrect active flags and dates"
**Symptom:** Old rows stay active, new rows inactive, dates overlap  
**Root Cause:** Processing multiple dates simultaneously without proper sequencing  
**Solution:** Reset dimension, replay all load_dates sequentially with loop

### Challenge 3: "MERGE matched row multiple times"
**Symptom:** BigQuery MERGE fails with duplicate match error  
**Root Cause:** Staging data not deduplicated or multiple active rows per product  
**Solution:** Dedupe `stg_run` per product per batch, ensure one active row per key


## ✅ Validation Queries

### View Complete History
```sql
-- Timeline for a specific product
SELECT 
  Product_Id,
  Product_Name,
  Product_Quantity,
  Modified_At,
  Is_Active
FROM scd_demo.dim_product
WHERE Product_Id = 3  -- Toothpaste
ORDER BY Modified_At;
```

### Current Active Snapshot
```sql
-- What's the current state of all products?
SELECT 
  Product_Id,
  Product_Name,
  Product_Quantity
FROM scd_demo.dim_product
WHERE Is_Active = 'Yes'
ORDER BY Product_Id;
```

### Point-in-Time Query
```sql
-- What did products look like on December 22, 2025?
SELECT 
  Product_Id,
  Product_Name,
  Product_Quantity
FROM scd_demo.dim_product
WHERE created_at <= '2025-12-22'
  AND modified_at > '2025-12-22';
```

### Change Frequency Audit
```sql
-- Which products change most often?
SELECT 
  Product_Id,
  Product_Name,
  COUNT(*) AS version_count,
  MIN(Modified_At) AS first_seen,
  MAX(CASE WHEN Is_Active = 'Yes' THEN Modified_At END) AS last_updated
FROM scd_demo.dim_product
GROUP BY Product_Id, Product_Name
HAVING COUNT(*) > 1
ORDER BY version_count DESC;
```

### Data Quality Check
```sql
-- Ensure only one active row per product
SELECT 
  Product_Id,
  COUNT(*) AS active_count
FROM scd_demo.dim_product
WHERE Is_Active = 'Yes'
GROUP BY Product_Id
HAVING COUNT(*) > 1;

-- Should return 0 rows
```

## 📊 Data Visualization & Analysis

One of the most powerful aspects of SCD Type 2 is the ability to visualize dimensional changes over time. BigQuery's built-in charting makes this incredibly accessible.

### Visualization 1: Which Products Changed the Most?

**Query:**
```sql
SELECT
  product_id,
  ANY_VALUE(product_name) AS product_name,
  COUNT(*) AS versions
FROM `slowly-chanding-dimension.scd_demo.dim_product`
GROUP BY product_id
ORDER BY versions DESC;
```

**Chart Output:**
![Multi-Product Timeline](docs/images/product.png)

**Insight:** This line chart reveals version count by product. In the example:
- **Conditioner** had the most changes (6 versions) - might indicate frequent restocking or pricing adjustments
- **Shampoo** dropped from 3 versions initially to 1, then spiked to 6 versions
- **Soap** and **Facewash** show relatively stable patterns with 1-2 versions
- **Lotion** peaked at 5 versions

**Business Value:** Identifies volatile products that may need:
- More stable inventory management
- Better forecasting models
- Data quality investigation (excessive changes might indicate upstream issues)

---

### Visualization 2: Current Active Quantity Per Product

**Query:**
```sql
SELECT
  product_name,
  product_quantity
FROM `slowly-chanding-dimension.scd_demo.dim_product`
WHERE is_active = TRUE
ORDER BY product_name;
```

**Chart Output:**

![Current Active Quantity](docs/images/chart_current_quantity.png)

**Insight:** This bar chart shows the current inventory snapshot:
- **Lotion** has the highest stock (≈26 units)
- **Soap** has strong inventory (≈22 units)
- **Shampoo** and **Conditioner** hover around 15-18 units
- **Facewash** and **Toothpaste** are lowest (~10 units each)

**Business Value:**
- Quick inventory health check
- Identifies reorder priorities (Facewash and Toothpaste need attention)
- Enables comparison with historical patterns

---

### Visualization 3: Multi-Product Quantity Timeline

**Query:**
```sql
SELECT
  DATETIME(created_at) AS created_at_dt,
  product_name,
  product_quantity
FROM `slowly-chanding-dimension.scd_demo.dim_product`
WHERE product_id IN (1,2,3,4)
ORDER BY created_at_dt;
```

**Chart Output:**

![Multi-Product Timeline](docs/images/chart_timeline.png)

**Insight:** This line chart compares quantity trends for 4 products over time:
- Shows one product starting at ~20 units (early December), dropping to ~10, then recovering to ~22 units (mid-December)
- Temporal patterns reveal seasonal trends, restocking cycles, or sales velocity
- The smooth line indicates proper SCD2 implementation—each point represents a version change

**Business Value:**
- Spot correlations (do certain products always change together?)
- Identify restocking patterns (weekly vs. monthly)
- Detect anomalies (unexpected spikes or drops)
- Support demand forecasting with historical trends

---

### Why Visualization Matters for SCD2

**"Seeing is believing"** - especially with large, constantly changing datasets:

1. **Pattern Recognition:** Humans spot visual patterns faster than reading query results
2. **Anomaly Detection:** Outliers jump out in charts but hide in tables
3. **Stakeholder Communication:** Non-technical users grasp charts instantly
4. **Data Quality Validation:** Weird chart shapes = potential data issues

### BigQuery Visualization Features Used

- **Line Charts:** Perfect for time series (quantity changes over time)
- **Bar Charts:** Great for comparisons (current quantities across products)
- **Interactive Tooltips:** Hover for exact values and dates
- **Color Coding:** Distinguish multiple products on the same timeline
- **Automatic Scaling:** BigQuery handles axes and legends


## ⚡ Performance Considerations

### For Production Scale

**Indexing:**
```sql
-- Create clustered table for better performance
CREATE TABLE scd_demo.dim_product_clustered
CLUSTER BY Product_Id, Is_Active
AS SELECT * FROM scd_demo.dim_product;
```

**Partitioning:**
```sql
-- Partition by Modified_At for efficient historical queries
CREATE TABLE scd_demo.dim_product_partitioned
PARTITION BY Modified_At
AS SELECT * FROM scd_demo.dim_product;
```

**Batch Processing:**
- Process staging in daily batches (not real-time)
- Dedupe staging before processing
- Monitor for products with excessive versions (data quality issue)




⭐ Star this repo if you found it helpful!

</div>
