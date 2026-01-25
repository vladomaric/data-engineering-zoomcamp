# Module 1 Homework: Docker & SQL

This folder contains my solutions for the first module of the Data Engineering Zoomcamp (2026 Cohort). The goal of this homework is to get familiar with Docker, SQL, and Terraform infrastructure.

---
## Question 1: Understanding Docker images
I verified the version of `pip` in the `python:3.13` image by running the following command in my terminal:

```bash
docker run python:3.13 pip --version
```
**Observed Output:**  

```pip 25.3 from /usr/local/lib/python3.13/site-packages/pip (python 3.13)``` 

**Answer:** 25.3

## Question 2: Understanding Docker networking
Based on the provided `docker-compose.yaml`, the hostname and port `pgadmin` should use to connect to the postgres database are:

* **Hostname:** `db`
* **Port:** `5432`
* **Answer:** `db:5432`

## Question 3: Trip Segmentation
During the period of November 1st 2025 (inclusive) through December 1st 2025 (exclusive), how many trips occurred with a distance up to 1 mile?

**SQL Query:**
```sql
SELECT
    count(*)
FROM
    'green_tripdata_2025-11.parquet'
WHERE
    lpep_pickup_datetime >= '2025-11-01'
    AND lpep_pickup_datetime < '2025-12-01'
    AND trip_distance <= 1;
```
**Observed output:**
```
   count_star()
0          8007
```
**Answer:** 8,007
## Question 4: Longest trip for each day
Which was the pick-up day with the longest trip distance? (Only considering trips with `trip_distance` < 100 miles).

**SQL Query:**
```sql
SELECT 
    CAST(lpep_pickup_datetime AS DATE) AS day,
    MAX(trip_distance) AS max_dist
FROM 'green_tripdata_2025-11.parquet'
WHERE trip_distance < 100
GROUP BY 1
ORDER BY max_dist DESC
LIMIT 1;
```
**Observed output:**
```
         day  max_dist
0 2025-11-14     88.03
```
**Answer:** 2025-11-14
## Question 5: Largest total amount
Which was the pickup zone with the largest `total_amount` (sum of all trips) on November 18th, 2025?

**SQL Query:**
```sql
SELECT 
    z."Zone",
    SUM(t.total_amount) AS sum_total
FROM 'green_tripdata_2025-11.parquet' t
JOIN 'taxi_zone_lookup.csv' z
  ON t."PULocationID" = z."LocationID"
WHERE CAST(t.lpep_pickup_datetime AS DATE) = '2025-11-18'
GROUP BY 1
ORDER BY sum_total DESC
LIMIT 1;
```
**Observed output:**
```
                Zone  sum_total
0  East Harlem North    9281.92
```
**Answer:** East Harlem North

## Question 6: Largest tip
For the passengers picked up in the zone named "East Harlem North" in November 2025, which was the drop-off zone that had the largest tip?

**SQL Query:**
```sql
SELECT 
    z_do."Zone" AS dropoff_zone,
    MAX(t.tip_amount) AS max_tip
FROM 'green_tripdata_2025-11.parquet' t
JOIN 'taxi_zone_lookup.csv' z_pu
  ON t."PULocationID" = z_pu."LocationID"
JOIN 'taxi_zone_lookup.csv' z_do
  ON t."DOLocationID" = z_do."LocationID"
WHERE z_pu."Zone" = 'East Harlem North'
GROUP BY 1
ORDER BY max_tip DESC
LIMIT 1;
```
**Observed output:**
```
     dropoff_zone  max_tip
0  Yorkville West    81.89
```
**Answer:** Yorkville West

## Question 7: Terraform Workflow
The correct lifecycle for initializing, applying, and then cleaning up infrastructure is:

**Answer:** `terraform init, terraform apply -auto-approve, terraform destroy`
