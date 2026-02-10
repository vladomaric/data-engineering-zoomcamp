# Module 3 Homework: Data Warehousing with BigQuery
This project focuses on data warehousing using Google BigQuery. The objective is to ingest 2024 Yellow Taxi Trip data, optimize storage through partitioning and clustering, and analyze performance differences between external and native tables.

---
## Direct Data Streaming to GCS
To keep the pipeline lean, the standard ingestion process was replaced with a Bash-based streaming pipe. This approach is significantly faster as it avoids intermediate storage steps. By piping curl directly into gsutil, the data moves through the network interface and straight into the GCS bucket.  

```bash
# A loop streams the first six months of 2024 directly to the bucket
for month in {01..06}; do
  curl -L "https://d37ci6vzurychx.cloudfront.net/trip-data/yellow_tripdata_2024-${month}.parquet" | \
  gsutil cp - gs://de-zoomcamp-2026-vlado-taxi-data/yellow_tripdata_2024/yellow_tripdata_2024-${month}.parquet
done
```

This method provided the fastest path to staging the 20,332,093 records, ensuring the raw data was ready for BigQuery without the latency of intermediate file handling.

---
## BigQuery Architecture Implementation
After successfully streaming the raw files into GCS, the focus shifted to building a high-performance analytical environment. This was achieved through a three-step progression: linking the raw data, materializing it for speed, and finally optimizing the physical storage.

### Step 1: Establishing the External Table
The process began by creating an External Table. This serves as a metadata layer that allows BigQuery to read the Parquet files directly from GCS. It is a cost-effective starting point because it requires zero storage costs within BigQuery while providing immediate access to the 2024 dataset.   

```sql
-- Linking the GCS Parquet files to BigQuery
CREATE OR REPLACE EXTERNAL TABLE `de-zoomcamp-2026-vlado.zoomcamp.yellow_tripdata_2024_ext`
OPTIONS (
  format = 'PARQUET',
  uris = ['gs://de-zoomcamp-2026-vlado-taxi-data/yellow_tripdata_2024/*.parquet']
);
```

### Step 2: Materialization to Native Storage
After that, the data was moved into a Native (Materialized) Table. Unlike external tables, native storage allows BigQuery to utilize its proprietary columnar format and manage metadata internally, which is essential for performance-heavy analytical queries.

```sql
-- Moving data from GCS into native BigQuery storage
CREATE OR REPLACE TABLE `de-zoomcamp-2026-vlado.zoomcamp.yellow_tripdata_2024` AS
SELECT * FROM `de-zoomcamp-2026-vlado.zoomcamp.yellow_tripdata_2024_ext`;
```

### Step 3: Performance Optimization via Partitioning and Clustering
The final and most critical step involved creating an Optimized Table. By partitioning the data by day and clustering it by vendor, the physical storage was reorganized to align with common query patterns. This minimizes the amount of data the execution engine must scan.
```sql
-- Creating the production-ready table with partitioning and clustering
CREATE OR REPLACE TABLE `de-zoomcamp-2026-vlado.zoomcamp.yellow_tripdata_2024_partitioned_clustered`
PARTITION BY DATE(tpep_dropoff_datetime)
CLUSTER BY VendorID AS
SELECT * FROM `de-zoomcamp-2026-vlado.zoomcamp.yellow_tripdata_2024_ext`;
```
---
## Homework Answers
### Q1: What is the count of records for the 2024 Yellow Taxi Data?
**Answer: 20,332,093  **
The total row count is determined by querying the external table which points to the Parquet files in your GCS bucket. This count serves as the baseline for verifying that the subsequent materialization into native storage is complete.
```sql
SELECT COUNT(*) 
FROM `de-zoomcamp-2026-vlado.zoomcamp.yellow_tripdata_2024_ext`
```
### Q2: What is the estimated amount of data that will be read when you query the distinct number of VendorIDs?
**Answer: 0 MB for the External Table and 155.12 MB for the Materialized Table.**  
When you perform a dry run on these two table types, BigQuery provides different estimates. For the external table, the estimate is 0 MB because BigQuery does not manage the underlying storage and cannot "peek" into the Parquet files in GCS to calculate column size before execution. For the native materialized table, BigQuery uses its internal metadata to provide an accurate estimate of 155.12 MB.
```sql
-- Dry run this to see 0 B estimate
SELECT COUNT(DISTINCT(VendorID)) 
FROM `de-zoomcamp-2026-vlado.zoomcamp.yellow_tripdata_2024_ext`;

-- Dry run this to see 155.12 MB estimate
SELECT COUNT(DISTINCT(VendorID)) 
FROM `de-zoomcamp-2026-vlado.zoomcamp.yellow_tripdata_2024`;
```
### Q3: Why are the estimated bytes different?
**Answer: BigQuery is a columnar store and native tables contain pre-calculated metadata.** 
Managed (native) tables are optimized into BigQuery’s internal format. This allows the engine to know exactly how much data is in each column without scanning it. External tables are just links to files; BigQuery doesn't know the size of a specific column until it actually begins reading the file at runtime.
### Q4: How many records have a fare_amount of 0?
**Answer: 8,333** 
```sql
SELECT COUNT(*) 
FROM `de-zoomcamp-2026-vlado.zoomcamp.yellow_tripdata_2024`
WHERE fare_amount = 0;
```
### Q5: What is the best strategy to make an optimized table?
**Answer: Partition by tpep_dropoff_datetime and Cluster on VendorID.** 
Partitioning splits the table into smaller segments by date, while clustering sorts the data within those partitions by the Vendor ID. This combination is the industry standard for taxi data which is almost always queried by time and service provider.
### Q6: What is the estimated bytes for distinct VendorIDs between 2024-03-01 and 2024-03-15?
**Answer: 310.24 MB for the non-partitioned table and 26.84 MB for the partitioned table.**   
A dry run on both tables demonstrates the efficiency of the "Divide and Conquer" approach. The partitioned table allows BigQuery to ignore all data outside of the 15-day window in March. Actionable Insight: This results in a 91% reduction in data processed, which directly lowers query costs and improves execution speed in your GCP environment.

```sql
-- Query run against the optimized partitioned table
SELECT DISTINCT(VendorID)
FROM `de-zoomcamp-2026-vlado.zoomcamp.yellow_tripdata_2024_partitioned_clustered`
WHERE tpep_dropoff_datetime BETWEEN '2024-03-01' AND '2024-03-15'
```
### Q7: Where is the data stored for the external table?
**Answer: GCP Bucket (GCS).**  
The external table is merely a metadata layer. The physical bytes live in Parquet files located at gs://de-zoomcamp-2026-vlado-taxi-data/.
### Q8: Is it always best to partition?
**Answer: False.**  
Partitioning adds metadata overhead. If the dataset is small (typically less than 1 GB), the cost of managing partitions can exceed the savings from data pruning. For very small tables, a standard native table is often faster and cheaper.
### Q9: What is the estimated bytes for a "SELECT " query?
**Answer: 0 B** 
While SELECT * will ultimately scan the entire table upon execution (approx. 2.62 GB), the dry run estimate in the BigQuery console often shows 0 B for this specific operation because it is treated as a metadata-only calculation in the query validator before the full column scan is initiated.
