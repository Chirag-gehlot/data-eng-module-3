# 🏢 Data Engineering Zoomcamp — Module 3: Data Warehouse & BigQuery

> My hands-on work for **Module 3** of the [DataTalksClub Data Engineering Zoomcamp](https://github.com/DataTalksClub/data-engineering-zoomcamp) — covering data warehousing concepts, BigQuery internals, performance optimisation with partitioning and clustering, and machine learning directly inside BigQuery using **BigQuery ML**.

---

## 📖 Module Overview

Module 3 dives deep into **data warehousing** using **Google BigQuery**. The module covers how BigQuery stores and processes data, how to optimise queries through partitioning and clustering, and how to build and deploy machine learning models without leaving SQL using BigQuery ML. NYC Taxi data (Yellow and FHV) is used as the working dataset throughout.

---

## 🗂️ What's Covered

### 1. External Tables (`bigquery.sql`)

**External tables** let BigQuery query files directly from GCS without copying data into BigQuery storage:

```sql
CREATE OR REPLACE EXTERNAL TABLE `taxi-rides-ny.nytaxi.external_yellow_tripdata`
OPTIONS (
    format = 'CSV',
    uris = [
        'gs://nyc-tl-data/trip data/yellow_tripdata_2019-*.csv',
        'gs://nyc-tl-data/trip data/yellow_tripdata_2020-*.csv'
    ]
);
```

- Wildcards (`*`) allow multiple files to be treated as a single table
- Querying a public table (`bigquery-public-data.new_york_citibike.citibike_stations`) to understand the format

---

### 2. Partitioning (`bigquery.sql`)

**Partitioning** divides a table into segments by a column, allowing BigQuery to skip irrelevant partitions and dramatically reduce bytes scanned:

| Table | Bytes Scanned (June 2019 query) |
|-------|-------------------------------|
| Non-partitioned | **1.6 GB** |
| Partitioned by `tpep_pickup_datetime` | **~106 MB** |

```sql
-- Non-partitioned
CREATE OR REPLACE TABLE taxi-rides-ny.nytaxi.yellow_tripdata_non_partitioned AS
SELECT * FROM taxi-rides-ny.nytaxi.external_yellow_tripdata;

-- Partitioned by pickup date
CREATE OR REPLACE TABLE taxi-rides-ny.nytaxi.yellow_tripdata_partitioned
PARTITION BY DATE(tpep_pickup_datetime) AS
SELECT * FROM taxi-rides-ny.nytaxi.external_yellow_tripdata;
```

Inspecting partitions via `INFORMATION_SCHEMA`:

```sql
SELECT table_name, partition_id, total_rows
FROM `nytaxi.INFORMATION_SCHEMA.PARTITIONS`
WHERE table_name = 'yellow_tripdata_partitioned'
ORDER BY total_rows DESC;
```

---

### 3. Clustering (`bigquery.sql`)

**Clustering** sorts data within each partition by one or more columns, further reducing bytes scanned when filtering on those columns:

| Table | Bytes Scanned (Jun 2019 – Dec 2020, VendorID=1) |
|-------|-------------------------------|
| Partitioned only | **1.1 GB** |
| Partitioned + Clustered by `VendorID` | **864.5 MB** |

```sql
CREATE OR REPLACE TABLE taxi-rides-ny.nytaxi.yellow_tripdata_partitioned_clustered
PARTITION BY DATE(tpep_pickup_datetime)
CLUSTER BY VendorID AS
SELECT * FROM taxi-rides-ny.nytaxi.external_yellow_tripdata;
```

---

### 4. Homework — FHV Tripdata (`bigquery_hw.sql`)

Applies the same external table, partitioning, and clustering concepts to the **For-Hire Vehicle (FHV)** 2019 dataset:

- Creates an **external table** from FHV CSV files in GCS
- Counts total records and distinct `dispatching_base_num` values
- Creates a **non-partitioned** and a **partitioned + clustered** version:
  - Partitioned by `dropoff_datetime`
  - Clustered by `dispatching_base_num`
- Benchmarks query performance between the two table types — filtering by a Q1 2019 date range and specific base numbers (`B00987`, `B02279`, `B02060`)

---

### 5. BigQuery ML (`bigquery_ml.sql`)

Builds a **Linear Regression model** to predict taxi tip amounts — entirely in SQL using BigQuery ML:

#### Feature Preparation
- Selects relevant columns: `passenger_count`, `trip_distance`, `PULocationID`, `DOLocationID`, `payment_type`, `fare_amount`, `tolls_amount`, `tip_amount`
- Casts location IDs and payment type to `STRING` (BigQuery ML treats these as categorical features)
- Filters out rows where `fare_amount = 0`

#### Model Training
```sql
CREATE OR REPLACE MODEL `taxi-rides-ny.nytaxi.tip_model`
OPTIONS (
    model_type='linear_reg',
    input_label_cols=['tip_amount'],
    DATA_SPLIT_METHOD='AUTO_SPLIT'
) AS
SELECT * FROM `taxi-rides-ny.nytaxi.yellow_tripdata_ml`
WHERE tip_amount IS NOT NULL;
```

#### Model Operations

| Step | Function | Purpose |
|------|----------|---------|
| Inspect features | `ML.FEATURE_INFO` | View feature statistics and types |
| Evaluate | `ML.EVALUATE` | Get RMSE, MAE, R² and other metrics |
| Predict | `ML.PREDICT` | Generate tip amount predictions |
| Explain predictions | `ML.EXPLAIN_PREDICT` | Show top 3 most influential features per prediction |

#### Hyperparameter Tuning
Runs automated hyperparameter optimisation with 5 trials (2 in parallel), tuning:
- `l1_reg` in range `[0, 20]`
- `l2_reg` from candidates `[0, 0.1, 1, 10]`

```sql
CREATE OR REPLACE MODEL `taxi-rides-ny.nytaxi.tip_hyperparam_model`
OPTIONS (
    model_type='linear_reg',
    input_label_cols=['tip_amount'],
    DATA_SPLIT_METHOD='AUTO_SPLIT',
    num_trials=5,
    max_parallel_trials=2,
    l1_reg=hparam_range(0, 20),
    l2_reg=hparam_candidates([0, 0.1, 1, 10])
) AS SELECT * FROM `taxi-rides-ny.nytaxi.yellow_tripdata_ml`
WHERE tip_amount IS NOT NULL;
```

---

## 🗂️ Repository Structure

```
.
└── bigquery/
    ├── bigquery.sql        # External tables, partitioning, clustering + benchmarks
    ├── bigquery_hw.sql     # Homework: FHV data — external table, partition, cluster
    └── bigquery_ml.sql     # BigQuery ML: feature prep, model training, eval, predict, hypertuning
```

---

## 🛠️ Tech Stack

| Tool | Purpose |
|------|---------|
| Google BigQuery | Cloud data warehouse |
| Google Cloud Storage | Raw data lake (CSV source files) |
| BigQuery ML | In-warehouse machine learning |
| SQL | All queries, table definitions, and ML operations |

---

## 📚 Resources

- [DataTalksClub DE Zoomcamp — Module 3](https://github.com/DataTalksClub/data-engineering-zoomcamp/tree/main/03-data-warehouse)
- [BigQuery Documentation](https://cloud.google.com/bigquery/docs)
- [BigQuery ML Documentation](https://cloud.google.com/bigquery/docs/bqml-introduction)
- [NYC TLC Trip Data](https://github.com/DataTalksClub/nyc-tlc-data)
- [Course YouTube Playlist](https://www.youtube.com/playlist?list=PL3MmuxUbc_hJed7dXYoJw8DoCuVHhGEQb)

---

## 🙌 Acknowledgements

Thanks to [Alexey Grigorev](https://linkedin.com/in/agrigorev) and the DataTalksClub team for this excellent free course.
