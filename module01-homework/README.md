# Module 1 Homework: Docker & SQL


## Question 1. Understanding docker first run 

Run docker with the `python:3.12.8` image in an interactive mode, use the entrypoint `bash`.

What's the version of `pip` in the image?

- **24.3.1**
- 24.2.1
- 23.3.1
- 23.2.1

```bash
(base) shaojun@Shaojuns-MacBook-Air docker_sql % docker build -t q1:pandas . 
[+] Building 17.2s (7/7) FINISHED                                                                                                docker:desktop-linux
...

View build details: docker-desktop://dashboard/build/desktop-linux/desktop-linux/fd836zcpau1o457ds5qeouwj2
(base) shaojun@Shaojuns-MacBook-Air docker_sql % docker run -it q1:pandas
root@badf962b6826:/# pip --version
pip 24.3.1 from /usr/local/lib/python3.12/site-packages/pip (python 3.12)
```

## Question 2. Understanding Docker networking and docker-compose

Given the following `docker-compose.yaml`, what is the `hostname` and `port` that **pgadmin** should use to connect to the postgres database?

```yaml
services:
  db:
    container_name: postgres
    image: postgres:17-alpine
    environment:
      POSTGRES_USER: 'postgres'
      POSTGRES_PASSWORD: 'postgres'
      POSTGRES_DB: 'ny_taxi'
    ports:
      - '5433:5432'
    volumes:
      - vol-pgdata:/var/lib/postgresql/data

  pgadmin:
    container_name: pgadmin
    image: dpage/pgadmin4:latest
    environment:
      PGADMIN_DEFAULT_EMAIL: "pgadmin@pgadmin.com"
      PGADMIN_DEFAULT_PASSWORD: "pgadmin"
    ports:
      - "8080:80"
    volumes:
      - vol-pgadmin_data:/var/lib/pgadmin  

volumes:
  vol-pgdata:
    name: vol-pgdata
  vol-pgadmin_data:
    name: vol-pgadmin_data
```

- postgres:5433
- localhost:5432
- db:5433
- postgres:5432
- **db:5432**

If there are more than one answers, select only one of them

##  Prepare Postgres

Run Postgres and load data as shown in the videos
We'll use the green taxi trips from October 2019:

Download this data and put it into Postgres.

```bash
(base) shaojun@Shaojuns-MacBook-Air docker_sql % docker build -f Dockerfile.ingest -t taxi_ingest:v001 .

(base) shaojun@Shaojuns-MacBook-Air docker_sql % docker-compose up -d

(base) shaojun@Shaojuns-MacBook-Air docker_sql % URL="https://github.com/DataTalksClub/nyc-tlc-data/releases/download/green/green_tripdata_2019-10.csv.gz"

docker run -it \
  --network=pg-network \
  taxi_ingest:v001 \
    --user=root \
    --password=root \
    --host=pgdatabase \
    --port=5432 \
    --db=ny_taxi \
    --table_name=green_trip_data \
    --url=${URL}
```
**Files are in [folder `docker_sql`](docker_sql/).**

## Question 3. Trip Segmentation Count

During the period of October 1st 2019 (inclusive) and November 1st 2019 (exclusive), how many trips, **respectively**, happened:
1. Up to 1 mile
2. In between 1 (exclusive) and 3 miles (inclusive),
3. In between 3 (exclusive) and 7 miles (inclusive),
4. In between 7 (exclusive) and 10 miles (inclusive),
5. Over 10 miles 

Answers:

- 104,802;  197,670;  110,612;  27,831;  35,281
- **104,802;  198,924;  109,603;  27,678;  35,189**
- 104,793;  201,407;  110,612;  27,831;  35,281
- 104,793;  202,661;  109,603;  27,678;  35,189
- 104,838;  199,013;  109,645;  27,688;  35,202

```sql
SELECT
    CASE
        WHEN trip_distance <= 1 THEN 'Up to 1 mile'
        WHEN trip_distance > 1 AND trip_distance <= 3 THEN '1-3 miles'
        WHEN trip_distance > 3 AND trip_distance <= 7 THEN '3-7 miles'
        WHEN trip_distance > 7 AND trip_distance <= 10 THEN '7-10 miles'
        ELSE 'Over 10 miles'
    END AS distance,
    COUNT(*) AS trips
FROM green_trip_data
WHERE (lpep_dropoff_datetime >= '2019-10-01' AND lpep_dropoff_datetime < '2019-11-01') 
  AND (lpep_pickup_datetime >= '2019-10-01' AND lpep_pickup_datetime < '2019-11-01') 
GROUP BY distance;
```

## Question 4. Longest trip for each day

Which was the pick up day with the longest trip distance?
Use the pick up time for your calculations.

Tip: For every day, we only care about one single trip with the longest distance. 

- 2019-10-11
- 2019-10-24
- 2019-10-26
- **2019-10-31**

```sql
SELECT DATE(lpep_pickup_datetime), MAX(trip_distance) 
FROM green_trip_data
GROUP BY DATE(lpep_pickup_datetime) 
ORDER BY MAX(trip_distance) DESC;
```

## Question 5. Three biggest pickup zones

Which were the top pickup locations with over 13,000 in
`total_amount` (across all trips) for 2019-10-18?

Consider only `lpep_pickup_datetime` when filtering by date.
 
- **East Harlem North, East Harlem South, Morningside Heights**
- East Harlem North, Morningside Heights
- Morningside Heights, Astoria Park, East Harlem South
- Bedford, East Harlem North, Astoria Park

```sql
SELECT 
    z."Zone" AS pickup_zone,
    SUM(total_amount) AS total_amount
FROM public.green_trip_data g
JOIN zones z
ON g."PULocationID" = z."LocationID"
WHERE DATE(lpep_pickup_datetime) = '2019-10-18'
GROUP BY z."Zone"
HAVING SUM(total_amount) > 13000
ORDER BY total_amount DESC;
```

## Question 6. Largest tip

For the passengers picked up in October 2019 in the zone
named "East Harlem North" which was the drop off zone that had
the largest tip?

Note: it's `tip` , not `trip`

We need the name of the zone, not the ID.

- Yorkville West
- **JFK Airport**
- East Harlem North
- East Harlem South

```sql
SELECT 
    z_dropoff."Zone" AS dropoff_zone,
    MAX(tip_amount) AS largest_tip 
FROM green_trip_data g 
JOIN zones z_pickup 
ON g."PULocationID" = z_pickup."LocationID" 
JOIN zones z_dropoff 
ON g."DOLocationID" = z_dropoff."LocationID"
WHERE 
    z_pickup."Zone" = 'East Harlem North'
    AND DATE(lpep_pickup_datetime) BETWEEN '2019-10-01' AND '2019-10-31'
GROUP BY z_dropoff."Zone"
ORDER BY largest_tip DESC
LIMIT 1;
```

## Terraform

In this section homework we'll prepare the environment by creating resources in GCP with Terraform.

In your VM on GCP/Laptop/GitHub Codespace install Terraform. 
Copy the files from the course repo
[here](../../../01-docker-terraform/1_terraform_gcp/terraform) to your VM/Laptop/GitHub Codespace.

Modify the files as necessary to create a GCP Bucket and Big Query Dataset.

**See [folder `terraform`](terraform/).**

## Question 7. Terraform Workflow

Which of the following sequences, **respectively**, describes the workflow for: 
1. Downloading the provider plugins and setting up backend,
2. Generating proposed changes and auto-executing the plan
3. Remove all resources managed by terraform`

Answers:
- terraform import, terraform apply -y, terraform destroy
- teraform init, terraform plan -auto-apply, terraform rm
- terraform init, terraform run -auto-approve, terraform destroy
- **terraform init, terraform apply -auto-approve, terraform destroy**
- terraform import, terraform apply -y, terraform rm
