# Loading Taxi Data into Google Cloud SQL

https://googlepluralsight.qwiklabs.com/focuses/10991610?parent=lti_session

## Setting up 

- get active account name

```sh
gcloud auth list
```

- list the project ID with this command:

```sh
gcloud config list project
```



## Preparing  Environment

- Create environment variables that will be used later in the lab for  your project ID and the storage bucket that will contain your data:

```sh
export PROJECT_ID=$(gcloud info --format='value(config.project)')
export BUCKET=${PROJECT_ID}-ml
```

## Create Cloud SQL instance

```sh 
gcloud sql instances create taxi \
    --tier=db-n1-standard-1 --activation-policy=ALWAYS
```

- Set a root password for the Cloud SQL instance:

```sh
gcloud sql users set-password root --host % --instance taxi \
 --password 123456
```

- Now create an environment variable with the IP address of the Cloud Shell:

```sh
export ADDRESS=$(wget -qO - http://ipecho.net/plain)/32
```

- Whitelist the Cloud Shell instance for management access to your SQL instance.

```sh
gcloud sql instances patch taxi --authorized-networks $ADDRESS
```

- Get the IP address of your Cloud SQL instance

```sh
MYSQLIP=$(gcloud sql instances describe \
taxi --format="value(ipAddresses.ipAddress)")
```

- Check the variable MYSQLIP:

```sh
echo $MYSQLIP
```

- this will give us IP address as an output.



- Create the taxi trips table by logging into the `mysql` command line interface.

```sh
mysql --host=$MYSQLIP --user=root \
      --password --verbose
```

- to create the schema for the `trips` table

```sql
create database if not exists bts;
use bts;

drop table if exists trips;

create table trips (
  vendor_id VARCHAR(16),		
  pickup_datetime DATETIME,
  dropoff_datetime DATETIME,
  passenger_count INT,
  trip_distance FLOAT,
  rate_code VARCHAR(16),
  store_and_fwd_flag VARCHAR(16),
  payment_type VARCHAR(16),
  fare_amount FLOAT,
  extra FLOAT,
  mta_tax FLOAT,
  tip_amount FLOAT,
  tolls_amount FLOAT,
  imp_surcharge FLOAT,
  total_amount FLOAT,
  pickup_location_id VARCHAR(16),
  dropoff_location_id VARCHAR(16)
);
```

- In mySQL cmd interface check the import by entering the following:

```sql
describe trips;
```

Query the `trips` table:

```sql
select distinct(pickup_location_id) from trips;
```

Exit the `mysql` interactive console:

```sql
exit
```



## Add data to Cloud SQL instance

- copy the New York City taxi trips CSV files stored on  Cloud Storage locally

```sh
gsutil cp gs://cloud-training/OCBL013/nyc_tlc_yellow_trips_2018_subset_1.csv trips.csv-1
gsutil cp gs://cloud-training/OCBL013/nyc_tlc_yellow_trips_2018_subset_2.csv trips.csv-2
```

Import the CSV file data into Cloud SQL using `mysql`:

```sh
mysqlimport --local --host=$MYSQLIP --user=root --password \
--ignore-lines=1 --fields-terminated-by=',' bts trips.csv-*
```

- enter `Passw0rd`.

- Connect to the `mysql` interactive console:

```
mysql --host=$MYSQLIP --user=root  --password
```

- enter `Passw0rd`.



## Checking for data integrity

- this means making sure the data meets our expectations.

- In the `mysql` interactive console select the database:

```sql
use bts;
```

- Query the `trips` table for unique pickup location regions:

```SQL
select distinct(pickup_location_id) from trips;
```

- Let's start by digging into the `trip_distance` column

```sql
select
  max(trip_distance),
  min(trip_distance)
from
  trips;
```

- This return maximum trip distance of 85 miles seems reasonable but the  minimum trip distance of 0 seems buggy. 
- lets see How many trips in the dataset  have a trip distance of 0?

```sql
select count(*) from trips where trip_distance = 0;
```

- There are 155 such trips in the database. Perhaps these are fraudulent transactions? 

- Let's  see if we can find more data that doesn't meet our expectations. We  expect the `fare_amount` column to be positive.

```SQL
select count(*) from trips where fare_amount < 0;
```

- There are 14 such trips returned.
- There may be a reasonable explanation for why the fares  take on negative numbers. 
- However, it's up to the data engineer to  ensure there are no bugs in the data pipeline that would cause such a  result.

- let's investigate the `payment_type` column.

```sql
select
  payment_type,
  count(*)
from
  trips
group by
  payment_type;
```

The results of the query indicate that there are four different payment types, with:

- payment type = 1 has 13863 rows
- payment type = 2 has 6016 rows
- payment type = 3 has 113 rows
- payment type = 4 has 32 rows

By Digging into [the documentation](https://www1.nyc.gov/assets/tlc/downloads/pdf/data_dictionary_trip_records_yellow.pdf), a payment type of 1 refers to credit card use, payment type of 2 is  cash, and a payment type of 4 refers to a dispute. The figures make  sense.

Exit the 'mysql' interactive console:

```sql
exit
```