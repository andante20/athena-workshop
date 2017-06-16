# Lab 4 - Partitioning Data

>*During this lab you will learn how to create and query partitioned tables.*
>
>**^^^Please make sure your AWS Management Console is set on US-East Region (N.Virginia)^^^**

![alt tag](../images/region.png)

## Create a Spark EMR cluster

- Follow the instructions [at this link](http://docs.aws.amazon.com/emr/latest/ManagementGuide/emr-plan-access-ssh.html) to create the SSH key you will require later

## Create an S3 bucket

- In a new browser window/tab open S3 console with [this link](https://console.aws.amazon.com/s3/home?region=us-east-1)
- Click Create Bucket
- Enter your username as the Bucket Name
- Select ‘US Standard’ as the Region
- Click Create
- Close the S3 browser window/tab.
- Wait while the cluster is being provisioned and running. (Approx. 5-10 minutes)

## Connect to your EMR cluster master

- **Linux/Mac machines** - Set SSH KeepAlive by default for your user:
  - Open a terminal
  - Enter `nano ~/.ssh/config`
  - Copy the lines below to the file
  ```xml
  Host *
   ServerAliveInterval 30
  ```
  - Save the file to disk
  - Close the terminal
- When the EMR cluster is ready click the SSH link to the left of the Master public DNS.
- Follow the instructions to SSH into the master instance.
- Once connected run the command `screen` at the prompt

## Create a partitioning batch job

- Run the command `spark-shell` at the bash prompt
- Wait for the Spark Shell to load
- Run the following command line by line into the Spark Shell prompt

```java
import org.apache.spark.sql.hive.orc._
import org.apache.spark.sql._

val hiveContext = new org.apache.spark.sql.hive.HiveContext(sc)
```

- Define the external CSV table to read from.

```java
hiveContext.sql("CREATE EXTERNAL TABLE IF NOT EXISTS yellow_trips_csv(" +
  "pickup_timestamp BIGINT, dropoff_timestamp BIGINT, vendor_id STRING, pickup_datetime TIMESTAMP, dropoff_datetime TIMESTAMP, pickup_longitude FLOAT, pickup_latitude FLOAT, dropoff_longitude FLOAT, dropoff_latitude FLOAT, rate_code STRING, passenger_count INT, trip_distance FLOAT, payment_type STRING, fare_amount FLOAT, extra FLOAT, mta_tax FLOAT, imp_surcharge FLOAT, tip_amount FLOAT, tolls_amount FLOAT, total_amount FLOAT, store_and_fwd_flag STRING) " +
  "ROW FORMAT DELIMITED FIELDS TERMINATED BY ',' " +
  "LOCATION 's3://nyc-yellow-trips/csv/'")
```

- Read the data from the CSV hive table

```java
val data=hiveContext.sql("select *,date_sub(from_unixtime(pickup_timestamp),0) as pickup_date from yellow_trips_csv limit 100")
```

- Perform the conversion. **Replace `<BUCKET NAME>` with the bucket you created.**

```java
data.write.mode("overwrite").partitionBy("pickup_date").parquet("s3://<BUCKET NAME>/parquet-partitioned/")
```

- Wait for the job to finish
- Look for the PARQUET files in your S3 bucket 

## Query partitioned files within Athena

- Open Athena console
- Create an external table from the partitioned PARQUET files.
>**Replace `<BUCKET NAME>` and `<USERNAME>`**

```sql
CREATE EXTERNAL TABLE IF NOT EXISTS <USERNAME>.yellow_trips_parquet_partitioned(pickup_timestamp BIGINT, dropoff_timestamp BIGINT, vendor_id STRING, pickup_datetime TIMESTAMP, dropoff_datetime TIMESTAMP, pickup_longitude FLOAT, pickup_latitude FLOAT, dropoff_longitude FLOAT, dropoff_latitude FLOAT, rate_code STRING, passenger_count INT, trip_distance FLOAT, payment_type STRING, fare_amount FLOAT, extra FLOAT, mta_tax FLOAT, imp_surcharge FLOAT, tip_amount FLOAT, tolls_amount FLOAT, total_amount FLOAT, store_and_fwd_flag STRING)
PARTITIONED BY (pickup_date TIMESTAMP)
STORED AS parquet
LOCATION 's3://<BUCKET NAME>/parquet-partitioned/'
```

- Load the table partitions

```sql
MSCK REPAIR TABLE yellow_trips_parquet_partitioned
```

- Run a query against the partitioned table. Replace `<USERNAME>`

```sql
select *
from
(
  select date_trunc('day',from_unixtime(pickup_timestamp)) as pickup_date1,* 
  from <USERNAME>.yellow_trips_parquet_partitioned 
  where pickup_date between timestamp '2009-04-12' and timestamp '2009-04-22'
)
where pickup_date1 between timestamp '2009-04-12' and timestamp '2009-04-22'
```

- Create an external table for non-partitioned PARQUET. Replace `<USERNAME>`

 ```sql
 CREATE EXTERNAL TABLE IF NOT EXISTS <USERNAME>.yellow_trips_parquet(pickup_timestamp BIGINT, dropoff_timestamp BIGINT, vendor_id STRING, pickup_datetime TIMESTAMP, dropoff_datetime TIMESTAMP, pickup_longitude FLOAT, pickup_latitude FLOAT, dropoff_longitude FLOAT, dropoff_latitude FLOAT, rate_code STRING, passenger_count INT, trip_distance FLOAT, payment_type STRING, fare_amount FLOAT, extra FLOAT, mta_tax FLOAT, imp_surcharge FLOAT, tip_amount FLOAT, tolls_amount FLOAT, total_amount FLOAT, store_and_fwd_flag STRING)
 STORED AS parquet
 LOCATION 's3://nyc-yellow-trips/parquet/';
 ```

- Compare the time and data scanned to the following query. Replace `<USERNAME>`

```sql
select *
from
(
  select date_trunc('day',from_unixtime(pickup_timestamp)) as pickup_date1,* 
  from <USERNAME>.yellow_trips_parquet
)
where pickup_date1 between timestamp '2009-04-12' and timestamp '2009-04-22'
```
