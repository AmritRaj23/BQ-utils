For the setup below, I am using 3 projects: 
* A BQ Admin project which is used for creating commitments (flex slot), reservations and assignments
* A project 'a' which uses the reservations when querying 
* A project 'b' which uses the on-demand model when querying

```
ADMIN_PROJECT_ID=<project-id>
PROJECT_A_SLOTS=<project-id>
PROJECT_B_ONDEMAND=<project-id>
LOCATION=US
RESERVATION_NAME=r1
```

In the admin project enable the bq reservation api 
```
gcloud services enable bigqueryreservation.googleapis.com
```

### making a flex commitment 

In the admin project create a commitment
```
bq mk \
  --project_id=$ADMIN_PROJECT_ID \
  --location=$LOCATION \
  --capacity_commitment \
  --plan=FLEX \
  --slots=500
```
### making a reservation 
```
bq mk \
  --project_id=$ADMIN_PROJECT_ID \
  --location=$LOCATION \
  --reservation \
  --slots=500 \
  --ignore_idle_slots=false \
  $RESERVATION_NAME
```
### making an assignment to project 
```
bq mk \
  --project_id=$ADMIN_PROJECT_ID \
  --location=$LOCATION \
  --reservation_assignment \
  --reservation_id=$RESERVATION_NAME \
  --job_type=QUERY \
  --assignee_id=$PROJECT_A_SLOTS \
  --assignee_type=PROJECT
```

## Query via reservation 

```
ORIGIN_PROJECT=$PROJECT_A_SLOTS
```

Query the public dataset 

```
bq query \
  --project_id=$ORIGIN_PROJECT \
  --location=$LOCATION \
  --nouse_legacy_sql \
  --nouse_cache \ 'SELECT  unique_key, complaint_type, complaint_description FROM `bigquery-public-data.austin_311.311_service_requests` LIMIT 10'
```

get latest job id

```
LATEST_JOB_ID=$(bq ls -j -a -n 1 $ORIGIN_PROJECT | awk '{if(NR>2)print}' | awk '{print $1}')
```

analyse job

```
bq show \
  --project_id=$ORIGIN_PROJECT \
  --location=$LOCATION \
  --format=prettyjson \
  -j $LATEST_JOB_ID  
```

## Query via on-demand project  

Change Project origin to on demand 

```
ORIGIN_PROJECT=$PROJECT_B_ONDEMAND
```

Query the dataset again 

```
bq query \
  --project_id=$ORIGIN_PROJECT \
  --location=$LOCATION \
  --nouse_legacy_sql \
  --nouse_cache \ 'SELECT  unique_key, complaint_type, complaint_description FROM `bigquery-public-data.austin_311.311_service_requests` LIMIT 10'
```

```
LATEST_JOB_ID=$(bq ls -j -a -n 1 $ORIGIN_PROJECT | awk '{if(NR>2)print}' | awk '{print $1}')
```

```
bq show \
  --project_id=$ORIGIN_PROJECT \
  --location=$LOCATION \
  --format=prettyjson \
  -j $LATEST_JOB_ID  
```

## Cleaning up 

Get latest assignment id 
```
LATEST_ASSIGNMENT_ID=$(bq ls --project_id=$ADMIN_PROJECT_ID --location=$LOCATION --reservation_assignment $ADMIN_PROJECT_ID:US.r1 | awk '{if(NR>2)print}' | awk '{print $1}')
```

Remove reservation from project 
```
bq rm \
--project_id=$ADMIN_PROJECT_ID \
--location=$LOCATION \
--reservation_assignment $ASSIGNMENT_ID
```

Delete the reservation 
```
bq rm \
--project_id=$ADMIN_PROJECT_ID \
--location=$LOCATION \
--reservation $RESERVATION_NAME
```

## Resources

Documentation [[here]](https://cloud.google.com/bigquery/docs/reservations-tasks#working_with_commitments)
