# Module 2 Homework: Workflow Orchestration with Kestra
This folder contains the solution for Module 2 of the Data Engineering Zoomcamp. This module focuses on using Kestra to orchestrate data pipelines that ingest NYC Taxi data into Google Cloud Storage (GCS) and BigQuery.

---
### Project Overview
The pipeline handles data extraction from GitHub, uncompresses the files, uploads them to GCS, and finally creates/merges the data into partitioned BigQuery tables.

Namespace Architecture: Because I am using the Free tier of Kestra, the "Secrets" management feature is unavailable. To solve this, I adopted a modular approach by storing all my sensitive GCP credentials and environment variables in a dedicated KV Store within a separate namespace called vlado.base. This allowed me to:

Keep my credentials isolated from the flow logic.

Reference keys like GCP_PROJECT_ID and GCP_SERVICE_ACCOUNT across multiple different flows using the kv() function.

Maintain a clean, production-like separation of concerns despite the tier limitations.

---
## Question 1: File size of the yellow taxi data
To determine the uncompressed size of the December 2020 Yellow Taxi dataset, I executed the workflow in Kestra with the inputs set to ```yellow, 2020, and 12```.  

Once the run finished, I inspected the Metrics tab for the upload_to_gcs task to find the exact byte count before the data hit my Google Cloud Storage bucket.

Calculation:  
Raw Metric: 134,481,400 bytes  
Conversion: $134,481,400 \div (1024 \times 1024) = 128.24 \text{ MiB}$  
Result: 128.3 MiB  

---
## Question 2: Variable Interpolation
I analyzed how Kestra handles string templating by testing the ```file``` variable definition. When the flow receives ```green, 2020, and 04``` as inputs, the engine renders the filename by injecting those values into the placeholders.

Result: ```green_tripdata_2020-04.csv```

---
## Question 3: Yellow Taxi Row Count (2020)  
To determine the total volume of Yellow Taxi records for the entire year of 2020, I utilized Kestra's backfill feature to process all 12 months into my BigQuery dataset. Once the ingestion was complete, I ran a validation query in the Google Cloud Console to get the final tally. 

SQL Query:
```SELECT COUNT(*) 
FROM `de-zoomcamp-2026-vlado.zoomcamp.yellow_tripdata` 
WHERE filename LIKE 'yellow_tripdata_2020%';
```
Result: The query returned exactly 24,648,499 rows. This figure represents the total number of trips successfully migrated from the source CSVs to my persistent storage on GCP.

---
## Question 4: Green Taxi Row Count (2020)  
I followed the same procedure for the Green Taxi dataset, executing a backfill for 2020 and verifying the outcome in BigQuery. Given the smaller fleet size, the row count is significantly lower than that of the Yellow taxi data.

SQL Query:
```SELECT COUNT(*) 
FROM `de-zoomcamp-2026-vlado.zoomcamp.yellow_tripdata` 
WHERE filename = 'yellow_tripdata_2021-03.csv';
```
Result: The specific count for March 2021 is 1,925,152 rows.

---
## Question 6: Configuring the Schedule Timezone  
Finally, I looked into how to adjust the scheduling behavior of the flow. Since Kestra defaults to UTC, I needed to ensure my triggers aligned with New York time to match the dataset's local context.  

Implementation: I modified the triggers section of the YAML to include the timezone property. I found that this must be placed specifically within the Schedule trigger block to be effective.

YAML Snippet:
```
triggers:
  - id: yellow_schedule
    type: io.kestra.plugin.core.trigger.Schedule
    cron: "0 10 1 * *"
    timezone: "America/New_York" # This ensures the run follows NY local time
    inputs:
      taxi: yellow
```

---
### Technical Environment
- Infrastructure: Google Cloud Platform (GCP)

- Orchestrator: Kestra (Free Tier)

- Configuration: Credentials stored in KV Store (vlado.base namespace)

- Storage: Google Cloud Storage (Bucket: de-zoomcamp-2026-vlado-taxi-data)

- Data Warehouse: BigQuery (Dataset: zoomcamp)
